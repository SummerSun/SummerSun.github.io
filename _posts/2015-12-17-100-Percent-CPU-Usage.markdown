---
title:  "100 percent CPU Usage"
date:   2015-12-17 16:00:00
categories: work
---

Recently we meet a 100 percent CPU usage problem in production. I never met this kind of problem before, not to mention solved it. It costs me a long time to do the investigation. But after issue solved, everything suddenly became easy. Even though, there is still unclear field for me. I am sharing the thought and the process of the issue fixing, hope it helps if someone else meet the same error and correct me if I am wrong about anything.

First of all, big thanks to Tess, for her fantastic blog and reply for my help email.
Here is the Blog [https://blogs.msdn.microsoft.com/tess/](https://blogs.msdn.microsoft.com/tess/), anything you google about ASP.NET debug, hang, or performance issue, you may be led to this blog by almost all means.

A high CPU issue is generally one of 3 things:


- An infinite loop
- Too much load (i.e. too many requests doing lots of small things, so no one single thing is a culprit)
- Too much churning in the Garbage Collector.

Keep these 3 things in mind and think deep. An infinite loop, check it not only in your code, but also the libraries you leverage. There is detailed discussion in Tess's blog about each scenario.
Read her posts, strong recommand.

Now I am coming back to talk about my problem: Our production web server meets 100 percent CPU usage from time to time, and each time, ops team fix it by resetting iis, then create dump file and performance log for dev team to analyze the root cause.

Basic information about the server, and our prod is built in .NET 4.5.
<div class="windbg-results">
<p>Operating System: Windows Server 2008 R2Service Pack 1</p>
<p>Number Of Processors: 24</p>
<p>Process ID: 4740</p>
<p>Process Image: c:\Windows\System32\inetsrv\w3wp.exe</p>
<p>System Up-Time: 10 day(s) 18:27:51</p>
<p>Process Up-Time: 3 day(s) 07:20:48</p>
<p>Processor Type: X64</p>
<p>Process Bitness: 64-Bit</p>
</div>
Tools I use to do the trouble shooting are WinDBG (64bits), Debug dialog 2.0, and a little hang analyzer tool shared by Tess: [HangAnalyzer](https://blogs.msdn.microsoft.com/tess/2007/12/12/automated-net-hang-analysis/).

From the report of debug dialog tools, the dump was created when the process was in the middle of the garbage collection. 

We could confirm this with command **!threadpool** in windbg:
<div class="windbg-results">
<p>0:000> !threadpool</p>
<p>CPU utilization: 100%</p>
<p>Worker Thread: Total: 44 Running: 1 Idle: 23 MaxLimit: 32767 MinLimit: 24</p>
<p>Work Request in Queue: 0</p>
<p>Number of Timers: 2</p>
<p>Completion Port Thread:Total: 9 Free: 9 MaxFree: 48 CurrentLimit: 9 MaxLimit: 1000 MinLimit: 24</p>
</div>
Make sure you get the right dump before starting! It would be better to get several right dumps.

When you get a high cpu usage issue, the first command you would type in is **!runaway**, and with this command, we got this:

<div class="windbg-results">
<p>0:000> !runaway</p>
<p><span>User Mode</span>Time</p>
<p><span>Thread </span>         Time</p>
<p><span>62:1ef0</span>         0 days 1:10:42.228</P>
<p><span>37:ca0 </span>         0 days 0:18:55.593</P>
<p><span>81:103c</span>         0 days 0:17:50.728</P>
<p><span>67:187c </span>        0 days 0:17:15.331</P>
<p><span>82:2118 </span>        0 days 0:15:31.622</P>
<p><span>84:2110 </span>        0 days 0:15:27.394</P>
<p><span>80:484 </span>         0 days 0:15:23.135</P>
<p><span>76:478 </span>         0 days 0:15:22.309</P>
<p><span>78:1684</span>         0 days 0:15:22.168</P>
<p><span>86:20f4</span>         0 days 0:15:13.448</p>
<p><span>88:20a8 </span>        0 days 0:15:06.740</span>   </P>
<p><span>53:1a10</span>         0 days 0:15:04.103</span>   </P>
<p>......</p>
<p><span>119:2f28</span>      0 days 0:00:01.778</span>   </p>
<p><span>130:3a6c</span>         0 days 0:00:01.747   </p>
<p class="suspect"><span>109:3cbc    </span>   0 days 0:00:01.731  </p>
<p><span>113:42e0</span>         0 days 0:00:01.684</p>
<p><span>106:109c</span>         0 days 0:00:01.669</p>
<p><span>131:3954 </span>        0 days 0:00:01.575</p>
<p><span>112:40fc </span>        0 days 0:00:01.575</p>
<p><span>115:4af0 </span>        0 days 0:00:01.544</p>
<p>.......</p>
</div>
We could see so many high cpu usage threads. Check the threads state and type with command **!threads**:
<div class="windbg-results">
<p>0:000> !threads</p>
<p>ThreadCount:      101</p>
<p>UnstartedThread:  0</p>
<p>BackgroundThread: 85</p>
<p>PendingThread:    0</p>
<p>DeadThread:       16</p>
<p>Hosted Runtime:   no</p>

<span>ID</span>    OSID  ThreadOBJ<span> </span>
<span>State</span><span> GC Mode</span><span> </span><span> </span><span> </span><span>
GC Alloc Context</span><span> </span><span> </span><span>Domain</span>
<span>Lock Count</span><span>Apt Exception</span>
<p>26    1 1dfc 0000000006f10080    28220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn</p>
<p>57    2 17f8 0000000006fc6610    2b220 Preemptive  0000000380196A78:00000003801980D0 00000000036b6e50 0     MTA (Finalizer)</p> 
<p>59    5 1850 000000001ae68b80  102a220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker)</p> 
<p>60    6 1e78 000000001bdc9dd0    21220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn</p>
<p>61    7  970 000000001c9c04a0  1020220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn (Threadpool Worker)</p> 
<p>62   11 1ef0 000000001d8d1230  1029220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker)</p> 
<p>64   12 209c 000000001d8d3170  202b220 Preemptive  0000000240271B48:00000002402739B0 000000001af951c0 1     MTA</p> 
<p>65   43  cf4 000000001e211a10    21220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn</p> 
<p>.......</p>
<p>104   87 4208 0000000029994d50    20220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     Ukn</p>
<p>105  226 3268 0000000025fccb30  1029220 Preemptive  00000002016BD808:00000002016BEF18 00000000036b6e50 0     MTA (Threadpool Worker)</p>
<p>106  120 109c 0000000025fcd300  1029220 Preemptive  0000000741074138:0000000741075C10 00000000036b6e50 0     MTA (Threadpool Worker) </p>
<p>107   30 4ab8 000000001e186c70  1029220 Preemptive  0000000280B82560:0000000280B83698 00000000036b6e50 0     MTA (Threadpool Worker)</p> 
<p>108  246 4250 000000001e188bb0  1029220 Preemptive  00000005814BB7A8:00000005814BBAC8 00000000036b6e50 0     MTA (Threadpool Worker)</p> 
<p class="suspect">109   13 3cbc 000000001e185cd0  1029220 Cooperative 00000004807B6338:00000004807B7B90 000000001af951c0 1     MTA (Threadpool Worker)</p>
<p>110   99 3d28 000000001e187440  1029220 Preemptive  0000000000000000:0000000000000000 00000000036b6e50 0     MTA (Threadpool Worker)</p>
<p>111  112 29a8 000000001e189380  1029220 Preemptive  00000005009385C8:0000000500938F48 00000000036b6e50 0     MTA (Threadpool Worker)</p>
<p>......</p>
</div> 

