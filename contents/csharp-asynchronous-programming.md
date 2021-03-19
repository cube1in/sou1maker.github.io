> 本文是抄记于B站杨旭老师的《C# 异步编程》视频的笔记。
>
> 老师的详细视频讲解请移步B站观看：https://www.bilibili.com/video/BV1Zf4y117fs



### 一、线程(Thread)：创建线程

#### 1. 什么是线程 Thread

线程是一个可执行路径，它可以独立于其它线程执行。
每个线程都在操作系统的进程(Process)内执行，而操作系统进程提供了程序运行的独立环境。
有了这两个概念之后，就可以将应用程序简单的分为两类：

1. 单线程应用，在进程的独立环境里只跑一个线程，所以该线程拥有独立权。
2. 多线程应用，单个进程中会跑多个线程，它们共享当前的执行环境（尤其是内存）
> 例如，一个线程在后台读取数据，另一个线程在数据到达后进行展示。
> 这个数据就被称作是共享的状态



例子

```csharp
class Program
{
    static void Main()
    {
        Thread t = new Thread(WriteY); // 开辟了一个新的线程 Thread
        t.Name = "Y Thread...";
        t.Start(); // 运行 WriteY()
        
        // 同时在主线程也做一些工作
        for (int i = 0; i < 1000; i++)
        	Console.WriteLine("x");
    }
    
    static void WriteY()
    {
        for (int i = 0; i < 1000; i++)
            Console.WriteLine("y");
    }
}
```

显示结果如下：

![consoleresult1](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609122359.png)



在单核计算机上，操作系统必须为每个线程分配“时间片”（在Windows中通常为20毫秒）来模拟并发，从而导致重复的`x`和`y`块。
在多核或多处理器计算机上，这两个线程可以真正地并行执行（可能受到计算机上其他活动进程的竞争）。
> 在本例中，由于控制台处理并发请求的机制的微妙性，您仍然会得到重复的x和y块。

![image-20200609122838599](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609122838.png)





#### 2. 术语：线程被抢占

线程在这个时候就可以被称为**被抢占**了：
> 它的执行与另外一个线程上代码的执行交织的那一点。



#### 3. 线程的一些属性

线程一旦开始执行，`IsAlive`就是`true`，线程结束就变成`false`。
线程结束的条件就是：线程构造函数传入的委托结束了执行。
线程一旦结束，就无法再重启。
每个线程都有一个`Name`属性，通常用于调试。
> 线程`Name`只能设置一次 ，以后更改会抛出异常。
静态的`Thread.CurrentThread`属性，会返回当前执行的线程。



例子



```csharp
class Program
{
    static void Main()
    {
        Thread.CurrentThread.Name = "Main Thread...";
        Thread t = new Thread(WriteY); // 开辟了一个新的线程 Thread
        t.Name = "Y Thread...";
        t.Start(); // 运行 WriteY()
        
        Console.WriteLine(Thread.CurrentThread.Name);
        // 同时在主线程也做一些工作
        for (int i = 0; i < 1000; i++)
        	Console.WriteLine("x");
    }
    
    static void WriteY()
    {
        Console.WriteLine(Thread.CurrentThread.Name);
        for (int i = 0; i < 1000; i++)
            Console.WriteLine("y");
    }
}
```



显示结果如下：

![image-20200609123757637](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609123757.png)





### 二、Thread.Join() & Thread.Sleep()

#### 1. Join and Sleep

调用`Join`方法，就可以等待另一个线程结束。



例子1

```csharp
class Program
{
    static void Main()
    {
        Thread t = new Thread(Go);
        t.Start();
        t.Join();
        Console.WriteLine("Thread t has ended!");
    }
    
    static void Go()
    {
        for (int i = 0; i < 1000; i++)
            Console.WriteLine("y");
    }
}
```



例子1显示结果如下：

![image-20200609124245590](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609124245.png)



例子2

