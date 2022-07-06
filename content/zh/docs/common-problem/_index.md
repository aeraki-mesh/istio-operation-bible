---
title: "Istio 常见问题"
linkTitle: "Istio 常见问题"
weight: 2
description: >
  介绍在使用 Istio 过程中可能遇到的一些常见问题的解决方法
---

Package 用于组织一组逻辑上紧密相关的 go 文件。是 go 语言中代码重用的基础单元。在文件系统中，一个 Package 对应一个文件夹，文件夹中包含该 Packag 中的多个 go 文件。在 go 语言模型中，一个 Packag 中包含了多个紧密相关的变量，结构体和方法。

Package中包含的内容：

```bash
└── package                           
    ├── variable
    ├── function
    └── struct
        ├── variable
        └── method
```