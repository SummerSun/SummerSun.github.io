---
layout: post
title:  "100 percent CPU Usage"
date:   2015-12-17 16:00:00
tags: [Work]
image:
  feature: abstract-7.jpg
---

Recently we met a 100 percent CPU usage problem in production. I never met this kind of problem before, not to mention solved it. It costs me a long time to do the investigation. But after issue solved, everything suddenly became easy. Even though, there is still unclear field for me. I am sharing the thought and the process of the issue fixing, hope it helps if someone else meet the same error and correct me if I am wrong about anything.

First of all, big thanks to Tess, for her fantastic blog and reply for my help email.
Here is the Blog [https://blogs.msdn.microsoft.com/tess/](https://blogs.msdn.microsoft.com/tess/), anything you google about ASP.NET debug, hang, or performance issue, you may be led to this blog by almost all means.

A high CPU issue is generally one of 3 things:

1. An infinite loop
2. Too much load (i.e. too many requests doing lots of small things, so no one single thing is a culprit)
3. Too much churning in the Garbage Collector.

Keep these 3 things in mind and think deep. An infinite loop, check it not only in your code, but also the libraries you leverage. There is detailed discussion in Tess's blog about each scenario.
Read her posts, strong recommand.

Now I am coming back to talk about my problem: Our production web server meets 100 percent CPU usage from time to time, and each time, ops team fix it by resetting iis, then create dump file and performance log for dev team to analyze the root cause.

Basic information about the server, and our prod is built in .NET 4.5.
Operating System: Windows Server 2008 R2Service Pack 1
Number Of Processors: 24
Process ID: 4740
Process Image: c:\Windows\System32\inetsrv\w3wp.exe
System Up-Time: 10 day(s) 18:27:51
Process Up-Time: 3 day(s) 07:20:48
Processor Type: X64
Process Bitness: 64-Bit

Tools I use to do the trouble shooting are WinDBG (64bits), Debug dialog 2.0, and a little hang analyzer tool shared by Tess: [HangAnalyzer](https://blogs.msdn.microsoft.com/tess/2007/12/12/automated-net-hang-analysis/).

From the report of debug dialog tools, the dump was created when the process was in the middle of the garbage collection. 

We could confirm this with command **!threadpool** in windbg:

0:000> !threadpool


CPU utilization: 100%


Worker Thread: Total: 44 Running: 1 Idle: 23 MaxLimit: 32767 MinLimit: 24


Work Request in Queue: 0


Number of Timers: 2


Completion Port Thread:Total: 9 Free: 9 MaxFree: 48 CurrentLimit: 9 MaxLimit: 1000 MinLimit: 24


Make sure you get the right dump before starting! It would be better to get several right dumps.

When you get a high cpu usage issue, the first command you would type in is **!runaway**, and with this command, we got this:

0:000> !runaway


User Mode          Time


Thread             Time


62:1ef0         0 days 1:10:42.228


37:ca0          0 days 0:18:55.593


81:103c         0 days 0:17:50.728


67:187c         0 days 0:17:15.331


82:2118         0 days 0:15:31.622


84:2110         0 days 0:15:27.394


80:484          0 days 0:15:23.135


76:478          0 days 0:15:22.309


78:1684         0 days 0:15:22.168


86:20f4         0 days 0:15:13.448


88:20a8         0 days 0:15:06.740 


53:1a10         0 days 0:15:04.103   
......


119:2f28      0 days 0:00:01.778  


130:3a6c         0 days 0:00:01.747   
<p style="color:#d22323">109:3cbc       0 days 0:00:01.731  </p>
113:42e0         0 days 0:00:01.684


106:109c         0 days 0:00:01.669


131:3954         0 days 0:00:01.575


112:40fc         0 days 0:00:01.575


115:4af0         0 days 0:00:01.544


.......

We could see so many high cpu usage threads. Check the threads state and type with command **!threads**:

0:000> !threads


ThreadCount:      101


UnstartedThread:  0


BackgroundThread: 85


PendingThread:    0


DeadThread:       16


Hosted Runtime:   no


ID    OSID  ThreadOBJ 


State GC Mode  


GC Alloc Context  Domain


Lock CountApt Exception


26    1 1dfc 0000000006f10080    28220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn


57    2 17f8 0000000006fc6610    2b220 Preemptive  0000000380196A78:00000003801980D0 00000000036b6e50 0     MTA (Finalizer) 


59    5 1850 000000001ae68b80  102a220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker)


60    6 1e78 000000001bdc9dd0    21220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn


61    7  970 000000001c9c04a0  1020220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn (Threadpool Worker)


62   11 1ef0 000000001d8d1230  1029220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker) 


64   12 209c 000000001d8d3170  202b220 Preemptive  0000000240271B48:00000002402739B0 000000001af951c0 1     MTA 


