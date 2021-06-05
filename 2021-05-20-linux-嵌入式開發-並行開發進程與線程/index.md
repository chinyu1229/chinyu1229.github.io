# Linux 嵌入式開發 – 並行開發—進程與線程

# Linux 嵌入式開發 -- 並行開發---進程與線程
## Process概念
* program
    * 存放在disk上的指令和數據的有序集合（文件）
    * 靜態的
* process
    * 執行一個program所分配的資源總稱
    * 是program一次執行的總稱
    * 動態的，包括創建，調度，執行，死亡
    * 有獨立的地址空間
    * linux為每個進程創建task_struct

## Process 內容
              
 process   - | 正文段      |--------
           --------- | 用戶數據段   |----program
           --------- | 系統數據段   |

## Process control block
* PID
* process user
* process status / priority
* file description table

## Process 類型
* 交互進程：在shell下啟動，可在前臺/後台運行
* 批處理進程：和在終端無關，被提交到一個作業隊列中以便順序執行
* 守護進程：和終端無關，一直在後台運行

## Process status
* running / ready
* waiting
  * interrupt
  * not interrupt
* terminated : 收到signal後可以繼續運行
* zombie ： pcb沒有被釋放

![](https://i.imgur.com/0LavzmI.png)

## Thread
* process在切換時系統開銷大
* 同一個process中的thread共享相同的空間
* Linux 不區分thread process
### 特點
* 通常thread指的是共享地址空間的多個任務
* 大大提高了任務切換的效率
* 避免額外的TLB & cache的刷新

thread共享：
* 可執行的命令
* 靜態資料
* 進程中打開的file descriptor 
* 當前工作目錄
* user id
* user group id


thread私有：
* tid
* pc & 相關暫存器
* stack & heap
* errno
* priority
* 執行狀態和屬性

## Linux thread lib
pthread提供如下基本操作
* 創建
* 回收
* 結束

同步和互斥機制
* 信號量
* 互斥鎖

### 創建
```c
#include<pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                    void *(*routine)(void *), void *arg);
```
* thread 線程對象
* attr 線程屬性，NULL表示默認
* routine線程執行的函數
* arg 傳遞給routine的參數
* 成功返回0

### 回收
```c
#include<pthread.h>
int pthread_join(pthread_t thread, void **retval);
```
* thread要回收的對象
* 調用thread阻塞直到thread結束
* *retval接收thread 的返回值

### 結束
```c
#include<pthread.h>
int pthread_exit(void *retval);
```
* 結束當前線程
* retval可以被其他thread通過pthread_join獲取
* thread私有資源將被釋放

### 範例
```c
/*thread_demo.c*/
#include<pthread.h>
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>

char msg[32] = "Hello World";

void *thread_func(void *arg)
{
	sleep(1);
	strcpy(msg, "mark by thread");
	pthread_exit("3Q for waiting for me");

}
int main(void)
{
	pthread_t a_thread;
	void *result;

	if(pthread_create(&a_thread,NULL,thread_func,NULL)!=0){
		printf("fail to pthread_create");
		exit(-1);
	}
	pthread_join(a_thread, &result);
	printf("result is %s\n",result);
	printf("msg is %s\n",msg);
	return 0;
}
```
編譯：`gcc -o thread_demo thread_demo.c -lpthread`
`$ ./thread_demo `
result is 3Q for waiting for me
msg is mark by thread

## Thread通信 -- 同步synchronization
* 由信號量來決定線程是繼續運行還是阻塞

### 信號量
* 信號量代表某一類資源，其值表示系統中該資源的數量
* 是一個受保護的變數，只能通過三種操作來訪問
    * 初始化
    * P操作（申請資源）
    * V操作（釋放資源）=> 一定會阻塞
#### P/V操作
* P(S)：
    if(信號量>0){
        申請資源的任務繼續運行; 
        信號量的值減1;
    }
    else{ 申請資源的任務阻塞 };

* V(S):
    信號值加一
    if(有任務在等待資源){
        喚醒等待的任務，讓其讓其繼續運行
    }
    
#### Posix 信號量
* posix有兩種信號量
    * 無名信號量（基於memory的信號量）=> 用於線程
    * 有名信號量 => 可用於線程或是進程

* pthread庫常用的信號量操作函數：
```c=
int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_wait(sem_t *sem); // P操作
int sem_post(sem_t *sem); // V操作
```
**初始化**
```c
#include<semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
```
* 成功返回0,失敗時EOF
* sem指向要初始化的信號量對象
* pshared 0-thread間，1-process間
* val信號量初值

**P/V操作**
```c
#include<semaphore.h>
int sem_wait(sem_t *sem); // P操作
int sem_post(sem_t *sem); // V操作
```
* 成功返回0,失敗時EOF
* sem指向要初始化的信號量對象

## 線程同步--範例一
兩個線程同步讀寫緩衝區（生產者與消費者問題）
```c
/*不嚴謹*/
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<semaphore.h>
#include<string.h>

char buf[32];
sem_t sem;
void *function(void *arg);

int main(void)
{
        pthread_t a_thread;
        if(sem_init(&sem,0,0)<0){
                perror("sem_init");
                exit(-1);
        }
        if(pthread_create(&a_thread,NULL,function,NULL)!=0){
                printf("fail\n");
                exit(-1);
        }
        printf("input 'quit'to exit\n");
        do{ //stdin write
                fgets(buf,32,stdin);
                sem_post(&sem);
        }while(strncmp(buf,"quit",4)!=0);
        return 0;
}
void *function(void *arg)//read 
{
        while(1)
        {
                sem_wait(&sem);
                printf("you enter %d characters\n",strlen(buf));
        }
}
```

```c
/*嚴謹*/
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<semaphore.h>
#include<string.h>

char buf[32];
sem_t sem_r,sem_w;
void *function(void *arg);

int main(void)
{
        pthread_t a_thread;
        if(sem_init(&sem_r,0,0)<0){
                perror("sem_init");
                exit(-1);
        }
        if(sem_init(&sem_w,0,1)<0){
                perror("sem_init");
                exit(-1);
        }

        if(pthread_create(&a_thread,NULL,function,NULL)!=0){
                printf("fail\n");
                exit(-1);
        }
        printf("input 'quit'to exit\n");
        do{ //stdin write
                sem_wait(&sem_w);
                fgets(buf,32,stdin);
                sem_post(&sem_r);
        }while(strncmp(buf,"quit",4)!=0);

        return 0;
}
void *function(void *arg)//read 
{
        while(1)
        {
                sem_wait(&sem_r);
                printf("you enter %d characters\n",strlen(buf));
                sem_post(&sem_w);
        }
}
```

## 互斥
* 臨界資源 （共享資源）
    * 一次只允許一個任務（進程、線程）訪問的共享資源 
* 臨界區
    * 訪問臨界資源的code 
    
* 互斥機制
    * mutex互斥鎖
    * 任務訪問臨界資源前，申請鎖，訪問完後釋放鎖 


### 互斥鎖初始化
```c
#include<pthread.h>
int pthread_mutex_init(pthread_mutex_t *mutex,
    const pthread_mutexattr_t * attr);
```
* 成功時返回0，失敗返回錯誤
* mutex指向要初始化的互斥鎖對象
* attr互斥鎖屬性，NULL表示缺少屬性

### 申請鎖
```c
#include<pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
```
* 成功時返回0，失敗返回錯誤
* mutex指向要初始化的互斥鎖對象
* 如果無法獲得鎖，任務阻塞

### 釋放鎖
```c
#include<pthread.h>
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
* 成功時返回0，失敗返回錯誤
* mutex指向要初始化的互斥鎖對象
* 執行完臨界區必須即時釋放鎖

### 範例
```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>
#define _LOCK_
unsigned int count, value1,value2;
pthread_mutex_t lock;

void *func(void *arg);
int main(){
    pthread_t a_thread;
    if(pthread_mutex_init(&lock, NULL) != 0){
        printf("fail to pthread init\n");
        exit(-1);
    } 
    if(pthread_create(&a_thread,NULL,func,NULL)!=0)
    {
        printf("fail to pthread create");
        exit(-1);
    }
    while(1){
        count++;
        #ifdef _LOCK_
            pthread_mutex_lock(&lock);
        #endif
            value1 = count;
            value2 = count;
        #ifdef _LOCK_
            pthread_mutex_unlock(&lock);
        #endif
    }
    return 0;
}
void *func(void *arg)
{
    while(1)
    {
        count++;
        #ifdef _LOCK_
            pthread_mutex_lock(&lock);
        #endif
            if(value1 != value2)
            {
                printf("value1 = %u,value2 = %u\n",value1,value2);
                usleep(1000000);
            }
        #ifdef _LOCK_
            pthread_mutex_unlock(&lock);
        #endif
    }
}
```
編譯：

使用互斥鎖
gcc -o test test.c -lpthread -D_LOCK_
不使用互斥鎖
gcc -o test test.c -lpthread 
