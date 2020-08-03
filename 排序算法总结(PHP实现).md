---
title: "排序算法总结(PHP实现)"
date: "2017-09-26"
tags: ["排序","PHP"]
toc: "true"
categories: ["学习笔记", "算法"]
---
排序算法看了忘忘了看，干脆写一遍。


- 时间复杂度：希尔、快排、归并、堆 nlogn  ，快排可退化为n^2
- 空间复杂度：快排nlogn，归并n
- 稳定性：冒泡、插入、归并稳定

# 冒泡排序
> n-1次冒泡，每次一轮循环交换出一个最大/最小的

``` c++
//C++
void maopao(int r[],int n)
{
    for(int i=0;i<n-1;i++)
        for(int j=0;j<n-i-1;j++)
        {
            if(r[j]>r[j+1])
            {
                int temp=r[j];
                r[j]=r[j+1];
                r[j+1]=temp;
            }
        }
}
```

``` php
//php 含有标志优化
function BubbleSort($arr) {
    $len = count($arr);
    //设置一个空数组 用来接收冒出来的泡
    //该层循环控制 需要冒泡的轮数
    for ($i = 1; $i < $len; $i++) {
        $flag = false;    //本趟排序开始前，交换标志应为假
        //该层循环用来控制每轮 冒出一个数 需要比较的次数
        for ($k = 0; $k < $len - $i; $k++) {
            //从小到大排序
            if ($arr[$k] > $arr[$k + 1]) {
                $tmp = $arr[$k + 1];
                $arr[$k + 1] = $arr[$k];
                $arr[$k] = $tmp;
                $flag = true;
            }
        }
        if(!$flag) return $arr;
    }
}
```

# 选择排序
>n-1次选择，每次从无序中选一个最大/最小的放到第一位

``` c++
void select_sort(int * a,int n)
{
    register int i,j,min,t;
    for(i=0;i<n-1;i++)
    {
        min=i;//查找最小值
        for(j=i+1;j<n;j++)
            if(a[min]>a[j])
                min=j;//交换
        if(min!=i)
        {
            t=a[min];
            a[min]=a[i];
            a[i]=t;
        }
    }
}
```
``` php
function selectSort($array){
    $temp = 0;
    for($i = 0;$i < count($array) - 1;$i++){
        $min = $i;
        for($j = $i+1;$j < count($array);$j++){
            if($array[$min] > $array[$j]){ //从小到大排列
                $min = $j;
            }
        }
        $temp = $array[$i];
        $array[$i] = $array[$min];
        $array[$min] = $temp;
    }
}

```

# 插入排序
>n-1次插入，默认0有序，每次将当前的元素插入到有序数组中

``` c++
void insert_sort(int array[],int n)
{
    int i,j;
    int temp;
    for(i=1;i<n;i++)
    {
        temp=array[i];
        j=i-1;
        //与已排序的数逐一比较，大于temp时，该数移后
        while((j>=0)&&(array[j]>temp))
        {
            array[j+1]=array[j];
            j--;
        }
        //存在大于temp的数
        if(j!=i-1) {array[j+1]=temp;}
    }

}
```

``` php
function insertSort($array){ //从小到大排列
//先默认$array[0]，已经有序，是有序表
    for($i = 1;$i < count($array);$i++){
        $insertVal = $array[$i]; //$insertVal是准备插入的数
        $insertIndex = $i - 1; //有序表中准备比较的数的下标
        while($insertIndex >= 0 && $insertVal < $array[$insertIndex]){
            $array[$insertIndex + 1] = $array[$insertIndex]; //将数组往后挪
            $insertIndex--; //将下标往前挪，准备与前一个进行比较
        }
        if($insertIndex + 1 !== $i){
            $array[$insertIndex + 1] = $insertVal;
        }
    }
}
```

# 希尔排序
>插入排序的优化，设置增量，按照增量间隔插入，期间不断缩小增量

``` c++
void shellSort(int a[], int n)  
{  
    int i, j, gap;  
  
    for (gap = n / 2; gap > 0; gap /= 2)  
        for (i = gap; i < n; i++)  
            for (j = i - gap; j >= 0 && a[j] > a[j + gap]; j -= gap)  
                Swap(a[j], a[j + gap]);  
}  
```

``` php
function shellSort($array){
    $count = count($array);
    $gap = floor($count/2);
    while($gap>0){
        for($i=$gap;$i<$count;$i++)
            for($j=$i-$gap;$j>=0&&$array[$j]>$array[$j+$gap];$j-=$gap)
            {
                $temp = $array[$j];
                $array[$j]=$array[$j+$gap];
                $array[$j+$gap]=$temp;
            }
            $gap=floor($gap/2);
    }
    return $array;
}
```

