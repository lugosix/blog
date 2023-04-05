---
title: "c++ std::sort() 使用方式不对引起的 coredump"
date: 2021-06-22T16:00:39+08:00
---
# 一种 coredump 复现方式
```c++
#include <algorithm>
#include <iostream>
#include <vector>

struct Order {
  Order(long t) : timestamp(t) {}
  long timestamp;
};

int main() {
  std::vector<Order> t;
  for (int i = 0; i < 16; i++) {
    t.push_back({i});
  }
  for (int i = 0; i < 16; i++) {
    t.push_back({16});
  }
  std::sort(t.begin(), t.end(), [](const Order &a, const Order &b) {
    if (a.timestamp <= b.timestamp) {
      return true;
    }
    return false;
  });
  return 0;
}
```

执行结果
```bash
g++ -std=c++11 test1.cpp
./a.out 
Segmentation fault (core dumped)
```
# 原因
![](/cpp_sort/1.png)  
截图来自 [cppreference](https://zh.cppreference.com/w/cpp/algorithm/sort)，标准要求仅在小于时返回 true，即在相等时也应返回 false，而上述例子比较函数相等时返回的是 true。具体 coredump 原因为迭代器越界，与标准库实现相关。

总的来说

vector 内元素个数大于16
自定义的比较函数相等时返回 true
满足这两个条件，在 sort 时就有可能触发 coredump

详细源码分析可参考 [coredump 源码分析](https://zhuanlan.zhihu.com/p/364361964)
# 解决方案
修改比较函数，相等时应返回 false。

上述例子中将 “<=“ 修改为 “<” 即可