---
title: lua table.sort方法
date: 2019-09-03 21:15:43
categories:
- Programming
tags:
- Lua
---

# 问题背景

- 排序出现报错
- 排序顺序不正确的现象

**报错**

```
invalid order function for sorting
```

出现了一次报错，但是在测试过程中没有复现，之后项目中出现了相同报错导致场景打不开。

**顺序不正确**

例子（功能：对一系列名称进行排序）：

```lua
-- 输入：a1, a2, a3, b1, b2, b3, chusheng, a5, a10, a4, b7, b8, marker15, marker16, marker17, marker18, marker19
local function compfunc(s1, s2)
    -- s1,s2分为数字段和非数字段，依次进行对比，比如b1和b2，先比较b和b，再比较1和2
    for i = 1, math.min(#s1tokens, #s2tokens) do
        if s1tokens[1] ~= s1tokens[2] then
            return -- 返回比较结果
        end
    end
    return #s1tokens <= #s2tokens
end

table.sort(names, compfunc)
-- 期待的结果：a1, a2, a3, a4, a5, a10, b1, b2, b3, b7, b8, chusheng, marker15, marker16, marker17, marker18, marker19
-- 实际的结果：a1, a2, a3, a4, a5, a10, b1, b2, b8, b3, b7, chusheng, marker15, marker16, marker17, marker18, marker19
```

**问题原因**

感谢冥王星在WT中留了table.sort的使用问题

对于同样的输入，加不加等号结果不正确和正确之分

# 查问题原因

在每次比较中加打印，发现有等号的比较次数多，而且出现了一次**b7和b7**的比较

![1567563813014](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1567563813014.png)

我们的排序表中没有连个b7，但是这里出现了b7和b7的比较

打印每次比较后表中的顺序，将出错的过程缩到下面的范围

```lua
local markersname = {
	'b8',
	'b2',
	'b3',
	'b1',
	'b7',
}
-- 排序的结果：b1, b2, b8, b3, b7
```

提取简单排序，从自定义的字符串比较中分出来

```lua
local tb = {8, 2, 3, 1, 7}
table.sort(tb, function(a, b)
	return a <= b
end)
-- 排序的结果：1, 2, 8, 3, 7
```

正确的比较过程和错误的比较过程对比如下：

![1567569473515](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1567569473515.png)

查网上的table排序使用的方法是快排，用常见快排过程分析失败

看lua源码(5.3.5)中的sort方法

```C
static IdxT partition (lua_State *L, IdxT lo, IdxT up) {
  IdxT i = lo;  /* will be incremented before first use */
  IdxT j = up - 1;  /* will be decremented before first use */
  /* loop invariant: a[lo .. i] <= P <= a[j .. up] */
  for (;;) {
    /* next loop: repeat ++i while a[i] < P */
    while (lua_geti(L, 1, ++i), sort_comp(L, -1, -2)) {
      if (i == up - 1)  /* a[i] < P  but a[up - 1] == P  ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* remove a[i] */
    }
    /* after the loop, a[i] >= P and a[lo .. i - 1] < P */
    /* next loop: repeat --j while P < a[j] */
    while (lua_geti(L, 1, --j), sort_comp(L, -3, -1)) {
      if (j < i)  /* j < i  but  a[j] > P ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* remove a[j] */
    }
    /* after the loop, a[j] <= P and a[j + 1 .. up] >= P */
    if (j < i) {  /* no elements out of place? */
      /* a[lo .. i - 1] <= P <= a[j + 1 .. i .. up] */
      lua_pop(L, 1);  /* pop a[j] */
      /* swap pivot (a[up - 1]) with a[i] to satisfy pos-condition */
      set2(L, up - 1, i);
      return i;
    }
    /* otherwise, swap a[i] - a[j] to restore invariant and repeat */
    set2(L, i, j);
  }
}
```

7和7比较后就会报错，与实际现象不符合

网上看到这个报错贴出来的源码和上面的不同，所以搜引擎中的lua库的.h文件，显示5.1.5

然后看lua源码(5.1.5)中的sort方法

```C
    /* a[l] <= P == a[u-1] <= a[u], only need to sort from l+1 to u-2 */
    i = l; j = u-1;
    for (;;) {  /* invariant: a[l..i] <= P <= a[j..u] */
      /* repeat ++i until a[i] >= P */
      while (lua_rawgeti(L, 1, ++i), sort_comp(L, -1, -2)) {
        if (i>u) luaL_error(L, "invalid order function for sorting");
        lua_pop(L, 1);  /* remove a[i] */
      }
      /* repeat --j until a[j] <= P */
      while (lua_rawgeti(L, 1, --j), sort_comp(L, -3, -1)) {
        if (j<l) luaL_error(L, "invalid order function for sorting");
        lua_pop(L, 1);  /* remove a[j] */
      }
      if (j<i) {
        lua_pop(L, 3);  /* pop pivot, a[i], a[j] */
        break;
      }
      set2(L, i, j);
    }
```

这里可以看到它不是通过i、j的值控制是否停止比较，而是一直进行++和--操作，靠比较函数来判断是否停止比较

