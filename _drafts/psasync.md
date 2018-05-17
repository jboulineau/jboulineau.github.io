The series I did on Concurrency in PowerShell contains by far the most popular posts on my blog (getting tens of views every day!), particularly the one on using runspaces. The code demonstrated in the post was an early attempt at using the method. Not long after I refactored the functions I was using into something more modular and flexible. Thus, the psasync module was born. It has proved to be a very simple way for me to kick off background work in runspaces in scripts or from the shell. It still has room for improvement (the functions don’t process pipeline input, for instance), but it’s a start. Since I hope to be updating it as suggestions come in or as I find ways to make it more robust, I’ve started a Codeplex project (my first, so be gentle). If you would like to be a contributor feel free to send me something and, if it’s good, I’ll give you credit on the project.

The module contains the following functions:
Get-RunspacePool

This should have, perhaps, been named Create-RunspacePool. As its name suggests, it returns a pool of runspaces that can be used (and reused).
Invoke-Async

This is a much improved version of the function introduced in the initial post. The big improvement was the addition of the AsyncPipeline class definition, which allows the creation of a simple object to keep track of both the pipeline and the AsyncResult handle which is returned by BeginInvoke(). This allows the process of looking at statuses of running processes and consuming results to be much simpler. The function also allows passing an array of parameters for script blocks with multiple arguments.
Receive-AsyncResults

This function wraps the code for pulling results (or errors, as the case may be) off the pipelines in the runspace pool utilizing the AsyncPipeline objects output from Invoke-Async.
Receive-AsyncStatus

A handy function for working with runspaces from the shell, Receive-AsyncStatus simply returns information about the status of the operations running in the pipelines you have invoked. Since Receive-AsyncResults is synchronous, this allows you to continue to work until your last process completes or selectively use Receive-AsyncResults on those that have completed.
Example Code

To demonstrate the use of the module, consider the same scenario presented in the Concurrency series: You have a series of Excel documents that you need to load into an SQL Server database. As before, set up the script block that will execute the real work.

<code>
Import-Module psasync

$AsyncPipelines = @()

$ScriptBlock = `
{
    Param($File)
    
    . <your_path>\Import-ExcelToSQLServer.ps1
    
    Import-ExcelToSQLServer -ServerName 'localhost' -DatabaseName 'SQLSaturday' -SheetName "SQLSaturday_1" `
        -TableName $($File.BaseName) -FilePath $($File.FullName)
}

# Create a pool of 3 runspaces
$pool = Get-RunspacePool 3

$files = Get-ChildItem <path-to-files> 

foreach($file in $files)
{
	 $AsyncPipelines += Invoke-Async -RunspacePool $pool -ScriptBlock $ScriptBlock -Parameters $file
}

Receive-AsyncStatus -Pipelines $AsyncPipelines

Receive-AsyncResults -Pipelines $AsyncPipelines -ShowProgress
</code>
You’ll notice there is nothing particularly complex here in the code. But, all the warnings from the runspace post apply. Multi-threading is awesome and powerful, but use it with care. 


---

 In my last post I looked at using background jobs to execute PowerShell code concurrently, concluding that for many tasks the large amount of overhead makes this method counter productive. Fortunately, there is a better way. What I present here is an expansion on the work of Oisin Grehan (B | T), who deserves the credit for introducing this method. This blog post introduced me to the concepts upon which I expound in this post.

A quick note to begin. I am presenting something that is not well documented and outside the ‘normal’ operations of PowerShell. I don’t think PowerShell was designed to be used in this way, as evidenced by the lack of thread safety in cmdlets and no native synchronization mechanisms (that I can find). I’d love it if someone reading this blog can provide more color around PowerShell’s philosophy of multi-threading, but judging by the built in mechanisms (jobs) the designers wanted to avoid the issue, and for good reasons! Use this at your own risk and only after testing EXTENSIVELY.

To review; the test scenario for this series involves a series of Excel documents that must be loaded into an SQL Server database. The goal is to speed up the process by loading more than one file at a time. So, I need to gather a collection of the file objects and execute a PowerShell script block to execute the ETL (Extraction Transform Load) code against each file. As you can see, this is very simple code, but it must be executed many times … the ideal (but not only) use case for this pattern.
1
2
3
4
5
6
7
8
9
	
$ScriptBlock = `
{
    Param($File)
     
    . <your_path>\Import-ExcelToSQLServer.ps1
     
    Import-ExcelToSQLServer -ServerName 'localhost' -DatabaseName 'SQLSaturday' -SheetName "SQLSaturday_1" `
        -TableName $($File.BaseName) -FilePath $($File.FullName)
}

What I need is some way for PowerShell to act as a dispatcher to generate other threads on which these import processes can operate. The key elements to this are the RunspacePool and PowerShell classes in the System.Management.Automation namespace. These are classes meant to enable applications to utilize PowerShell processes, but I am using it for a different purpose. Yep, it’s about to get very developery on this blog. But, have no fear non-developers (like my fellow DBA’s) I’m working on making this easier for you.

Every PowerShell pipeline, defined by the PowerShell class, must have an environment of resources and session constructs (environment variables, loaded cmdlets, etc.) in which to run. In other words, every pipeline needs a runspace. A pipeline can only exist on one runspace. However, pipelines can also be queued onto runspace pools. It is this ability to create runspace pools that allows for (albeit clumsy) multi-threading capabilities.

