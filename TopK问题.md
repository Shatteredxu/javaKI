

##### [如何从大量的 URL 中找出相同的 URL？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-common-urls.md)

>   **使用分治策略：**将URL分为多个小文件，每一个不超过4G,将每个小文件加载进来，遍历其中URL, 对遍历到的 URL 求 `hash(URL) % 1000` ，根据计算结果把遍历到的 URL 存储到 a0, a1, a2, ..., a999，这样每个大小约为 300MB。使用同样的方法遍历文件 b，把文件 b 中的 URL 分别存储到文件 b0, b1, b2, ..., b999 中。这样处理过后，所有可能相同的 URL 都在对应的小文件中，即 a0 对应 b0, ..., a999 对应 b999，不对应的小文件不可能有相同的 URL。那么接下来，我们只需要求出这 1000 对小文件中相同的 URL 就好了。
>
>   接着就找有相同hash的URL查看是否相同

>   **使用前缀树：**URL 的长度差距不会不大，而且前面几个字符，绝大部分相同。这种情况下，非常适合使用**字典树**（trie tree）

##### [如何从大量数据中找出高频词？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-top-100-words.md)

>   分成多个小文件，然后对每个文件中的词进行hash，得到的结果存入文件中，然后使用hashmap记录每个词出现的次数，再采用大顶堆得到top100

##### [如何找出某一天访问百度网站最多的 IP？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-top-1-ip.md)

>   大致就是先对 IP 进行哈希映射，接着使用 HashMap 统计重复 IP 的次数，最后计算出重复次数最多的 IP。用一个max去遍历hashmap

##### [如何在大量的数据中找出不重复的整数？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-no-repeat-number.md)

>   1.   hash+hashmap
>   2.   位图
>   3.   

##### [如何在大量的数据中判断一个数是否存在？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-a-number-if-exists.md)

>   1.   分治，就是计算hash
>
>   2.   **位图：**用 `1<<32=4,294,967,296` 个 bit 来表示每个数字。初始位均为 0，那么总共需要内存：4,294,967,296b≈512M。
>
>        我们读取这 40 亿个整数，将对应的 bit 设置为 1。接着读取要查询的数，查看相应位是否为 1，如果为 1 表示存在，如果为 0 表示不存在。

##### [如何查询最热门的查询串？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-hotest-query-string.md)

>   1.   分治：划分为多个小文件，加载到内存求出出现次数最多的字符串，对于每个小文件都是，然后计算每个小文件出现次数最多的串
>   2.   hashMap法：直接使用hashmap存储出现次数，再利用大顶堆找到出现次数最多的
>   3.   前缀树：在叶子节点存储出现的次数，在用小顶堆统计(前缀树经常被用来统计字符串的出现次数。它的另外一个大的用途是字符串查找，判断是否有重复的字符串等。)

##### [如何统计不同电话号码的个数？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/count-different-phone-numbers.md)

>   1.   前缀树
>
>   2.   hashmap
>   3.   位图法(数据重复问题，首选**位图法**)



##### [如何从 5 亿个数中找出中位数？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-mid-value-in-500-millions.md)

>   1.   双堆法：维护两个堆，一个大顶堆，一个小顶堆。大顶堆中最大的数**小于等于**小顶堆中最小的数；保证这两个堆中的元素个数的差不超过 1。
>
>        若数据总数为**偶数**，当这两个堆建好之后，**中位数就是这两个堆顶元素的平均值**。当数据总数为**奇数**时，根据两个堆的大小，**中位数一定在数据多的堆的堆顶**。
>
>   2.   分治法，也就是topk问题，将五亿个数按二进制最高位为0,1分为两个文件，然后文件多的中位数一定在这里面，然后再进行划分
>
>   注意：当数据总数为偶数，如果划分后两个文件中的数据有相同个数，那么**中位数就是数据较小的文件中的最大值与数据较大的文件中的最小值的平均值。**

##### [如何按照 query 的频度排序？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/sort-the-query-strings-by-counts.md)

>   1.   hashmap法
>
>   2.   分治法：划分小文件，分别统计次数，放到hashmap中，写到单独文件中，可以采用外排序归并法进行排序

##### [如何找出排名前 500 的数？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-rank-top-500-numbers.md)

>   也就是topk问题，可以考虑计数，快排，归并排序。

##### [讲讲大数据中 TopK 问题的常用套路？](https://github.com/doocs/advanced-java/blob/main/docs/big-data/topk-problems-and-solutions.md)

>   1.   堆排序时间复杂度为O(nlogn),优势是可以维持一个k大小的小顶堆，每次不断和堆顶元素比较，并不断淘汰最小元素。
>   2.   快排法：
>
>   ```java
>   class Solution {
>       public int findKthLargest(int[] nums, int k) {
>           int n = nums.length;
>           return quickSort(nums,0,n-1,n-k);//寻找第n-k
>       }
>       int quickSort(int[] arr,int low,int high,int k){
>           int i,j,temp,t;
>         // 1 随机确定一个基准值
>           int p = (int)(low + Math.random() * (high - low + 1));
>           temp = arr[p];
>           arr[p] =arr[low];
>           arr[low] = temp;
>           if(low>high){return 0;}
>           i = low;
>           j = high;
>         // 2 小数放左 大数放右
>           while(i<j){
>               while(temp<=arr[j]&&i<j){
>                   j--;
>               }
>               if(i<j){
>                   arr[i]=arr[j];
>                   ++i;
>               }
>               while(temp>=arr[i]&&i<j){
>                   i++;
>               }
>              if(i<j){
>                   arr[j]=arr[i];
>                   --j;
>               }
>           }
>           arr[j] = temp;
>         // 3 检查当前值的位置
>         // =k-1 当前数就是第K大
>           // <k-1 target在右区间
>           // >k-1 target在左区间
>           if(k==j){return arr[k];}
>          // 在目标区间继续寻找
>            return j < k ? quickSort(arr,j+1,high,k): quickSort(arr,low,j-1,k);
>       }
>   }
>   ```
>
>   3.   使用bitmap，bitmap每个字节位置代表数字，而0,1值代表数字有没有出现
>   4.   使用hash:遇到String类型找最大最小, 可以考虑将大文件分为小文件，然后对每个小文件中的字符串进行长度hash，找到最长的字符，然后再对每个最长的字符高位进行查询，依次进行找到最长的10个。（这种方式比较适合网址或者电话号码的查询）
>   5.   字典树
>   6.   混合查询(比如先做快排(得到无序的k个数)，然后再将每个机器上的k个数做快排得到最大的k个数，在进行快排)(或者用堆排序找到每个机器最大的kk个数，这些数字已经是有序的了，然后依次取出 10 台机器中，耗时最高的 5 条放入小顶堆中。最后，遍历 10 台机器上的数据，每台机器从第 6 个数据开始往下循环，如果这个值比堆顶的数据大，则抛掉堆顶数据并且把它加入，继续用下一个值进行同样比较。如果这个值比堆顶的值小，则结束当前循环，并且在下一台机器上做同样操作。

##### 给你10亿个数，选择10个最小的

>   分治法，分成小文件，每个小文件加载到内存找到最小的10个，对于每个文件都是如此，最后在进行堆排序，找到10个最最小的

##### 10亿数据如何快速找到某个数

>   1.   bitMap
>   2.   压缩， 