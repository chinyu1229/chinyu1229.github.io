# Linux 嵌入式開發 – 並行開發—Unix進程間的通訊方式


# Linux 嵌入式開發 -- 並行開發--- Unix進程間的通訊方式
## Process間通訊介紹
早期：
* pipe
* fifo
* signal

System V IPC
* share memory
* message queue
* semphore set

---------------------------本地通信↑

---------------------------本地/網路通信↓
Linux
* socket

## 無名管道
* 只能用於有親緣關係的進程之間的通信
* 單工的通信模式，具有固定的讀端和寫端
* 創建時返回兩個文件描述符，分別用於讀寫管道

### 創建 - pipe
```c
#include<unistd.h>
int pipe(int pfd[2]);
```
* 成功時返回0，失敗時返回EOF
* pfd 包含兩個元素的int 陣列，用來保存文件描述符
* pfd[0]用於read管道，pfd[1]用於write管道



### 範例
子進程1和2分別往管道寫入字串，父進程讀管道內容並打印
```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/types.h>

int main()
{
    pid_t pid1, pid2; // save return value of fork()
    char buf[32];
    int pfd[2];
    if(pipe(pfd)<0)
    {
        perror("pipe");
        exit(-1);
    }
    if((pid1 == fork()) < 0 )
    {
        perror("fork");
        exit(-1);
    }
    else if(pid1 == 0) // child process1
    {
        strcpy(buf,"process1");
        write(pfd[1],buf,32);
        exit(0);
    }
    else // parents process
    {
        if((pid2 = fork()) < 0)
        {
            perror("fork");
            exit(-1);
        }
        else if(pid == 0) // child process 2
        {
            sleep(1);
            strcpy(buf,"process2");
            write(pfd[1],buf,32);
        }
        else
        {
            wait(NULL);
            read(pfd[0],buf,32);
            printf("%s\n",buf);
            wait(NULL);
            read(pfd[0],buf,32);
            printf("%s\n",buf);
        }
    }
    
    return 0;
}
```

## 讀無名管道
* 寫端存在 
    * 管道中有數據  read返回實際讀取的byte數
    * 管道中無數據  進程read阻塞

* 寫端不存在
    * 管道中有數據 read返回實際讀取的byte數
    * 管道中無數據 read返回0

## 寫無名管道
* 讀端存在 
    * 管道中有空間  write返回實際寫入的byte數
    * 管道中無空間  不保證原子操作（有多少空間先寫多少空間）進程寫阻塞

* 讀端不存在
    * 管道中有空間 
    * 管道中無數據 
    * 會發生管道斷裂（被信號結束）
    
無名管道的大小獲取
```c
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>

int main()
{
    int count = 0, pfd[2];
    char buf[1024];
    if(pipe(pfd) < 0)
    {
        perror("pipe");
        exit(-1);
    }
    while(1)
    {
        write(pfd[1],buf,1024);
        printf("wrote %dk bytes\n",++count);
    }
    return 0;
}
```

測試管道斷裂 -- pipe_broken
```c
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/wait.h>

int main()
{
    pid_t pid;
    int pfd[2],status;
    char buf[32];
    
    if(pipe(pfd) < 0)
    {
        perror("pipe");
        exit(-1);
    }
    close(pfd[0]); // close read
    if((pid = fork()) < 0)
    {
        perror("fork");
        exit(-1);
    }
    else if(pid == 0)
    {
        write(pfd[1], buf, 32);
        exit(0);
    }
    else
    {
        wait(&status);
        printf("status = %x \n",status);
    }
    return 0;
}

```

## 有名管道
* 對應管道文件，可以用於任意進程之間進行通信
* 打開管道時可以指定讀寫方式
* 透過文件I/O操作，內容存放在內存中

### 創建
```c
#include<unistd.h>
#include<fcntl.h>
int mkfifo(const char *path, mode_t mode);
```

* 成功返回0，失敗返回EOF
* mode管道文件的權限

### 範例
ProcessA:循環從鍵盤輸入並寫入有名管道，quit時退出 
ProcessB:循環統計processA每次寫入管道的字串長度