65   43  cf4 000000001e211a10    21220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn 


.......


104   87 4208 0000000029994d50    20220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn


105  226 3268 0000000025fccb30  1029220 Preemptive  00000002016BD808:00000002016BEF18 00000000036b6e50 0     MTA (Threadpool Worker)


106  120 109c 0000000025fcd300  1029220 Preemptive  0000000741074138:0000000741075C10 00000000036b6e50 0     MTA (Threadpool Worker)


107   30 4ab8 000000001e186c70  1029220 Preemptive  0000000280B82560:0000000280B83698 00000000036b6e50 0     MTA (Threadpool Worker) 


108  246 4250 000000001e188bb0  1029220 Preemptive  00000005814BB7A8:00000005814BBAC8 00000000036b6e50 0     MTA (Threadpool Worker) 
<p style="color:#d22323">109   13 3cbc 000000001e185cd0  1029220 Cooperative 00000004807B6338:00000004807B7B90 000000001af951c0 1     MTA (Threadpool Worker)</p>
110   99 3d28 000000001e187440  1029220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker)


111  112 29a8 000000001e189380  1029220 Preemptive  00000005009385C8:0000000500938F48 00000000036b6e50 0     MTA (Threadpool Worker)


......


we found two interesting things through the above threads state:
1. Most of the high cpu usage threads are GC threads
2. Thread 109 whose GC mode is cooperative. 

GC mode **preemptive disabled** (or cooperative mode) is something really worth paying attention to.

Here is the difference of GC mode, Preemptive mode VS Cooperative mode:

<div style="background:antiquewhite">In cooperative models, once a thread is given control it continues to run until it explicitly yields control or it blocks.
In a preemptive model, the virtual machine is allowed to step in and hand control from one thread to another at any time.
Both models have their advantages and disadvantages.</div>


Back to our thread 109, check the clrstack:

clr!ObjectNative::Equals+f


System.Collections.Generic.ObjectEqualityComparer`1[[System.\__Canon, mscorlib]].IndexOf(System.\__Canon[], System.\__Canon, Int32, Int32)


System.Collections.Generic.List`1[[System.\__Canon, mscorlib]].IndexOf(System.\__Canon)


System.Collections.Generic.List`1[[System.\__Canon, mscorlib]].Remove(System.\__Canon)


......


Microsoft.Ajax.Utilities.JSParser.ParseLeftHandSideExpression(Boolean)


Microsoft.Ajax.Utilities.JSParser.ParseUnaryExpression(BooleanByRef, Boolean)


Microsoft.Ajax.Utilities.JSParser.ParseReturnStatement()


Microsoft.Ajax.Utilities.JSParser.ParseStatement(Boolean)


Microsoft.Ajax.Utilities.JSParser.ParseBlock(Microsoft.Ajax.Utilities.ContextByRef)


Microsoft.Ajax.Utilities.JSParser.ParseBlock()


......


Microsoft.Ajax.Utilities.JSParser.ParseStatements()


Microsoft.Ajax.Utilities.JSParser.Parse(Microsoft.Ajax.Utilities.CodeSettings)


StoShared.ShrinkWrap.Shrink.MinifyJavaScript(System.String)


StoShared.ShrinkWrap.Shrink.Wrap()
<p style="color:#d22323">WidgetWeb.Areas.V1.Controllers.CoreController.ShrinkWrap(System.String, StoShared.ShrinkWrap.ResourceType)</p>
<p style="color:#d22323">WidgetWeb.Areas.V1.Controllers.CoreController.Script(System.String)</p>
DynamicClass.lambda_method(System.Runtime.CompilerServices.Closure, System.Web.Mvc.ControllerBase, System.Object[])


......


clr!UnManagedPerAppDomainTPCount::DispatchWorkItem+11a


clr!ThreadpoolMgr::ExecuteWorkRequest+4c


clr!ThreadpoolMgr::WorkerThreadStart+f3


clr!Thread::intermediateThreadProc+7d


kernel32!BaseThreadInitThunk+d


ntdll!RtlUserThreadStart+1d



The colored line indicates the entry to production code which is responsible of minifying a javascript file.

Let's check out what is being minified and what happens during the process.
Search on the intenet, it seems like several people have gotten high cpu in the minification process.
Then we find the culprit: the library we used to do the minify, it's so out of date and has been officially announced that there was [infinite loop issue fixed when upgrade in 2012](http://ajaxmin1.rssing.com/chan-4185279/all_p2.html).
But unfortunally, we could not find the detailed information about the issue and how to reproduce it. We will deploy the fix code to production soon and I will come back to update.

Any complex issue, the debugging process is all about define the scope of the issue properly, pick up the right tools, and do it.