# 快速排序
>划分，不多说

``` c++
void qsort(int a[],int l,int r)
{
    if(l>=r) return;


    int first=l;
    int last=r;
    int key=a[first];

    while(first<last)
    {
        while(first<last&&a[last]>=key) last--;
        a[first]=a[last];
        while(first<last&&a[first]<=key) first++;
        a[last]=a[first];


    }
    a[first]=key;
    qsort(a,l,first-1);
    qsort(a,first+1,r);
}
```

``` php
function quickSort(&$array,$l,$r){
    if($l>=$r) return;
    $key = $array[$l];
    $left = $l;
    $right = $r;
    while($left<$right)
    {
        while($left<$right&&$array[$right]>=$key) $right--;
        $array[$left]=$array[$right];
        while($left<$right&&$array[$left]<=$key) $left++;
        $array[$right]=$array[$left];


    }
    $array[$left]=$key;
    quickSort($array,$l,$right-1);
    quickSort($array,$left+1,$r);
}
```

# 堆排序
>先建堆：从n/2个节点往前依次维护堆。维护：调整该节点与左右子节点关系，若有变动则维护变动子节点堆。做n次取节点，每次交换后维护剩下堆。


``` c++
void headadjust(int a[],int i,int n)
{
    int lc=i*2;
    int rc=i*2+1;
    int max=i;

    if(i<=n/2){
        if(lc<=n&&a[max]<a[lc]) max=lc;
        if(rc<=n&&a[max]<a[rc]) max=rc;
        if(max!=i)
        {
            swap(a[i],a[max]);
            hheadadjust(a,max,n);
        }
    }
}
void heapbuild(int a[],int n)
{
    for(int i=n/2;i>=1;i--)
        headadjust(a,i,n);

}
void heapsort(int a[],int n)
{
    heapbuild(a,n);
    for(int i=n;i>=1;i--) {
        swap(a[1], a[i]);
        headadjust(a,1,i-1);
    }
}
```

# 归并排序
>递归二分数组，然后将数组交叉合并

``` c++  
void Merge(int sourceArr[],int tempArr[], int startIndex, int midIndex, int endIndex)
{
    int i = startIndex, j=midIndex+1, k = startIndex;
    while(i!=midIndex+1 && j!=endIndex+1)
    {
        if(sourceArr[i] > sourceArr[j])
            tempArr[k++] = sourceArr[j++];
        else
            tempArr[k++] = sourceArr[i++];
    }
    while(i != midIndex+1)
        tempArr[k++] = sourceArr[i++];
    while(j != endIndex+1)
        tempArr[k++] = sourceArr[j++];
    for(i=startIndex; i<=endIndex; i++)
        sourceArr[i] = tempArr[i];
}
 
//内部使用递归
void MergeSort(int sourceArr[], int tempArr[], int startIndex, int endIndex)
{
    int midIndex;
    if(startIndex < endIndex)
    {
        midIndex = (startIndex + endIndex) / 2;
        MergeSort(sourceArr, tempArr, startIndex, midIndex);
        MergeSort(sourceArr, tempArr, midIndex+1, endIndex);
        Merge(sourceArr, tempArr, startIndex, midIndex, endIndex);
    }
}
```

``` php
function mergeSortRecursion(&$arr, $begin, $end) {
    if ($begin < $end) {
        $mid = floor(($begin + $end) / 2);
        // 把数组不断拆分，只到只剩下一个元素，对于一个元素的数组，一定是排序好的
        mergeSortRecursion($arr, $begin, $mid);
        mergeSortRecursion($arr, $mid + 1, $end);
        $i = $begin;
        $j = $mid + 1;
        $k = 0;
        $temp = array();
        // 开始执行归并，遍历两个数组，从它们里面取较小的元素插入到新数组中
        while ($i <= $mid && $j <= $end) {
            if ($arr[$i] < $arr[$j]) {
                $temp[$k++] = $arr[$i++];
            } else {
                $temp[$k++] = $arr[$j++];
            }
        }
        while ($i <= $mid) {
            $temp[$k++] = $arr[$i++];
        }
        while ($j <= $end) {
            $temp[$k++] = $arr[$j++];
        }
        for ($i = 0; $i < $k; $i++) {
            $arr[$i + $begin] = $temp[$i];
        }
    }
   
}
```
