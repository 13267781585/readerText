### 动态规划   

122. 买卖股票的最佳时机 II

给定一个数组 prices ，其中 prices[i] 是一支给定股票第 i 天的价格。

设计一个算法来计算你所能获取的最大利润。你可以尽可能地完成更多的交易（多次买卖一支股票）。

注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

 

示例 1:

输入: prices = [7,1,5,3,6,4]
输出: 7
解释: 在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
     随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6-3 = 3 。

示例 2:

输入: prices = [1,2,3,4,5]
输出: 4
解释: 在第 1 天（股票价格 = 1）的时候买入，在第 5 天 （股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
     注意你不能在第 1 天和第 2 天接连购买股票，之后再将它们卖出。因为这样属于同时参与了多笔交易，你必须在再次购买前出售掉之前的股票。

示例 3:

输入: prices = [7,6,4,3,1]
输出: 0
解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。

* 解法   
1. 动态规划   


```java
 public int maxProfit(int[] prices) {
        if(prices==null||prices.length==0||prices.length==1){return 0;}
        int len = prices.length;
        //profit[i][0]表示i天没有持有股票获得最大利润
        //profit[i][1]持有股票最大利润
        int[][] profit = new int[len][2];
        profit[0][1] = -prices[0];
        for(int i=1;i<len;i++){
            profit[i][0] = Math.max(profit[i-1][0],profit[i-1][1]+prices[i]);
            profit[i][1] = Math.max(profit[i-1][1],profit[i-1][0]-prices[i]);
        }
        return profit[len-1][0];
    }
```

状态转移只与前一个有关，减少空间复杂度   

```java
    public int maxProfit(int[] prices) {
        if(prices==null||prices.length==0||prices.length==1){return 0;}
        int len = prices.length;
        int no = 0;
        int yes = -prices[0];
        for(int i=1;i<len;i++){
            int no1 = Math.max(no,yes+prices[i]);
            int yes1 = Math.max(yes,no-prices[i]);
            no = no1;
            yes = yes1;
        }
        return no;
    }
```

2. 贪心  
利润最大化:相隔两天只要利润为正，就可以交易，这样总的利润最大   
```java

    public int maxProfit(int[] prices) {
        if(prices==null||prices.length==0||prices.length==1){return 0;}
        int profit = 0;
        for(int i=1;i<prices.length;i++){
            profit += Math.max(0,prices[i]-prices[i-1]);
        }
        return profit;
    }
```

* 证明
![122-1.jpg](.\image\122-1.jpg)

作者：LeetCode-Solution
链接：https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii/solution/mai-mai-gu-piao-de-zui-jia-shi-ji-ii-by-leetcode-s/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。