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


486. 预测赢家

给你一个整数数组 nums 。玩家 1 和玩家 2 基于这个数组设计了一个游戏。

玩家 1 和玩家 2 轮流进行自己的回合，玩家 1 先手。开始时，两个玩家的初始分值都是 0 。每一回合，玩家从数组的任意一端取一个数字（即，nums[0] 或 nums[nums.length - 1]），取到的数字将会从数组中移除（数组长度减 1 ）。玩家选中的数字将会加到他的得分上。当数组中没有剩余数字可取时，游戏结束。

如果玩家 1 能成为赢家，返回 true 。如果两个玩家得分相等，同样认为玩家 1 是游戏的赢家，也返回 true 。你可以假设每个玩家的玩法都会使他的分数最大化。

 

示例 1：

输入：nums = [1,5,2]
输出：false
解释：一开始，玩家 1 可以从 1 和 2 中进行选择。
如果他选择 2（或者 1 ），那么玩家 2 可以从 1（或者 2 ）和 5 中进行选择。如果玩家 2 选择了 5 ，那么玩家 1 则只剩下 1（或者 2 ）可选。 
所以，玩家 1 的最终分数为 1 + 2 = 3，而玩家 2 为 5 。
因此，玩家 1 永远不会成为赢家，返回 false 。

示例 2：

输入：nums = [1,5,233,7]
输出：true
解释：玩家 1 一开始选择 1 。然后玩家 2 必须从 5 和 7 中进行选择。无论玩家 2 选择了哪个，玩家 1 都可以选择 233 。
最终，玩家 1（234 分）比玩家 2（12 分）获得更多的分数，所以返回 true，表示玩家 1 可以成为赢家。

1. 解析  
这里和一般的dp不太一样，因为先手和后手都有相关联，并且在选择的时候并不是选择边界最大的一个，例如 70 100 1 1，先手选择并不会选择70，所以先手在选择时存在两种情况，后手在先手的情况下又存在四种情况。这里会定义两个dp方法。(一般只有一个)   
int first(int[] nums,int left,int right)  //先手选择   
int second(int[] nums,int left,int right)  //后手选择  
状态方程： 
玩家一：先手  
a.选择左边，等于左边的值加上后手选择的最大值  first(nums,left,right) = nums[left] + second(nums,left+1,right)  
b.选择右边，等于右边的值加上后手选择的最大值  first(nums,left,right) = nums[right] + second(nums,left,right-1);   
最后结果为 res = Math.max(左边，右边);
终止条件:left==right return nums[left]   
玩家二：后手   
a.先手选了左边 second(nums,left,right) = first(nums,left+1,right)    
b.先手选了右边 second(nums,left,right) = first(nums,left,right)   
最后结果 res = Math.min(左边，右边)  
终止条件：left==right return 0   
对后手min的解释:先手玩家一定会让我在后手拿到最小的数   
![486-1.jpg](.\image\486-1.jpg)
```java
    public boolean PredictTheWinner(int[] nums) {
        if(nums==null||nums.length==0){
            return false;
        }
        int sum = 0;
        for(int i=0;i<nums.length;i++){
            sum += nums[i];
        }
        int first = firstChoose(nums,0,nums.length-1);
        //不能用 sum / 2 去比较  会省略小数点
        return 2 * first - sum >= 0;
    }

    public int firstChoose(int[] nums,int left,int right){
        if(left==right){
            return nums[left];
        }
        return Math.max(nums[left] + secondChoose(nums,left+1,right),nums[right] + secondChoose(nums,left,right-1));
    }

    public int secondChoose(int[] nums,int left,int right){
        if(left==right){
            return 0;
        }
        return Math.min(firstChoose(nums,left+1,right),firstChoose(nums,left,right-1));
    }
```

