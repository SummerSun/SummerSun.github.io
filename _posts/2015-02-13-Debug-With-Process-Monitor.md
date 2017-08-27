---
layout: post
title:  "Debug with Process Monitor"
---

记录第一次用Process Monitor debug成功解决问题。
<!--excerpt-->

新接了一个项目。 Dev machine setup完了以后成功build，结果发现site打不开。从IIS manager尝试启动site，直接抛出这样一个信息量稀少的提示：W3SVC service没有启动。

<img src="http://7xq1tb.com1.z0.glb.clouddn.com/w3svcError.jpg"> 

尝试启动W3SVC，失败！Deplendented service WAS未启动。启动WAS，可是无论是在servicemsc里面，还是以管理员身份运行的console里面，都是同样的错误：Access Denied。

Google到scott大神的[Blog](http://www.hanselman.com/blog/FixedWindowsProcessActivationServiceWASIsStoppingBecauseItEncounteredAnError.aspx)，继续google WAS access denied，得到解决思路: Process Monitor。

按照link里的方法设置好filter， 强调一下作者说的：filter对process monitor <b>很重要</b>。

<img src="http://7xq1tb.com1.z0.glb.clouddn.com/filter.jpg">

Filter出来的具体的access denied信息，问题就很清楚了：

<img src="http://7xq1tb.com1.z0.glb.clouddn.com/processMonitor.jpg">

启动WAS要创建一个文件（空文件，不知道要做什么用）：bindingInfo.tmp。打开error中的目录下才发现，之所以Access Denied因为这个目录下已经存在了这样一个文件，也不知道什么时候做了什么创建的这个文件。Delete，restart，Done。

引用作者的一段话：

- Debugging is 95% tools and 5% intuition. Know what tools can get you that next bit of information you need to take the next step in your analysis.
- If you feel you've hit a wall in your analysis, knock that wall down. Your process is doing IO to a file/registry/device/network/etc. Watch it. Look for failures.