---
title: "Best way to generate version string by cmake"
date: 2024-04-25T08:47:14+08:00
draft: false
---

使用 cmake 生成软件编译所需要的版本信息，比如 git commit hash 等，把这些信息写入到对应的头文件中，然后在编译的时候使用。
cmake 中 [configure_file](https://cmake.org/cmake/help/latest/command/configure_file.html) 就可以用于此目的

```
configure_file(<input> <output>
               [NO_SOURCE_PERMISSIONS | USE_SOURCE_PERMISSIONS |
                FILE_PERMISSIONS <permissions>...]
               [COPYONLY] [ESCAPE_QUOTES] [@ONLY]
               [NEWLINE_STYLE [UNIX|DOS|WIN32|LF|CRLF] ])
```

最重要的参数有两个，input 和 output
input 可以指定输入文件的模版，比如输入文件中可以如此写：


```C
// version.h.in
#pragma once
#define VERSION "@VERSION@"
```

```cmake
execute_process(
  COMMAND git log --pretty=format:'%cd' -n 1
  OUTPUT_VARIABLE VERSION
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  ERROR_QUIET)
configure_file(version.h.in version.h)
```

这样的话，在 cmake configure 阶段， version.h.in 中的`@VERSION@` 就会被 cmake 中的 VERSION 变量替换

```C
// version.h
#pragma once
#define VERSION "abcd123"
```



