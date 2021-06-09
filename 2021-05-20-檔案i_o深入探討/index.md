# Linux Chapter 5 - 檔案IO深入探討


# 檔案I/O深入探討
## 原子與競速條件
#### 以互斥的方式建立檔案
在`open()`中設定`O_EXCL`與`O_CREAT`可以在檔案已存在的時候回傳錯誤，確保process本身是檔案的建立者

#### 將資料附加到檔案
當我們有多個process要增加資料到同一個檔案
會有下列這種寫法
```c
if(lseek(fd,0,SEEK_END)==-1)
    errExit("lseek");
if(write(fd, buf, len)!=len)
    fatal("Partial/failed write");
```
會有race condtion問題，第一個process在lseek()與write()之間被第二個相同的行程中斷，那兩個行程會在寫入之前將他們的file offset設定到相同位置，因此會產生互相覆蓋的現象
* **為了避免此問題，必須讓移往檔案結尾下一個byte操作與寫入操作都是原子操作，我們可以使用`O_APPEND`flag達成**

## 檔案操作控制
```c
#include<fcntl.h>
int fcntl(int fd, int cmd,...);
        Return value on success depends on cmd, returns -1 on error
```
用途之一是用來取得或修改access mode,以及一個開啟檔案的開啟檔案狀態flag
**取得設定的cmd : F_GETFL**
```c
int flags, accessMode;
flags = fcntl(fd, F_GETFL);
if(flags == -1) errExit("fcntl");
```
可以使用 `&` 操作去判斷目前的開啟檔案狀態flag
```c
if(flags & O_SYNC)
    printf("write are synchronized\n");
```
若要檢查檔案的access mode,必須使用如下的操作
```c
accessMode = flags & O_ACCMODE;
if(accessMode == O_WRONLY || accessMode == O_RDWR)
    printf("file is writable\n");
```
* 詳細的原因可以參考書p.104

修改開啟檔案狀態flag
```c
int flags;
flags = fcntl(fd, F_GETFL);
if(flags == -1)
    errExit("fcntl");
flags |= O_APPEND; //新增一個新的狀態flag
if(fcntl(fd, F_SETFL,flags) == -1) //更新狀態
    errExit("fcntl");
```














