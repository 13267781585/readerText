## 回溯  
https://leetcode-cn.com/problems/permutations/solution/hui-su-suan-fa-python-dai-ma-java-dai-ma-by-liweiw/

39. 组合总和

给定一个无重复元素的正整数数组 candidates 和一个正整数 target ，找出 candidates 中所有可以使数字和为目标数 target 的唯一组合。

candidates 中的数字可以无限制重复被选取。如果至少一个所选数字数量不同，则两种组合是唯一的。 

对于给定的输入，保证和为 target 的唯一组合数少于 150 个。

 

示例 1：

输入: candidates = [2,3,6,7], target = 7
输出: [[7],[2,2,3]]

示例 2：

输入: candidates = [2,3,5], target = 8
输出: [[2,2,2,2],[2,3,3],[3,5]]

示例 3：

输入: candidates = [2], target = 1
输出: []

示例 4：

输入: candidates = [1], target = 1
输出: [[1]]

示例 5：

输入: candidates = [1], target = 2
输出: [[1,1]]

方法一：dp   
```java
    //空间复杂度和时间复杂度很高
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        Map<Integer,List<List<Integer>>> map = new HashMap<>();
        List<List<Integer>> empty = new ArrayList<>();
        empty.add(new ArrayList<>());
        map.put(0,empty);
        for(int n:candidates){
            for(int i=1;i<=target;i++){
                if(i>=n){
                    List<List<Integer>> list = map.get(i-n);
                    List<List<Integer>> cur = map.get(i);
                    if(cur==null){
                        cur = new ArrayList<>();
                        map.put(i,cur);
                    }
                    if(list!=null){
                        for(List<Integer> l:list){
                            List<Integer> temp = new ArrayList<Integer>();
                            temp.addAll(l);
                            temp.add(n);
                            cur.add(temp);
                        }
                    }
                }
            }
        }
        return map.get(target) == null ? new ArrayList<>() : map.get(target);
    }

```

方法二：回溯  
```java
   public List<List<Integer>> combinationSum(int[] candidates, int target) {
        List<List<Integer>> res = new ArrayList<>();
        List<Integer> count = new ArrayList<>();
        cal(res,candidates,target,count,0);
        return res;
    }

    //重点：利用begin索引记录遍历的num，防止出现重复  
    //事件复杂度和空间复杂度几乎 100
    public void cal(List<List<Integer>> res,int[] nums,int target,List<Integer> count,int begin){
        if(target==0){
            List<Integer> r = new ArrayList<>();
            r.addAll(count);
            res.add(r);
        }else{
            for(int i=begin;i<nums.length;i++){
                int n = nums[i];
                if(target>=n){
                    count.add(n);
                    cal(res,nums,target-n,count,i);
                    count.remove(count.size()-1);
                }
            }
        }
    }
```

参考：https://leetcode-cn.com/problems/combination-sum/solution/hui-su-suan-fa-jian-zhi-python-dai-ma-java-dai-m-2/


51. N 皇后
n 皇后问题 研究的是如何将 n 个皇后放置在 n×n 的棋盘上，皇后的行列斜线上不能有其他的皇后，给你一个整数 n ，返回所有不同的 n 皇后问题 的解决方案。

每一种解法包含一个不同的 n 皇后问题 的棋子放置方案，该方案中 'Q' 和 '.' 分别代表了皇后和空位。

输入：n = 4
输出：[[".Q..","...Q","Q...","..Q."],["..Q.","Q...","...Q",".Q.."]]
解释：如上图所示，4 皇后问题存在两个不同的解法。

示例 2：

输入：n = 1
输出：[["Q"]]


方法一：回溯   
用int[] record一维数组表示皇后放置的位置 表示为 x(i,reocrd[i]) i为行号  
```java
    public List<List<String>> solveNQueens(int n) {
        List<List<String>> res = new ArrayList<>();
        int[] record = new int[n];
        process(n,0,record,res);
        return res;
    }

    //皇后的坐标 x(a,record[a])
    public void process(int n,int row,int[] record,List<List<String>> res){
        if(row==n){
            //处理将结果转化为List<String>
            List<String> r = new ArrayList<>();
            for(int i=0;i<n;i++){
                int value = record[i];
                String str = "";
                for(int j=0;j<n;j++){
                    if(j!=value){
                        str += ".";
                    }else{
                        str += "Q";
                    }
                }
                r.add(str);
            }
            res.add(r);
        }
        for(int i=0;i<n;i++){
            if(!isValid(row,i,record)){
                record[row] = i;
                process(n,row+1,record,res);
            }
        }
    }

    public boolean isValid(int row,int index,int[] record){
        for(int i=0;i<row;i++){
            if(record[i]==index||Math.abs(i-row)==Math.abs(record[i]-index)){
                return true;
            }
        }
        return false;
    }
```

方法二：位运算  