因而会因为 i 超过边界而引发异常 invalid order function for sorting，或者发生排序出现错误的情况

首先是排序出现不正确的地方，7和7比较之后，8和7又做了一次比较，87交换，后续都出了问题

如果这个地方是7和7比较，那么就会超过边界，抛出错误



首先看`{8, 2, 3, 1, 7}`发生排序顺序不正确的情况

**排序过程：**

- 8 3 7排序，`{3, 2, 7, 1, 8}`

- 中间值作为哨兵，并放在倒数第二位（up - 1），`{3, 2, 1, 7, 8}`

- 从左到右++i与哨兵比较

  `{3, 2, 1, 7, 8}`

  i从1到4，这时7和7比较，应该返回false，i = 4，这里返回了true，所以继续比较，i = 5

- 从右（up - 2）到左--j与哨兵比较

  `{3, 2, 1, 7, 8}`

  7与1比较，不满足，所以 j = 3

- j < i，停止比较

- 把哨兵放到它该在的位置上`swap pivot (a[u-1]) with a[i]`，**出错**，算法中按照i的位置来判断它该交换的位置，应该7和7交换，这里8和7交换

- 之后`i - low = 4 < up - i = 0`不成立

  （6, 5）小区间递归排序

  （1, 4）大区间循环排序`{3, 2, 1, 8}`

  正确的情况`i - low = 3 < up - i = 1`不成立

  （5, 5）小区间递归排序

  （1, 3）大区间循环排序`{3, 2, 1}`

- 对于`{3, 2, 1, 8}`

  - 3、2、8排序`{2, 3, 1, 8}`
  - 3放到 up - 1的位置`{2, 1, 3, 8}`
  - i从1加到3，3与3比较，返回true，继续比较，过程同上i = 4
  - 3与8交换`{2, 1, 8, 3}`
  - `{2, 1, 8}`排序`{1, 2, 8}`
  - 结果为`{1, 2, 8, 3}`

- 最终结果为`{1, 2, 8, 3, 7}`

把8换成7对数组排序`{7, 2, 3, 1, 7}`，报错`invalid order function for sorting`



最后贴一下两个版本的源码

区别

1. 结构上，5.3.5版本将获得哨兵索引的过程提取了出来
2. 新版本中抛出错误的判断进行了改进
3. 新版本对数量较大的排序做了优化，随机取哨兵
4. 递归调用有修改，对大的一边区间做了选取哨兵的优化

```C
// 5.1.5

static void auxsort (lua_State *L, int l, int u) {
  while (l < u) {  /* for tail recursion */
    int i, j;
    /* sort elements a[l], a[(l+u)/2] and a[u] */
    lua_rawgeti(L, 1, l);
    lua_rawgeti(L, 1, u);
    if (sort_comp(L, -1, -2))  /* a[u] < a[l]? */
      set2(L, l, u);  /* swap a[l] - a[u] */
    else
      lua_pop(L, 2);
    if (u-l == 1) break;  /* only 2 elements */
    i = (l+u)/2;
    lua_rawgeti(L, 1, i);
    lua_rawgeti(L, 1, l);
    if (sort_comp(L, -2, -1))  /* a[i]<a[l]? */
      set2(L, i, l);
    else {
      lua_pop(L, 1);  /* remove a[l] */
      lua_rawgeti(L, 1, u);
      if (sort_comp(L, -1, -2))  /* a[u]<a[i]? */
        set2(L, i, u);
      else
        lua_pop(L, 2);
    }
    if (u-l == 2) break;  /* only 3 elements */
    lua_rawgeti(L, 1, i);  /* Pivot */
    lua_pushvalue(L, -1);
    lua_rawgeti(L, 1, u-1);
    set2(L, i, u-1);
    /* a[l] <= P == a[u-1] <= a[u], only need to sort from l+1 to u-2 */
    i = l; j = u-1;
    for (;;) {  /* invariant: a[l..i] <= P <= a[j..u] */
      /* repeat ++i until a[i] >= P */
      while (lua_rawgeti(L, 1, ++i), sort_comp(L, -1, -2)) {
        if (i>u) luaL_error(L, "invalid order function for sorting");
        lua_pop(L, 1);  /* remove a[i] */
      }
      /* repeat --j until a[j] <= P */
      while (lua_rawgeti(L, 1, --j), sort_comp(L, -3, -1)) {
        if (j<l) luaL_error(L, "invalid order function for sorting");
        lua_pop(L, 1);  /* remove a[j] */
      }
      if (j<i) {
        lua_pop(L, 3);  /* pop pivot, a[i], a[j] */
        break;
      }
      set2(L, i, j);
    }
    lua_rawgeti(L, 1, u-1);
    lua_rawgeti(L, 1, i);
    set2(L, u-1, i);  /* swap pivot (a[u-1]) with a[i] */
    /* a[l..i-1] <= a[i] == P <= a[i+1..u] */
    /* adjust so that smaller half is in [j..i] and larger one in [l..u] */
    if (i-l < u-i) {
      j=l; i=i-1; l=i+2;
    }
    else {
      j=i+1; i=u; u=j-2;
    }
    auxsort(L, j, i);  /* call recursively the smaller one */
  }  /* repeat the routine for the larger one */
}
```