we found two interesting things through the above threads state:

1. Most of the high cpu usage threads are GC threads
2. Thread 109 whose GC mode is cooperative. 

GC mode **preemptive disabled** (or cooperative mode) is something really worth paying attention to.

Here is the difference of GC mode, Preemptive mode VS Cooperative mode:

>In cooperative models, once a thread is given control it continues to run until it explicitly yields control or it blocks.
>In a preemptive model, the virtual machine is allowed to step in and hand control from one thread to another at any time.
>Both models have their advantages and disadvantages.

Back to our thread 109, check the clrstack:
<div class="windbg-results">
<p>clr!ObjectNative::Equals+f</p>
<p>System.Collections.Generic.ObjectEqualityComparer`1[[System.\__Canon, mscorlib]].IndexOf(System.\__Canon[], System.\__Canon, Int32, Int32)</p>
<p>System.Collections.Generic.List`1[[System.\__Canon, mscorlib]].IndexOf(System.\__Canon)</p>
<p>System.Collections.Generic.List`1[[System.\__Canon, mscorlib]].Remove(System.\__Canon)</p>
<p>......</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseLeftHandSideExpression(Boolean)</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseUnaryExpression(BooleanByRef, Boolean)</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseReturnStatement()</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseStatement(Boolean)</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseBlock(Microsoft.Ajax.Utilities.ContextByRef)</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseBlock()</p>
<p>......</p>
<p>Microsoft.Ajax.Utilities.JSParser.ParseStatements()</p>
<p>Microsoft.Ajax.Utilities.JSParser.Parse(Microsoft.Ajax.Utilities.CodeSettings)</p>
<p>StoShared.ShrinkWrap.Shrink.MinifyJavaScript(System.String)</p>
<p>StoShared.ShrinkWrap.Shrink.Wrap()</p>
<p class="suspect">WidgetWeb.Areas.V1.Controllers.CoreController.ShrinkWrap(System.String, StoShared.ShrinkWrap.ResourceType)</p>
<p class="suspect">WidgetWeb.Areas.V1.Controllers.CoreController.Script(System.String)</p>
<p>DynamicClass.lambda_method(System.Runtime.CompilerServices.Closure, System.Web.Mvc.ControllerBase, System.Object[])</p>
<p>......</p>
<p>clr!UnManagedPerAppDomainTPCount::DispatchWorkItem+11a</p>
<p>clr!ThreadpoolMgr::ExecuteWorkRequest+4c</p>
<p>clr!ThreadpoolMgr::WorkerThreadStart+f3</p>
<p>clr!Thread::intermediateThreadProc+7d</p>
<p>kernel32!BaseThreadInitThunk+d</p>
<p>ntdll!RtlUserThreadStart+1d</p>
</div>

The colored line indicates the entry to production code which is responsible of minifying a javascript file.

Let's check out what is being minified and what happens during the process.

Search on the intenet, it seems like several people have gotten high cpu in the minification process.
Then we find the culprit: the library we used to do the minify, it's so out of date and has been officially announced that there was [infinite loop issue fixed when upgrade in 2012](http://ajaxmin1.rssing.com/chan-4185279/all_p2.html).
But unfortunally, we could not find the detailed information about the issue and how to reproduce it. We will deploy the fix code to production soon and I will come back to update.

Any complex issue, the debugging process is all about define the scope of the issue properly, pick up the right tools, and do it.