```csharp
class Program
{
    static Thread thread1, thread2;
    
    static void Main()
    {
        thread1 = new Thread(ThreadProc);
        thread1.Name = "Thread1";
        thread1.Start();
        
        thread2 = new Thread(ThreadProc);
        thread2.Name = "Thread2";
        thread2.Start();
    }
    
    static void ThreadProc()
    {
        Console.WriteLine("\nCurrent thread: {0}", Thread.CurrentThread.Name);
        if (Thread.CurrentThread.Name == "Thread1" &&
           	thread2.ThreadState != ThreadState.Unstarted)
            thread2.Join();
        
        Thread.Sleep(4000);
        Console.WriteLine("\nCurrent thread: {0}", Thread.CurrentThread.Name);
        Console.WriteLine("Thread1: {0}", thread1.ThreadState);
        Console.WriteLine("Thread1: {0}\n", thread2.ThreadState);
    }
}
```



例子2显示结果如下：

![image-20200609125206397](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609125206.png)





#### 2. 添加超时

调用`Join`的时候，可以设置一个超时，用毫秒或者`TimeSpan`都可以。
> 如果返回`true`，那就是线程结束了；如果超时了，就返回`false`。

`Thread.Sleep()`方法会暂停当前的线程，并等待一段时间。



注意：
1. `Thread.Sleep(0)`这样调用会导致线程立即放弃本身当前的时间片，自动将CPU移交给其他线程。
2. `Thread.YieId()`做同样的事情，但是它只会把执行交给同一处理器上的其它线程。
3. 当等待`Sleep`或`Join`的时候，线程处于阻塞的状态。
4. `Sleep(0)`或`YieId()`有时在高级性能调试的生产代码中很有用。它也是一个很好的诊断工具，有助于发现线程安全问题：
> 如果在代码中的任何地方插入`Thread.YieId()`就破坏了程序，那么你的程序几乎肯定有bug。



例子1

```csharp
class Program
{
    static Thread thread1, thread2;
    
    static void Main()
    {
        thread1 = new Thread(ThreadProc);
        thread1.Name = "Thread1";
        thread1.Start();
        
        thread2 = new Thread(ThreadProc);
        thread2.Name = "Thread2";
        thread2.Start();
    }
    
    static void ThreadProc()
    {
        Console.WriteLine("\nCurrent thread: {0}", Thread.CurrentThread.Name);
        if (Thread.CurrentThread.Name == "Thread1" &&
           	thread2.ThreadState != ThreadState.Unstarted)
            if (thread2.Join(2000))
              	Console.WriteLine("Thread2 has termminated.");
        	else
              	Console.WriteLine("The timeout has elapsed and Thread1 will resume.");
        
        Thread.Sleep(4000);
        Console.WriteLine("\nCurrent thread: {0}", Thread.CurrentThread.Name);
        Console.WriteLine("Thread1: {0}", thread1.ThreadState);
        Console.WriteLine("Thread1: {0}\n", thread2.ThreadState);
    }
}
```



例子1的结果请自行执行。



例子2

```csharp
class Program
{
    static TimeSpan waitTime = new TimeSpan(0, 0, 1);
    
    static void Main()
    {
        Thread newThread = new Thread(Work);
        newThread.Start();
        
        if (newThread.Join(waitTime + waitTime))
        {
            Console.WriteLine("New Thread termminated.");
        }
        else
        {
            Console.WriteLine("Join timed out.");
        }
    }
    
    static void Work()
    {
        Thread.Sleep(waitTime);
    }
}
```



例子2显示结果如下：

![image-20200609130256797](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609130256.png)



### 三、阻塞 Blocking

#### 1. 阻塞

如果线程的执行由于某种原因导致暂停，我们就认为该线程被阻塞了。
> 例如在`Slepp`或者通过`Join`等待其他线程结束

