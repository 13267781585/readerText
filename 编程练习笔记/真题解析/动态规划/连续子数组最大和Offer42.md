输入一个整型数组，数组中的一个或连续多个整数组成一个子数组。求所有子数组的和的最大值。

要求时间复杂度为O(n)。

 

示例1:

输入: nums = [-2,1,-3,4,-1,2,1,-5,4]
输出: 6
解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。

 

提示：

    1 <= arr.length <= 10^5
    -100 <= arr[i] <= 100

* 解法一  
```java
//空间O(1) 时间O(n)
//本质上也是动态规划，只是优化了空间复杂度
public int maxSubArray(int[] nums) {
        int max = Integer.MIN_VALUE;
        int sum = 0;
        for(int i=0;i<nums.length;i++){
            sum+=nums[i];
            if(sum>max){
                max=sum;
            }
            if(sum<0){
                sum=0;
            }
        }
        return max;
    }
```
![Offer42-1](../image/Offer42-1.jpg)

* 解法二:动态规划  
迭代
```java
     public int maxSubArray(int[] nums) {
        int[] record = new int[nums.length];
        record[0] = nums[0];
        int max=nums[0];
        for(int i=1;i<nums.length;i++){
            int value = Math.max(nums[i],nums[i]+record[i-1]);
            if(value>max)
                max = value;
            record[i]=value;
        }
        return max;
    }
```
![Offer42-2](../image/Offer42-2.jpg)

递归  
```java
 public int maxSubArray(int[] nums) {
        int[] record = new int[nums.length];
        int max = Integer.MIN_VALUE;
        for(int i=0;i<nums.length;i++){
            max = Math.max(max,maxValue(record,nums,i));
        }
        return max;
    }

    public int maxValue(int[] record,int[] nums,int i){
        if(i == nums.length -1) return nums[i];
        if(record[i] != 0) return record[i];
        int value = maxValue(record,nums,i+1)+nums[i];
        record[i] = Math.max(nums[i],value);
        return record[i];

    }
```
![Offer42-3](../image/Offer42-3.jpg)
来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/lian-xu-zi-shu-zu-de-zui-da-he-lcof
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。