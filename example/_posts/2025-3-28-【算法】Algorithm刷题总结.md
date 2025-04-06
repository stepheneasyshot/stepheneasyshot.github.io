---
layout: post
description: > 
  本文介绍了近期在leetcode的一些刷题过程的记录与总结，主要为C++语音来实现。
image: 
  path: /assets/img/blog/blogs_cpp.png
  srcset: 
    1920w: /assets/img/blog/blogs_cpp.png
    960w:  /assets/img/blog/blogs_cpp.png
    480w:  /assets/img/blog/blogs_cpp.png
accent_image: /assets/img/blog/blogs_cpp.png
excerpt_separator: <!--more-->
sitemap: false
---
# Algorithm刷题总结
## 背景
近期决定补齐一下算法方面的知识，因为是非计算机科班出身，没有系统学过 `C++数据结构和算法` 这门课，在算法方面的知识很薄弱，想要从应用层往下深入就很困难。

Android 应用开发做需求时完全用不到算法，性能优化时是规避JVM内存泄漏，按照Android系统的规则来处理，相当于是在别人预设好的算法环境上来适配自己的代码。所以打算借着刷算法题，把一些算法的实现记录下来，方便以后回顾。然后对于重难点进行系统的学习。

## 环境搭建
为了省去前期环境配置和开发工具选取的耗时，我直接在此前做JNI的项目中，分出了一块来做算法的测试和学习，JNI环境中大部分的C++库都是可以支持的，还可以同时练习C++，Kotlin和java三种。

![](/assets/img/blog/blogs_algorithm_project_env.png)

