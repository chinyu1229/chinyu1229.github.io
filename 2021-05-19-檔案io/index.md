# Linux Chapter 4 - 檔案IO


# 檔案I/O

#### 概述
全部執行I/O的系統呼叫都是透過檔案描述符來控制開啟的檔案，各類型的檔案在開啟之後，都能以檔案描述符進行操控。
### 標準檔案符
shell平時會將以下三個檔案描述符保持一直開啟，可以使用I/O redirection進行適當的修改
 

| file descriptor | 用途    | POSIX 名稱    | stdio stream |
| --------------- | ------- | ------------- | ------------ |
| 0               | 標準輸入 | STDIN_FILENO  | stdin        |
| 1               | 標準輸出 | STDOUT_FILENO | stdout       |
| 2               | 標準錯誤 | STDERR_FILENO | stderr       |
 
### 開啟檔案
open()
```c
#include<sys/stat.h>
#include<fcntl.h>
int open(const char *pathname, int flags, ... /*mode_t mode*/);
                                            
                 Return file descriptor on success, -1 on error
```
flags -> 為位元mask 指定檔案的存取模式
mode -> 指定了檔案的存取權限（如果open()沒有指定`O_CTEAT`flags可以省略）
* 由於flags各個參數互相獨立（除了必選項不可重複）皆可以使用`|`來新增性質

#### 必選項：以下三個常數中必須指定一個，且僅允許指定一個。



| flags    | 用途 |
| -------- | -------- |
| O_RDONLY | 以唯讀模式開啟     |
| O_WRONLY | 以唯寫模式開啟     |
| O_RDWR   | 以讀寫模式開啟     | 

`以可讀寫方式打開文件。上述三種旗標是互斥的，也就是不可同時使用，但可與下列的旗標利用OR(|)運算符組合。`

`以下可選項可以同時指定0個或多個，和必選項按位`|`起來作為flags參數。`


| flags       | 用途                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------- |
| O_CLOEXEC   | 設定close-on-exec flags                                                                     |
| O_CREAT     | 若檔案不存在則建立                                                                          |
| O_DIRECTORY | 如果pathname不是目錄，失敗                                                                  |
| O_EXCL      | 結合O_CREAT參數使用，專門用來建立檔案，若創建文件時，文件已存在，會返回錯誤訊息                                                       |
| O_LARGEFILE | 在32位元系統中使用，可以開啟大檔案                                                          |
| O_NOCTTY    | 不要讓pathname所指向的終端設備，成為控制終端機                                              |
| O_NOFOLLOW  | 如果參數pathname 所指的文件為一符號連接，則會令打開文件失敗                                 |
| O_TRUNC     | 若文件存在並且以可寫的方式打開時，此旗標會令文件長度清為0，而原來存於該文件的資料也會消失。 |
| O_APPEND    | 總在檔案結尾新增資料                                                                        |
| O_ASYNC     | 進行I/O操作時，產生signal                                                                   |
| O_DIRECT    | 檔案I/O跳過 buffer cache                                                                    |
| O_DSYNC     | 提供同步I/O的資料完整性                                                                     |
| O_NOATIME   | 在`read()`時不更新最近的存取時間                                                            |
| O_NONBLOCK  | 以非阻塞的方式開啟                                                                          |
| O_SYNC      | 以同步的方式寫入檔案                                                                        |
 
 
詳細內容可以參考The linux programming interface 國際中文版p.84
man page : https://man7.org/linux/man-pages/man2/open.2.html

  
### 創建檔案
creat()
根據pathname建立並開啟一個檔案，若檔案存在=>開啟並清空檔案內容
```c
#include<fcntl.h>
int creat(const char* pathname, mode_t mode);
```
可以等價於以下的open()函數
```c
fd = open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

 
### 讀取檔案
read()
```c
#include<unist.h>
ssize_t read(int fd, void *buffer, size_t count);

/*returns number of bytes read,0 on EOF,or -1 on error*/
```
count 指定最多可以讀取的byte數量
buffer 提供用來存放輸入資料的buffer cache位址
buffer cache至少有count個bytes
 
man page : https://man7.org/linux/man-pages/man2/read.2.html
 
 ### 寫入檔案
 write()
 ```c
 #include<unistd.h>
 ssize_t write(int fd, const void *buffer, size_t count);
 
                 Return number of bytes written, or -1 on error
 ```
count 指定最多可以要從buffer寫入檔案的byte數量
buffer 要寫入檔案的資料bytes數
 
### 關閉檔案
```c
#include<unistd.h>
int close(int fd);
                    Return 0 on success, or -1 on error
```                    
 
### 改變檔案偏移量
有時候也稱為讀寫偏移量或指標。指的是**執行下一個read() or write()操作的檔案位置**。檔案開啟時或指向開頭（偏移量 = 0）
針對fd參數所代表的已開啟檔案，可以使用lseek() function
```c
#include<unistd.h>
off_t lseek(int fd,off_t offset, int whence);
                Return new file offset if successful, or -1 on error
```
offset指定以byte為單位的數值,whence表示應參考哪個基準點來解釋offset參數，如下列表格



| whence   | 用途 |
| -------- | -------- |
| SEEK_SET | 將檔案偏移量從檔案的起始點開始算 offset 個 bytes         |
| SEEK_CUR | 相對於目前的偏移量，調整offset個 bytes         |
| SEEK_END | 將檔案偏移量設定為檔案大小加上offset 個 bytes，可以說offset應從檔案最後一個byte之後的下一個byte開始算起     |
 
一些lseek()的應用案例

```c
lseek(fd,0,SEEK_SET);     /*Start of file*/
lseek(fd,0,SEEK_END);     /*Next byte after the end of the file*/
lseek(fd,-1,SEEK_END);    /*Last byte of file*/
lseek(fd,-10,SEEK_CUR);   /*Ten bytes prior to current location*/
lseek(fd,10000,SEEK_END); /*10001 past last byte of file*/
```
**lseek()不允許應用於pipe,FIFO,socket or terminal**
 
### 檔案空洞(file hole)
如果檔案偏移量跨越了檔案結尾，然後再執行I/O操作，`read()`會return 0，但`write()`可以在檔案結尾後任意寫資料進去。
從檔案結尾之後到新寫入資料之間的*這段空間*就稱為檔案空洞。
* **檔案空洞不佔用任何空間，優勢在於相較於為實際需要的空bytes分配磁碟區塊，稀疏填充的檔案會佔用較少的空間。**

以下為apue範例
```c
#include "apue.h"
#include <fcntl.h>

char	buf1[] = "abcdefghij";
char	buf2[] = "ABCDEFGHIJ";

int
main(void)
{
	int	fd;

	if ((fd = creat("file.hole", FILE_MODE)) < 0)
		err_sys("creat error");

	if (write(fd, buf1, 10) != 10)
		err_sys("buf1 write error");
	/* offset now = 10 */

	if (lseek(fd, 16384, SEEK_SET) == -1)
		err_sys("lseek error");
	/* offset now = 16384 */

	if (write(fd, buf2, 10) != 10)
		err_sys("buf2 write error");
	/* offset now = 16394 */

	exit(0);
}
```
**running result**
```shell
$ ./a.out
$ ls -l file.hole
-rw-r--r-- 1 sar    16394   <date>  file.hole
$ od -c file.hole
0000000  a b c d e f g h i j \0 \0 \0 \0 \0 \0
0000020 \0 \0 \0 \0 \0 \0 \0 \0 \0 \0 \0 \0 \0
*
0040000  A B C D E F G H I J
0040012
```
可以使用du -h 指令 => 顯示檔案佔用的block是多少






