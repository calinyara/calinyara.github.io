---
layout: post
title:  "在CPU计算资源中占个位"
categories: Technology
tags: CPU占用率 top htop 系统占用率 性能分析 占用率 cpu-utilization performance csp 云计算
author: Calinyara
description:
---

<br>

“占位子”是国人生活里必不可少的一个部分，不管是排队吃饭还是买票候车 :( ，早已成了生活的日常。最近收到某云计算提供商通知，*“如果XX天内，免费计算实例的CPU使用率有*95%*的时间低于*10%*，则该实例将被当作空闲资源回收”*。哎，看来不得不在CPU上也占个位子，以免想用的时候却发现计算实例已被回收了。

<br>

分析需求应该是：

- 需要让CPU始终看起来很忙
- 不影响真正工作负载
- 用户无感，用户在运行负载时无需额外操作

<br>

**[dtop](https://github.com/calinyara/dtop)** 就是一个满足该需求的占位工具。该程序原本用于统计系统中CPU使用率，**其原理是先占据系统全部CPU计算资源，当有真正负载运行时再让渡出其所需的计算资源**。举个例子，当系统中没有负载运行时，该程序占据100%的CPU，当有一个程序负载需要运行并占据10%的CPU时，**dtop**则让出10%的CPU资源，并占据剩下的90%。从外界看来，该系统的CPU使用率一直保持在100%，并没有任何变化。在这一过程中，**dtop**也统计出了系统中CPU的利用率及空闲率。

<br>

<div align="center"><img src="/assets/images/20230401-bag-seats-in-cpu/cpu_utilization.png"/></div>
<br>

具体操作时，可以利用**tmux**在计算实例上开一个新的session，并运行**dtop**。这样可以保持计算实例的CPU利用率一直处于100%。而当有工作负载要运行时，**dtop**会主动让出CPU计算资源，不会对工作负载产生任何影响，且无需用户进行任何额外操作。算是给工作负载在CPU中占了个座，正主来了主动让座。

<br>

```shell
$ tmux new -s dtop
$ dtop -i 5 
```

<br>

<!-- Global site tag (gtag.js) - Google Analytics -->

<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'UA-66555622-4');
</script>

<br>

<!-- Google tag (gtag.js) -->

<script async src="https://www.googletagmanager.com/gtag/js?id=G-27WH7FZ7KT"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-27WH7FZ7KT');
</script>
