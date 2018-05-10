---
title: LeetCode 算法题刷题心得（JavaScript）
date: 2018-05-09
---

> 花了十几天，把《算法》看了一遍然后重新 AC 了一遍 LeetCode 的题，收获颇丰。这次好好记录下心得。
我把所有做题的代码都放在 github 上以供参考。
项目地址：https://github.com/violetjack/LeetCodeACByJS
题目地址：https://leetcode.com/problemset/top-interview-questions/

说来惭愧，之前写的《LeetCode 逻辑题分享》其实自己动手做的比较少，都是看解决方案。更加关键的是**我没有系统地去学习过算法**（自学的编程）。所以导致以下几个问题：

* 看题不懂方法论，理解他人方案困难。
* 解题方法通过看别人的方案去归纳，照着抄。（其实都是有系统的算法写法的）
* 很多题目看了答案只是知其然而不知其所以然。
* 很多答案（讨论区的方案）是有错误的，却把它当正确答案来发。

之后，我看了《算法（第4版）》一书，重新去做并且试着去 AC 题目，问题又是一堆堆的。所以这次比第一次刷题时间要久不少。

# 各类题的解决方案

话不多说，系统整理下解题的一些算法和解决方案

## 二叉树

二叉树大多使用递归的方式左右两个元素向下递归。比如：

**计算二叉树最大深度**
```js
var maxDepth = function (root) {
    if (root == null) return 0
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right))
};
```
**将二叉树以二维数组形式表现**
```js
var levelOrder = function(root) {
    let ans = []
    helper(root, ans, 0)
    return ans
};

function helper(node, ans, i){
    if (node == null) return
    if (i == ans.length) ans.push([])
    ans[i].push(node.val)

    helper(node.left, ans, i + 1)
    helper(node.right, ans, i + 1)
}
```
都是通过递归方式逐层向下去查找二叉树数据。

## 可能性问题

这类题一般是告诉你一组数据，然后求出可能性、最小值或最大值。比如：

**给定几种面额的硬币和一个总额，使用最少的硬币凑成这个总额。**
```js
var coinChange = function (coins, amount) {
    let max = amount + 1
    let dp = new Array(amount + 1)
    dp.fill(max)
    dp[0] = 0

    for (let i = 1; i < max; i++) {
        for (let j = 0; j < coins.length; j++) {
            if (coins[j] <= i) {
                dp[i] = Math.min(dp[i], dp[i - coins[j]] + 1)
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount]
};
```
使用了动态规划（DP），将从 0 到目标额度所需的最小硬币数都列出来。

**求出从矩阵左上角走到右下角，且只能向右向下移动，一共有多少种可能性。**
```js
var uniquePaths = function (m, n) {
    const pos = new Array(m)
    for (let i = 0; i < m; i++) {
        pos[i] = new Array(n)
    }
    for (let i = 0; i < n; i++) {
        pos[0][i] = 1
    }
    for (let i = 0; i < m; i++) {
        pos[i][0] = 1
    }
    for (let i = 1; i < m; i++) {
        for (let j = 1; j < n; j++) {
            pos[i][j] = pos[i - 1][j] + pos[i][j - 1]
        }
    }
    return pos[m - 1][n - 1]
};
```
这题就是使用了动态规划逐步列出每一格的可能性，最后返回右下角的可能性。

**获取给定数组连续元素累加最大值**
```js
var maxSubArray = function (nums) {
    let count = nums[0], maxCount = nums[0]
    for (let i = 1; i < nums.length; i++) {
        count = Math.max(count + nums[i], nums[i])
        maxCount = Math.max(maxCount, count)    
    }
    return maxCount
};
```
上面这题通过不断对比最大值来保留并返回最大值。

其实，可能性问题使用**动态规划**要比使用 DFS、BFS 算法更加简单而容易理解。（我使用 DFS 经常报 TLE）

## 查找

一般遇到的查找问题，如查找某个值一般会用到一下方法：

* 排序算法（排序便于查找）
* 二分查找
* 索引移动查找（这个方法名自己想的，大概就这个意思~）

