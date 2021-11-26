# Golang - 並行開發 --- Mutex

# Golang 並行（concurrent）開發

## 共享資源訪問控制

### Mutex

#### 結構
```go
type Mutex struct{
  state int32
  sema uint32
}
```

state紀錄Mutex的四種狀態：主要兩個為`Locked`:表示是否被鎖定、`Waiter`:表示blocked中且等待拿鎖的goroutine數量

#### Method

`Lock()`：加鎖
`Unlock()`：解鎖

加鎖過程：
* 去判斷Locked值是否為0，若是0，把Locked set為1，代表拿到鎖
* 若是1，Waiter的值+1，並且此goroutine blocked 直到 Locked == 0才會被喚醒

解鎖過程：
* 若沒有其他goroutine blocked，只要把Locked set 為 0，不需要釋放 semaphore
* 若有其他blocked goroutine，首先先把Locked set 為 0，然後查看到Waiter>0，釋放一個 semaphore，喚醒一個blocked goroutine，被喚醒的goroutine把Locked set 為1



#### 自旋

lock時，如果Locked為1，嘗試lock的goroutine不是馬上進入blocked，而是會持續地(時間很短)探測Locked是否為0，好處是可以**避免goroutine的切換**


* 相當於CPU的PAUSE指令，過程中會持續探測Locked（與sleep不同的是不需要將goroutine轉成睡眠狀態）
* 自旋要滿足
  * 自旋次數夠小，通常設定為4
  * CPU核心數>1
  * scheduler 的可運行queue必須為空

也就是CPU不忙的時候啟用自旋

問題：
如果自旋過程中獲得鎖，避免了goroutine切換，那之前被blocked的goroutine很難獲得鎖，造成starving
因此新版的go新增了Mutex -- starving欄位


#### Mutex模式

state有個Starving的欄位，預設情況下為Normal，goroutine如果加鎖不成功，會判斷是否滿足自旋的條件，滿足即嘗試搶鎖。

如果沒有自旋的goroutine可能會造成starving，因此若block超過1ms將Starving欄位設定成starving模式，並且再次進入blocked狀態。

處於starving模式，一旦有goroutine unlock，一定會喚醒goroutine


#### Mutex 簡單示範
```go
var m *sync.Mutex

func mutexDemo(i int){
	fmt.Println(i, " try to obtain Lock")
	m.Lock()
	fmt.Println(i, " in Lock")
	time.Sleep(1000)
	fmt.Println(i, " release Lock")
	m.Unlock()
}

func main()  {
	m = new(sync.Mutex)
	go mutexDemo(1)
	go mutexDemo(2)

	time.Sleep(time.Second)

}
```

result:

```shell
2  try to obtain Lock
1  try to obtain Lock
2  in Lock
2  release Lock
1  in Lock
1  release Lock

```


#### RWMutex

此鎖解決了goroutine concurrent時只有一個有互斥鎖的goroutine可以執行，但其他被blocked，通常被運用在一寫多讀的情況下

#### 結構 & Method
```go
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}

func (*RWMutex) Lock 
func (*RWMutex) Unlock
func (*RWMutex) RLock   
func (*RWMutex) RUnlock
```

#### 邏輯

1. Write Lock
   * 獲取 w 的互斥鎖
   * block 等待所有的read操作結束
     * 依靠檢測readerCount(readerWait)

2. Write UnLock
   * wake 因為 read lock而被block的goroutine
   * 解除 w 互斥鎖

3. Read Lock (RLock())
   * 增加readerCount
   * blocked並等待write操作結束
     * readerCount最大為2^30，因此在w鎖定時先將readerCount減去2^30，因此檢測到為負值時就會知道w在鎖住，便會blocked等待

4. Read Unlock (RUnlock())
   * readerCount --
   * wake等待write操作的goroutine (最後一位reader)


#### 避免write lock餓死

通過 RWMutex.readerWait解決

當write操作時，會把RWMutex.readerCounter copy到RWMutex.readerWait ，用來標前方有幾個讀者，當RWMutex.readerWait變成0時喚醒write操作



#### Mutex v.s RWMutex

Mutexv下，讀寫互相鎖（一次只能讀或寫）

```go
var m sync.Mutex

func writer(val *int){
	m.Lock()
	*val += 100
	time.Sleep(5000)
	m.Unlock()
}
func reader(val *int){
	m.Lock()

	fmt.Println("reader: " ,*val)
	time.Sleep(1000)

	m.Unlock()
}

func main()  {
	var wg sync.WaitGroup
	n := 10
	num := 0

	wg.Add(n)
	for i:=0; i < n ; i ++{
		go func() {
			writer(&num)
			wg.Done()
		}()
	}
	wg.Add(n)
	for i:=0; i < n ; i ++{
		go func() {
			reader(&num)
			wg.Done()
		}()
	}
	wg.Wait()
}
```

RWMutex 下，多讀一寫

```go
var m sync.RWMutex

func writer(val *int){
	m.Lock()
	*val += 100
	fmt.Println("writer: ", *val)
	time.Sleep(1000)
	m.Unlock()
}

func reader(val *int){
	m.RLock()

	fmt.Println("reader: " ,*val)
	time.Sleep(3000)

	m.RUnlock()
}

func main()  {

	var wg sync.WaitGroup
	n := 50
	num := 0

	wg.Add(n)
	for i:=0; i < n ; i ++{
		go func() {
			writer(&num)
			wg.Done()
		}()
	}
	wg.Add(n)
	for i:=0; i < n ; i ++{
		go func() {
			reader(&num)
			wg.Done()
		}()
	}
	wg.Wait()
}
```
