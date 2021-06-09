# Linux Chapter 21 Signal 處理常式


# Linux Chapter 21 Signal 處理常式

## 訊號處理常式設計
1. 設定一個全域flag，接著離開。主程式會週期性的檢測此flag，若有被設定為flag，會進行適當的反應
2. 會做出幾種類型的清理動作，接著結束行程或使用非區域跳躍（nonlocal goto）解開unwind the stack，並將控制權交回給主程式（是先定義的位置）

### 可重入函式與非同步訊號安全函式

#### 可重入與不可重入
訊號處理常式與multiple thread的概念有關，前者可能會在**任意時間點非同步的**中斷程式執行，所以主程式與訊號處理常式會變成在同一個行程中，2個獨立的thread(非同步執行)
若函式可以在同一個行程的各thread中**同步且安全的**執行，此函式稱為是可重入的（reentrant）
* SUSv3對可重入的定義是: 當兩條或多條thread呼叫函式時，即便是彼此互相交叉執行，也能保證效果與各thread以未定義順序呼叫時一致

若函式會更新全域或靜態的資料結構，可能就是不可重入的函式（只使用local變數的函式保證是可重入）。
* 可能發生的情況： 若主程式在呼叫`malloc()`期間，受到一個同樣呼叫`malloc()`的訊號處理常式中斷，則此linklist可能會遭到破壞，因此`malloc()`的函式家族與使用這些函式的其他函式庫函式都是不可重入。
* 其他會傳回靜態配置的記憶體，也是不可重入的，`crypt() getpwnam() gethostbyname() getservbyname()`
* 函式的內部紀錄是使用靜態的資料結構也是不可重入的，例如`scanf() printf()`，他們會有緩衝的I/O更新內部資料結構，所以在訊號處理常式使用`printf()`也在主程式呼叫`printf()`，緩衝區的資料會交錯，導致得到非期望的輸出結果，甚至是整個程式crush

#### 標準的非同步訊號安全函式（async-signal-safe functions）
* 指可以安全地從訊號處理常式進行呼叫。
* 當函式是reentrant函式，或是不會受到訊號處理常式中斷，可以稱函式為async-signal-safe function
* man page: https://man7.org/linux/man-pages//man7/signal-safety.7.html


#### 在訊號處理常式內使用errno
因為可能會更新errno變數，會導致函式reentrant，因為可能會覆寫主程式之前呼叫函式設定的errno值。
所以可以先儲存errno的值，並在函式執行完畢之後，回存errno

```c
void handler(int sig)
{
    int sErrno;
    sErrno = errno;
    /* can execute a function that might modify errno */
    errno = sErron;
}
```

### 全域變數與`sig_atomic_t`資料型別
全域變數有reetrant問題，但有時需要在主程式與訊號處理常式之間共用全域變數，要能正確處理即可以保證安全。
常見的設計是：
* 設定全域旗標，主程式會定期檢查，並採取動作來對收到的訊號做出反應（並且清除旗標），用此方式存取全域變數時，需要使用`volatile`來宣告，避免編譯器進行優化，而讓變數存在register。
  * 可以宣告成： `volatile sig_atomic_t flag`，其中sig_atomic_t 可以保證原子操作。
  * C語言中的`++` `--` 不在sig_atomic_t 的範圍，要避免使用
  * 在實作時要定義`SIG_ATOMIC_MIN` 、 `SIG_ATOMIC_MAX`，有號值域：-127 ~ 127、無號值域：0 ~ 255

## 終止訊號處理常式的其他方法
先前提到的訊號處理常式在完成後通常是傳回主程式，有時候也需要一些其他的處理，例如：
* 使用`_exit()`終止行程，處理常式先進行一些清理的動作，（不使用`exit()`是因為他是非安全函式)
* 使用`kill()` 、 `raise()`送出訊號，以刪除行程
* 從訊號處理常式執行nonlocal跳轉
* 使用`abort()`終止，並core dump

### Nonlocal 跳轉
可以利用`setjmp()` `longjmp()`進行跳轉，可以在收到硬體異常訊號時進行回復，並且可以catch訊號，同時可以將控制權傳回程式的特定位置，
例如：收到`SIGINT`，shell會執行nonlocal跳轉使得可以跳回主輸入迴圈中，以便讀取新的輸入。
 
