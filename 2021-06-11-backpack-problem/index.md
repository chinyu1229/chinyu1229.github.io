# Backpack_Problem


# Backpack 0 - 1 Problem
* 主問題：從n個物品選一些物品，在不超過最大容量下，使得價值最大。
* 解空間：{x<sub>1</sub>,x<sub>2</sub>....,x<sub>n</sub>}
  * x<sub>i</sub> : 0 or 1 (表示取或不取)
  * 共有 $2^n$ 可能的解
* 限制條件： $\sum_{i=1}^n$ $w_ix_i$ <= W
* 採用回溯法
* 限界條件：
  * 對於任何一個中間節點z，從root到z的分支所代表的狀態已經確定，從z到子孫的節點還未確定。如果z在第t層，說明第1種物品到t-1種物品（是否裝入背包）確定，t可以沿著分支擴展確認狀態，t+1到n不確定。
  * 目前裝入背包的物品總價用cp表示，因為還不確定t+1到n物品的狀態，先假設全部都放入背包，也就是剩餘的總價值，用rp表示。
  * cp + rp是所有從root出發經過中間節點z的可行解的價值上界。如果價值上界小於或等於目前的最優值，則說明節點z沒有繼續搜尋的必要
  * 即 cp + rp > bestp

## solution

假設有4個物品，每個物品w [2,5,4,2], 價值v [6,3,5,4], W = 10
1. 初始化：sumw , sumv 統計所有物品的總重和總價 -> sumw = 13, sumw = 18,目前放入背包的物品重量cw = 0, 總價cp = 0, 最優值 bestp = 0
2. 第一層：t = 1,
   * 判斷cw + w[1] = 2 < W （滿足限制條件）向左擴展分支，令x[1] = 1, cw = cw + w[1] = 2, cp = cp+v[1] = 6
   * <img src="pic1.png" width = "70%" />
