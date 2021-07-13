# Linux Chapter 24 CreateProcess


# CreateProcess

## fork() exit() wait() execev() 簡介
1. fork()
   * parent process經由呼叫fork()建立一個 child process 
   * child獲得parent的stack segment、data segment、heap segment、text segment
   * 可說是把parent process一分為二
2. exit()
   * terminate a process 
   * 將佔用的所有資源歸還給kernel
   * parent 可以利用 wait()來取得結束的狀態(status)
3. wait()
   * 若child process還未呼叫exit()，那wait()會suspend parent process，直到有任一child process terminated
   * 可以取得status
4. execve()
   * Load a new program到目前process的記憶體
   * 丟去現存的text segment
   * 重新建立 stack segment、data segment、heap segment

大致流程圖
<img src="process.png" width = "70%" /> 

## fork()
```c
#include <unistd.h>
pid_t fork(void);
    In parent: returns process ID of child on success, or –1 on error; in successfully created child: always returns 0
```



