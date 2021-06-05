# Linux Chapter 21 Signal 處理常式


# Linux Chapter 21 Signal 處理常式

## 常式設計
1. 設定一個全域flag，接著離開。主程式會週期性的檢測此flag，若有被設定為flag，會進行適當的反應
2. 會做出幾種類型的清理動作，接著結束行程或使用非區域跳躍（nonlocal goto）解開unwind the stack，並將控制權交回給主程式（是先定義的位置）

## 可重入函式與非同步訊號安全函式

### 可重入與不可重入
訊號處理常式與multiple thread的概念有關，前者可能會在**任意時間點非同步的**中斷程式執行，所以主程式與訊號處理常式會變成在同一個行程中，2個獨立的thread(非同步執行)
若函式可以在同一個行程的各thread中**同步且安全的**執行，此函式稱為是可重入的（reentrant）
* SUSv3對可重入的定義是: 當兩條或多條thread呼叫函式時，即便是彼此互相交叉執行，也能保證效果與各thread以未定義順序呼叫時一致

若函式會更新全域或靜態的資料結構，可能就是不可重入的函式（只使用local變數的函式保證是可重入）。
* 可能發生的情況： 若主程式在呼叫`malloc()`期間，受到一個同樣呼叫`malloc()`的訊號處理常式中斷，則此linklist可能會遭到破壞，因此`malloc()`的函式家族與使用這些函式的其他函式庫函式都是不可重入。
* 其他會傳回靜態配置的記憶體，也是不可重入的，`crypt() getpwnam() gethostbyname() getservbyname()`
* 函式的內部紀錄是使用靜態的資料結構也是不可重入的，例如`scanf() printf()`，他們會有緩衝的I/O更新內部資料結構，所以在訊號處理常式使用`printf()`也在主程式呼叫`printf()`，緩衝區的資料會交錯，導致得到非期望的輸出結果，甚至是整個程式crush

### 標準的非同步訊號安全函式（async-signal-safe functions）
* 指可以安全地從訊號處理常式進行呼叫。
* 當函式是reentrant函式，或是不會受到訊號處理常式中斷，可以稱函式為async-signal-safe function
* 我們**不可以**在訊號處理常式呼叫**不安全的函式**
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