進入訊號處理函式時，核心會自動將觸發的訊號以及在`act.sa_mask`欄位指定的每個訊號都加到行程的signal mask，並在正常返回時，將訊號從mask中解除。
因此若使用標準的`longjmp()`離開常式，在某些平台，不會將阻塞的訊號解除。

所以在POSIX.1-1990中，另外定義了新的函式
```c
#include<setjmp.h>
int sigsetjmp(sigjmp_buf env, int savesigs);
                            Return 0 on initial call, nonzero on return via siglongjmp()
void siglongjmp(sigjmp_buf env, int val);
```

* 若savesigs設定為nonzero，則呼叫`sigsetjmp()`時，行程目前signal mask會存在env中，之後可將env傳給`siglongjmp`進行還原，若設定為0，則不會儲存。
* 兩個函式都**不是**非同步訊號安全函式
* 若訊號處理常式中斷的主程式正在更新資料結構，而訊號處理常式再進行nonlocal跳轉結束，使得主程式未完成更新，可以使用`sigprocmask()`避免此問題，暫時將訊號堵塞。

### abort()
終止行程，並core dump
```c
#include<stdlib.h>
void abort(void);
```
* 產生`SIGABRT`來終止行程
* 無論阻塞或者忽略`SIGABRT`訊號，`abort()`不受影響，除非行程catch到`SIGABRT`訊號後handler未返回，否則`abort()`必須終止行程
* 若成功終止，還會刷新stdio stream並關閉
* 使用nonlocal跳轉可以抵消`abort()`的效果

## 替代堆疊中處理訊號：sigaltstack()
呼叫訊號處理常式，核心會在行程stack中為其建立frame，若stack超過大小限制時，則無法為處理常式新增frame，因此不會被呼叫。
會有以下的做法：
1. 配置一塊**替代訊號堆疊**的記憶體區域，作為訊號處理常式的frame
2. 呼叫`sigaltstack()`，以通知核心替代訊號堆疊的存在
3. 在建立訊號處理常式時，指定`SA_ONSTACK`旗標，通知核心在替代堆疊上建立frame

```c
#include<signal.h>
int sigaltstack(const stack_t *sigstack, stack_t *old_sigstack);
                        Return 0 on success, or -1 on error
```

* sigstack指向新替代訊號堆疊的位置與屬性
* old_sigstack指向上一替代訊號堆疊的位置與屬性
* 指向的stack_t如下


To establish a new alternate signal stack, the fields of this structure are set as follows:

man page: `sigaltstack(2)`

       ss.ss_flags
              This field contains either 0, or the following flag:

              SS_AUTODISARM (since Linux 4.7)
                     Clear the alternate signal stack settings on entry
                     to the signal handler.  When the signal handler
                     returns, the previous alternate signal stack
                     settings are restored.

                     This flag was added in order make it safe to switch
                     away from the signal handler with swapcontext(3).
                     Without this flag, a subsequently handled signal
                     will corrupt the state of the switched-away signal
                     handler.  On kernels where this flag is not
                     supported, sigaltstack() fails with the error
                     EINVAL when this flag is supplied.

       ss.ss_sp
              This field specifies the starting address of the stack.
              When a signal handler is invoked on the alternate stack,
              the kernel automatically aligns the address given in
              ss.ss_sp to a suitable address boundary for the underlying
              hardware architecture.

       ss.ss_size
              This field specifies the size of the stack.  The constant
              SIGSTKSZ is defined to be large enough to cover the usual
              size requirements for an alternate signal stack, and the
              constant MINSIGSTKSZ defines the minimum size required to
              execute a signal handler.
```c
typedef struct{
    void   *ss_sp;    /* Starting address of alternate stack */
    int    ss_flags;  /* Flags: SS_ONSTACK, SS_DISABLE */
    size_t ss_size;   /* Size of alternate stack */
}   
```
* `ss_sp` `ss_size` 指定了位置和大小
* `ss_flag`包含
  * SS_ONSTACK
    * The process is currently executing on the alternate signal stack. Attempts to modify the alternate signal stack while the process is executing on it fail. This flag shall not be modified by processes.

  * SS_DISABLE
    * The alternate signal stack is currently disabled.