RunspacePools are created through the CreateRunspacePool static method of the RunspaceFactory class. This method has 9 overloads, so there’s plenty of options to explore. The simplest method is:
1
	
$pool = [RunspaceFactory]::CreateRunspacePool(1, 3)

This simple line of code creates a pool of 3 runspaces upon which pipelines can run. You can do a lot more with runspace pools, such as establishing session state configurations that can be shared by all the runspaces in the pool. This is handy for, say, loading specific modules or establishing shared variables. But, it is beyond the scope of this post. Choosing the size of your runspace pool is very important. Too many and you will find diminishing (or worse) performance. Too few and you will not reap the full benefits. This is a decision that must be made per computer and per workload. More on this later.

Part of the configuration of the runspace pool is the apartment state value. With this code, I specify that all of the runspaces will be in single-threaded apartments and then open the pool for use.
1
2
	
$pool.ApartmentState = "STA"
$pool.Open()

Apartment states are very complicated topics and I’m not going to attempt to describe them here. I will only say that this is an attempt to force thread synchronization of COM objects. You also need to be aware of these since certain code will only work in a multi-threaded or single-threaded apartment. You should also be aware of what your IDE uses. For instance, the ISE uses STA, while the shell itself (in v2) is MTA. This can be confusing! Since this is a COM mechanism that doesn’t really ‘exist’ in Windows per say, it is not sufficient to solve your thread safety concerns. But, it is my attempt to provide what automatic synchronization I can. With that, a quick word on thread safety.

If you compare the script block above with the script block from my post on background jobs, you will see that the properties of the file objects are quite different. This is because the RunspacePool method does *not* serialize / de-serialize objects, but passes the objects to the runspaces by reference. This means that an object on thread B that was created by thread A points to precisely the same memory location. So, if thread B calls a method of the object at the same time thread A is executing code in the same method, thread B’s call could be making modifications to local variables within the method’s memory space that change the outcome of, or break, thread A’s execution and vice versa. This is generally considered to be a bad thing. Be careful with this. You should take care in your code to ensure that the same object cannot be passed to more than one thread. Again, use at your own risk.

At this point, I can begin creating pipelines and assigning them to the runspace pool. In the code download you will see that I run this in a loop to add a script block for every file to the pool, but I’m keeping it simple here. There are a few other bits in the sample code that I don’t expound on in this post, too.
1
2
3
	
$pipeline  = [System.Management.Automation.PowerShell]::create()
$pipeline.RunspacePool = $pool
$pipeline.AddScript($ScriptBlock).AddArgument($File)

Here the PowerShell pipeline object is captured and then assigned to the previously created run pool. The script is then added to the pipeline and a file object is passed as an argument. (Note that you can pass n parameters to a script block by appending additional AddArgument() calls. You can also queue n scripts or commands to a pipeline and they will be executed synchronously within the runspace.) The script is not executed immediately. Rather, two methods exist that cause the pipeline to begin executing. The Invoke() method is the synchronous version, which causes the dispatching thread to wait on the pipeline contents to process and return. BeginInvoke() is the asynchronous method that allows for the pipeline to be started and control returned to the dispatching thread.
1
	
$AsyncHandle = $pipeline.BeginInvoke()

BeginInvoke() returns an asynchronous handle object the properties of which include useful information such as the execution state of the pipeline. It’s also how you are able to hook into the pipeline at the appropriate time. To do so, the EndInvoke() method is used. EndInvoke() accepts the handle as it’s argument and will wait for the pipeline to complete before returning whatever contents (errors, objects, etc.) that were generated. In this code sample, the results are simply returned to the host pipeline. Also note that since the PowerShell class is un-managed code, calling Dispose() is wise. Otherwise, garbage collection will not release the memory grants and your powershell.exe process will be bloated until such time as the object is disposed or the process is closed (just for fun you can test this using [GC]::Collect()). Closing the RunspacePool is also good practice.
1
2
3
	
$pipeline.EndInvoke($AsyncHandle)
$pipeline.Dispose()
$pool.Close()
Notes on usage

You shouldn’t use this method for every task and when you do every decision should be carefully considered. Take the size of your runspace pool, for instance. Think carefully about how and where your code will be executed. Where are the bottlenecks? Where is the resource usage occuring? And, of course, how many CPU cores are on the machine where the code will be executed (both host machine and remote)?

For example, I have used this method to perform index maintenance on SQL Servers. But, consider all of the pieces. If you didn’t know that index rebuilds (but not reorgs!) could be multi-threaded by SQL Server, you could get into some trouble. I came across a database tool that professes to multi-thread index rebuilds, but it’s method is to simply calculate the number of cores available to the server and kick off that number of rebuilds. Ignoring for a moment that you have not left any processors for Windows to use, you’ve also not considered the operations of the index rebuilds themselves. If the max degree of parallelism setting is 0 on the index definition (or any number other than 1), you could be looking at serious resource conflict. Imagine an 8 core server. That’s potentially 64 simultaneous threads! It will work, but the scheduler yields, CPU cache thrashing, context changes, ( cross-NUMA node access costs?) may have serious impact to your system.

So, be careful and think through the impact of the decisions you make when using this method.