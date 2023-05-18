---
layout: post
title:  "Linux cgroups"
date:   2021-07-29 23:33:27 +0800
categories: linux
---
# Linux Cgroups

[TOC]

cgroup全称为Control Groups是Linux 内核提供的一种用于限制一个或者一组进程使用的资源的一种机制。cgroups提供了包括诸如cpu，memory，io等系统资源的精细化控制。大家耳熟能详的Docker正是使用的cgroups来限制容器的资源使用。

## cgroup子系统

cgroups为每种资源提供了一个子系统。他们分别是

- cpu子系统，用于控制进程的cpu使用率
- cpuset子系统，为组中的进程单独分配cpu和内存节点
- couacct子系统，生成关于cpu使用量的报告
- io子系统，设置进程的读/写到块设备或者从块设备读写的限制
- memory子系统，限制进程的内存用量
- device子系统，限制组中的进程对设备的访问
- freezer子系统，允许暂停/继续组中的人物
- net_cls子系统，可以标记组中的数据包
- net_prio子系统，提供一种可以动态设置组中每个接口网络流量的优先级的方法
- perf_event子系统，提供组中perf event的访问
- hugtlb子系统，为组提供huge pages支持
- pid子系统，设置组中进程的限制

## cgroups层次结构 （Hierarchy）

内核使用 cgroup 结构体来表示一个 control group 对某一个或者某几个 cgroups 子系统的资源限制。cgroup 结构体可以组织成一颗树的形式，每一棵cgroup 结构体组成的树称之为一个 cgroups 层级结构。cgroups层级结构可以 attach 一个或者几个 cgroups 子系统，当前层级结构可以对其 attach 的 cgroups 子系统进行资源的限制。每一个 cgroups 子系统只能被 attach 到一个 cpu 层级结构中。

<!-- ![cgrooups hierarchy](/assets/images/image-20210731002311458.png) -->

比如上图表示两个cgroups层级结构，每一个层级结构中是一颗树形结构，树的每一个节点是一个 cgroup 结构体（比如cpu_cgrp, memory_cgrp)。第一个 cgroups 层级结构 attach 了 cpu 子系统和 cpuacct 子系统， 当前 cgroups 层级结构中的 cgroup 结构体就可以对 cpu 的资源进行限制，并且对进程的 cpu 使用情况进行统计。 第二个 cgroups 层级结构 attach 了 memory 子系统，当前 cgroups 层级结构中的 cgroup 结构体就可以对 memory 的资源进行限制。

在每一个 cgroups 层级结构中，每一个节点（cgroup 结构体）可以设置对资源不同的限制权重。比如上图中 cgrp1 组中的进程可以使用60%的 cpu 时间片，而 cgrp2 组中的进程可以使用20%的 cpu 时间片。

## cgroups 实现

Control Groups 对内核进行了如下的扩展

- 系统中的每个任务都有一个指向`css_set`的引用计数指针
- 每个`css_set`包含了一组指向`cgourp_subsys_state`对象的引用计数指针，系统中注册的每个cgroups子系统一个。没有从任务的每个层次结构的成员的cgroups的直接链接，但是可以通过`cgroups_subsys_state`对象的指针来确定。这是因为访问subsystem state在性能关键代码中是预期高频率的行为，而需要任务的实际 cgroup 分配(特别是在 cgroups 之间移动)的操作则不太常见。一个链表使用 `css_set` 贯穿每个` task_struct` 的 `cg_list` 字段，锚定在 css_set->tasks。
- 可以挂载 cgroup 层次结构文件系统，以便从用户空间进行浏览和操作。
- 可以列出被attach到任意cgourp的所有任务（通过PID）