被阻塞的线程会立即将其处理器的时间片生成给其它线程，从此就不再消耗处理器的时间，直到满足其阻塞条件为止。
可以通过`ThreadState`这个属性（枚举）来判断线程是否处于被阻塞的状态：

```csharp
bool blocked = (someThread.ThreadState & ThreadState.waitSleepJoin) != 0;
```



#### ThreadState

`ThreadState`是一个`flags enum`，通过按位的形式，可以合并数据的选项。

![image-20200609144922410](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609144922.png)



`ThreadState`状态的变化：

![image-20200609145501240](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609145501.png)

例子

```csharp
class Program
{
    static void Main()
    {
        var state = ThreadState.Unstarted | ThreadState.Stopped | ThreadState.WaitSleepJoin;
        Console.WriteLine($"{Convert.ToString((int)state, 2)}");
    }
}
```



显示结果如下：

![image-20200609145317113](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609145317.png)



但是它的大部分的枚举值都没什么用，下面的代码将`ThreadState`剥离为四个最有用的值之一：`Unstarted`、`Running`、`WaitSleepJoin`和`Stopped`

```csharp
public static ThreadState SimpleThreadState (ThreadState ts)
{
    return ts & (ThreadState.Unstarted |
                 ThreadState.WaitSleepJoin |
                 ThreadState.Stopped);
}
```



`ThreadState`属性可用于诊断的目的，但不适用于同步，因为线程状态可能会在测试`ThreadState`和对该信息进行操作之间发生变化。





#### 2. 解除阻塞Unblocking

当遇到下列四种情况的时候，就会解除阻塞：
1. 阻塞条件被满足
2. 操作超时（如果设置了超时的话）
3. 通过`Thread.Interrupt()`进行打断
4. 通过`Thread.Abort()`进行中止



#### 3. 上下文切换

当线程阻塞或解除阻塞时，操作系统将执行上下文切换。这会产生少量开销，通常为`1`或`2`微秒。



#### I/O-bound vs Compute-bound（或 CPU-Bound）

一个花费大部分时间等待某事发生的操作称为 I/O-bound
> I/O-bound 绑定操作通常涉及输入或输出，但这并不是硬性要求：`Thread.Sleep()`也被视为 I/O-bound
相反，一个花费大部分时间执行 CUP 密集型工作的操作称为 Compute_bound。



#### 4. 阻塞 vs 忙等待（自旋）-  Blocking vs Spinning

I/O-bound 操作的工作方式有两种：
1. 在当前线程上同步的等待
> `Console.ReadLine()`，`Thread.Sleep()`，`Thread.Join()`...
2. 异步的操作，在稍后操作完成时触发一个回调动作。
同步等待的 I/O-bound 操作将大部分时间花在阻塞线程上。
它们也可以周期性的在一个循环里进行“打转（忙等待或叫做自旋）”



忙等待（自旋）例子

![image-20200609152422889](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609152422.png)



在忙等待和阻塞方面有一些细微差别
1. 首先，如果您希望条件很快得到满足（可能在几微秒之内），则短暂自旋可能会很有效，因为它避免了上下文切换的开销和延迟。
> .NET Framework提供了特殊的方法和类来提供帮助`SpinLock**和`SpinWait。
2. 其次，阻塞也不是零成本。这是因为每个线程在生存期间会占用大约 **1 MB** 的内存，并会给`CLR`和操作系统带来持续的管理开销。
> 因此，在需要处理成百上千个并发操作的大量 I/O-bound 程序的上下文，阻塞可能会很麻烦。
> 所以，此类程序需要使用基于回调的方法，在等待时完全撤销其线程。



### 四、什么是线程安全

本地 vs 共享的状态 - Local vs Shared State

#### 1. Local 本地独立

`CLR`为每个线程分配自己的内存栈（`Stack`），以便使本地变量保持独立。



例子

