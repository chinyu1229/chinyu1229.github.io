# Linux 嵌入式開發 – 並行開發—SystemV IPC

# Linux 嵌入式開發 -- 並行開發--- System V IPC
## System V IPC
* 包含共享內存、消息對列、和信號燈集
* 每個IPC對象有唯一的ID
* IPC對象創建後一直存在，直到被顯式的刪除（主動刪除）
* 每個IPC對象有一個關聯的KEY
* ipcs / ipcrm
* 實現進程間通訊

### Key
![](https://i.imgur.com/KnPMrFD.png)

### ftok
```c
#include<sys/types.h>
#include<sys/ipc.h>
key_t ftok(const char *path,int proj_id);
```
* 成功時返回合法的key值，失敗時返回EOF
* path存在且可以被訪問文件的路徑
* proj_id 用於生成key的數字，不能為0

### 範例
```c
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<sys/ipc.h>
#include<unistd.h>

int main(int argc, char *argv[])
{
    key_t key;
    if((key = ftok(".",'a')) == -1)
    {
        perror("key");
        exit(-1);
    }
}
```

## 共享內存
* 是一種最高校的進程間通信方式，進程可以直接讀寫內存，而不需要任何數據的拷貝
* 在內核空間創建，可以被進程映射到用戶空間訪問，使用靈活
* 由於多個進程可以同時訪問共享內存，印此需要同步和互斥機制配合使用

### 使用步驟
* 創建/打開共享內存
* 映射共享內存，即把指定的共享內存映射到進程的地址空間用於訪問
* 讀寫共享內存
* 撤銷共享內存映射
* 刪除共享內存對象

#### 創建
```c
#include<sys/ipc.h>
#include<sys/shm.h>
int shmget(key_t key, int size, int shmflg);
```
* 成功時返回共享內存的id，失敗時返回EOF
* key和共享內存關聯的key, IPC_PRIVATE或ftok生成
* shmflg共享內存標誌位 IPC_CREAT\|0666

範例：創建一個私有的共享內存，大小為512 bytes,權限0666
```c
int shmid;
if((shmid = shmget(IPC_PRIVATE,512,0666)) < 0)
{
    perror("shmget");
    exit(-1);
}
```

範例：創建/打開一個和key關聯的共享內存，大小為1024 bytes,權限0666
```c
key_t key;
int shmid;
if((key = ftok(".",'m')) == -1)
{
    perror("ftok");
    exit(-1);
}
if((shmid = shmget(key,1024,IPC_CREAT|0666)) < 0)
{
    perror("shmget");
    exit(-1);
}
```

## 共享內存映射
```c
#include<sys/ipc.h>
#include<sys/shm.h>
void *shmat(int shmid, const void *shmaddr, int shmflg);
```
* 成功時返回映射的地址，失敗返回(void*)-1
* shmid 要映射的共享內存id
* shmaddr 映射後的地址，NULL表示由系統自動映射
* shmflag 0 表示可讀寫，SHM_RDONLY表示只讀

### 範例
* 通過指針訪問共享內存，指針類型取決於共享內存中存放的數據類型

在共享內存中存放鍵盤輸入的字串
```c
char *addr;
int shmid;
......
if((addr = (char *)shmat(shmid,NULL,0)) == (char *) -1)
{
    perror("shmat");
    exit(-1);
}
fgets(addr, N, stdin);
....
```
### 撤銷映射
```c
#include<sys/ipc.h>
#include<sys/shm.h>
int shmdt(void *shmaddr);
```
* 成功返回0，失敗返回EOF
* 不使用共享內存時應即時撤銷映射
* 進程結束時自動撤銷

### 內存控制
```c
#include<sys/ipc.h>
#include<sys/shm.h>
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```
* 成功返回0，失敗返回EOF
* shmid要操作的共享內存id
* cmd要執行的操作 IPC_STAT(獲取共享內存的屬性存於struct shmid_ds結構buf中) IPC_SET(設置) IPC_RMID(刪除共享內存ID)

### 注意事項
* 每個共享內存大小都有限制
    * ipcs -l
    * cat /proc/sys/kernel/shmmax

* 共享內存刪除的時間點
    * shmctl(shmid, IPC_RMID, NULL)添加刪除標記
    * nattach變成0時真正刪除
    * 所有映射都取消時刪除


## 消息隊列
* system V IPC對象的一種
* 由消息隊列ID來唯一標示
* 可以按照類型來發送接收消息

### 使用步驟
* 打開創建 msgget
* 發送消息 msgsnd
* 接收消息 msgrcv
* 控制消息 msgctl

#### 創建/打開
```c
#include<sys/ipc.h>
#include<sys/msg.h>
int msgget(key_t key, int msgflg);
```
* 成功時返回消息隊列的id,失敗時返回EOF
* key 和消息隊列關聯的key IPC_PRIVATE或ftok
* msgflg標誌位 IPC_CREAT|0666

創建範例：
```c
#include<sys/ipc.h>
#include<sys/msg.h>
int main()
{
    int msgid;
    key_t key;
    if((key = ftok(".",'q')) == -1)
    {
        perror("ftok");
        exit(-1);
    }
    if((msgid = msgget(key, IPC_CREAT|0666)) < 0)
    {
        perror("msgget");
        exit(-1);
    }
    ....
    return 0;
}
```
#### 發送
```c
#include<sys/ipc.h>
#include<sys/msg.h>
int msgsnd(int msgid,const void * msgp, size_t size, int msgflg);
```
* 成功時返回 0,失敗時返回 -1
* msgp 消息緩衝區地址
* size 消息正文長度
* msgflg標誌位 0 或 IPC_NOWAIT

#### 消息格式
* 通信雙方首先定義好統一的消息格式
* 用戶根據應用需求定義結構體類型
* 首（第一個）成員類型為long 代表消息類型（正整數）
* 其他成員都屬於消息正文
```
typedef struct
{
    long mytype;//類型(首位)
    char mtext[64];
    ...
    
}MSG;
```

範例：
```c
typedef struct
{
    long mytype;//類型(首位)
    char mtext[64];
}MSG;
#define LEN(sizeof(MSG) - sizeof(long))

int main()
{
    MSG buf;
    ....
    buf.mytype = 100;
    fgets(buf.mtext, 64, stdin);
    msgsnd(msgid, &buf, LEN, 0);
    ....
    return 0;
}
```

#### 接收
```c
#include<sys/ipc.h>
#include<sys/msg.h>
int msgrcv(int msgid,const void * msgp, size_t size, long msgtype, int msgflg);
```
* 成功時返回 0,失敗時返回 -1
* msgp 消息緩衝區地址
* size 消息正文長度
* msgtype 指定接收消息的類型
* msgflg標誌位 0 或 IPC_NOWAIT

範例：

```c
typedef struct
{
    long mytype;//類型(首位)
    char mtext[64];
}MSG;
#define LEN(sizeof(MSG) - sizeof(long))

int main()
{
    MSG buf;
    ....
    if(msgrcv(msgid, &buf, LEN, 200, 0) < 0)
    {
        perror("msgrcv");
        exit(-1);
    }
    ....
    return 0;
}
```
#### 控制
```c
#include<sys/ipc.h>
#include<sys/msg.h>
int msgctl(int msgid, int cmd, struct msqid_ds *buf);
```
* 成功時返回 0,失敗時返回 -1
* cmd 要執行的操作 IPC_STAT/IPC_SET/IPC_RMID
* buf 存放隊列屬性的地址

## 練習
兩個進程通過消息隊列輪流將鍵盤輸入的字串發送給對方，接收並打印對方發送的消息
```c
/*clientA.c*/
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/type.h>
#include<sys/ipc.h>
#include<sys/msg.h>

typedef struct
{
    long mytype;
    char mtext[64];
}MSG;
#define LEN(sizeof(MSG) - sizeof(long))
#define TypeA 100
#define TypeB 200
int main()
{
    key_t key;
    int msgid;
    MSG buf;
    if((key = ftok(".",'q')) == -1)
    {
        perror("ftok");
        exit(-1);
    }
    if((msgid = msgget(key, IPC_CREAT|0666)) < 0)
    {
        perror("msgget");
        exit(-1);
    }
    while(1)
    {
       buf.mytpe = TypeB;
       printf("input > ");
       fgets(buf.mtext,64,stdin);
       msgsnd(msgid,&buf, LEN, 0);
       if(strcmp(buf.mtext, "quit\n")==0) break;
       if(msgrcv(msgid, &buf, LEN, TypeA, 0) < 0)
       {
           perror("msgrcv");
           exit(-1);
       }
       if(strcmp(buf.mtext, "quit\n")==0) 
       {
           msgctl(msgid, IPC_RMID, 0);
           exit(0);
       }
       printf("recv from clientB : %s\n",buf.mtext);
    }
    return 0;
}
```
```c
/*clientB.c*/
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/type.h>
#include<sys/ipc.h>
#include<sys/msg.h>

typedef struct
{
    long mytype;
    char mtext[64];
}MSG;
#define LEN(sizeof(MSG) - sizeof(long))
#define TypeA 100
#define TypeB 200
int main()
{
    key_t key;
    int msgid;
    MSG buf;
    if((key = ftok(".",'q')) == -1)
    {
        perror("ftok");
        exit(-1);
    }
    if((msgid = msgget(key, IPC_CREAT|0666)) < 0)
    {
        perror("msgget");
        exit(-1);
    }
    while(1)
    {
       if(msgrcv(msgid, &buf, LEN, TypeB, 0) < 0)
       {
           perror("msgrcv");
           exit(-1);
       }
       if(strcmp(buf.mtext, "quit\n")==0)
       {
           msgctl(msgid,IPC_RMID,0);
           exit(0);
       }
       
       printf("recv from clientA : %s\n",buf.mtext);
       
       buf.mytpe = TypeA;//對方的類型
       printf("input > ");
       fgets(buf.mtext,64,stdin);
       msgsnd(msgid,&buf, LEN, 0);
       if(strcmp(buf.mtext, "quit\n")==0) break;
       

    }
    return 0;
}
```

## 信號燈
* 也叫做信號量，用於進程或線程的同步或互斥機制
* 信號燈的類型
    * POSIX 無名信號燈
    * POSIX有名信號燈
    * system v 信號燈
* 信號燈代表資源的數量

### 特點
* system V 信號燈是一個或多個信號燈的集合
* 可同時操作集合中多個信號燈
* 申請多個資源避免死鎖

### 使用步驟
* 打開/創建信號燈 semget
* 信號燈初始化 semctl
* P/V操作 semop
* 刪除信號燈 semctl

#### 創建
```c
#include<sys/ipc.h>
#inlcude<sys/sem.h>
int semget(key_t key, int nsems, int semflg);
```
* 成功時返回信號燈的id，失敗時返回-1
* key 和消息隊列關聯的key IPC_PRIVATE或 ftok
* nsems 集合中包含計數信號燈的個數
* semflg 標誌位 IPC_CREAT|0666 IPC_EXCL

#### 初始化
```c
#include<sys/ipc.h>
#inlcude<sys/sem.h>
int semctl(int semid, int semnum, int cmd,....);
```
* 成功時返回0，失敗時返回EOF
* semid 要操作的信號燈集
* semnum 要操作的集合中的信號燈編號
* cmd 執行的操作 SETVAL IPC_RMID
* union semun 取決於 cmd

範例：
假設信號燈集合中包含兩個信號燈，第一個初始化為2,第二個初始化0

```c
union semun myun;
myun.val = 2;
if(semctl(semid, 0, SETVAL, myun) < 0)
{
    perror("semctl");
    exit(-1);     
}
myun.val = 0;
if(semctl(semid, 1, SETVAL, myun) < 0)
{
    perror("semctl");
    exit(-1);     
}
```

## 信號燈P/V操作
```c
#include<sys/ipc.h>
#inlcude<sys/sem.h>
int semop(int semid, struct sembuf *sops, unsigned nsops);
```
* 成功時返回0，失敗時返回EOF
* semid 要操作的信號燈集id
* sops 描述對信號燈操作的結構體（陣列）

sembuf: 系統中已經定義

```
struct sembuf
{
    short semnum;
    short sem_op;
    short sem_flg;
};
```
* semnum 信號燈編號
* sem_op -1:P操作 1:V操作
* sem_flg 0/IPC_NOWAIT

範例：
父子進程通過system V信號燈同步對共享內存的讀寫
* 父進程從鍵盤輸入字串到共享內存
* 子進程刪除字串中的空格並打印
* 父進程輸入quit後刪除共享內存的信號燈集，程序結束

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<signal.h>
#include<signal.h>
#include<sys/types.h>
#include<sys/ipc.h>
#include<sys/shm.h>
#include<sys/sem.h>

#define N 64
#define READ 0
#define WRITE 1
union semun
{
	int val;
	struct semid_ds *buf;
	unsigned short *array;
	struct seminfo *__buf;
};

void init_sem(int semid, int s[], int n)
{
	int i;
	union semun myun;
	for(i = 0;i < n; i++)
	{
		myun.val = s[i];
		semctl(semid, i, SETVAL, myun);
	}
}
void pv(int semid, int num, int op)
{
	struct sembuf buf;
	buf.sem_num = num;
	buf.sem_op = op;
	buf.sem_flg = 0;
	semop(semid, &buf, 1);
}
int main()
{
	int shmid, semid, s[]={0,1};
	pid_t pid;
	key_t key;
	char *shmaddr;
	if((key = ftok(".",'s')) == -1)
	{
		perror("ftok");
		exit(-1);
	}
	if((shmid = shmget(key, N, IPC_CREAT|0666)) < 0)
	{
		perror("shmget");
		exit(-1);
	}

	if((semid = semget(key, 2, IPC_CREAT|0666)) < 0)
	{
		perror("shmget");
		goto _error1;
	}
	init_sem(semid, s, 2);
	if((shmaddr = (char *)shmat(shmid, NULL,0)) == (char *) -1)
	{
		perror("shmat");
		goto _error2;
	}
	//create child process
	if((pid = fork()) < 0)
	{
		perror("fork");
		goto _error2;
	}
	else if(pid == 0)
	{
		char *p,*q;
		while(1)
		{
			pv(semid, READ, -1);
			p = q = shmaddr;
			while(*q)
			{
				if(*q != ' ' )
				{
					*p++ = *q;
				}
				q++;
			}
			*p = '\0';
			printf("%s",shmaddr);
			pv(semid, WRITE,1);
		}

	}
	else
	{
		while(1)
		{
			pv(semid, WRITE, -1);
			printf("input > ");
			fgets(shmaddr, N,stdin);
			if(strcmp(shmaddr, "quit\n") == 0) break;
			pv(semid,READ,1);
		}
		kill(pid, SIGUSR1);
	}

_error2:
	semctl(semid, 0, IPC_RMID);
_error1:
	shmctl(shmid,IPC_RMID,NULL);

	return 0;
}
```








