**查找横向和纵向都递增的二维矩阵中的某个值**
```js
var searchMatrix = function (matrix, target) {
    if (matrix.length == 0) return false
    let row = 0, col = matrix[0].length - 1
    while (true) {
        if (matrix[row][col] > target && col > 0) {
            col--
        } else if (matrix[row][col] < target && row < matrix.length - 1) {
            row++
        } else if (matrix[row][col] == target) {
            return true
        } else {
            break
        }
    }
    return false
};
```
先将位置定位在右上角，通过改变位置坐标来找到目标值。使用了索引移动查找法来找到结果。

**找到数组中最左边和最右边的某个数字所在位置**
```js
var searchRange = function (nums, target) {
    let targetIndex = binarySearch(nums, target, 0, nums.length - 1)
    if (targetIndex == -1) return [-1, -1]
    let l = targetIndex, r = targetIndex
    while(l > 0 && nums[l - 1] == target){
        l--
    }
    while(r < nums.length - 1 && nums[r + 1] == target){
        r++
    }
    return [l, r]
};

function binarySearch(arr, val, lo, hi) {
    if (hi < lo) return -1
    let mid = lo + parseInt((hi - lo) / 2)

    if (val < arr[mid]) {
        return binarySearch(arr, val, lo, mid - 1)
    } else if (val > arr[mid]) {
        return binarySearch(arr, val, mid + 1, hi)
    } else {
        return mid
    }
}
```
这题使用**二分法**来查找到某个目标数字的索引值，然后**索引移动法**分别向左和向右查找字符。获取左右两侧的索引值返回。

## 回文

所谓回文，就是正着读反着读是一样的。使用索引两边向中间移动的方式来判断是否为回文。

**找到给定字符串中某段最长的回文**
```js
var longestPalindrome = function (s) {
    let maxLength = 0, left = 0, right = 0
    for (let i = 0; i < s.length; i++) {
        let singleCharLength = getPalLenByCenterChar(s, i, i)
        let doubleCharLength = getPalLenByCenterChar(s, i, i + 1)
        let max = Math.max(singleCharLength, doubleCharLength)
        if (max > maxLength) {
            maxLength = max
            left = i - parseInt((max - 1) / 2)
            right = i + parseInt(max / 2)
        }
    }
    return s.slice(left, right + 1)
};

function getPalLenByCenterChar(s, left, right) {
    // 中间值为两个字符，确保两个字符相等
    if (s[left] != s[right]){
        return right - left
    }
    while (left > 0 && right < s.length - 1) {
        left--
        right++
        if (s[left] != s[right]){
            return right - left - 1
        }
    }
    return right - left + 1
}
```

## 路径题

路径题可以使用深度优先（DFS）和广度优先（BFS）算法来做。我比较常用的是使用 DFS 来做。通过递归将走过的路径进行标记来不断往前找到目标路径。如：