```csharp
class Program
{
    static void Main()
    {
        new Thread(Go).Start(); // 在新线程上调用 Go()
        Go(); // 在 main 线程上调用 Go()
    }
    
    static void Go()
    {
        // cycles 是本地变量
        // 在每个线程的内存栈上，都会创建 cycles 独立的副本
        for (int cycles = 0; cycles < 5; cycles++)
        {
            Console.WriteLine("?");
        }
    }
    
    // 结果会输出 10 个 ？。
}
```



#### 2. Shared 共享

如果多个线程都引用到同一个对象的实例，那么它们就共享了数据。



例子

```csharp
class ThreadTest
{
    
    bool _done;
    
    static void Main()
    {
        ThreadTest tt = new ThreadTest(); // 创建了一个共同的实例
        
        // new Thread(tt.Go).Start() 和 tt.Go()用的都是同一个tt实例
        new Thread(tt.Go).Start();
        tt.Go();
    }
    
    void Go() // 这是一个实例方法（非静态）
    {
        if (!_done)
        {
            _done = true;
            Console.WriteLine("Done");
        }
    }
    
    // 由于两个线程是在同一个 ThreadTest 实例上调用调用的 Go()，所以它们共享 _done
    // 结果就是只打印一次 Done
}
```



被 Lambda 表达式或匿名委托所捕获的本地变量，会被编译器转化为字段（`field`），所以也会被共享。



例子

```csharp
class ThreadTest
{    
    static void Main()
    {
        bool done = false;
        
        ThreadStart action = () =>
        {
            if (!done)
            {
                done = true;
                Console.WriteLine("Done");
            }
        };
        
        new Thread(action).Start();
        action();
    }
}
```



显示结果如下：

![image-20200609175340413](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609175340.png)



静态字段（`field`）也会在线程间共享数据。



例子

```csharp
class ThreadTest
{
    
    static bool _done; // 静态字段在同一个应用域下的所有线程中被共享
    
    static void Main()
    {
        new Thread(Go).Start();
        Go();
    }
    
    static void Go()
    {
        if (!_done)
        {
            _done = true;
            Console.WriteLine("Done");
        }
    }
    
}
```



显示结果如下：

![image-20200609175758004](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609175758.png)





### 五、线程安全（Thread Safety）

后三个例子就引出了**线程安全**这个关键概念（或者说缺乏线程安全）
上述例子的输出实际上是无法确定的：
1. 有可能（理论上）“**Done**”会被打印两次。
2. 如果交换 **Go** 方法里语句的顺序，那么“Done”被打印两次的几率会大大增加。
3. 因为一个线程可能正在评估`if`，而另一个线程正在执行`WriteLine`语句，它还没来得及把 done 设为`true`。



例子

```csharp
class ThreadTest
{
    
    static bool _done; // 静态字段在同一个应用域下的所有线程中被共享
    
    static void Main()
    {
        new Thread(Go).Start();
        Go();
    }
    
    static void Go()
    {
        if (!_done)
        {
            Console.WriteLine("Done");
            Thread.Sleep(100); // 假设在处理上一个语句的时候花费了较长时间
            
             _done = true;
        }
    }
    
}
```



显示的结果如下：

![image-20200609180344028](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609180344.png)



###### 所以尽可能的避免使用共享状态。





#### 1. 锁定与线程安全（Locking & Thread Safety）

在读取和写入共享数据的时候，通过使用一个互斥锁（exclusive lock），就可以修复前面的例子的问题。
C#使用`lock`语句来加锁。
当两个线程同时竞争一个锁的时候（锁可以基于任何引用类型对象），一个线程会等待或阻塞，直到锁变成可用状态。



例子

```csharp
class ThreadTest
{
    
    static bool _done; // 静态字段在同一个应用域下的所有线程中被共享
    
    // 锁可以基于任何引用类型对象，可以换成 string、StringBuilder 等引用类型都可以
    static readonly object _locker = new object(); 
    
    static void Main()
    {
        new Thread(Go).Start();
        Go();
    }
    
    static void Go()
    {
        lock(_locker)
        {
           if (!_done)
        	{
            	Console.WriteLine("Done");
            	_done = true;
        	} 
        }        
    }    
}
```



