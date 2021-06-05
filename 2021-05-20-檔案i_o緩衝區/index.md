# Linux Chapter 13 - 檔案IO緩衝區

# 檔案I/O緩衝區
The Linux programming interface chapter 13
## 核心緩衝區概述
system call read() & write()在操作磁碟檔案時不會直接對磁碟存取，而是單純在使用者空間緩衝區與核心緩衝區快取（kernel buffer cache）之間複製資料。
例如： `write(fd, "abc",3)`
在後續的某個時間點，核心會將其*緩衝區*的資料寫入磁碟
 
**如果在此期間，另外一個process試圖讀取檔案的這幾個bytes，則核心將自動從緩衝區提供資料，而不是從檔案讀取。同理於read()期間有write() process的情況**

## 緩衝區大小對效能的影響
![](https://i.imgur.com/OVUHxqa.jpg)
* system cpu time：主要是測量使用者空間與核心空間之間資料傳輸所消耗的時間
    * 包含CPU的處理, 等待I/O I/O的處理,作業系統處理提供一些服務額外的時間，以及系統閒置的時間
* Elapsed time : 完整的程式在這個系統裡面執行的時間
 
當緩衝區大小為1byte，需要呼叫大約一億次的system call，而緩衝區大小為4096bytes時，system call次數約為24000，當超過此設定值，效能的改善就開始不明顯，是因為在使用者空間和核心空間之間複製資料以及執行實際I/O所需的時間相較之下，read()和write()的system call 成本就顯得微不足道。

＊ 結論： 若我們正對檔案進行大量的資料讀寫，則以大區塊緩衝資料來減少system call 即可提升系統效能

## stdio函式庫的緩衝
操作磁碟檔案時，對大型區塊資料使用緩衝可以降低system call。
### 設定一個stdio stream的緩衝模式
```c
#include<stdio.h>
int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```

**在開啟串流之後，必須在對串流進行任何其他的stdio函式之前先呼叫setbuf() 因為setbuf()將影響後續的stdio操作對串流的行為**
* 參數
    * stream:可以識別要修改的檔案stream緩衝區
    * buf & size 指定使用的緩衝區
        * 若buf不為NULL則以其指向的size大小區塊作為stream的緩衝區，因為stdio接著會使用buf指向的緩衝區，所以應該以靜態配置或使用malloc()於heap做動態配置，不應該配置為stack裡面的區域變數，否則當函式返回時會釋放stack frame，導致混亂
        * 若buf為NULL，stdio會自動配置一個緩衝區提供stream使用（除非我們選擇無緩衝的I/O)。在SUSv3允許但不要求實作時使用size來決定緩衝區的大小
    * mode指定緩衝區的類型![](https://i.imgur.com/sIsu0YR.jpg)

### 範例
```c
#defind BUF_SIZE 1024
static char buf[BUF_SIZE];
if(setvbuf(stdout, buf, _IOFBF) != 0)
    errexit("setbuf");
```
＊setvbuf()出錯時會傳回非0值
 
setbuf()位於setvbuf上層
```c
#include<stdio.h>
void setbuf(FILE *stream, char *buf);
```
* 呼叫`setbuf(fp,buf)`等同於`setvbuf(fp, buf, (buf != NULL) ? _IOFBF: _IONBF, BUFSIZ);`
* 參數
    * buf可以指定為NULL表示無緩衝，或指向由呼叫者配置的BUFSIZ大小的緩衝區（BUFSIZ定義在<stdio.h>通常是8192）
 
setbuffer()與setbuf()相似，但允許呼叫者指定buf緩衝區的大小
```c
#define _BSD_SOURCE
#include<stdio.h>
void setbuffer(FILE *stream, char *buf, size_t size);
```
* SUSv3並未規範此函式，但多數UNIX有提供

### 刷新stdio緩衝區
我們可以用fflush()函式強制寫入stdio的輸出串流資料（如透過`write()`刷新寫入核心緩衝區）。此函式會刷新指定stream的輸出緩衝區
```c
#include<stdio.h>
int fflush(FILE *stream);
```
* 參數
    * 若stream為NULL，則fflush將刷新與**output stream**相關的每個stdio緩衝區
* 也能應用於輸入串流，這將捨棄緩衝區裡面的所有輸入資料（當程序下次嘗試從串流讀資料，將重頭開始寫入緩衝區）
* 當關閉對應的串流時，會自動刷新stdio的緩衝區
* 在許多c函式庫實作中，若stdin stdout是參考到terminal，則從stdin讀取輸入時，都會進行一次`fflush(stdout)`呼叫，效果是刷新寫入stdout的任何提示，但不包含結尾的換行符號
範例1:
```c
#include<stdio.h>
int main()
{
    printf("hello");
    fflush();//fflush(stdout);
    return 0;
}
```
常用於立刻將輸入值印出來。

範例2：
```c
#include<stdio.h>
int main()
{
    int i;
    while(1)
    {
        printf("Enter a integer:");
        scanf("%d",&i);
        printf("%d\n",i);
    }
    return 0;
}
```
此範例如果輸入非int類型的值，則輸入值會放入緩衝區，而導致你下個迴圈的輸入無法被讀到，因為scanf會一直去緩衝區讀取，而不會理會使用者輸入。可以在scanf()前面加上，fflush(stdin)，但不是每個系統都有支援fflush(stdin)。這時候可以利用一些特殊的函數（內部本身就包含fflush()），來解決這種問題。

### 控制檔案I/O的核心緩衝

可以強制將核心緩衝區刷新到輸出檔，例如資料庫的日誌行程，要確保在繼續操作前將結果真正的寫入到硬碟（或硬碟的快取）

#### 同步I/O資料完整性與同步I/O檔案完整性
SUSv3將同步I/O完成(synchronized IO completion) 定義為： 一種I/O操作，若非已成功傳輸（到硬碟），及診斷為同步失敗

有兩種不同類型的同步I/O完成
1. synchronized I/O data integrity completion(資料完整性)
    * 對於read操作，意思是所需的檔案已經（從硬碟）傳輸給process。若有任何未完成的write操作並且會影響資料的read時，會在進行read前，將資料寫入磁碟
    * 對於write操作，意思是在write請求的指定資料已經傳輸（到硬碟）完成，而且用以取得資料所需的全部檔案中的**中繼資料**也已經全部傳輸（到硬碟）完成。
        * 關鍵在於不需要將全部的已修改檔案中的中繼資料屬性傳輸完成，就可以檢索檔案資料
        * 需要傳輸的其中一個中繼資料屬性是檔案的大小 
        
2.  synchronized I/O file integrity compliction(檔案完整性)
    * 與資料完整性的差異在於，在檔案更新期間，會將**全部已更新的檔案中繼資料傳輸到硬碟**，即使後續的檔案read操作不需要檔案中繼資料

**中繼資料（metadata）：檔案擁有者與所屬群組 檔案權限，檔案大小，檔案的（hard）連結數量，檔案最後的存取時間，最後修改時間，中繼資料發生變化的時間，檔案資料區塊指標**


#### 用於控制檔案I/O核心緩衝的系統呼叫
```c
#include<unistd.h>
int fsync(int fd);
                        成功返回0,失敗返回-1
```
* 將緩衝資料與開啟檔案的fd相關的metadate刷新到磁碟
* 會強制檔案處於I/O檔案完整性狀態
* 會在完成磁碟裝置（或快取記憶體）的傳輸之後才返回

```c
#include<unistd.h>
int fdatasync(int fd);
                       成功返回0,失敗返回-1  
```
* 會強制檔案處於I/O資料完整性狀態
* 操作與fsync()類似
* 可能會降低磁碟操作的次數

```c
#include<unistd.h>
void sync(int void);
```
* 會將包含更新檔案資訊的全部核心緩衝區（資料區塊，指標區塊，中繼資料等）刷新到磁碟
* 在已將全部資料傳輸到磁碟裝置（或者是快取）之後返回
* 在SUSv3中，實作時只是單純對I/O傳遞進行排班，因此可以在未完成之前返回

#### 同步全部的寫入檔案
在呼叫open()時指定O_SYNC flag可以讓全部的後續輸出同步
```
fd = open(pathname,O_WRONLY | O_SYNC);
```
每次write()檔案都會自動將檔案資料與中繼資料刷新到硬碟（依照檔案完整性進行寫入操作）

關於O_SYNC對效能的影響可以參考 書籍p.265

#### I/O緩衝摘要
![](https://i.imgur.com/EanlGPj.jpg)

#### 繞過緩衝區快取：直接I/O
在進行磁碟I/O時，繞過緩衝區快取，可以直接從使用者空間將資料傳輸到檔案或磁碟裝置。
direct I/O有時反而會大幅降低效能，因為核心為了改善效能，進行的許多優化。因此只適用於有特定I/O需求的應用，例如資料庫系統有自己的快取和I/O優化，所以不需要核心消耗CPU的時間與記憶體去進行同樣的工作。

* 可以對個別的檔案或block device進行直接I/O，可以在open()指定O_DIRECT flag。

direct I/O的對齊限制
* 傳虛用途的資料緩衝區，必須對齊符合block size倍數的記憶體邊界
* 資料傳輸的開始點，即檔案或裝置的偏移量，必須是block size的倍數
* 待傳輸資料的長度必須是block sizek的倍數

#### 為檔案I/O混搭函式庫函式與system call
```c
#include<stdio.h>
int filno(FILE *stream);
                            Returns file descriptor on sucess, or -1 on error
FILE *fdopen(int fd, const char *mode);
                            Returns (new) file pointer on success, or NULL on error
```
對同一個檔案進行I/O操作時，可以混搭system call與標準C函式庫的函式

* fdopen()是fileno()的反函式，此函式會建立一個與此檔案描述符對應的stream給I/O使用
* fdopen()對非正規檔案的描述符特別有用，例如建立通訊（socket）和 pipe的system call

