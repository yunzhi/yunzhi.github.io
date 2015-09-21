---
layout: post
title: SEAndroid的相关知识
category: blog
description: SEAndroid 基于SELinux,是android4.4上引入的一种安全机制，本文介绍了该机制的一些相关知识．
---

＃SElinux

SELinux则是由美国NSA（国安局）和一些公司（RedHat、Tresys）设计的一个针对Linux的安全加强系统。NSA最初设计的安全模型叫FLASK，全称为Flux Advanced Security Kernel（由Uta大学和美国国防部开发，后来由NSA将其开源），当时这套模型针对DTOS系统。后来，NSA觉得Linux更具发展和普及前景，所以就在Linux系统上重新实现了FLASK，称之为SELinux。Linux Kernel中，SELinux通过Linux Security Modules实现。在2.6之前，SElinux通过Patch方式发布。从2.6开始，SELinux正式入驻内核，成为标配。

## 1. DAC和MAC

SELinux出现之前，Linux上的安全模型叫DAC，全称是Discretionary Access Control，翻译为自主访问控制。DAC的核心思想很简单，就是：

>> 进程理论上所拥有的权限与执行它的用户的权限相同。比如，以root用户启动Browser，那么Browser就有root用户的权限，在Linux系统上能干任何事情。

MAC（Mandatory Access Control），翻译为强制访问控制。MAC的处世哲学非常简单：即任何进程想在SELinux系统中干任何事情，都必须先在安全策略配置文件中赋予权限。凡是没有出现在安全策略配置文件中的权限，进程就没有该权限。

## 待整理
１．根据SELinux规范，完整的allow相关的语句格式为：

　　　　rule_name source_type target_type : class perm_set