在多线程上下文，以这种方式避免不确定性的代码就叫做**线程安全**。
Lock不是线程安全的银弹，很容易忘记对字段加锁，`lock`也会引起一些问题（死锁）



###### i++ 这种表达式是否是线程安全的？

不是线程安全的，它不是原子操作。

它包含三步：

1. 读取i的值

2. 计算i+1的值

3. 把i+1的值赋值给i



### 六、向线程传递数据 & 异常处理

#### 1. 向线程传递数据

如果你想往线程的启动方法里传递参数，最简单的方法就是使用`lambda`表达式，在里面使用参数调用方法。



例子

```csharp
class Program
{
    static void Main()
    {
        Thread t = new Thread(() => Print("Hello frome t!"));
        t.Start();
    }
    
    static void Print(string message)
    {
        Console.WriteLine(message);
    }
}
```



甚至可以把整个逻辑都放在`lambda`里面。



例子

```csharp
class Program
{
    static void Main()
    {
        new Thread(() => 
        {
            Console.WriteLine("I'm running on another thread!");
            Console.WriteLine("This is so easy!");
        }).Start();
    }
}
```



#### 2. 向线程传递数据（在 C# 3.0 之前）

在 **C# 3.0** 之前，没有`lambda`表达式。可以使用 **Thread** 的`Start`方法来传递参数。
**Thread** 的重载构造函数可以接受下列两个委托之一作为参数：

![image-20200609190246928](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609190247.png)



例子

```csharp
class Program
{
    static void Main()
    {
        Thread t = new Thread(Print);
        t.Start("Hello from t!");
    }
    
    static void Print(object messageObj)
    {
        string message = (string)messageObj; // 参数只能是object，需要转换
        Console.WriteLine(message);
    }
}
```



### 七、Lambda 表达式与被捕获的变量

使用 **Lambda** 表达式可以简单的给 **Thread** 传递参数。但是线程开始后，可能会不小心修改了被捕获的变量，这要多加注意。



例子

```csharp
class Program
{
    static void Main()
    {
        for (int i = 0; i < 10; i++)
        {
            new Thread(() => Console.WriteLine(i)).Start();
        }
    }
    
    // i 在循环的整个生命周期内指向的是同一个内存地址
    // 每个线程对 Console.WriteLine() 的调用都会在它运行的时候对它进行修改。
}
```



显示结果如下（运行四次，每次结果都不一样）：

![image-20200609191227001](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609191227.png)



解决方案例子：

```csharp
class Program
{
    static void Main()
    {
        for (int i = 0; i < 10; i++)
        {
            int temp = i;
            new Thread(() => Console.WriteLine(temp)).Start();
        }
    }
    
    // 但是顺序仍然无法保证
}
```



显示结果如下（不重复，但顺序无法保证）：

![image-20200609191546732](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609191546.png)





#### 1. 异常处理

创建线程时在作用范围内的`try/catch/finally`块，在线程开始执行后就与线程无关了。



例子

```csharp
class Program
{
    static void Main()
    {
        try
        {
            new Thread(Go).Start();
        }
        catch (Exception ex)
        {
            Console.WriteLine("Exception");
        }
    }
    
    static void Go() { throw null; }
    
    // 异常不会被捕获
    // 补救办法就是把异常处理放在 Go 方法里
}
```



解决方案例子：

```csharp
class Program
{
    static void Main()
    {
		new Thread(Go).Start();
    }
    
    static void Go() 
    {
        try
        {
			throw null; 
        }
        catch (Exception ex)
        {
         	Console.WriteLine("Exception");
        }
    }
}
```



