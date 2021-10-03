### [3. 无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

```go
func lengthOfLongestSubstring(s string) int {
    if len(s)==0{return 0}
    var fre[256]int
    var res,left,right=0,0,-1;
    for left<len(s){
        if right+1<len(s)&&fre[s[right+1]-'a']==0 {
            fre[s[right+1]-'a']++;
            right++;
        }else{
            fre[s[left]-'a']--;
            left++;
        }
        if res<right-left+1{
            res = right-left+1
        }
    }
    return res;
}
```

### [5. 最长回文子串](https://leetcode-cn.com/problems/longest-palindromic-substring/)

##### 中⼼扩散法

```go
func longestPalindrome(s string) string {
    if len(s)==0||len(s)==1 {
        return s;
    }
    start,end:=0,0
    for i:=0;i<len(s);i++{
        left1,right1 :=expandAroundCenter(s ,i,i );
        left2,right2 :=expandAroundCenter(s ,i,i+1);
        if(right1-left1>end-start){
            start,end = left1,right1;
        }
        if(right2-left2>end-start){
            start,end = left2,right2;
        }
    }
    return s[start:end+1];//前闭后开
}
func expandAroundCenter(s string,left,right int) (int,int){
    for ;left>=0&&right<len(s)&&s[left] == s[right];left,right= left-1,right+1 {
			//前面初始化的分号需要由，后面left和right在一起赋值，left可以为0
    }
    return left+1,right-1;
}
```

##### 动态规划

### [6. Z 字形变换](https://leetcode-cn.com/problems/zigzag-conversion/)

```java
func convert(s string, numRows int) string {
    if numRows==1 {return s;}
    matrix,down := make([][]byte,numRows,numRows),0;
    direct:=true;
    for i:=0;i<len(s);i++{
        matrix[down] = append(matrix[down],byte(s[i]))
        if down==numRows-1 {
            direct=false;
        }else if down==0{
            direct=true;
        }
        if direct {
            down++;
        }else{
            down--;
        }
    }
    res:=make([]byte,0);
    for _,row:=range matrix{
        for _,col:=range row{
            res = append(res,col);
        }
    }
    return string(res);
}
```

### [7. 整数反转](https://leetcode-cn.com/problems/reverse-integer/)

```go
//负数取余还是负数
func reverse(x int) (res int) {//先生命res作为返回值
   for x!=0 {
     	if res < math.MinInt32/10 || res > math.MaxInt32/10 {//判断是否溢出
            return 0
    	}
       k:=x%10;
       res=res*10+k;
       x/=10;
   }
   return ;//不需要考虑正负值
}
```



### [17. 电话号码的字母组合](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/)

```go
func letterCombinations(digits string) []string {
    if len(digits)==0{return []string{}}
    dmap:=[10]string{
        "","", "abc", // 2
        "def", // 3
        "ghi", // 4
        "jkl", // 5
        "mno", // 6
        "pqrs", // 7
        "tuv", // 8
        "wxyz", // 9
    }
    res := make([]string,0);//res使用make定义，需要传指针进去
    backTrack(digits,"",0,dmap,&res);
    return res;
}
func backTrack(digits,temp string,pos int,dmap [10]string,res *[]string){
    if pos== len(digits){
        *res = append(*res,temp);//指针变量需要加*改变
        return;
    }
    k := digits[pos]-'0';
    letter := dmap[k];
    for i:=0;i<len(letter);i++ {
        backTrack(digits,temp+string(letter[i]),pos+1,dmap,res)
    }
}
```

