---
title: "push_back VS emplace_back"
date: 2021-08-04T21:37:51+08:00
draft: false
tags: [C++]
---
C++ 容器 vector, list, deque 都提供了针对序列容器尾部的操作，比如 push_back，pop_back。C++11 提供了新的操作: emplace_back。两者有何区别呢

```c++
c.push_back(x)
c.pop_back()
c.emplace_back(args)
```

push_back 的输入参数为要插入的容器元素对象，容器通过拷贝或者移动该对象把它放入容器的末尾。emplace_back 则不同，它的接受参数为构造容器元素所需要的参数，容器会使用这些参数直接在容器内的对应位置创建容器对象。


举个🌰：
```c++
class Student {
    Student(string name, int age, string address);

private:
    std::string address;
    std::string name;
    int age;
}
```

如果是使用 `push_back`，需要先构造 Student 对象然后把该对象通过 push_back 拷贝到容器中
```C++
void use(vector<Student>& students) {
    Student s ("Carina", 7, "GUANHEJINTING");
    students.push_back(s);
}
```

如果是使用 `emplace_back`，直接把构造 Student 需要的参数传递给 emplace_back 即可
```C++
void use(vector<Student>& students) {
    students.emplace_back("Carina", 7, "GUANHEJINTING");
}    
```


由此我们可以看到使用 push_back 会多了一步对对象拷贝或者移动的工作。如果对于一些对象这个操作（拷贝或者移动）比较耗时的情况，优先使用 emplace_back 会比较好一点。对于大部分情况来说，这个效率问题都应该不是问题。

emplace_back 有另外的一个缺点，由于它要根据参数构造一个容器元素，它构造元素的路径可能不是我们的原意，比如可能经过了不易觉察的隐式转换。比如下面的代码，容器会使用 20 来构造一个 std::vector，这样在 vs 中就插入了一个包含 20 个元素的 vector，而不是插入了 20 本身。这种错误只有在运行时才能得到体现。

```C++
std::vector<std::vector<int>> vs;
vs.emplace_back(20);
```

此外，insert 和 emplace （相比于 push_back, emplace_back), 也有类似的区别

[参考](https://abseil.io/tips/112)