在 **WPF、WinForm** 里，可以订阅全局异常处理事件：
1. `Application.DispatcherUnhandledException`
2. `Application.ThreadException`
3. 在通过消息循环调用的程序的任何部分发生未处理的异常（这相当于应用程序处于活动状态时在主线程上运行的所有代码）后，将触发这些异常。
4. 但是非 **UI** 线程上的未处理异常，并不会触发它。
而任何线程有任何未处理的异常都会触发`AppDomain.CurrentDomain.UnhandledException`





### 八、前台线程 & 后台线程

#### 1. 前台线程 & 后台线程（Foreground vs Background Threads）

默认情况下，你手动创建的线程就是前台线程。
只要有前台线程在运行，那么应用程序就会一直处于活动状态。
1. 但是只有后台线程运行却不行。
2. 一旦所有的前台线程停止了，那么应用程序就停止了
> 任何的后台线程也会突然终止。

注意：线程的前台、后台状态与它的优先级无关（所分配的执行时间）
> 可以通过`IsBackground`属性判断线程是否是后台线程。



例子

```csharp
class Program
{
    static void Main(string[] args)
    {
		Thread worker = new Thread(() => Console.ReadLine());
        if (args.Length > 0)
        {
            worker.IsBackground = true;
        }
        worker.Start();
    }
}
```



显示结果如下：

![image-20200609194709375](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609194709.png)



解释：

1. 如果没有带参数运行，那么 **worker** 线程就是前台线程，也就会一直等待我们输入
2. 如果带参数运行，**worker** 线程就被设置为后台线程，那当 **Main** 这个前台线程执行完之后，前台线程结束，后台线程也就结束了，也就无法输入了



进程以这种形式终止的时候，后台线程执行栈中的`finally`块就不会被执行了。
> 如果想让它执行，可以在退出程序时使用`Join`来等待后台线程（如果是你自己创建的线程），或者使用`signal construct`，如果是线程池...



应用程序无法正常退出的一个常见原因是还有活跃的前台线程。



### 九、线程优先级

#### 1. 线程优先级

线程的优先级（**Thread** 的`Priority`属性）决定了相对于操作系统中其它活跃线程所占的执行时间。
优先级（`Priority`属性）分为：
> `enum ThreadPriority { Lowest, BelowNormal, Normal, AboveNormal, Highest }`



#### 2. 提升线程优先级

提升线程优先级的时候需要特别注意，因为它可能“饿死”其他线程。
如果想让某线程（**Thread**）的优先级比其它进程（**Process**）中的线程（**Thread**）高，那就必须提升进程（**Process**）的优先级
> 使用`System.Diagnostics`下的`Process`类。

![image-20200609203953516](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609203953.png)

这可以很好地用于只做少量工作且需要较低延迟的非 **UI** 进程。
对于需要大量计算的应用程序（尤其是有 **UI** 的应用程序），提高进程优先级可能会使其他进程饿死，从而降低整个计算机的速度。



### 十、信号简介

#### 1. 信号（Signaling）

有时，你需要让某线程一直处于等待的状态，直至接收其它线程发来的通知。这就叫做 **signaling** （发送信号）。
最简单的信号结构就是 **ManualResetEvent**。
> 调用它上面的`WaitOne`方法会阻塞当前的线程，直到另一个线程通过调用`Set`方法来开启信号。



例子

```csharp
class Program
{
    static void Main(string[] args)
    {
		var signal = new ManualResetEvent(false);
        
        new Thread(() =>
        {
			Console.WriteLine("Waiting for signal ...");
            signal.WaitOne();
            
            // 以下语句在获得信号后才会执行
            signal.Dispose();
            Console.WriteLine("Got signal!");
        }).Start();
        
        Thread.Sleep(3000);
        signal.Set(); // 打开了信号
    }
}
```



结果显示如下：

![image-20200609205404686](https://blogcore.oss-cn-shenzhen.aliyuncs.com/markdown/img/20200609205404.png)

调用玩`Set`之后，信号会处于“打开”状态。可以通过调用`Reset`方法将其再次关闭。