```go

var res []string//全局数组，可以直接append
func letterCombinations(digits string) []string {
    if len(digits)==0{return []string{}}
    dmap:=[10]string{
        "","", "abc", // 2
        "def", // 3
        "ghi", // 4
        "jkl", // 5
        "mno", // 6
        "pqrs", // 7
        "tuv", // 8
        "wxyz", // 9
    }
    res = []string{};
    backTrack(digits,"",0,dmap,);
    return res;
}
func backTrack(digits,temp string,pos int,dmap [10]string){
    if pos== len(digits){
        res = append(res,temp);
        return;
    }
    k := digits[pos]-'0';
    letter := dmap[k];
    for i:=0;i<len(letter);i++ {
        backTrack(digits,temp+string(letter[i]),pos+1,dmap)
    }
}
```

### [39. 组合总和](https://leetcode-cn.com/problems/combination-sum/)

```java
var res [][]int
func combinationSum(candidates []int, target int) [][]int {
    res = [][]int{};
    backTrack(candidates,[]int{},target,0,0)
    return res;
}
func backTrack(candidates,list []int,target,sum ,pos int){
    if sum>target {return;}
    if sum==target{
        res = append(res,append([]int{},list...))
        return
    }
    for i:=pos;i<len(candidates);i++ {
        sum+=candidates[i];
        list = append(list,candidates[i])
        backTrack(candidates,list,target,sum,i)
        sum-=candidates[i];
        list = list[:len(list)-1]
    }
}
```

### [40. 组合总和 II](https://leetcode-cn.com/problems/combination-sum-ii/)

```go
var res [][]int
func combinationSum2(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    res = [][]int{};
    backTrack(candidates,[]int{},target,0,0)
    return res;
}
func backTrack(candidates,list []int,target,sum ,pos int){
    if sum>target {return;}
    if sum==target{
        res = append(res,append([]int{},list...))
        return
    }
    for i:=pos;i<len(candidates);i++ {
        if i>pos&&candidates[i]==candidates[i-1]{
            continue;//避免同一个递归深度的数字被加到同一个数组，保证本层不重复 但不同层可以重复
        }
        sum+=candidates[i];
        list = append(list,candidates[i])
        backTrack(candidates,list,target,sum,i+1)
        sum-=candidates[i];
        list = list[:len(list)-1]
    }
}
```

### [77. 组合](https://leetcode-cn.com/problems/combinations/)

##### 简单写法

```go
func combine(n int, k int) [][]int {
	var res [][]int
	var backtrace func(num int)
	var arr = make([]int, 0)//创建长度为0的数组
	backtrace = func(num int) {
		if len(arr) == k {
			res = append(res, append([]int{}, arr...))//res里面追加一个或多个元素，然后返回一个和res一样类型
		} else {
			for i := num; i <= n-(k-len(arr))+1; i++ {
				arr = append(arr, i)
				backtrace(i + 1)
				arr = arr[:len(arr)-1]
			}
		}
	}
	backtrace(1)
	return res
}
```

##### 常规写法

```go
var res [][]int 
func combine(n int, k int) [][]int {
    res = [][]int{};
    backtrack(n,k,1,[]int{});
    return res
}
func backtrack(n,k,start int,list []int){
    if len(list)==k{
        temp:= make([]int,k)
        copy(temp,list)
        res = append(res,temp)
        return
    }
    for i:=start;i<=n-(k-len(list))+1;i++ {
        list = append(list,i)
        backtrack(n,k,i+1,list);
        list = list[:len(list)-1]
    }
}
```

### [216. 组合总和 III](https://leetcode-cn.com/problems/combination-sum-iii/)

```go
var res [][]int
func combinationSum3(k int, n int) [][]int {
    res = [][]int{}//用之前需要初始化
    backTrack(n,k,0,1,[]int{})
    return res;
}
func backTrack(n,k,sum,start int,list []int){
    if sum>n {return}
    if sum==n&&len(list)==k{
        res = append(res,append([]int{},list...))//创建新的数组append
        return
    }
  for i:=start;i<=9-(k-len(list))+1;i++ {
        list = append(list,i)
        backTrack(n,k,sum+i,i+1,list)
        list = list[:len(list)-1]
    }
}
```

