---
layout: darft
title: LLDB指令
date: 2020-02-24 16:40:16
tags:
---

## image

```c
      add          -- Add a new module to the current target's modules.
      dump         -- Commands for dumping information about one or more target
                      modules.
      list         -- List current executable and dependent shared library
                      images.
      load         -- Set the load addresses for one or more sections in a
                      target module.
      lookup       -- Look up information within executable and dependent
                      shared library images.
      search-paths -- Commands for managing module search paths for a target.
      show-unwind  -- Show synthesized unwind instructions for a function.
```

### add

### dump

### list

### load

### lookup

```

```



打印符号信息，例如类的结构:

```c
image lookup --t ViewController
```

![image-20200224165050109](/Users/joey.cao/Desktop/Learning/MyBlog/joey520.github.io/source/_posts/LLDB指令/image-20200224165050109.png)打印参数的符号信息，例如查询`ViewController`这个符号：

```c
image lookup --symbol ViewController
```

![image-20200224165014911](/Users/joey.cao/Desktop/Learning/MyBlog/joey520.github.io/source/_posts/LLDB指令/image-20200224165014911.png)

查找符号的地址，可以帮助快速查找到符号声明处，可以方便快速找到方法的定义位置：

```c
image lookup --name myMethod
```

![image-20200224164946578](/Users/joey.cao/Desktop/Learning/MyBlog/joey520.github.io/source/_posts/LLDB指令/image-20200224164946578.png)

### search-paths

### show-unwind

