---
title: ReactOS 数据文件和PE文件映射
date: 2017-07-07 17:15:33
tags:
- ReactOS
- Memory
categories:
- ReactOS
---

上一篇总结了一下内存管理的基础知识，里面也有NtAllocateVirtualMemory的解析，但是在日常工作中用文件映射更多。文件映射又分为普通数据文件映射和PE文件映射两种。本篇以ReactOS的数据文件和PE文件映射为例总结一下这个过程，细致地了解一下过程中的细节，防止工作中碰到坑。




** 更新记录 **

1. 文章完成于 2017-07-07 09:24

** 参考资料 **

1. 《Windows内核情景分析-采用开源代码ReactOS》
2. ReactOS 0.4.5源码


By Andy @2017-07-07 17:24

