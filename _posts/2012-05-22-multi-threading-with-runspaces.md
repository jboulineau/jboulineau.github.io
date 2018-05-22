---
layout: post
title: "Multi-Threading with Runspaces"
date: 2012-05-22
series: "Concurrency in PowerShell"
categories: "PowerShell"
excerpt: "There are multiple methods to achieve concurrency in PowerShell. This post covers Multi-Threading with Runspaces"
---

Download the code for this post <a href="/assets/code/runspaces.zip">here</a>.

In my last post I looked at using background jobs to execute PowerShell code concurrently, concluding that for many tasks the large amount of overhead makes this method counter productive. Fortunately, there is a better way. What I present here is an expansion on the work of Oisin Grehan ( [B](http://www.nivot.org/) - [T](http://www.twitter.com/oising)), who deserves the credit for introducing this method. This blog post introduced me to the concepts upon which I expound in [this post](http://www.nivot.org/nivot2/post/2009/01/22/CTP3TheRunspaceFactoryAndPowerShellAccelerators.aspx).

A quick note to begin. I am presenting something that is not well documented and outside the ‘normal’ operations of PowerShell. I don’t think PowerShell was designed to be used in this way, as evidenced by the lack of thread safety in cmdlets and no native synchronization mechanisms (that I can find). I’d love it if someone reading this blog can provide more color around PowerShell’s philosophy of multi-threading, but judging by the built in mechanisms (jobs) the designers wanted to avoid the issue, and for good reasons! Use this at your own risk and only after testing EXTENSIVELY.

To review; the test scenario for this series involves a series of Excel documents that must be loaded into an SQL Server database. The goal is to speed up the process by loading more than one file at a time. So, I need to gather a collection of the file objects and execute a PowerShell script block to execute the ETL (Extraction Transform Load) code against each file. As you can see, this is very simple code, but it must be executed many times … the ideal (but not only) use case for this pattern.

``` PowerShell
$ScriptBlock = `
{
    Param($File)
     
    . <your_path>\Import-ExcelToSQLServer.ps1
     
    Import-ExcelToSQLServer -ServerName 'localhost' -DatabaseName 'SQLSaturday' -SheetName "SQLSaturday_1" `
        -TableName $($File.BaseName) -FilePath $($File.FullName)
}
```

What I need is some way for PowerShell to act as a dispatcher to generate other threads on which these import processes can operate. The key elements to this are the RunspacePool and PowerShell classes in the [System.Management.Automation](http://msdn.microsoft.com/en-us/library/system.management.automation(v=vs.85).aspx) namespace. These are classes meant to enable applications to utilize PowerShell processes, but I am using it for a different purpose. Yep, it’s about to get very developery on this blog. But, have no fear non-developers (like my fellow DBA’s) I’m working on making this easier for you.

Every PowerShell pipeline, defined by the PowerShell class, must have an environment of resources and session constructs (environment variables, loaded cmdlets, etc.) in which to run. In other words, every pipeline needs a runspace. A pipeline can only exist on one runspace. However, pipelines can also be queued onto runspace pools. It is this ability to create runspace pools that allows for (albeit clumsy) multi-threading capabilities.

RunspacePools are created through the CreateRunspacePool static method of the [RunspaceFactory](http://msdn.microsoft.com/en-us/library/system.management.automation.runspaces.runspacefactory(v=vs.85).aspx) class. This method has 9 overloads, so there’s plenty of options to explore. The simplest method is:

``` PowerShell	
$pool = [RunspaceFactory]::CreateRunspacePool(1, 3)
```

This simple line of code creates a pool of 3 runspaces upon which pipelines can run. You can do a lot more with runspace pools, such as establishing session state configurations that can be shared by all the runspaces in the pool. This is handy for, say, loading specific modules or establishing shared variables. But, it is beyond the scope of this post. Choosing the size of your runspace pool is very important. Too many and you will find diminishing (or worse) performance. Too few and you will not reap the full benefits. This is a decision that must be made per computer and per workload. More on this later.

Part of the configuration of the runspace pool is the apartment state value. With this code, I specify that all of the runspaces will be in single-threaded apartments and then open the pool for use.

``` PowerShell
$pool.ApartmentState = "STA"
$pool.Open()
```
Apartment states are very complicated topics and I’m not going to attempt to describe them here. I will only say that this is an attempt to force thread synchronization of COM objects. You also need to be aware of these since certain code will only work in a multi-threaded or single-threaded apartment. You should also be aware of what your IDE uses. For instance, the ISE uses STA, while the shell itself (in v2) is MTA. This can be confusing! Since this is a COM mechanism that doesn’t really ‘exist’ in Windows per say, it is not sufficient to solve your thread safety concerns. But, it is my attempt to provide what automatic synchronization I can. With that, a quick word on thread safety.

If you compare the script block above with the script block from my post on background jobs, you will see that the properties of the file objects are quite different. This is because the RunspacePool method does *not* serialize / deserialize objects, but passes the objects to the runspaces by reference. This means that an object on thread B that was created by thread A points to precisely the same memory location. So, if thread B calls a method of the object at the same time thread A is executing code in the same method, thread B’s call could be making modifications to local variables within the method’s memory space that change the outcome of, or break, thread A’s execution and vice versa. This is generally considered to be a bad thing. Be careful with this. You should take care in your code to ensure that the same object cannot be passed to more than one thread. Again, use at your own risk.

At this point, I can begin creating pipelines and assigning them to the runspace pool. In the code download you will see that I run this in a loop to add a script block for every file to the pool, but I’m keeping it simple here. There are a few other bits in the sample code that I don’t expound on in this post, too.

``` PowerShell
$pipeline  = [System.Management.Automation.PowerShell]::create()
$pipeline.RunspacePool = $pool
$pipeline.AddScript($ScriptBlock).AddArgument($File)
```
Here the PowerShell pipeline object is captured and then assigned to the previously created run pool. The script is then added to the pipeline and a file object is passed as an argument. (Note that you can pass n parameters to a script block by appending additional AddArgument() calls. You can also queue n scripts or commands to a pipeline and they will be executed syncronously within the runspace.) The script is not executed immediately. Rather, two methods exist that cause the pipeline to begin executing. The Invoke() method is the synchronous version, which causes the dispatching thread to wait on the pipeline contents to process and return. BeginInvoke() is the asynchronous method that allows for the pipeline to be started and control returned to the dispatching thread.

``` PowerShell	
$AsyncHandle = $pipeline.BeginInvoke()
```

BeginInvoke() returns an asynchronous handle object the properties of which include useful information such as the execution state of the pipeline. It’s also how you are able to hook into the pipeline at the appropriate time. To do so, the EndInvoke() method is used. EndInvoke() accepts the handle as it’s argument and will wait for the pipeline to complete before returning whatever contents (errors, objects, etc.) that were generated. In this code sample, the results are simply returned to the host pipeline. Also note that since the PowerShell class is unmanaged code, calling Dispose() is wise. Otherwise, garbage collection will not release the memory grants and your powershell.exe process will be bloated until such time as the object is disposed or the process is closed (just for fun you can test this using [GC]::Collect()). Closing the RunspacePool is also good practice.

``` PowerShell	
$pipeline.EndInvoke($AsyncHandle)
$pipeline.Dispose()
$pool.Close()
```
## Notes on usage

You shouldn’t use this method for every task and when you do every decision should be carefully considered. Take the size of your runspace pool, for instance. Think carefully about how and where your code will be executed. Where are the bottlenecks? Where is the resource usage occuring? And, of course, how many CPU cores are on the machine where the code will be executed (both host machine and remote)?

For example, I have used this method to perform index maintenance on SQL Servers. But, consider all of the pieces. If you didn’t know that index rebuilds (but not reorgs!) could be multi-threaded by SQL Server, you could get into some trouble. I came across a database tool that professes to multi-thread index rebuilds, but it’s method is to simply calculate the number of cores available to the server and kick off that number of rebuilds. Ignoring for a moment that you have not left any processors for Windows to use, you’ve also not considered the operations of the index rebuilds themselves. If the max degree of parallelism setting is 0 on the index definition (or any number other than 1), you could be looking at serious resource conflict. Imagine an 8 core server. That’s potentially 64 simultaneous threads! It will work, but the scheduler yields, CPU cache thrashing, context changes, ( [cross-NUMA node access](http://sqlblog.com/blogs/linchi_shea/archive/2012/01/30/performance-impact-the-cost-of-numa-remote-memory-access.aspx) costs?) may have serious impact to your system.

So, be careful and think through the impact of the decisions you make when using this method.