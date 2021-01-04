---
layout: post
title: 跨年期间的诡异Bug
---

十二月三十一号我忙了一整天，准备各种美味佳肴，各种酒，邀请朋友们来家里跨年。正当我们收拾妥当准备推杯换盏的时候，我老板打电话来了。可能没有比这个更扫兴的事情了吧。

通常外资企业，老板是不会轻易给你打电话的，除非出了线上问题，俗称 LSI：Live site issue。问题很简单，香港客户打新，下单成功，但是在平台上看不到历史订单。
首先，这样的问题之前没出现过，Duty Watcher看来看去也看不出什么明显的问题，只能找最后是谁check-in了什么，可是最后check-in的代码他们也看不明白，逻辑绕来绕去怎么看都像错的。当然关于这段代码，可以再讲另外一个有趣的故事。

接通电话会议，就看到一群大佬正在共享桌面看我之前check-in的代码。我只能强装镇定和duty watcher确认问题和现象，确认完我觉得99.99% 不是我的锅。
可是他们不相信，也找不到其他可疑的点，于是大家一起扯皮了半小时，他们执意要roll back，我只能表示 you go ahead。
接下来的follow-up邮件，果然不出我所料，roll back也没能解决问题，只能继续trouble-shoot，谁知跨年结束，到了一月一号，就好了，the issue is gone。

一号下午，印度同事上线开始检查这个问题，终于找到了真凶。今天节后第一天上班，我也把那段代码打开来重新看了下，发现特别有趣。
Scenario 非常简单，为了给客户显示历史订单，platform -> OAPI -> Cache Service。Cache Service没有问题，Cache Service收到的request有点问题。
31号那天，OAPI发来的request竟然指定FromDate=2020/9/30，ToDate=2020/12/30。所以31号的打新订单没有显示。OAPI的逻辑是这样的：

```C#
// Logic in Controller
if (string.IsNullOrEmpty(request.OrderId) && !request.FromDateTime.HasValue && !request.ToDateTime.HasValue)
{
  request.FromDateTime = UtcDateTime.Now.AddMonths(-3);
}

// Logic in Service
if (request.ToDateTime.HasValue)
{
  orderActivitySearchRequest.ToDate = request.ToDateTime.Value.ToDateTime();
}
else
{
  orderActivitySearchRequest.ToDate = orderActivitySearchRequest.FromDate.AddMonths(3);
}
```

FromDate = 31-Dec-2020 – 3 Months = 30 September 2020

ToDate = 30-Sep-2020 + 3 Months = 30 December 2020

OAPI这样一操作，完美的避开了12/31/2020。按照这个逻辑，2021年还有几次机会出现这个问题，5/29/2021, 5/30/2021, 5/31/2021, 7/31/2021 和 12/31/2021。

我怀疑是不是印度同事准备的跨年彩蛋。