---

```C
// 5.3.5

/*
** Does the partition: Pivot P is at the top of the stack.
** precondition: a[lo] <= P == a[up-1] <= a[up],
** so it only needs to do the partition from lo + 1 to up - 2.
** Pos-condition: a[lo .. i - 1] <= a[i] == P <= a[i + 1 .. up]
** returns 'i'.
*/
static IdxT partition (lua_State *L, IdxT lo, IdxT up) {
  IdxT i = lo;  /* will be incremented before first use */
  IdxT j = up - 1;  /* will be decremented before first use */
  /* loop invariant: a[lo .. i] <= P <= a[j .. up] */
  for (;;) {
    /* next loop: repeat ++i while a[i] < P */
    while (lua_geti(L, 1, ++i), sort_comp(L, -1, -2)) {
      if (i == up - 1)  /* a[i] < P  but a[up - 1] == P  ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* remove a[i] */
    }
    /* after the loop, a[i] >= P and a[lo .. i - 1] < P */
    /* next loop: repeat --j while P < a[j] */
    while (lua_geti(L, 1, --j), sort_comp(L, -3, -1)) {
      if (j < i)  /* j < i  but  a[j] > P ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* remove a[j] */
    }
    /* after the loop, a[j] <= P and a[j + 1 .. up] >= P */
    if (j < i) {  /* no elements out of place? */
      /* a[lo .. i - 1] <= P <= a[j + 1 .. i .. up] */
      lua_pop(L, 1);  /* pop a[j] */
      /* swap pivot (a[up - 1]) with a[i] to satisfy pos-condition */
      set2(L, up - 1, i);
      return i;
    }
    /* otherwise, swap a[i] - a[j] to restore invariant and repeat */
    set2(L, i, j);
  }
}

/*
** Choose an element in the middle (2nd-3th quarters) of [lo,up]
** "randomized" by 'rnd'
*/
static IdxT choosePivot (IdxT lo, IdxT up, unsigned int rnd) {
  IdxT r4 = (up - lo) / 4;  /* range/4 */
  IdxT p = rnd % (r4 * 2) + (lo + r4);
  lua_assert(lo + r4 <= p && p <= up - r4);
  return p;
}

/*
** QuickSort algorithm (recursive function)
*/
static void auxsort (lua_State *L, IdxT lo, IdxT up,
                                   unsigned int rnd) {
  while (lo < up) {  /* loop for tail recursion */
    IdxT p;  /* Pivot index */
    IdxT n;  /* to be used later */
    /* sort elements 'lo', 'p', and 'up' */
    lua_geti(L, 1, lo);
    lua_geti(L, 1, up);
    if (sort_comp(L, -1, -2))  /* a[up] < a[lo]? */
      set2(L, lo, up);  /* swap a[lo] - a[up] */
    else
      lua_pop(L, 2);  /* remove both values */
    if (up - lo == 1)  /* only 2 elements? */
      return;  /* already sorted */
    if (up - lo < RANLIMIT || rnd == 0)  /* small interval or no randomize? */
      p = (lo + up)/2;  /* middle element is a good pivot */
    else  /* for larger intervals, it is worth a random pivot */
      p = choosePivot(lo, up, rnd);
    lua_geti(L, 1, p);
    lua_geti(L, 1, lo);
    if (sort_comp(L, -2, -1))  /* a[p] < a[lo]? */
      set2(L, p, lo);  /* swap a[p] - a[lo] */
    else {
      lua_pop(L, 1);  /* remove a[lo] */
      lua_geti(L, 1, up);
      if (sort_comp(L, -1, -2))  /* a[up] < a[p]? */
        set2(L, p, up);  /* swap a[up] - a[p] */
      else
        lua_pop(L, 2);
    }
    if (up - lo == 2)  /* only 3 elements? */
      return;  /* already sorted */
    lua_geti(L, 1, p);  /* get middle element (Pivot) */
    lua_pushvalue(L, -1);  /* push Pivot */
    lua_geti(L, 1, up - 1);  /* push a[up - 1] */
    set2(L, p, up - 1);  /* swap Pivot (a[p]) with a[up - 1] */
    p = partition(L, lo, up);
    /* a[lo .. p - 1] <= a[p] == P <= a[p + 1 .. up] */
    if (p - lo < up - p) {  /* lower interval is smaller? */
      auxsort(L, lo, p - 1, rnd);  /* call recursively for lower interval */
      n = p - lo;  /* size of smaller interval */
      lo = p + 1;  /* tail call for [p + 1 .. up] (upper interval) */
    }
    else {
      auxsort(L, p + 1, up, rnd);  /* call recursively for upper interval */
      n = up - p;  /* size of smaller interval */
      up = p - 1;  /* tail call for [lo .. p - 1]  (lower interval) */
    }
    if ((up - lo) / 128 > n) /* partition too imbalanced? */
      rnd = l_randomizePivot();  /* try a new randomization */
  }  /* tail call auxsort(L, lo, up, rnd) */
}
```

