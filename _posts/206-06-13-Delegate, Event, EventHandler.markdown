---
layout: post
title: C# Delegate, Event and EventHandler
date: 2016-06-13 18:21:28 +08:00
tags: [PluralSight]
image:
  feature: abstract.jpg
---
Being a .NET developer, one must have used delegates and eventhanders.
But sometimes in an unconcious way. I have decided several times to make clear those concepts, but gave up for all kinds of reasons.
With the help of [Pluralsight](https://www.pluralsight.com), by the way, it is fantastic, I finally did it.
Blogs tagged **PluralSight** summarize my learning via this tool. MSDN helps a lot too though the website is a little unfancy.
Thanks to [MSDN](https://msdn.microsoft.com/en-us/) and the pluralsight [video](https://app.pluralsight.com/library/courses/csharp-events-delegates/table-of-contents).
All credits go to them.

This blog covers the concepts of delegates, event handers, events, and event args.
I use lots of code snippets to demonstrate how to create delegates,
how to create event handlers, how to connect events and eventhandler via delegates,
the build-in delegates in c# and the asynchronous nature of delegates, and finally, the way to make thread-safe call to windows form controls when using delegates in asychronous way. 
Hope this post is helpful. And correct me if anything wrong.
<!--break-->
## Overview
First of all, let's use the image in the video to see a general overview:

![](http://7xq1tb.com1.z0.glb.clouddn.com/Event_Delegate_EventHandler.JPG)

## Summaries
Put the summaries in the beginning maybe a little weired, but I think it's important.
Have these summaries in mind first, then verify it after finish the post.

1. A delegate is a specialized class based on a MulticaseDelegate class
2. Delegates act as the glue/pipeline between events and event handlers
3. Events provide notifications and send data using EventArgs
2. Event Handlers receive and process EventArgs data from a delegate
5. Objects that raise events don't need to explicitly know the object that will handle the event
6. Without delegates, event won't be useful at all
7. There are several built-in delegates in .NET:
   - Action / Action\<T>: accept one or more parameters and return no value, one-way delegate
   - Func / Func<T, TResult>: accept one or more parameters and return a value of type TResult
8. Lambdas provide a way to define inline method using a concise syntax
9. Events and delegates could be used in many scenarios:
   - Makes different components communicate in a loosely coupled way
   - To start asynchronous processes
   - To handle backgroundworker functionality
   - To start threads
   - ...

## Create delegates
There are four ways of creating delegates:

- Definition
{% highlight c# %}
class Program
{
    public delegate void MyDel(string message);
    public static void DelegateMethond(string message)
    {
        // cw+Tab shortcut to input, new skill
        Console.WriteLine(message);
    }
    public MyDel my_del = new MyDel(DelegateMethond);
}
{% endhighlight  %}
- Delegate Inference
{% highlight c# %}
class Program
{
    public delegate void MyDel(string message);
    public MyDel my_del = DelegateMethod;
    public static void DelegateMethod(string message)
    {
        Console.WriteLine(message);
    }
}
{% endhighlight %}
- Anonymous Method
{% highlight c# %}
class Program
{
    public delegate void MyDel(string message);
    public MyDel my_del = delegate(string message)
    {
        Console.WriteLine(message);
    };
}
{% endhighlight %}
- Lambda Expression
{% highlight c# %}
class Program
{
    public delegate void MyDel(string message);
    public MyDel my_del = (m) => Console.WriteLine(m);
}
{% endhighlight %}

The four ways of creating delegates involve several .NET features/concepts: delegate inference, anonymous method and lambda expression.

### Asychronous nature of delegates
>> Delegates enable you to call a synchronous method in an asynchronous manner.
When you call a delegate synchronously, the Invoke method calls the target method directly on the current thread.
If the BeginInvoke method is called, the CLR queues the request and returns immediately to the caller.
The target method is called asynchronously on a thread from the thread pool.
The original thread, which submitted the request, is free to continue executing in parallel with the target method.

>> If a callback method has been specified in the call to the BeginInvoke method, the callback method is called when the target method ends.
In the callback method, the EndInvoke method obtains the return value and any input/output or output-only parameters.
If no callback method is specified when calling BeginInvoke, EndInvoke can be called from the thread that called BeginInvoke.

The above description from MSDN explains everything so clear.

Here's an example:
{% highlight c# %}
class Program
{
    public delegate string MyDel(string message);
    public static MyDel my_del = (m) => { Thread.Sleep(5000); return m + " world"; };
    
    static void call_back(IAsyncResult asyncResult)
    {
        var result = my_del.EndInvoke(asyncResult);
        Console.WriteLine(result);
    }
    
    static void Main(string[] args)
    {
        my_del.BeginInvoke("Hello", call_back, null);
        Console.WriteLine("Do whatever u want ...");
        Console.ReadLine();
    }
}
{% endhighlight%}

### Build-in delegates of .NET
There are two major built-in delegate types in C#.
Action is a delegate type that accepts one to many parameters and returns no value.
Func is a delegate type that accespts one to many parameters and return a value.
Both could used as parameters to help in de-coupling.

Action Example:
{% highlight c# %}
Action<int, int> a1 = (x, y) => x + y;
Console.WriteLine(a1(2, 3)); // output 5
{% endhighlight %}

Func Example:
{% highlight c# %}
Func<int, int, string> f = (x, y) => (x * y).ToString();
var result = f(2, 3); // result is "6"
{% endhighlight %}

## EventHandlers
EventHandler represents the method that will handle an event that has no event data.
EventHandlers\<TEventArgs> represents the method that will handle an event when the event provides data.

The event model in the .NET Framework is based on having an event delegate that connects an event with its handler.
To raise an event, two elements are needed:

- A delegate that refers to a method that provides the response to the event.
- Optionally, a class that holds the event data, if the event provides data.
If the event does not generate event data, the second parameter is simply the value of the EventArgs.Empty field.
Otherwise, the second parameter is a type derived from EventArgs and supplies any fields or properties needed to hold the event data.

We could define an event handler in two ways:
{% highlight c# %}
public event EventHandler<WorkPerformedEventArgs> workPerformed;
{% endhighlight %}

{% highlight c# %}
public delegate void WhatEverDelegateName(object sender, WorkPerformedEventArgs e);
public WhatEverDelegateName workPerformed;
{% endhighlight %}

Then we could use a delegate to connect the event and the event handler.

{% highlight c# %}
workPerformed += (s, e) => Console.WriteLine("Do Work: " + e.Hours + " " + e.WorkType);
{% endhighlight %}
There is an useful and safe built-in feature for EventHandler: a public eventhandler must be invoked by the method which is defined in the same class.
Otherwise it will cause the compile failure.

For the EventHandler definition, there is a little confusing about with or without the keyword 'event'.
I will not cover this here cause the aurthor of <<C# in Depth>> explains very well in this [article](http://csharpindepth.com/Articles/Chapter2/Events.aspx).

## Thread-safe asynchronous with delegates
Delegates enable you to call a synchronous method in an asynchronous manner.
But one must to make sure to make calls to a control in a thread-safe way when using it in the winform applciations.
It is unsafe to call a control from the a thread other that the one that created the control.

Unsafe call example, we will get 'InvalidOperationExcaption' with additional information: Cross-thread operation not valid when click button.
{% highlight c#%}
public partial class Form1 : Form
{
    public Form1()
    {
        InitializeComponent();
    }
    private void Button_Click(object sender, EventArgs e)
    {
        var del = new SetTextDelegate(SetText);
        del.BeginInvoke("This text was set unsafely", null, null);
    }
    private void SetText()
    {
        this.TextBox.Text = "This text was set unsafely.";
    }
}
{% endhighlight %}

Control's InvokeRequired property could solve the problem and achieve to make a thread-safe call.
We just need to update the SetText method above as following:
{% highlight c# %}
private void SetText(string message)
{
    // call Invoke with a delegate that makes the actual call to the control
    // otherwise call the control directly
    if(this.TextBox.InvokeRequired)
    {
        var self_del = new SetTextDelegate(SetText);
        this.BeginInvoke(self_del, new object[] { message });
    }
    else
    {
        this.TextBox.Text= "This text was set safely.";
    }
}
{% endhighlight %}

Another preferred way to perform asynchronous operations is to use the BackgroundWorker component.

{% highlight c# %}
public partial class Form1 : Form
{
    public BackgroundWorker backgroundWorker = new BackgroundWorker();
    
    public Form1()
    {
        InitializeComponent();
        InitBackgroundWorker();
    }
    public void InitBackgroundWorker()
    {
        // pretend to do some work
        this.backgroundWorker.DoWork += (s, e) => { Thread.Sleep(2000); };

        // You could also subscribe to the event by creating a DoWorkEventHander instance with a method which accepts the correct parameters
        // this.backgroundWorker.DoWork += new DoWorkEventHandler(Dowork);
        
        this.backgroundWorker.RunWorkerCompleted += (s, e) =>
        {
            this.TextBox.Text = "This text was set safely by BackgroundWorker.";
        };
    }
    
    public void Button_Click(object sender, EventArgs e)
    {
        this.backgroundWorker.RunWorkerAsync();
    } 
}
{% endhighlight %}

BackgroundWorker has some other features such as calcelation and reporting progress which are not covered in this topic.
I want to concentrate on how to make thread-safe call with background worker rather than its powerful features.

Now we have went through the concepts and usages of delegates and event handlers.
They are used to solve many programming problems and there even a pattern about them: Event-based Asynchronous Pattern.
Some people like to mapping the c# delegates to c++ function pointers. They are alike. But they also have important differences. c# delegates are safe-type.

Spent much time on this blog, reading, texting and preparing the code snipptes. But sure there must be something missing.
You could google if interested or you need them. And I will keep updating the post once I find anything important.