```c
/* create_fifo.c */
#include<unistd.h>
#include<fcntl.h>
#include<stdlib.h>

int main()
{
    if(mkfifo("myfifo",0666) < 0)
    {
        perror("mkfifo");
        exit(-1);
    }
    return 0;
}
```

```c
/* write_fifo.c*/
#include<stdio.h>
#include<unistd.h>
#include<fcntl.h>
#include<stdlib.h>

#define N 32
int main()
{
    char buf[N];
    int pfd;
    if(pfd = open("myfifo",O_WRONLY)) < 0)
    {
        perror("open");
        exit(-1);
    }
    while(1)
    {
        fgets(buf,N,stdin);
        if(strcmp(buf,"quit\n") == 0) break;
        write(pfd,buf,N);
    }
    close(pfd);
    return 0;
}
```

```c
/*read_fifo.c*/
#include<stdio.h>
#include<unistd.h>
#include<fcntl.h>
#include<stdlib.h>

#define N 32
int main()
{
    char buf[N];
    int pfd;
    if(pfd = open("myfifo",O_WRONLY)) < 0)
    {
        perror("open");
        exit(-1);
    }
    while(read(pfd,buf,N) > 0)
    {
        printf("the length of string is %d\n",strlen(buf));
    }
    close(pfd);
    return 0;
}
```

```shell
$ gcc -o create_fifo create_fifo.c
$ gcc -o write_fifo write_fifo.c
$ gcc -o read_fifo read_fifo.c
```

```shell
$ ./create_fifo
$ ll 
$ ./read_fifo
$ ./write_fifo
```
**有名管道可能會阻塞 -- 當只有寫端或讀端其中一個開啟時會阻塞，只有當兩個端同時開啟open才會繼續運行**

## 信號
* 在軟體層次上對中隊機制的一種模擬，是一種異步通信方式
* linux kernal 通過信號通知user process，不同的信號代表不同的事件
 ![](https://i.imgur.com/AFRLFNL.png)
* process 對信號有不同的響應方式
    * 缺省方式
    * 忽略信號
    * 捕捉信號（註冊信號）

常用信號：
![](https://i.imgur.com/0BGQHys.png)
![](https://i.imgur.com/Cp1T8gQ.png)
 
信號相關命令
* kill[-signal] pid
    * 默認發送SIGTERM
    * -sig 可指定信號
    * pid指定發送對象

* killall[-u user | prog] 
    * prog 指定進程名
    * user指定用戶名

## 信號發送
```c
#include<unistd.h>
#include<signal.h>
int kill(pid_t pid, int sig);
int raise(int sig); // 給自己發信號
```
* pid接收進程的進程號，0代表同組進程，-1代表所有進程

## 信號相關函數
```c
/*定時器*/
int alarm(unsigned int seconds);
```
* 成功時返回上個定時器的剩餘時間
* seconds定時器的時間，0取消定時器
* 一個進程中只能設定一個定時器，時間到會產生SIGALRM

```c
int pause(void);
```
* 進程一直阻塞，直到被信號中斷
* 被信號中斷後返回-1，errno為EINTR

### 範例
```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
int main()
{
    alarm(3);
    pause();
    printf("I have been waken up\n");
    return 0;
}
```
```
收到定時器signal時，進程就終止，因此不會打印line 8
$ a.out
Alarm clock
```

**alarm常用於超時檢測**

## 設置信號的響應方式
```c
#include<unistd.h>
#include<signal.h>
void(* signal(int signo,void(*handler)(int)))(int);
```
* 成功返回原先信號處理函數
* signo要設置的信號類型
* handler指定的信號處理函數：SIG_DFL代表缺省方式 ,SIG_IGN代表忽略信號

### 範例
```c
#include<unistd.h>
#include<signal.h>

void handler(int signo)
{
    if(signo == SIGINT) printf("I have got SIGINT\n");
    if(signo == SIGQUIT) printf("I have got SIGINT\n");
}
int main()
{
    signal(SIGINT,hardler);
    signal(SIGQUIT,handler);
    while(1) pause();
    return 0;
}
```




















