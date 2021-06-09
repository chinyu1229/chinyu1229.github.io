# Linux Chapter 20 Signal 基本概念

# Linux Chapter 20 Signal 基本概念

## 概念
signal 可以通知行程（process）有事件發生，也可以稱為軟體中斷，多數情況下都無法預測訊號抵達的時間。
 
行程可以送出訊號給另外一個行程，在此可以做為synchronization 或者 IPC 的技術。

## 訊號類型與預設行為

可以在signal(7)使用手冊列出訊號名稱，或是參考書上p.430
下圖列出簡易版訊號表：
![](https://i.imgur.com/5bYVDns.jpg)

![](https://i.imgur.com/WzFNTuu.jpg)

## 改變訊號處置
Unix系統提供兩種 1.signal() 2. sigaction()
* signal()提供設定訊號的原始API 介面比sigaction()簡單
* sigaction()是建立訊號處置常式首推的API
* signal() 是基於 sigaction()實作的函式

```c
#include<signal.h>
void (*signal(int sig, void (*handler)(int)));
        Return previous signal disposition on success,or SIG_ERR on error
```
* handler可以用以下的值取代
    * SIG_DFL:將訊號處置 重新設定為預設值，適用於還原先前signal()呼叫改變的訊號處置
    * SIG_IGN:忽略該訊號，行程也不會知道有此訊號

## 訊號處理常式

範例：
```c
#include<signal.h>
#include"tlpi_hdr.h"

static void sigHandler(int sig)
{
    printf("Ouch!\n");
}
int main(int argc[], char *argv[])
{
    int j;
    if(signal(SIGINT, sigHandler) == SIG_ERR)
        errExit("signal");
    for(j = 0;;j++)
    {
        printf("%d\n", j);
        sleep(3);
    }
}
```
result:
```
$ ./ouch
0
*Type Control-C*
Ouch!
1
2
*Type Control-C*
Ouch!
3
'Type Control-\'(the terminal quit char)
Quit(core dumped)
```

## 發送訊號 kill()
```c
#include<signal.h>
int kill(pid_t pid, int sig);
            Return 0 on success, or -1 on error
```
* 若pid > 0，會將訊號發給pid指定的行程
* 若pid == 0，會將訊號發送至與呼叫的行程**同一個群組的每一個行程**
* 若pid < -1，在行程群組ID與pid絕對值相同的行程群組中的每個行程都會收到此訊號
* 若pid == -1，會將訊號發送到呼叫的行程有權將訊號送出的每個行程（不包含init，以及呼叫行程本身）。若特權級行程執行此呼叫，則系統上每個行程都會收到此訊號

要讓行程將訊號發送給另外一個行程需要的權限，規則如下：
* 特權級CAP_KILL行程可以發送訊號給任何行程
* 以root使用者身份與群組身份執行的init行程是一個特例，只能發送已安裝的處理常式之訊號，可以避免系統管理員意外殺死init行程。

* SIGCONT，非特權行程可以將此訊號發送給相同作業階段（session）的任何其他行程，而不需理會使用者ID的檢查。此規則可以讓進行工作控制的shell重新啟動已停止的工作。

## 其他發送訊號的方式：raise()、killpg()

發送訊號給自己
```c
#include<signal.h>
int raise(int sig);
            Returns 0 on success, or nonzero on error
```
等同於：`kill(gitpid(), sig);`

-------
送出一個訊號給一個行程群組裡的每個成員

```c
#include<signal.h>
int killpg(pid_t pgrp, int sig);
                Return 0 on success, or -1 on error
```

等同於：`kill(-pgrp, sig);`

## 顯示訊號的說明
訊號的說明文字在sys_siglist陣列中，例如可以使用sys_siglist[SIGPIPE]，獲得訊號說明。但建議使用strsignal()函式。

```c
#define _BSD_SOURCE
#include<signal.h>

extern const char *const sys_siglist[];

#define _GNU_SOURCE
#include<string.h>
char *strsingnal(int sig);
                Returns pointer to signal description string
```

* 會檢查sig的邊界，並傳回一個指標指向該訊號的說明字串，若此訊號編號無效，就會指向錯誤字串
* 比起直接使用sys_siglist[] 的優點在於strsignal()是local-sensitive，因此訊號的說明會根據地域語言來呈現。

psignal()顯示msg參數所指的字串，後面接著一個冒號，如同strsignal()一樣，也是local-sensitive。
```c
#include<signal.h>
void psignal(int sig, const char *msg);
```
* strsignal() psigna() sys_siglist並未納入SUSv3的標準中，但許多UNIX的系統有這些功能（SUSv4已納入）


## 訊號集(Signal Set)

可以使用一個singal set的資料結構表示多個訊號，型態為sigset_t

-----

### 初始化訊號

sigemptyset()初始化成一個未包含任何成員的訊號集
sigfillset()初始化一個訊號集，使其包含全部的訊號
```c
#include<signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
                Return 0 on success, or -1 on error
```

* 一定要用其中一個函式來初始化一個訊號集，因為C語言不會對automatic variable進行初始化
* 並且將靜態變數初始化為0的清空訊號集的方法不具可攜性，因此使用memset清空的方法也不可行

初始化之後，可以使用sigaddset()、sigdelset()將訊號添加或者刪除

```c
#include<signal.h>

int sigaddset(const sigset_t *sig, int sig);
int sigdelset(const sigset_t *sig, int sig);
                    Return 0 on success, or -1 on error
```
* 可以用sigismember()測試sig指定的訊號是否為set的成員

```c
#include<signal.h>
int sigismember(const sigset_t *set, int sig);
                Return 1 if sig is a member of set, 0 if it is not, or -1 on error
```

GNU C 函式庫實作了三個非標準函式
```c
#define _GNU_SOURCE
#include<signal.h>

int sigandset(sigset_t *dest, sigset_t *left, sigset_t *right);
int sigorset(sigset_t *dest, sigset_t *left, sigset_t *right);
                        Return 0 on success, or -1 on error
int sigisemptyset(sigset_t *set);
                        Return 1 if set is empty, otherwise 0
```
* sigandset()將left和right sets 交集 存在dest set中
* sigandset()將left和right sets 聯集 存在dest set中
* 若set內沒有signal則sigisemptyset return 1


## 訊號遮罩(signal mask)

kernel會為每個process維護一個singal mask，即一組傳遞給行程並會受到blocked的訊號，若將受到阻塞的訊號發送給行程，則訊號會被延遲傳遞，直到從行程的signal mask移除此訊號才會解除blocked。

可用下列方式將訊號新增到訊號遮罩：
* 當呼叫訊號處理常式，可以將觸發呼叫的訊號，自動新增到訊號遮罩中。
* 使用sigaction()，可以指定一組要阻塞的額外訊號，handler會將其阻塞
* 在任意時間點，sigprocmask()，直接將訊號新增(刪除)到signal mask

```c 
#include<signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
                        Return 0 on success, or -1 on error
```
* how參數
    * SIG_BLOCK 會將mask設定為 現值`OR`set => 將set指向的signal set添加到mask中
    * SIG_UNBLOCK 將set指向的signal set從mask中刪除
    * SIG_SETMASK 給set指向的signal set設定 signal mask
    

## 擱置的訊號 (Pending Signal)
行程收到一個目前被blocked的訊號，會加入pending signal set，可以用sigpending()來獲得行程目前的擱置訊號

```c
#include<signal.h>
int sigpending(sigset_t *set);
                    Return 0 on success, or -1 on error
```

## 等待訊號 pause()
會使process暫停執行，直到受到**handler中斷**為止，或是一個未經處理的訊號終止process

```c
#include<unistd.h>

int pause(void);
            return -1 with errno set to EINTR
```