**通过给定单词在二维字母数组中查找是否能使用邻近字母组成这个单词**([212题](https://leetcode.com/problems/word-search-ii/description/))
```js
let hasWord = false

var findWords = function (board, words) {
    var ans = []
    for (let word of words) {
        for (let j = 0; j < board.length; j++) {
            for (let i = 0; i < board[0].length; i++) {
                if (board[j][i] == word[0]) {
                    hasWord = false
                    DFS(word, board, 0, j, i, "")
                    if (hasWord) {
                        if (!ans.includes(word))
                            ans.push(word)
                    }
                }
            }
        }
    }
    return ans
};

function DFS(word, board, index, j, i, subStr) {
    if (word[index] == board[j][i]) {
        subStr += board[j][i]
        board[j][i] = "*"
        if (j < board.length - 1)
            DFS(word, board, index + 1, j + 1, i, subStr)
        if (j > 0)
            DFS(word, board, index + 1, j - 1, i, subStr)
        if (i < board[0].length - 1)
            DFS(word, board, index + 1, j, i + 1, subStr)
        if (i > 0)
            DFS(word, board, index + 1, j, i - 1, subStr)
        board[j][i] = word[index]
    }
    if (index >= word.length || subStr == word) {
        hasWord = true
    }
}
```

由于 DFS 是一条路走到黑，如果每个元素都去使用 DFS 来找会出现超时的情况。如果条件允许（如查找递增数组）可以通过**设置缓存**来优化 DFS 查找超时问题。

**获取二维矩阵中最大相邻递增数组长度。**
```js
const dirs = [[0, 1], [1, 0], [0, -1], [-1, 0]]

var longestIncreasingPath = function (matrix) {
    if (matrix.length == 0) return 0
    const m = matrix.length, n = matrix[0].length
    let max = 1

    let cache = new Array(m)
    for (let i = 0; i < m; i++){
        let child = new Array(n)
        child.fill(0)
        cache[i] = child
    }

    for (let i = 0; i < m; i++) {
        for (let j = 0; j < n; j++) {
            let len = dfs(matrix, i, j, m, n, cache)
            max = Math.max(max, len)
        }
    }
    return max
}

function dfs(matrix, i, j, m, n, cache){
    if (cache[i][j] != 0) return cache[i][j]
    let max = 1
    for (let dir of dirs){
        let x = i + dir[0], y = j + dir[1]
        if(x < 0 || x >= m || y < 0 || y >= n || matrix[x][y] <= matrix[i][j]) continue;
        let len = 1 + dfs(matrix, x, y, m, n, cache)
        max = Math.max(max, len)
    }
    cache[i][j] = max
    return max
}
```
将已使用 DFS 查找过的长度放入缓存，如果有其他元素走 DFS 走到当前值，直接返回缓存最大值即可。

## 链表

链表从 JS 的角度来说就是一串对象使用指针连接的数据结构。合理使用 `next` 指针改变指向来完成对链表的一系列操作。如：

**链表的排序：**
```js
var sortList = function (head) {
    if (head == null || head.next == null) return head

    let prev = null, slow = head, fast = head
    while (fast != null && fast.next != null) {
        prev = slow
        slow = slow.next
        fast = fast.next.next
    }

    prev.next = null;

    let l1 = sortList(head)
    let l2 = sortList(slow)

    return merge(l1, l2)
};

function merge(l1, l2) {
    let l = new ListNode(0), p = l;

    while (l1 != null && l2 != null) {
        if (l1.val < l2.val) {
            p.next = l1;
            l1 = l1.next;
        } else {
            p.next = l2;
            l2 = l2.next;
        }
        p = p.next;
    }

    if (l1 != null)
        p.next = l1;

    if (l2 != null)
        p.next = l2;

    return l.next;
}
```
使用了**自上而下的归并排序方法**对链表进行了排序。使用 `slow.next` 和 `fast.next.next` 两种速度获取链表节点，从而获取中间值。

**链表的倒序**
```js
var reverseList = function(head) {
    let ans = null,cur = head
    while (cur != null) {
        let nextTmp = cur.next
        cur.next = ans
        ans = cur
        cur = nextTmp
    }
    return ans
};
```


## 排序

排序和查找算是算法中最重要的问题了。常用的排序算法有：

* 插入排序
* 选择排序
* 快速排序
* 归并排序
* 计数排序

更多排序算法的知识点可参考[《JS家的排序算法》](https://www.jianshu.com/p/1b4068ccd505)，文章作者图文并茂的讲解了各种排序算法，很容易理解。
举几个排序算法的栗子：

**求数组中第K大的值**
```js
/**
 * @param {number[]} nums
 * @param {number} k
 * @return {number}
 */
var findKthLargest = function (nums, k) {
    for (let i = 0; i <= k; i++) {
        let max = i
        for (let j = i; j < nums.length; j++) {
            if (nums[j] > nums[max]) max = j
        }
        swap(nums, i, max)
    }
    return nums[k - 1]
};

function swap(arr, a, b) {
    let tmp = arr[a]
    arr[a] = arr[b]
    arr[b] = tmp
}
```
使用了**选择排序**排列了前 K 个值得到结果。

**对有重复值的数组 `[2,0,2,1,1,0]` 排序**
```js
var sortColors = function (nums) {
    sort(nums, 0, nums.length - 1)
};

function sort(arr, lo, hi) {
    if (hi <= lo) return
    let lt = lo, i = lo + 1, gt = hi;
    let v = arr[lo]
    while (i <= gt) {
        if (arr[i] < v) swap(arr, lt++, i++)
        else if (arr[i] > v) swap(arr, i, gt--)
        else i++
    }
    sort(arr, lo, lt - 1)
    sort(arr, gt + 1, hi)
}

function swap(arr, a, b) {
    let x = arr[a]
    arr[a] = arr[b]
    arr[b] = x
}
```
这种有重复值的使用**三向切分的快速排序**是非常好的解决方案。当然，**计数排序**法可是不错的选择。
还有之前提到的链表的排序使用的是**归并排序**。

## 算术题

算术题看似简单，但是遇到最大的问题就是：如果使用累加、累成这种常熟级别的增长，遇到很大的数字会出现 TLE （超出时间限制）。所以，我们要用指数级别的增长来找到结果。如：

**计算 x 的 n 次方**
```js
var myPow = function (x, n) {
    if (n == 0) return 1
    if (n < 0) {
        n = -n
        x = 1 / x
    }
    return (n % 2 == 0) ? myPow(x * x, parseInt(n / 2)) : x * myPow(x * x, parseInt(n / 2));
};
```
一开始我使用了 x*x 这么乘上 n 次，但是遇到 n 太大就直接超时了。使用以上方案：2^9^ = 2 * 4^4^ = 2 * 8^2^ = 2 * 64 = 128
直接从常熟级变化变为指数级变化，这一点在数学运算中是需要注意的。

**求 x 的平方根**
```js
var mySqrt = function (x) {
    let l = 0, r = x
    while (true) {
        let mid = parseInt(l + (r - l) / 2)
        if (mid * mid > x) {
            r = mid - 1
        } else if (mid * mid < x) {
            if ((mid + 1) * (mid + 1) > x) {
                return mid
            }
            l = mid + 1
        } else {
            return mid
        }
    }
};
```
这题使用二分法来找到结果。

## 二进制问题

二进制问题，一般使用[按位运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators)和二进制转换 `Number.parseInt()` 和 `Number.prototype.toString()`来解决。

**将一个32位数字的二进制进行倒序**
```js
var reverseBits = function(n) {
    var t = n.toString(2).split("");
    while(t.length < 32) t.unshift("0"); // 插入足够的 0
    return parseInt(t.reverse().join(""), 2);
};
```

# 常用算法

讲了这么多，其实除了常用的排序、搜索，其他最常用的就是 DP、DFS、BFS 这三个算法了。可以这么说：掌握了排序和这三个算法，可以 AC 大多数的算法问题。这么牛逼的算法了解一下？

## 简单说说几种排序和查找

* **冒泡排序**：遍历数组，对比元素和后面相邻元素，如果当前元素大于后面元素，调换位置。这样从头遍历到尾，获取最后一位排序玩的元素。然后在 1 到 n - 1 中再次重复以上步骤。直到最后第一和第二个元素对比大小。是一种从后往前的排序。
* **选择排序**：遍历数组，找到最小的元素位置，与第一个元素调换位置，然后缩小范围从第二个元素开始遍历，如此重复到最后一个元素。可以从后往前也可以从前往后排序。
```js
function sort(arr) {
    const len = arr.length
    for (let i = 0; i < len; i++) {
        let min = i
        for (let j = i + 1; j < len; j++) {
            if (arr[j] < arr[min]) min = j
        }
        swap(arr, i, min)
        console.log(arr)
    }
    return arr
}
```
* **插入排序**：遍历数组，选中某一个元素，与前面相邻元素对比，如果当前元素小于之前元素，调换位置，继续对比直到当前元素前的元素小于当前元素（或者到最前面），如此对所有元素排序一遍。是一种从前往后的排序。
```js
function sort(arr) {
    const len = arr.length
    for (let i = 1; i < len; i++) {
        for (let j = i; j > 0 && arr[j] < arr[j - 1]; j--) {
            swap(arr, j, j - 1)
            console.log(arr)
        }
    }
    return arr
}
```
* **希尔排序**：类似于插入排序，选中一个元素与元素前 n 个元素进行比大小和调换位置。之后再缩小 n 的值。这种方法可以减少插入排序中最小值在最后面，然后需要一个一个调换位置知道最前面这类问题。减少调换次数。是一种从前往后的排序。
* **归并排序**：在《算法》中提到了两种归并排序：一种是自上而下的归并排序。将数组不断二分到最小单位（1到2个元素）将他们进行排序，之后将前两个和后两个元素对比，如此往上最后完成整个数组的排序。还有一种自下而上的归并排序是直接将数组分割为若干个子数组进行排序然后合并。
```js
let aux = new Array(arr.length)
function sort(arr, lo, hi) {
    if (hi <= lo) return
    let mid = lo + (parseInt((hi - lo) / 2))

    sort(arr, lo, mid)
    sort(arr, mid + 1, hi)
    merge(arr, lo, mid, hi)
}

function merge(arr, lo, mid, hi) {
    let i = lo, j = mid + 1
    for (let k = lo; k <= hi; k++) {
        aux[k] = arr[k]
    }
    for (let k = lo; k <= hi; k++) {
        if (i > mid) arr[k] = aux[j++]
        else if (j > hi) arr[k] = aux[i++]
        else if (aux[j] < aux[i]) arr[k] = aux[j++]
        else arr[k] = aux[i++]
    }
    console.log(arr)
}
```
* **快速排序**：选定第一个值为中间值，然后将小于中间值的元素放在中间值的左侧而大于中间值的元素放在中间值右侧，然后对两侧的元素分别再次切割，直到最小单位。
```js
function sort(arr, lo, hi) {
    if (hi <= lo + 1) return
    let mid = partition(arr, lo, hi) // 切分方法
    sort(arr, lo, mid)
    sort(arr, mid + 1, hi)
}

function partition(arr, lo, hi) {
    let i = lo, j = hi + 1
    let v = arr[lo]
    while(true) {
        while(arr[++i] < v) if (i == hi) break
        while(v < arr[--j]) if (j == lo) break
        if ((i >= j)) break
        swap(arr, i, j)
        console.log(arr)
    }
    swap(arr, lo, j)
    console.log(arr)
    return j
}
```
* **三向切分的快速排序**：类似于快速排序，优化点在于如果某个元素等于切分元素，元素位置不变。最后小于切分元素的到左边，等于切分元素的根据数量放在中间，大于切分元素的放在右边。**适用于有大量相同大小元素的数组。**
```js
function sort(arr, lo, hi) {
    if (hi <= lo) return
    let lt = lo, i = lo + 1, gt = hi;
    let v = arr[lo]
    while (i <= gt) {
        if (arr[i] < v) swap(arr, lt++, i++)
        else if (arr[i] > v) swap(arr, i, gt--)
        else i++
        console.log(arr)
    }
    sort(arr, lo, lt - 1)
    sort(arr, gt + 1, hi)
}
```
* **堆排序**：堆排序可以说是一种利用堆的概念来排序的选择排序。使用优先队列返回最大值的特性逐个返回当前堆的最大值。
* **计数排序**：就是将数组中所有元素的出现次数保存在一个数组中，然后按照从小到大返回排序后的数组。
* **桶排序**：其实就是字符串排序的 LSD 和 MSD 排序。LSD 使用索引计数法从字符串右边向左边移动，根据当前值进行排序。而 MSD 是从左到右使用索引计数法来排序，在字符串第一个字符后，将字符串数组分为若干个相同首字符串的数组各自进行第二、第三次的 MSD 排序。
* **二分查找**： 对有序数组去中间值与目标值相比对。如果目标值小于中间值，取前一半数组继续二分。如果目标值大于中间值，取后一半数组继续二分。如果目标值等于中间值，命中！

## DP

关于动态规划，可以看下[详解动态规划——邹博讲动态规划](http://www.cnblogs.com/little-YTMM/p/5372680.html)一文，其中讲了路径、硬币、最长子序列。都是 LeetCode 中有的题目。
我的理解：动态规划就是下一状态可以根据上一状态，或之前几个状态获取到的一种推理过程。

## DFS

深度优先搜索（DFS）就是选中某条从条件1到条件2的某条可能性进行搜索，之后返回搜索其他一条可能性，如此一条条升入。举个栗子，如果有5条路，那么 DFS 算法就是只排出一个斥候先走一条路走到底去侦察，如果走不通那么返回走下一条路径。
```js
DFS（顶点v）
{
  标记v为已遍历；
  for（对于每一个邻接v且未标记遍历的点u）
      DFS（u）;
}
```
DFS 使用的是递归的方式进行搜索的。

**示例：**在二维字母矩阵中查找是否能够使用相邻字母组成目标单词。
```js
var exist = function (board, word) {
    for (let y = 0; y < board.length; y++) {
        for (let x = 0; x < board[0].length; x++) {
            if (find(board, word, y, x, 0)) return true
        }
    }
    return false
};

function find(board, word, y, x, d) {
    if (d == word.length) return true
    if (y < 0 || x < 0 || y == board.length || x == board[y].length) return false;
    if (board[y][x] != word[d]) return false
    let tmp = board[y][x]
    board[y][x] = "*"
    let exist = find(board, word, y, x + 1, d + 1)
        || find(board, word, y, x - 1, d + 1)
        || find(board, word, y + 1, x, d + 1)
        || find(board, word, y - 1, x, d + 1)
    board[y][x] = tmp
    return exist
}
```

## BFS

广度优先搜索（BFS）就是将从条件1到条件2的所有可能性都列出来同步搜索的过程。适用于查找最短路径。举个栗子，如果有5条路，那么 BFS 算法就是分别向5条路排出斥候去侦察。
```js
BFS()
{
  输入起始点；
  初始化所有顶点标记为未遍历；
  初始化一个队列queue并将起始点放入队列；

  while（queue不为空）
  {

    从队列中删除一个顶点s并标记为已遍历； 
    将s邻接的所有还没遍历的点加入队列；
  }
}
```
BFS是使用数组存储下一顶点的方式。

**示例：**每次改变一次字母，通过给定数组中的单词，从单词 A 变为单词 B。（[127题](https://leetcode.com/problems/word-ladder/description/)）
```js
/**
 * @param {string} beginWord
 * @param {string} endWord
 * @param {string[]} wordList
 * @return {number}
 */
var ladderLength = function (beginWord, endWord, wordList) {
    if (!wordList.includes(endWord)) return 0
    let set = new Set(),
        visited = new Set(),
        len = 1

    set.add(beginWord)
    visited.add(beginWord)
    while (set.size != 0) {
        let tmp = new Set([...set])

        for (let w of tmp) {
            visited.add(w)
            set.delete(w)

            if (changeOneChar(w, endWord))
                return len + 1
            
            for (let word of wordList){
                if (changeOneChar(w, word) && !visited.has(word)){
                    set.add(word)
                }
            }
        }
        len++
    }
    return 0
};

function changeOneChar(a, b) {
    let count = 0
    for (let i = 0; i < a.length; i++)
        if (a[i] != b[i])
            count++
    return count == 1
}
```

# 最后

写下 AC 一遍题目之后的收获。

* 知道了方法论，做起题来轻松了不少。
* 遇到问题多找轮子，一定有某种方法论可以用。
* 不要耍小聪明用一些奇巧淫技，思路不对再怎么绕都是浪费时间。
* 不要想着自己造轮子（特别是算法方面），绝大多数问题前辈一定有更好更完善的方案在。自己造轮子费时费事又没太大意义。
* 看答案和自己做是两回事，自己动手实现了才能算是会了。
* 算法之所以存在，就是用来适应某些场景、解决某类问题的。在对的场景选择对的算法才能体现算法的价值，不要滥用算法。
* 没必要把所有算法都精通，但起码在遇到问题时可以找到最优算法解决问题。即知道算法的存在及其用途，按需深入学习。

其实刷算法题还是很有趣的事情，之后计划把 [LeetCode 题库](https://leetcode.com/problemset/all/)中的所有问题都刷一遍~

**PS：本文以及相关项目中有任何错误或者可以改进的地方，还请提出。共同进步~**