## 正文记录
[经典150题地址](https://leetcode.cn/studyplan/top-interview-150/)

### 数组和字符串常见处理
### 排序
#### 冒泡排序

```cpp
/**
 * 冒泡排序
 * 从前往后，每个元素都和后面的所有元素比较一次，将最小的元素移到前面来
 * @param arr
 * @param n
 */
void bubbleSort(std::vector<int> &arr) {
    for (int i = 0; i < arr.size(); i++) {
        for (int j = i; j < arr.size(); j++) {
            // 如果当前元素大于后面的元素，交换两个元素，将最小的元素移到前面来
            if (arr[i] > arr[j]) {
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
    }
}
```

#### 快速排序
```cpp
// 划分函数，选取一个基准元素，将数组分为两部分，走完之后，基准元素插入到其该在的位置，其右侧所有元素都比基准元素大，左侧所有元素都比基准元素小
int partition(std::vector<int>& arr, int low, int high){
    int base = arr[high];
    int baseIndex = low;
    // 遍历数组，将所有小于基准元素的元素都移动到基准元素的左侧
    for(int i=low;i<=high;i++){
        if(arr[i]<base){
            std::swap(arr[i], arr[baseIndex]);
            baseIndex++;
        }
    }
    // 将基准元素插入到其该在的位置
    std::swap(arr[high], arr[baseIndex]);
    return baseIndex;
}

// 递归快速排序，只要目标区域包含两个及以上的元素，就继续排序
void quickSort(std::vector<int>& arr, int low, int high){
    if(low<high){
        int baseIndex = partition(arr, low, high);
        quickSort(arr, low, baseIndex-1);
        quickSort(arr, baseIndex+1, high);
    }
}
```
在解题中，如果排序只是一个很小的部分，还可以直接使用库里的api：
```cpp
sort(nums1.begin(), nums1.end());
```

### 移除元素类
#### 经典双指针
两个下标变量，满足某个条件时，将slow慢指针移到快指针的位置，而快指针一般用来便历，每次都移动。

```cpp
int removeElement(int* nums, int numsSize, int val) {
    int slow = 0, fast = 0; //一对夫妇，原本都是零起点
    while (fast < numsSize) {   //但是有一个跑得快，一个跑得慢
        if (nums[fast] != val) {    //于是跑得快的那个先去寻找共同目标
            nums[slow] = nums[fast];    //如果找到了，就送给跑得慢的那个
            slow++;     //然后跑得慢的那个也就离目标近一点
        }
        fast++; //但是不管是否找得到，跑得快的那方都一直奔跑到生命的尽头
    }
    return slow;    //最终留下跑得慢的一方
}
```

#### vector的erase函数
‌vector的erase函数用于删除容器中的元素或元素范围。‌

##### 使用方法：
* ‌删除单个元素‌：使用erase(iterator pos)，其中pos是指向要删除元素的迭代器。删除后，迭代器指向的位置会向前移动，覆盖被删除的元素‌
* 删除一段元素‌：使用erase(iterator first, iterator last)，其中first是起始迭代器，last是结束迭代器的下一个位置。删除后，起始位置之前的所有元素都会向前移动‌

##### 返回值
erase函数返回一个迭代器，指向被删除元素范围之后的第一个元素。如果删除单个元素，返回的是指向下一个元素的迭代器；如果删除一段元素，返回的是指向最后一个被删除元素之后的位置的迭代器‌。

##### 示例
```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    // 删除单个元素
    vec.erase(vec.begin() + 2); // 删除元素3
    // 删除一段元素
    vec.erase(vec.begin(), vec.begin() + 3); // 删除前三个元素
    // 输出结果
    for (int num : vec) {
        std::cout << num << " ";
    }
    return 0;
}
```

### 翻转数组
```
给定一个整数数组 nums，将数组中的元素向右轮转 k 个位置，其中 k 是非负数。

示例 1:

输入: nums = [1,2,3,4,5,6,7], k = 3
输出: [5,6,7,1,2,3,4]
解释:
向右轮转 1 步: [7,1,2,3,4,5,6]
向右轮转 2 步: [6,7,1,2,3,4,5]
向右轮转 3 步: [5,6,7,1,2,3,4]
示例 2:

输入：nums = [-1,-100,3,99], k = 2
输出：[3,99,-1,-100]
解释: 
向右轮转 1 步: [99,-1,-100,3]
向右轮转 2 步: [3,99,-1,-100]使用三次遍历和交换，实现需求：
nums = "----->-->"; k =3
result = "-->----->";

reverse "----->-->" we can get "<--<-----"
reverse "<--" we can get "--><-----"
reverse "<-----" we can get "-->----->"
this visualization help me figure it out :)
```

解决方案，直接对数组遍历翻转三次

```cpp
class Solution {
public:
    void reverse(vector<int>& nums, int start, int end) {
        while (start < end) {
            swap(nums[start], nums[end]);
            start += 1;
            end -= 1;
        }
    }

    void rotate(vector<int>& nums, int k) {
        k %= nums.size();
        reverse(nums, 0, nums.size() - 1);
        reverse(nums, 0, k - 1);
        reverse(nums, k, nums.size() - 1);
    }
};
```

### 计数类型
#### 数青蛙
```
给你一个字符串 croakOfFrogs，它表示不同青蛙发出的蛙鸣声（字符串 "croak" ）的组合。由于同一时间可以有多只青蛙呱呱作响，所以 croakOfFrogs 中会混合多个 “croak” 。

请你返回模拟字符串中所有蛙鸣所需不同青蛙的最少数目。

要想发出蛙鸣 "croak"，青蛙必须 依序 输出 ‘c’, ’r’, ’o’, ’a’, ’k’ 这 5 个字母。如果没有输出全部五个字母，那么它就不会发出声音。如果字符串 croakOfFrogs 不是由若干有效的 "croak" 字符混合而成，请返回 -1 。

 

示例 1：

输入：croakOfFrogs = "croakcroak"
输出：1 
解释：一只青蛙 “呱呱” 两次
示例 2：

输入：croakOfFrogs = "crcoakroak"
输出：2 
解释：最少需要两只青蛙，“呱呱” 声用黑体标注
第一只青蛙 "crcoakroak"
第二只青蛙 "crcoakroak"
示例 3：

输入：croakOfFrogs = "croakcrook"
输出：-1
解释：给出的字符串不是 "croak" 的有效组合。
 

提示：

1 <= croakOfFrogs.length <= 105
字符串中的字符只有 'c', 'r', 'o', 'a' 或者 'k'
```

我的题解，是准备了一个二维数组，来当作青蛙的容器。然后对入参的字符串遍历一遍，再根据不同的字母的情况来遍历一遍青蛙容器，判断该怎么处理这个字母，是需要分配新的青蛙，还是有旧青蛙可以复用。逻辑上应该没问题，通过了80的测试用例。

但是这样的时间复杂度是O(n^2)，空间复杂度是O(n)，最后判题超时了。

```cpp
int myCountCroakOfFrogs(std::string croakOfFrogs) {
    int arrayLenth = croakOfFrogs.length() / 5 + 1;
    int frogsNumber = 0;
    // 检查字符串中五个字母数量是否相等
    int cCount = 0;
    int rCount = 0;
    int oCount = 0;
    int aCount = 0;
    int kCount = 0;
    char tempFrogsArray[arrayLenth][6];
    // 清空初始化
    for (int j = 0; j < arrayLenth; j++) {
        for (int k = 0; k < 6; k++) {
            tempFrogsArray[j][k] = '\0';
        }
    }
    // 遍历每一个字母，往临时二维数组里填充数据
    for (char croakOfFrog: croakOfFrogs) {
        // 遇到c，判断有没有满了的项，满了的话，清空重新接收，否则新开一个
        if (croakOfFrog == 'c') {
            cCount++;
            for (int j = 0; j < arrayLenth; j++) {
                if (std::string(tempFrogsArray[j]) == "croak") {
                    // 清空数组
                    for (int k = 0; k < 6; k++) {
                        tempFrogsArray[j][k] = '\0';
                    }
                    tempFrogsArray[j][0] = 'c';
                    break;
                }
                if (std::string(tempFrogsArray[j]).empty()) {
                    tempFrogsArray[j][0] = 'c';
                    break;
                }
            }
        } else if (croakOfFrog == 'r') {
            rCount++;
            for (int j = 0; j < arrayLenth; j++) {
                if (std::string(tempFrogsArray[j]) == "c") {
                    tempFrogsArray[j][1] = 'r';
                    break;
                }
            }
        } else if (croakOfFrog == 'o') {
            oCount++;
            for (int j = 0; j < arrayLenth; j++) {
                if (std::string(tempFrogsArray[j]) == "cr") {
                    tempFrogsArray[j][2] = 'o';
                    break;
                }
            }
        } else if (croakOfFrog == 'a') {
            aCount++;
            for (int j = 0; j < arrayLenth; j++) {
                if (std::string(tempFrogsArray[j]) == "cro") {
                    tempFrogsArray[j][3] = 'a';
                    break;
                }
            }
        } else if (croakOfFrog == 'k') {
            kCount++;
            for (int j = 0; j < arrayLenth; j++) {
                if (std::string(tempFrogsArray[j]) == "croa") {
                    tempFrogsArray[j][4] = 'k';
                    break;
                }
            }
        }
        // 检索数组里面满足croak的项数量
        int count = 0;
        for (int j = 0; j < arrayLenth; j++) {
            if (std::string(tempFrogsArray[j]) == "croak") {
                count++;
            }
        }
        frogsNumber = frogsNumber > count ? frogsNumber : count;
    }
    if (cCount != rCount || rCount != oCount || oCount != aCount || aCount != kCount) {
        frogsNumber = -1;
    }
    // 检查数组里面是否有不满足croak的项
    for (int j = 0; j < arrayLenth; j++) {
        if (!std::string(tempFrogsArray[j]).empty() && std::string(tempFrogsArray[j]) != "croak") {
            frogsNumber = -1;
            break;
        }
    }
    return frogsNumber;
}
```

评论区的优秀解：

```
将青蛙分成 5 种：
刚才发出了 c 的声音。
刚才发出了 r 的声音。
刚才发出了 o 的声音。
刚才发出了 a 的声音。
刚才发出了 k 的声音。

遍历 croakOfFrogs，例如当前遍历到 r，那么就看看有没有青蛙刚才发出了 c 的声音，如果有，那么让它接着发出 r 的声音。换句话说，我们需要消耗一个 c，产生一个 r。
```

```cpp
class Solution {
public:
    int minNumberOfFrogs(string croakOfFrogs) {
        int c = 0;
        int r = 0;
        int o = 0;
        int a = 0;
        int k = 0;
        int max = 0;
        for (char item : croakOfFrogs) {
            if (item == 'c') {
                k--;
                c++;
                if (k < max) {
                    max = k;
                }
            } else if (item == 'r') {
                c--;
                r++;
            } else if (item == 'o') {
                r--;
                o++;
            } else if (item == 'a') {
                o--;
                a++;
            } else if (item == 'k') {
                a--;
                k++;
            }
            if (c < 0 || r < 0 || o < 0 || a < 0) {
                return -1;
            }
        }
        if (c != 0 || r != 0 || o != 0 || a != 0) {
            return -1;
        }
        return -max;
    }
};
```

#### 分发糖果
```
n 个孩子站成一排。给你一个整数数组 ratings 表示每个孩子的评分。

你需要按照以下要求，给这些孩子分发糖果：

每个孩子至少分配到 1 个糖果。
相邻两个孩子评分更高的孩子会获得更多的糖果。
请你给每个孩子分发糖果，计算并返回需要准备的 最少糖果数目 。

示例 1：
输入：ratings = [1,0,2]
输出：5
解释：你可以分别给第一个、第二个、第三个孩子分发 2、1、2 颗糖果。

示例 2：
输入：ratings = [1,2,2]
输出：4
解释：你可以分别给第一个、第二个、第三个孩子分发 1、2、1 颗糖果。
     第三个孩子只得到 1 颗糖果，这满足题面中的两个条件。
```

官方题解：

```
我们可以将「相邻的孩子中，评分高的孩子必须获得更多的糖果」这句话拆分为两个规则，分别处理。
• 左规则：当 ratings[i−1]<ratings[i] 时，i 号学生的糖果数量将比 i−1 号孩子的糖果数量多。
• 右规则：当 ratings[i]>ratings[i+1] 时，i 号学生的糖果数量将比 i+1 号孩子的糖果数量多。
我们遍历该数组两次，处理出每一个学生分别满足左规则或右规则时，最少需要被分得的糖果数量。每个人最终分得的糖果数量即为这两个数量的最大值。
```

```cpp
class Solution {
public:
    int candy(vector<int>& ratings) {
        int n = ratings.size();
        vector<int> left(n);
        for (int i = 0; i < n; i++) {
            if (i > 0 && ratings[i] > ratings[i - 1]) {
                left[i] = left[i - 1] + 1;
            } else {
                left[i] = 1;
            }
        }
        int right = 0, ret = 0;
        for (int i = n - 1; i >= 0; i--) {
            if (i < n - 1 && ratings[i] > ratings[i + 1]) {
                right++;
            } else {
                right = 1;
            }
            ret += max(left[i], right);
        }
        return ret;
    }
};
```

### 棋盘上刺客到达目的地的问题
原问题：
![](/assets/img/blog/blogs_algorithm_assassin.jpg)

这题我做了前半部分，根据初始环境，标记出棋盘上所有不可达区域，后面就需要尝试移动 `A` ，看看能否顺利到达右下角。因为对二叉树，图等难点完全没有开始学习，所以就卡在这里了。

找到网上的java题解：

```java
package com.stephen.jnitest.knalgorithm;

public class AssassinSolution {

    private static final char EMPTY = '.';

    private static final char OBSTACLE = 'X';

    private static final char ASSASSIN = 'A';

    private static final char LEFT_GUARD = '<';

    private static final char RIGHT_GUARD = '>';

    private static final char UP_GUARD = '^';

    private static final char DOWN_GUARD = 'v';

    public boolean solution(String[] B) {

        int n = B.length;

        int m = B[0].length();

        char[][] board = new char[n][m];

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                board[i][j] = B[i].charAt(j);
            }
        }

        int[] assassinPosition = findAssassinPosition(board);

        if (assassinPosition == null) {
            // Assassin is not on the board
            return false;
        }

        if (isAssassinDetected(board, assassinPosition)) {
            // Assassin is detected by a guard
            return false;
        }

        boolean[][] visited = new boolean[n][m];

        return dfs(board, visited, assassinPosition[0], assassinPosition[1], n - 1, m - 1);

    }

    private int[] findAssassinPosition(char[][] board) {

        int n = board.length;

        int m = board[0].length;

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (board[i][j] == ASSASSIN) {
                    return new int[]{i, j};
                }
            }
        }
        return null;
    }

    private boolean isAssassinDetected(char[][] board, int[] assassinPosition) {

        int n = board.length;

        int m = board[0].length;

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (board[i][j] == LEFT_GUARD && i == assassinPosition[0] && j > assassinPosition[1]) {
                    // Assassin is to the left of the guard
                    return true;
                }
                if (board[i][j] == RIGHT_GUARD && i == assassinPosition[0] && j < assassinPosition[1]) {
                    // Assassin is to the right of the guard
                    return true;
                }

                if (board[i][j] == UP_GUARD && i > assassinPosition[0] && j == assassinPosition[1]) {
                    // Assassin is above the guard
                    return true;
                }

                if (board[i][j] == DOWN_GUARD && i < assassinPosition[0] && j == assassinPosition[1]) {
                    // Assassin is below the guard
                    return true;
                }
            }
        }
        return false;
    }

    private boolean dfs(char[][] board, boolean[][] visited, int i, int j, int n, int m) {
        if (i < 0 || i > n || j < 0 || j > m || board[i][j] == OBSTACLE || visited[i][j]) {
            return false;
        }
        if (i == n && j == m) {
            return true;
        }
        visited[i][j] = true;
        return dfs(board, visited, i + 1, j, n, m) || dfs(board, visited, i - 1, j, n, m) || dfs(board, visited, i, j + 1, n, m) || dfs(board, visited, i, j - 1, n, m);
    }
}
```

## 后续计划
边系统学习算法，边刷150经典题，本阶段关注完成度和广度。

初级阶段完成之后，尝试进阶更为复杂的算法，关注性能，复杂度。