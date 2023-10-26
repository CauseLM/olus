# Handler机制之Thread

## 一、线程概念

### (一)、进程

> 在现代的操作系统中，进程是资源分配的最小单位，而线城是CPU调度的基本单位。一个进程中最少有一个线程，名叫主线程。进程是程序执行的一个实例，比如说10个用户同时执行Chrome浏览器，那么就有10个独立的进程(尽管他们共享一个可执行代码)。

**1、进程的特点：**

每一个进程都有自己的独立的一块内存空间、一组资源系统。其内部数据和状态都是完全独立的。进程的优点是提高CPU的运行效率，在同一个时间内执行多个程序，即并发执行。但是从严格上将，也不是绝对的同一时刻执行多个程序，只不过CPU在执行时通过时间片等调度算法不同进程告诉切换。

> 所以总结来说：进程由操作系统调度，简单而且稳定，进程之间的隔离性好，一个进程的奔溃不会影响其他进程。单进程编程简单，在多个情况下可以把进程和CPU进行绑定，从分利用CPU。

当然多进程也有一些缺点

**2、进程的缺点：**

> 一般来说进程消耗的内存比较大，进程切换代价很高，进程切换也像线程一样需要保持上一个进程的上下文环境。比如在Web编程中，如果一个进程处理一个请求的话，如果提高并发量就要提高进程数，而进程数量受内存和切换代价限制。

### (二)、线程

> 线程是进程的一个实体，是CPU调度和分配的基本单位，它比进程更下偶读能独立运行的基本单位，线程自己基本上不拥有系统资源，只拥有一点在运行中不可少的资源(如程序计数器、栈)，但是它可与同属一个进程的其他线程共享进程所拥有的全部资源。

同类的多线程共享一块内存空间个一组系统资源，线程本身的数据通常只有CPU的寄存器数据，以及一个供程序执行的堆栈。线程在切换时负荷小，因此，线程也称为轻负荷进程。一个进程中可以包含多个线程。

在JVM中，本地方法栈、虚拟机栈和程序计数器是线程隔离的，而堆区和方法区是线程共享的。

### (三)、进程线程的区别

> 地址空间：线程是进程内的一个执行单元；进程至少有一个线程；一个进程内的多线程它们共享进程的地址空间；而进程自己独立的地址空间
>  资源有用：进程是资源分配和拥有的单位，同一个进程内的线程共享进程的资源。
>  线程是处理器调度的基本单位，但进程不是
>  二者均可并发执行(下面补充了并发和并行的区别)

并发：多个事件在同一个时间段内一起执行
 并行：多个事件在同一时刻同时执行

### (四)、多线程

为了进一步提高CPU的利用率，多线程便诞生了。一个程序可以运行多个线程，多个线程可以同时执行，从整个应用角度上看，这个应用好像独自拥有多个CPU一样。虽然多线程进一步提高了应用的执行效率，但是由于线程之间会共享内存资源，这会导致一些资源同步的问题，另外，线程之间切换也会对资源有所消耗。

这里需要注意的是，如果一台电脑只有一个CPU核心，那么多线也并没有真正的"同时"运行，它们之间需要通过相互切换来共享CPU核心，所以，只有一个CPU核心的情况下，多线程不会提高应用效率。但是，现在计算机一般都会有多个CPU，并且每个CPU可能还有会多个核心，所以现在硬件资源条件下，多线程编程可以极大的提高应用的效率。

![img](https:////upload-images.jianshu.io/upload_images/5713484-0f7a599138b3862f.png?imageMogr2/auto-orient/strip|imageView2/2/w/489/format/webp)



在Java程序中，JVM负责线程的调度。线程调度是按照特定的机制为多个线程分配CPU的使用权。

调度的模式有两种：分时调度和抢占式调度。分时调度是所有线程轮流获得CPU使用权，并平均分配每个线程占用CPU的时间；抢占式调度是根据线程的优先级别来获取CPU的使用权。JVM的线程调度模式采用了抢占式模式。

## 二、Android线程的实现

> Android线程，一般地就是指Android虚拟机线程，而虚拟机线程是由通过系统调用而创建的Linux线程。纯粹的的Linux线程与虚拟机线程区别在于虚拟机线程具有运行Java代码的runtime。

在Android 中当担也就对应一个类。从这一点上看Thread和其他类并没有任何区别，只不过Thread的属性和方法仅用于完成"线程管理"这个任务而已。在Android系统中，我们经常需要启动一个新的线程，这些线程大多从Thread这个类继承

### (一)、Thread类

[Thread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/libart/src/main/java/java/lang/Thread.java)

```java
public class Thread implements Runnable {
  .....
}
```

通过上面代码，我们可以知道Thread实现了[Runnable](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/lang/Runnable.java)，侧面也说明线程是"可执行的代码"。

```csharp
public  interface Runnable {
    public abstract void run();
}
```

[Runnable](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/lang/Runnable.java)是一个接口类，唯一提供的方法就是run()。

#### 1、Thread的使用：

一般情况下，我们是这样使用Thread的：

##### (1)、继承Thread：



```tsx
public  MyThread extends Thread{
}
MyThread  mt=new MyThread();
mt.start();
```

##### (2)、直接使用Runnable：

Thread的关键就是Runnable，因此下面的是另一个常见的用法。



```cpp
new Thread(Runnable runnable).start();
```

#### 2、Thread的常用方法：

![img](https:////upload-images.jianshu.io/upload_images/5713484-4657ce630772034a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

#### 3、Thread的常用字段：



```csharp
    volatile ThreadGroup group;
    volatile boolean daemon;
    volatile String name;
    volatile int priority;
    volatile long stackSize;
    Runnable target;
    private static int count = 0;

    /**
     * Holds the thread's ID. We simply count upwards, so
     * each Thread has a unique ID.
     */
    private long id;

    /**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;
```

我们就依次说下：

> - **group**：每一个线程都属于一个group，当线程被创建是，就会加入一个特定的group，当前程运行结束，会从这个group移除。
> - **deamon**：当前线程不是守护线程，守护线程只会在没有非守护线程运行下才会运行
> - name：线程名称
> - priority：线程优先级，Thread的线程优先级取值范围为[1,10]，默认优先级为5
> - stackSize：线程栈大小，默认是0，即使用默认的线程栈大小(由dalvik中的全局变量gDvm.stackSize决定)
> - target：一个Runnable对象，Thread的run()方法中会转调target的run()方法，这是线程真正处理事务的地方。
> - id：线程id，通过递增的count得到该id，如果没有显式给线程设置名字，那么久会使用Thread+id当前线程的名字。注意这里不是真正的线程id，即在logcat中打印的tid并不是这个id，那tid是指dalvik线程id
> - localValues：本地线程存储(TLS)数据 关于TLS后面会详细介绍

#### 4、create()方法：

> 为什么要研究create()方法？因为Thread一种有9个构造函数，其中8个里面最终都是调用了create()方法

在[Thread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/libart/src/main/java/java/lang/Thread.java) 402行



```dart
 /**
     * Initializes a new, existing Thread object with a runnable object,
     * the given name and belonging to the ThreadGroup passed as parameter.
     * This is the method that the several public constructors delegate their
     * work to.
     *
     * @param group ThreadGroup to which the new Thread will belong
     * @param runnable a java.lang.Runnable whose method <code>run</code> will
     *        be executed by the new Thread
     * @param threadName Name for the Thread being created
     * @param stackSize Platform dependent stack size
     * @throws IllegalThreadStateException if <code>group.destroy()</code> has
     *         already been done
     * @see java.lang.ThreadGroup
     * @see java.lang.Runnable
     */
    private void create(ThreadGroup group, Runnable runnable, String threadName, long stackSize) {
        //步骤一 
        Thread currentThread = Thread.currentThread();

        //步骤二 
        if (group == null) {
            group = currentThread.getThreadGroup();
        }

        if (group.isDestroyed()) {
            throw new IllegalThreadStateException("Group already destroyed");
        }

        this.group = group;

        synchronized (Thread.class) {
            id = ++Thread.count;
        }

        if (threadName == null) {
            this.name = "Thread-" + id;
        } else {
            this.name = threadName;
        }

        this.target = runnable;
        this.stackSize = stackSize;

        this.priority = currentThread.getPriority();

        this.contextClassLoader = currentThread.contextClassLoader;

        // Transfer over InheritableThreadLocals.
        if (currentThread.inheritableValues != null) {
            inheritableValues = new ThreadLocal.Values(currentThread.inheritableValues);
        }

        // add ourselves to our ThreadGroup of choice
        //步骤二 
        this.group.addThread(this);
    }
```

我把create内部代码大体上分为3个部分

> - 步骤一：通过静态函数currentThread获取创建线程所在的当前线程
> - 步骤二：将创新线程所在的当前线程的一些属性赋值给即将创建的线程
> - 步骤三：通过调用ThreadGroup的addThread方法将新线程添加到group中。

#### 4、Thread的生命周期：

> 线程共有6种状态；在某一时刻只能是这6种状态之一。这些状态由Thread.State这个枚举类型表示，并且可以通过getState()方法获得当前具体的状态类型。

Thread.State这个枚举类在在[Thread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/libart/src/main/java/java/lang/Thread.java)  78行



```php
  /**
     * A representation of a thread's state. A given thread may only be in one
     * state at a time.
     */
    public enum State {
        /**
         * The thread has been created, but has never been started.
         */
        NEW,
        /**
         * The thread may be run.
         */
        RUNNABLE,
        /**
         * The thread is blocked and waiting for a lock.
         */
        BLOCKED,
        /**
         * The thread is waiting.
         */
        WAITING,
        /**
         * The thread is waiting for a specified amount of time.
         */
        TIMED_WAITING,
        /**
         * The thread has been terminated.
         */
        TERMINATED
    }
```

我们在用说明下：

> - **NEW** : 起劲尚未启动的线程的状态。当使用new一个新线程时，如new Thread(runnable)，但还没有执行start()，线程还有没有开始运行，这时线程的状态就是NEW。
>
> - **RUNNABLE**：可运行线程的线程状态。此时的线程可能正在运行，也可能没有运行。
>
> - BLOCKED
>
>   ：受阻塞并且正在等待监视锁的某一线程的线程状态。
>
>   进入阻塞状态的情况：
>
>   - ① 等待某个操作的返回，例如IO操作，该操作返回之前，线程不会继续后面的代码
>   - ② 等待某个"锁"，在其他线程或程序释放这个"锁"之前，线程不会继续执行。
>   - ③ 等待一定的触发条件
>   - ④ 线程执行了sleep()方法
>   - ⑤ 线程被suspend()方法挂起
>        一个被阻塞的线程在下列情况下会被重新激活
>   - ① 执行了sleep()，随眠时间已到
>   - ② 等待的其他线程或者程序持有"锁"已经被释放
>   - ③ 正在等待触发条件的线程，条件已得到满足
>   - ④ 执行suspend()方法，被调用了resume()方法
>   - ⑤ 等待的操作返回的线程，操作正确返回。
>
> - **WAITING**：某一等待线程的线程状态。线程因为调用了Object.wait()或者Thread.join()而未运行，就会进入WAITING状态。
>
> - **TIMED_WAITING**：具有指定等待时间的某一等待线程的线程状态。线程是应为调用了Thread.sleep()，或者加上超时值在调用Object.wait()或者Thread.join()而未运行，则会进入TIMED_WAITING状态。
>
> - **TERMINATED**：已终止线程状态。线程已运行完毕，它的run()方法已经正常结束或者通过抛出异常而技术。线程的终止，run()方法结束，线程就结束了。

总结称为一幅图就是下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-d8bb315210babf51.png?imageMogr2/auto-orient/strip|imageView2/2/w/519/format/webp)

Thread的生命周期.png

#### 5、线程的启动：

> 上面说的这两种方法获取Thread，最终都通过start()方法启动。

代码在[Thread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/libart/src/main/java/java/lang/Thread.java)  1058行



```java
    /**
     * Starts the new Thread of execution. The <code>run()</code> method of
     * the receiver will be called by the receiver Thread itself (and not the
     * Thread calling <code>start()</code>).
     *
     * @throws IllegalThreadStateException - if this thread has already started.
     * @see Thread#run
     */
    public synchronized void start() {
         //保证线程只启动一次
        checkNotStarted();

        hasBeenStarted = true;

        nativeCreate(this, stackSize, daemon);
    }


    private void checkNotStarted() {
        if (hasBeenStarted) {
            throw new IllegalThreadStateException("Thread already started");
        }
    }
```

通过上面代码我们看到，start()方法里面首先是判断是不是启动过，如果没启动过直接调用nativeCreate(Thread , long, boolean)方法，通过方法名，我们知道是一个nativce方法

代码在[Thread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/libart/src/main/java/java/lang/Thread.java) 1066行



```java
  private native static void nativeCreate(Thread t, long stackSize, boolean daemon);
```

##### 5.1、nativeCreate()函数

nativeCreate()这是一个native方法，那么其所对应的JNI方法在哪里？在[java_lang_Thread.cc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/art/runtime/native/java_lang_Thread.cc)中国getMethods是一个JNINativeMethod数据

代码在[java_lang_Thread.cc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/art/runtime/native/java_lang_Thread.cc)   179行



```csharp
static JNINativeMethod gMethods[] = {
  NATIVE_METHOD(Thread, currentThread, "!()Ljava/lang/Thread;"),
  NATIVE_METHOD(Thread, interrupted, "!()Z"),
  NATIVE_METHOD(Thread, isInterrupted, "!()Z"),
  NATIVE_METHOD(Thread, nativeCreate, "(Ljava/lang/Thread;JZ)V"),
  NATIVE_METHOD(Thread, nativeGetStatus, "(Z)I"),
  NATIVE_METHOD(Thread, nativeHoldsLock, "(Ljava/lang/Object;)Z"),
  NATIVE_METHOD(Thread, nativeInterrupt, "!()V"),
  NATIVE_METHOD(Thread, nativeSetName, "(Ljava/lang/String;)V"),
  NATIVE_METHOD(Thread, nativeSetPriority, "(I)V"),
  NATIVE_METHOD(Thread, sleep, "!(Ljava/lang/Object;JI)V"),
  NATIVE_METHOD(Thread, yield, "()V"),
};
```

其中一项为：



```bash
NATIVE_METHOD(Thread, nativeCreate, "(Ljava/lang/Thread;JZ)V"),
```

这里的NATIVE_METHOD定义在java_lang_Object.cc文件中，如下：

代码在[java_lang_Object.cc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/art/runtime/native/java_lang_Object.cc)   25行



```cpp
#define NATIVE_METHOD(className, functionName, signature, identifier) \
    { #functionName, signature, reinterpret_cast<void*>(className ## _ ## identifier) }
```

将宏定义展开并带入，可得所对应的方法名为**Thread_nativeCreate**

##### 5.2、Thread_nativeCreate()函数

代码在[java_lang_Thread.cc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/art/runtime/native/java_lang_Thread.cc)   49行



```css
static void Thread_nativeCreate(JNIEnv* env, jclass, jobject java_thread, jlong stack_size,
                                jboolean daemon) {
  Thread::CreateNativeThread(env, java_thread, stack_size, daemon == JNI_TRUE);
}
```

看到 只是调用了Thread类的CreateNativeThread

##### 5.3、Thread::CreateNativeThread()函数

代码在[thread.cc](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/art/runtime/thread.cc)   388行



```cpp
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  CHECK(java_peer != nullptr);
  Thread* self = static_cast<JNIEnvExt*>(env)->self;
  Runtime* runtime = Runtime::Current();

  // Atomically start the birth of the thread ensuring the runtime isn't shutting down.
  bool thread_start_during_shutdown = false;
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    if (runtime->IsShuttingDownLocked()) {
      thread_start_during_shutdown = true;
    } else {
      runtime->StartThreadBirth();
    }
  }
  if (thread_start_during_shutdown) {
    ScopedLocalRef<jclass> error_class(env, env->FindClass("java/lang/InternalError"));
    env->ThrowNew(error_class.get(), "Thread starting during runtime shutdown");
    return;
  }

  Thread* child_thread = new Thread(is_daemon);
  // Use global JNI ref to hold peer live while child thread starts.
  child_thread->tlsPtr_.jpeer = env->NewGlobalRef(java_peer);
  stack_size = FixStackSize(stack_size);

  // Thread.start is synchronized, so we know that nativePeer is 0, and know that we're not racing to
  // assign it.
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer,
                    reinterpret_cast<jlong>(child_thread));

  // Try to allocate a JNIEnvExt for the thread. We do this here as we might be out of memory and
  // do not have a good way to report this on the child's side.
  std::unique_ptr<JNIEnvExt> child_jni_env_ext(
      JNIEnvExt::Create(child_thread, Runtime::Current()->GetJavaVM()));

  int pthread_create_result = 0;
  if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;
    child_thread->tlsPtr_.tmp_jni_env = child_jni_env_ext.get();
    CHECK_PTHREAD_CALL(pthread_attr_init, (&attr), "new thread");
    CHECK_PTHREAD_CALL(pthread_attr_setdetachstate, (&attr, PTHREAD_CREATE_DETACHED),
                       "PTHREAD_CREATE_DETACHED");
    CHECK_PTHREAD_CALL(pthread_attr_setstacksize, (&attr, stack_size), stack_size);

     /***这里是重点，创建线程***/
    pthread_create_result = pthread_create(&new_pthread,
                                           &attr,
                                           Thread::CreateCallback,
                                           child_thread);
    CHECK_PTHREAD_CALL(pthread_attr_destroy, (&attr), "new thread");

    if (pthread_create_result == 0) {
      // pthread_create started the new thread. The child is now responsible for managing the
      // JNIEnvExt we created.
      // Note: we can't check for tmp_jni_env == nullptr, as that would require synchronization
      //       between the threads.
      child_jni_env_ext.release();
      return;
    }
  }

  // Either JNIEnvExt::Create or pthread_create(3) failed, so clean up.
  {
    MutexLock mu(self, *Locks::runtime_shutdown_lock_);
    runtime->EndThreadBirth();
  }
  // Manually delete the global reference since Thread::Init will not have been run.
  env->DeleteGlobalRef(child_thread->tlsPtr_.jpeer);
  child_thread->tlsPtr_.jpeer = nullptr;
  delete child_thread;
  child_thread = nullptr;
  // TODO: remove from thread group?
  env->SetLongField(java_peer, WellKnownClasses::java_lang_Thread_nativePeer, 0);
  {
    std::string msg(child_jni_env_ext.get() == nullptr ?
        "Could not allocate JNI Env" :
        StringPrintf("pthread_create (%s stack) failed: %s",
                                 PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());
  }
}
```

这里面重点是**pthread_create()函数**，pthread_create是pthread库中的函数，通过syscall再调用到clone来创建线程。

> - 原型：int pthread_create((pthred_t thread,pthread_attr_t * attr, void * (start_routine) (void * ), void * arg))
> - 头文件：#include
> - 入参： thread(线程标识符)、attr(线程属性设置)、start_routine(线程函数的起始地址)、arg(传递给start_rountine参数)：
> - 返回值：成功则返回0；失败则返回-1
> - 功能：创建线程，并调用线程其实地址指向函数start_rountine。

再往下就到内核层了，由于篇幅限制，就先不深入，有兴趣的同学可以自行研究

## 三、线程的阻塞

> 线程阻塞指的是暂停一个线程的执行以等待某个条件发生(如某资源就绪)。Java提供了大量的方法来支持阻塞，下面让我们逐一分析。

### (一)、sleep()方法

> sleep()允许指定以毫米为单位的一段时间作为参数，它使得线程在指定的时间内进入阻塞状态，不能得到CPU时间，指定的时间已过，线程重新进入可执行状态。典型地，sleep()被用在等待某个资源就绪的情形：测试发现条件不满足后，让线程阻塞一段后重新测试，直到条件满足为止。

### (二) suspend()和resume()方法

> 两个方法配套使用，suspend()使得线程进入阻塞状态，并且不会自动恢复，必须其对应的resume()被调用，才能使得线程重新进入可执行状态。典型地，suspend()和resume()被用在等待另一个线程产生的结果的情形：测试发现结果还没有产生后，让线程阻塞，另一个线程产生了结果后，调用resume()使其恢复。

### (三) yield()方法

> yeield()使得线程放弃当前分得的CPU时间，但是不使线程阻塞，即线程仍处于可执行状态，随时可能再次分的CPU时间。调用yield()效果等价于调度程度认为该线程已执行了足够的时间从而转到另一个线程。

### (四) wait()和notify()方法

两个方法配套使用，wait()使得线程进入阻塞状态，它有两种形式，一种允许指定以毫秒为单位的一段时间作为参数，另一种没有惨呼是，当前对应的notify()被调用或者超出指定时间线程重新进入可执行状态，后者则必须对应的notify()被调用。初看起来它们与suspend()和resume()方法对没有什么分别，但是事实上它们是截然不同的。区别的核心在于，前面叙述的所有方法，阻塞时都不会释放占用的锁(如果占用的话)，而这一对方法则相反。

这里需要重点介绍下wait()和notify()

> - 首先，前面叙述的所有方法都隶属于Thread类，但是这一对却直接隶属于Object类，也就是说，所有对象都拥有这一对方法。初看起来十分不可思议，但是实际上却是很自然的，因为这一对方法阻塞时需要释放占用的锁，而锁是任何对象都具有的，调用对象wait()方法导致线程阻塞，并且该对象上的锁释放。而调用对象的notify()方法则导致因调用对象的wait()方法而阻塞线程中随机选择的一个解除阻塞(但要等待获得锁后才真正可执行)。
> - 其次，前面叙述的所有方法都可在任何位置调用，但是这一对方法却必须在synchronized方法或块中调用，理由也很简单，只有synchronized方法或块中当前线程才占有所，才有锁可以释放。同样的道理，调用这一对方法的对象上的锁必须为当前线程锁拥有，这样才有锁可以释放。因此，这一对方法调用必须防止在这样的synchronized方法或者块中，该方法或者块的上锁对象就是调用这对方法的对象。若不满足这一条件，则程序虽然仍能编译，但是在运行时会出现IllegalMonitorStateException异常。
> - 第三，调用notify()方法导致解除阻塞的线程是从因调用该对象的wait()方法而阻塞的线程中随机选取的，我们无法预料那个一个线程将会被选择，所以编程时需要特别小心，避免因这种不确定性而产生问题。
> - 最后，除了notify()，还有一个方法notifyAll()也可能其到类似作用，唯一的区别是在于，调用notifyAll()方法将把 因 调用该对象的wait()方法而阻塞的所有线程一次性全部解除阻塞。当然，只有获得锁的那一个线程才能进入可执行状态。

## 四、关于线程上下文切换

> 在多线程编程中，多个线程公用资源，计算机会多各个线程进行调度。因此各个线程会经历一系列不同的状态，以及在不同的线程间进行切换。
>  既然线程需要被切换，在生命周期中处于各种状态，如等待、阻塞、运行。吧线程就需要能够保存线程，在线程被切换后/回复后需要继续在上一个状态运行。这就是所谓的上下文切换。为了实现上下文切换，势必会消耗资源，造成性能损失。因为我们在进行多线程编程过程中需要减少上下文切换，提高程序运行性能。

一些常用的方法：

> - 无锁并发编程：可以采用一些方法避免锁的使用，如不同线程去处理不同段
> - CAS常用算法：原子操作不需要加锁
> - 使用最少线程：避免创建不需要的线程
> - 协程：在单线程里实现多任务调度

## 五、关于线程的安全问题

> 线程安全无非是要控制多个线程对某个资源的有序访问或修改。即多个线程，一个临界区，通过通知这些线程对临界区的访问，使得每个线程的每次执行结果都相同(搞清楚这个问题，可以避免多线程编程的狠多错误)

### (一)、实现线程安全的工具：

> - 1 隐式锁：synchronized
> - 2 显式锁：java.util.concurrent.lock
> - 3 关键字：volatile
> - 4 原子操作：java.util.concurrent.atomic

### (二)、线程同步阀：

> - 1 CountDownLatch类是一个同步计数器。(java.util.concurrent.CountDownLatch)
> - 2 CyclicBarrier是一个同步辅助类，它允许一组线程相互等待，直到到到某个公共屏蔽点(common barrier point)。在涉及一组固定大小的线程的程序中，这些线程不时地相互等待，这时候CyclicBarrier很有用。因为barrier在释放等待线程后可以重用，因此称为循环的barrier。
> - 3 信号量(java.util.concurrent.Semaphone)，计数信号量，该信号量维护一定大小的许可集合，规定最多允许多少个进程同时访问临界区。其中，semp.acquire()类似于操作系统中的P操作，semp.release类似于操作系统的V操作。
> - 4 任务机制(java.util.concurrent.Future->FutureTask)。结合Runnable使用！一般FutureTask多用于耗时的计算，主线程在完成自己的任务后再去获取结果；只有计算完成时获取，否则一直阻塞。

## 六、守护线程

### (一) 概念

> 守护线程我觉得还是很有用的。首先看看守护进程是什么?守护线程就是后台运行的线程。普通线程结束后，守护线程自动结束。一般main线程视为守护线程，以及GC、数据库连接池等，都做成守护进程。

### (二) 特点

> 守护线程就像备胎一样，JRE(女神)根本不会管守护进行有没有，在不在，只要前台线程结束，就算执行完毕了。

### (三) 如何使用

> 直接调用setDeamon() 即可。

### (四) 注意事项

> setDaemon(true) 必须在start()方法之前调用；在守护线程中开启的线程也是守护线程；守护线程中不能进行I/O操作。

## 七、线程的内存

### (一) Java内存模型

> Java内存模型规范了Java虚拟机与计算机内存是符合协同工作的。Java虚拟机是一个完整的计算机的一个模型，因此这个模型自然也包含一个内存模型——又称为Java内存模型。

如果你想设计表现良好的并发程序，理解Java内存模型是非常重要的。Java内存模型规定了如何和何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步的访问共享变量。

原始的Java内存模型存在一些不足，因此Java内存模型在Java 1.5时被重新修订。这个版本的Java内存模型在Java8中仍在使用。

### (二) Java 线程内存模型原理

Java内存模型把Java虚拟机内部划分为线程栈和堆。下面这张图演示了Java内存模型的逻辑视图。

![img](https:////upload-images.jianshu.io/upload_images/5713484-0793233a5b1454a5.png?imageMogr2/auto-orient/strip|imageView2/2/w/358/format/webp)

Java内存模型原理.png

> - 每一个运行在Java虚拟机李的线程都拥有自己的线程栈。这个线程栈包含了这个线程调用的方法当前执行点相关的信息。一个线程仅能访问自己的线程栈。一个线程创建的本地变量对其他线程不可见，仅自己可见。即使两个线程执行同样的代码，这两个线程仍然在自己的线程栈中的代码来创建本地变量。因此，每个线程拥有每个本地变量的独有版本。
> - 所有原始类型的本地变量都存放在线程栈上，因此对其他线程不可见。一个线程可能向另一个线程传递一个原始类型变量的拷贝，但是它不能共享这个原始类型变量自身。
> - 堆上包含在Java程序中创建的所有对象，无论是哪一个对象创建的。这包括原始类型的对象版本。如果一个对象被创建然后赋值给一个局部变量，或者用来作为另一个对象的成员变量，这个对象仍然是存在堆上。

下面这张图演示了调用栈和本地变量存放在线程栈上，对象存放在堆上。

![img](https:////upload-images.jianshu.io/upload_images/5713484-f8841c2a05539821.png?imageMogr2/auto-orient/strip|imageView2/2/w/486/format/webp)

堆与栈.png

所以大体可以分为以下几种情况：

> - 一个本地变量可能是原始类型，在这种情况下，它总是"待在"线程栈上。
> - 一个本地变量也可能是指向一个对象的一个引用。在这种情况下，引用(这个本地变量)存放在这个线程栈上，但是对象本身存放在堆上。
> - 一个对象可能包含方法，这些方法可能包含本地变量。这些本地变量仍然存放在线程栈上，即使这些方法是所属的对象存放在堆上。
> - 一个对象的成员变量可能随着这个对象自身存放在堆上。不管这个成员变量是原始类型还是引用类型。
> - 静态成员变量跟随着类定义一起也存放在堆上。
> - 存放在堆上的对象可以被所有吃持有这个对象引用的线程访问。当一个线程可以访问一个对象时，它也可以访问这个对象的成员变量。如果两个线程同时调用同一个对象上的同一个方法，它们将会都访问这个对象的成员变量，但是每一个线程都拥有这个本地变量的私有拷贝。

下图演示了上面提到的点：

![img](https:////upload-images.jianshu.io/upload_images/5713484-b52d73332088ecaa.png?imageMogr2/auto-orient/strip|imageView2/2/w/505/format/webp)

情况.png

PS:

> - 1、两个线程拥有一系列的本地变量。其中一个本地变量(Local Variable 2)执行堆上的一个共享对象(Object 3)。这两个线程分贝拥有同一个对象的不同引用。这些引用都是本地变量，因此存放在各自线程的线程栈上。这两个不同的引用指向堆上同一个对象。
> - 2、这个共享对象(Object 3)持有Object 2和Object 4一个引用作为其成员员变量(如图中Object 3 指向 Object 2和Object 4的箭头)。通过这在Object 3中这些成员变量引用，这两个线程就可以访问到Object 2和 Object 4。

上面这张图也展示了指向堆上两个不同对象的一个本地变量。在这种情况下，指向两个不同对象的引用不是同一个对象。理论上，两个线程都可以访问Object 1 和Object 5，如果两个线程都拥有两个对象的引用。但是在上图中，每个线程仅有一个引用指向两个对象其中之一。

### (三) 硬件内存架构

> 现代硬件内存模型与Java模型有一些不同。理解内存模型结构以及Java内存模型如何与它协同工作也是非常重要的。这部分描述了通用的硬件内存架构，下面的部分将会描述内存是如何与它"联合"工作的。

现代计算机硬件架构的简单图示：

![img](https:////upload-images.jianshu.io/upload_images/5713484-e82e4b7ca3ed6bdb.png?imageMogr2/auto-orient/strip|imageView2/2/w/452/format/webp)

现代计算机硬件架构.png

> - 一个现代计算机通常由两个或者多个CPU。其中一些CPU还有多核。从这一点可以看出，在一个或者多个CPU的现代计算上运行多个线程是可能的。每个CPU在某一时刻运行一个线程是没有问题的。这意味着，如果你的Java程序是多线程的，在你的Java程序中每个CPU上一个线程可能同时(并发)执行。
> - 每个CPU都包含一系列的寄存器，它们是CPU内存的基础。CPU在寄存器上执行操作的速度远大于主存上执行的速度。这是因为CPU访问寄存器的速度远大于主存。
> - 每个CPU可能还有一个CPU缓存层。实际上，绝地多数的现代CPU都有一定大小的缓存层。CPU访问缓存层的速度快于访问主存的速度，但通常比访问内部寄存器的速度还要慢一点。一些CPU还有多层缓存，但这些对理解Java内存模型如何和内存交互不是那么重要。只要知道CPU中可以由一个缓存层就可以了。
> - 一个计算机还包含一个主存。所有CPU都可以访问主存。主存通常比CPU缓存大得多。
> - 通常情况下，当一个CPU需要读取主存时，它会将主存的部分读到CPU缓存中。它甚至可能将缓存中的部分内容读到它的内部寄存器中，然后在寄存器中执行操作。当CPU需要将结果写回到主存中时，它会将内部寄存器值刷新到缓存中，然后在某个时间点将值刷新回主存。
> - 当CPU需要在缓存层存放一些东西的时候，存放在缓存中的内容通常会刷新回主存。CPU缓存可以在某一时刻将数据局部写到它的内存中，和在某一时刻局部刷新它的内存。它不会在某一时刻读/写整个缓存。通常，在一个被称作"cache lines"的更小内存块中缓存被更新。一个或者多个缓存行可能被读到缓存，一个或者多个缓存行可能再被刷新回主存。

### (四) Java内存模型和硬件内存架构之间的桥接

> 上面已经提到，Java内存模型与硬件内存架构之间存在差异。硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内中。部分线程栈和堆可能有时候会出现在CPU缓存中和CPU内部的寄存器中。

如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/5713484-be7bc7fca66ec8f8.png?imageMogr2/auto-orient/strip|imageView2/2/w/817/format/webp)

桥接.png

> 当对象和变量被存放在计算机中各个不同的内存区域中时，就可能会出现一些具体的问题。主要包含两个方面：
>
> - 线程对共享变量修改的可见性
> - 当读、写和检查共享变量时rece conditions(竞争条件)

下面我们专门来解释一下上面的两个问题

**1、对象的可见性**

> 如果两个或者更多的线程在没有正确使用volatile声明或者同步的情况下共享一个对象，一个线程更新这个共享对象可能对其他线程来说是不可见的。

想象一下，共享对象那个被初始化在主存中。跑在CPU上的一个线程将这个共享对象读到CPU缓存中。然后修改了这个对象。要CPU缓存没有被刷新到驻村，对象修改后的版本对跑在其他CPU上的线程都是不可见的。这种方式可能导致每个线程拥有这个共享对象的私有拷贝，每个拷贝停留在不同的CPU缓存中。

下面示意了这种情形。

![img](https:////upload-images.jianshu.io/upload_images/5713484-1bf466a04dd6dc6b.png?imageMogr2/auto-orient/strip|imageView2/2/w/520/format/webp)

跑在左边的CPU的线程拷贝这个共享对象到它的CPU缓存中，然后将count变量的值修改为2，这个修改对跑在右边的CPU上的其他线程是不可见的，因为修改后count的值还没有被刷新回主存中去。

> 解决这个问题你可以使用volatile关键字。volatile关键字可以保证直接从主存中读取一个变量，如果这个变量被修改后，总是会写回到主存中去。

**2、Race Conditions(竞争条件)**

> 如果两个或者更多的线程共享一个对象，多个线程在这个共享对象上更新变量，就可能放生Race Conditions(竞争条件)。想象一下，如果线程A读取一个共享对象的变量count到它的CPU缓存中。再想象一下，线程B也做了同样的事情，但是往一个不同的CPU缓存个中。现在线程A将count加1，线程B也做了同样的事情，现在count已经被增加了两个，每个CPU缓存中一次。如果这些增加操作被顺序执行，变量count应该被增加两次，然后原值+2倍写回到主存中区。然而，两次增加都是在没有适当的同步下并发执行的。无论线程A还是线程B将count修改后的版本写回到主存中去，修改后的值仅会被原值大1，尽管增加了两次。

下图演示了上面描述的情况：

![img](https:////upload-images.jianshu.io/upload_images/5713484-76a7b9ae5fe2ba91.png?imageMogr2/auto-orient/strip|imageView2/2/w/542/format/webp)

解决这个问题可以使用**Java同步块**。一个同步块可以保证在同一时刻仅有一个线程可以进入代码的临界区。同步块还可以保证代码块中所有被访问的变量将会从主存中读入，当线程退出同步代码块是，所有被更新的变量会被刷新回主存中区，不管这个变量是否被声明为volatile。



# Handler机制之ThreadLocal

本片文章的主要内容如下：

> - 1、Java中的ThreadLocal
> - 2、 Android中的ThreadLocal
> - 3、Android 面试中的关于ThreadLocal的问题
> - 4、ThreadLocal的总结
> - 5、思考

**这里先说下Java中的ThreadLocal和Android中的ThreadLocal的源代码是不一样的，为了让大家更好的理解Android中的ThreadLocal，我们先从Java中的ThreadLocal开始。**

> 注：基于Android 6.0(6.0.1_r10)/API 23  源码

## 一、 Java中的ThreadLocal：

### (一) ThreadLocal的前世今生

> 早在JDK1.2的版本中就提供java.lang.ThreadLocal，ThreadLocal为解决多线程程序的并发问题提供了一种新的思路，并在JDK1.5开始支持泛型。这个工具类可以很简洁地编写出优美的多线程程序。当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立的改变自己的副本，而不会影响其他线程所对应的副本。从线程的角度来看，目标变量就像是本地变量，这也是类名中"Local"所要表达的意思。所以，在Java中编写线程局部变量的代码相对来说要"笨拙"一些，因此造成了线程局部变量没有在Java开发者得到很好的普及。

### (二) ThreadLocal类简介

#### 1、Java源码描述

看Java源码中的描述：

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

翻译一下：

> ThreadLocal类用来提供线程内部的局部变量，这种变量在多线程环境下访问(通过get或set方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量。ThreadLocal实例通常来说都是private static 类型，用于关联线程。

如果大家还不明白，我们可以类比一下就是：

> 比如，今天是七夕，我们的计划是，先吃饭、看电影、去开房(你们懂的)，这里吃饭、看电影、开房好比同一个线程的三个函数，我就是一个线程，我完成这三个函数都需要一个东西"支付宝"(用来付钱)。但是支付宝(类比为ThreadLocal)这个东西其实是**支付宝公司**的，我只要吃完饭，看电影买票，开房付房费的时候才会使用，平时都是放在手机里面不动的吧。这样我就可以在何时何地用支付宝付款了。
>  当然你们会问，为什么不设置为全局变量，这样不也是可以实现何时何地都能去公交卡吗？但是如果有很多人(很多线程)呢？总不能大家都用我的支付宝吧，那样我不就成为雷锋了。这就是ThreadLocal设计的初衷：提供线程内部的局部变量，在本地线程内随时随地可取，隔离其他线程。

总结一下：

> ThreadLocal的作用是提供线程内的局部变量，这种局部变量仅仅在线程的生命周期中起作用，减少同一个线程内多个函数或者组件一些公共变量的传递的复杂度。

#### 2、ThreadLocal类结构

ThreadLocal的类图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-d89c4a36ff46b0be.png?imageMogr2/auto-orient/strip|imageView2/2/w/669/format/webp)

ThreadLocal的类图.png

ThreadLocal的结构图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-25b7d58cc48916c7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1120/format/webp)

ThreadLocal类结构.png

#### 3、ThreadLocal常用的方法

##### (1)、set方法

设置当前线程的线程局部变量的值
 源码如下：



```dart
   /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

代码流程很清晰：

> 先拿到保存键值对的ThreadLocalMap对象实例map，如果map为空(即第一次调用的时候map值为null)，则去创建一个ThreadLocalMap对象并赋值给map，并把键值对保存在map

我们看到首先是拿到当前先线程实例**t**，任何将**t**作为参数构造**ThreadLocalMap**对象，为什么需要通过Threadl来获取**ThreadLocalMap对象**？**Thread**类和**ThreadLocalMap**有什么联系，那我们来看下getMap()方法的具体实现



```dart
    //ThreadLocal.java
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

我们看到getMap实现非常直接，就是直接返回Thread对象的threadLocal字段。Thread类中的ThreadLocalMap字段声明如下：



```csharp
    //Thread.java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

ok，我们总结一下：

> **ThreadLocal**的**set(T)** 方法中，首先是拿到当前线程**Thread**对象中的**ThreadLocalMap**对象实例**threadLocals**，然后再将需要保存的值保存到threadLocals里面。换句话说，每个线程引用的**ThreadLocal**副本值都是保存在当前**Thread**对象里面的。存储结构为**ThreadLocalMap**类型，**ThreadLocalMap**保存的类型为**ThreadLocal**，值为**副本值**

##### (2)、get方法

该方法返回当前线程所对应的线程局部变量



```kotlin
   /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```

> 同样的道理，拿到当前线程**Thread**对象实例中保存的**ThreadLocalMap**对象**map**，然后从**ma**p中读取键为**this**(即ThreadLocal类实例)对应的值。

如果map不是null，直接从map里面读取就好了，如果map==null，那么我们需要对当前线程Thread对象实例中保存ThreadLocalMap对象new一下。即通过setInitialValue()方法来创建，setInitialValue()方法的具体实现如下：



```csharp
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

代码很清晰，通过createMap来创建ThreadLocalMap对象，前面set(T)方法里面的ThreadLocalMap也是通过createMap来的，我们看看createMap具体实现：



```dart
    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     * @param map the map to store.
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

这样我们就对ThreadLocal的存储机制彻底清楚了。

##### (3)、remove()方法



```csharp
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
```

代码很简单，就是直接将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK1.5新增的方法。需要指出的，当线程结束以后，对该应线程的局部变量将自动被垃圾回收，所以显示调用该方法清除线程的局部变量并不是必须的操作，但是它可以加速内存回收的数据。

#### 4、内部类ThreadLocalMap

先来看下内部类的注释



```dart
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {}
```

简单翻译一下：

> ThreadLocalMap是一个适用于维护线程本地值的自定义哈希映射(hash map)，没有任何操作可以让它超出ThreadLocal这个类的范围。该类是私有的，允许在Thread类中声明字段。为了更好的帮助处理常使用的，hash表条目使用了WeakReferences的键。但是，由于不实用引用队列，所以，只有在表空间不足的情况下，才会保留已经删除的条目

##### (1)、存储结构

通过注释我们知道ThreadLocalMap中存储的是ThreadLocalMap.Entry(后面用Entry代替)对象。因此，在ThreadLocalMap中管理的也就是Entry对象。也就是说，ThreadLocalMap里面大部分函数都是针对Entry。

首先ThreadlocalMap需要一个"容器"来存储这些Entry对象，ThreadLocalMap中定义了额Entry数据实例table，用于存储Entry



```php
        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
```

也就是说，ThreadLocalMap维护一张哈希表(一个数组)，表里存储Entry。既然是哈希表，那肯定会涉及到加载因子，即当表里面存储的对象达到容量的多少百分比的时候需要扩容。ThreadLocalMap中定义了threshold属性，当表里存储的对象数量超过了threshold就会扩容。如下：



```cpp
/**
 * The next size value at which to resize.
 */
private int threshold; // Default to 0

/**
 * Set the resize threshold to maintain at worst a 2/3 load factor.
 */
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

从上面代码可以看出，加载银子设置为2/3。即每次容量超过设定的len2/3时，需要扩容。

##### (2)、存储Entry对象

> hash散列的数据在存储过程中可能会发生碰撞，大家知道HashMap存储的是一个Entry链，当hash发生冲突后，将新的的Entry存放在链表的最前端。但是ThreadLocalMap不一样，采用的是index+1作为重散列的hash值写入。另外有一点需要注意key出现null的原因由于Entry的key是继承了弱引用，在下一次GC是不管它有没有被引用都会被回收。当出现null时，会调用replaceStaleEntry()方法循环寻找相同的key，如果存在，直接替换旧值。如果不存在，则在当前位置上重新创新的Entry。

看下代码：



```csharp
    /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        //设置当前线程的线程局部变量的值
        private void set(ThreadLocal key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();
                //替换掉旧值
                if (k == key) {
                    e.value = value;
                    return;
                }
                //和HashMap不一样，因为Entry key 继承了所引用，所以会出现key是null的情况！所以会接着在replaceStaleEntry()重新循环寻找相同的key
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }


       /**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
```

从上面源码中可以看出，通过key(ThreadLocal类型)的hashcode来计算存储的索引位置 i 。如果 i 位置已经存储了对象，那么就向后挪一个位置以此类推，直到找到空的位置，再讲对象存放。另外，在最后还需要判断一下当前的存储的对象个数是否已经超出了反之(threshold的值)大小，如果超出了，需要重新扩充并将所有的对象重新计算位置(rehash函数来实现)。那么我们看看rehash方法如何实现的。

##### (2.1)、rehash()方法



```cpp
        /**
         * Re-pack and/or re-size the table. First scan the entire
         * table removing stale entries. If this doesn't sufficiently
         * shrink the size of the table, double the table size.
         */
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
```

看到，rehash函数里面先是调用了expungeStaleEntries()函数，然后再判断当前存储对象的小时是否超出了阀值的3/4。如果超出了，再扩容。不过这样有点混乱，为什么不直接扩容并重新摆放对象?

上面我们提到了，ThreadLocalMap里面存储的Entry对象本质上是一个WeakReference<ThreadLcoal>。也就是说，ThreadLocalMap里面存储的对象本质是一个队ThreadLocal的弱引用，该ThreadLocal随时可能会被回收！即导致ThreadLocalMap里面对应的 value的Key是null。我们需要把这样的Entry清除掉，不要让他们占坑。

expungeStaleEntries函数就是做这样的清理工作，清理完后，实际存储的对象数量自然会减少，这也不难理解后面的判断的约束条件为阀值的3/4，而不是阀值的大小。

##### (2.2)、expungeStaleEntries()与expungeStaleEntry()方法



```csharp
        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
```

expungeStaleEntries()方法很简单，主要是遍历table，然后调用expungeStaleEntry()，下面我们来主要讲解下这个函数expungeStaleEntry()函数。

##### (2.3) expungeStaleEntry()方法

> ThreadLocalMap中的expungeStaleEntry(int)方法的可能被调用的处理有：

![img](https:////upload-images.jianshu.io/upload_images/5713484-1774a202dc7eebca.png?imageMogr2/auto-orient/strip|imageView2/2/w/702/format/webp)

expungeStaleEntry的调用.png



通过上面的图，不难看出，这个方法在ThreadLocal的set、get、remove时都会被调用。



```dart
        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

> 从上面代码中，可以看出先清理指定的Entry，再遍历，如果发现有Entry的key为null，就清理。Key==null，也就是ThreadLocal对象是null。所以当程序中，将ThreadLocal对象设置为null，在该线程继续执行时，如果执行另一个ThreadLocal时，就会触发该方法。就有可能清理掉Key是null的那个ThreadLocal对应的值。

所以说expungStaleEntry()方法清除线程ThreadLocalMap里面所有key为null的value。

##### (3)、获取Entry对象getEntry()



```dart
        /**
         * Get the entry associated with key.  This method
         * itself handles only the fast path: a direct hit of existing
         * key. It otherwise relays to getEntryAfterMiss.  This is
         * designed to maximize performance for direct hits, in part
         * by making this method readily inlinable.
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntry(ThreadLocal key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```

getEntry()方法很简单，直接通过哈希码计算位置 i ，然后把哈希表对应的 i 的位置Entry对象拿出来。如果对应位置的值为null，这就存在如下几种可能。

> - key 对应的值为null
> - 由于位置冲突，key对应的值存储的位置并不是 i 位置上，即 i 位置上的null并不属于 key 值

因此，需要一个函数去确认key对应的value的值，即getEntryAfterMiss()方法

##### (3.1)、getEntryAfterMiss()函数



```dart
   /**
         * Version of getEntry method for use when key is not found in
         * its direct hash slot.
         *
         * @param  key the thread local object
         * @param  i the table index for key's hash code
         * @param  e the entry at table[i]
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

所以ThreadlocalMap的getEntry()方法的整体流程如下：

> 第一步，从ThreadLocal的直接索引位置(通过ThreadLocal.threadLocalHashCode&(len-1)运算得到)获取Entry e，如果e不为null，并且key相同则返回e。
>  第二步，如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回应对应的Entry，否则，如果key值为null，则擦除该位置的Entry，否则继续向一个位置查询。

ThreadLocalMap整个get过程中遇到的key为null的Entry都被会擦除，那么value的上一个引用链就不存在了，自然会被回收。set也有类似的操作。这样在你每次调用ThreadLocal的get方法去获取值或者调用set方法去设置值的时候，都会去做这个操作，防止内存泄露，当然最保险的还是通过手动调用remove方法直接移除

##### (4)、ThreadLocalMap.Entry对象

前面很多地方都在说ThreadLocalMap里面存储的是ThreadLocalMap.Entry对象，那么ThreadLocalMap.Entry独享到底是如何存储键值对的?同时有是如何做到的对ThreadLocal对象进行弱引用？
 代码如下：



```dart
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
```

从源码的继承关系可以看到，Entry是继承WeakReference<ThreadLocal>。即Entry本质上就是WeakReference<ThreadLocal>，换而言之，Entry就是一个弱引用，具体讲，Entry实例就是对ThreadLocal某个实例的弱引用。只不过，Entry同时还保存了value

整体模型如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-bc893ac35da6fd7b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

模型1.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-fd837cb2b37254f1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1178/format/webp)

模型2.png

#### 5、总结

> ThreadLocal是解决线程安全的一个很好的思路，它通过为每个线程提供了一个独立的变量副本解决了额变量并发访问的冲突问题。在很多情况下，ThreadLocal比直接使用synchronized同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。ThreadLocal和synchronize用一句话总结就是一个用存储拷贝进行空间换时间，一个是用锁机制进行时间换空间。

其实补充知识:

> - ThreadLocal官方建议已定义成private static 的这样让Thread不那么容易被回收
> - 真正涉及共享变量的时候ThreadLocal是解决不了的。它最多是当每个线程都需要这个实例，如一个打开数据库的对象时，保证每个线程拿到一个进而去操作，互不不影响。但是这个对象其实并不是共享的。

## 二、 Android中的ThreadLocal

#### (一) ThreadLocal的作用

> Android中的ThreadLocal和Java 的ThreadLocal是一致的，每个线程都有自己的局部变量，一个线程的本地变量对其他线程是不可见的，ThreadLocal不是用与解决共享变量的问题，不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制

#### (二) 源码跟踪

ThreadLocal的源代码在[ThreadLocal.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/lang/ThreadLocal.java)

##### 1、ThreadLocal的结构

![img](https:////upload-images.jianshu.io/upload_images/5713484-ed7645a01b7d2ab9.png?imageMogr2/auto-orient/strip|imageView2/2/w/682/format/webp)

ThreadLocal的结构.png

可以直观地看到在android中的ThreadLocal类提供了一些方法和一个静态内部类Value，其中Values主要是用来保存线程的变量的一个类，它相当于一个容器，存储保存进来的变量。

##### 2、ThreadLocal的内部具体实现

首先看下成员变量

###### (1)、ThreadLocal的成员变量



```dart
    /** Weak reference to this thread local instance. */
    private final Reference<ThreadLocal<T>> reference
            = new WeakReference<ThreadLocal<T>>(this);

    /** Hash counter. */
    private static AtomicInteger hashCounter = new AtomicInteger(0);

    /**
     * Internal hash. We deliberately don't bother with #hashCode().
     * Hashes must be even. This ensures that the result of
     * (hash & (table.length - 1)) points to a key and not a value.
     *
     * We increment by Doug Lea's Magic Number(TM) (*2 since keys are in
     * every other bucket) to help prevent clustering.
     */
    private final int hash = hashCounter.getAndAdd(0x61c88647 * 2);
```

我们发现成员变量主要有3个，那我们就依次来说下

> - **reference**：通过弱引用存储ThreadLocal本身，主要是防止线程自身所带的数据都无法释放，避免OOM。
> - **hashCounter**：是线程安全的加减操作，getAndSet(int newValue)取当前的值，并设置新的值
> - **hash**：里面的0x61c88647 * 2的作用是：在Value存在数据的主要存储数组table上，而table被设计为下标为0，2，4...2n的位置存放key，而1，3，5...(2n-1)的位置存放的value，0x61c88647*2保证了其二进制最低位为0，也就是在计算key的下标时，一定是偶数位。

###### (2)、ThreadLocal的方法

ThreadLocal除了构造函数一共6个方法，我们就依次说下

**① get()方法**

该方法返回线程局部变量的当前线程副本中的值



```kotlin
    /**
     * Returns the value of this variable for the current thread. If an entry
     * doesn't yet exist for this variable on this thread, this method will
     * create an entry, populating the value with the result of
     * {@link #initialValue()}.
     *
     * @return the current value of the variable for the calling thread.
     */
    @SuppressWarnings("unchecked")
    public T get() {
        // Optimized for the fast path.
        // 获取当前线程
        Thread currentThread = Thread.currentThread();
        // 获取当前线程的Value 实例
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            //如果键值的key的索引为index，则所对应到的value索引为index+1
            //所以 hash&value.mask获取就是key的索引值
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
           // 如果当前Value实例为空，则创建一个Value实例
            values = initializeValues(currentThread);
        }
        return (T) values.getAfterMiss(this);
    }
```

> 通过代码，我们得出**get()**是通过value.table这个数据通过索引值来找到值的

上面代码中调用了额initializeValues(Thread)方法，下面我们就来看一下

**② initializeValues(currentThread)方法**



```cpp
    /**
     * Creates Values instance for this thread and variable type.
     */
    Values initializeValues(Thread current) {
        return current.localValues = new Values();
    }
```

> 通过上面代码我们知道initializeValues(Thread)方法主要是直接new出一个新的Values对象。

**③ initialValue()方法**

该方法返回此线程局部变量的当前线程的"初始值"



```php
    /**
     * Provides the initial value of this variable for the current thread.
     * The default implementation returns {@code null}.
     *
     * @return the initial value of the variable.
     */
    protected T initialValue() {
        return null;
    }
```

> 也就是默认值为Null，当没有设置数据的时候，调用get()的时候，就返回Null,可以在创建ThreadLocal的时候复写initialValue()方法可以定义初始值。

**④ initialValue()方法**

将此线程局部变量的当前线程副本中的值设置为指定值。



```csharp
    /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        //获取当前线程
        Thread currentThread = Thread.currentThread();
        // 获取 当前线程的Value
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        // 将数据设置到Value
        values.put(this, value);
    }
```

**⑤ initialValue()方法**



```java
    /**
     * Removes the entry for this variable in the current thread. If this call
     * is followed by a {@link #get()} before a {@link #set},
     * {@code #get()} will call {@link #initialValue()} and create a new
     * entry with the resulting value.
     *
     * @since 1.5
     */
    public void remove() {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            values.remove(this);
        }
    }
```

移除此线程局部变量的当前线程的值。

**⑥ initialValue()方法**



```cpp
    /**
     * Gets Values instance for this thread and variable type.
     */
    Values values(Thread current) {
        return current.localValues;
    }
```

> 通过线程获取Values对象。

##### 3、ThreadLocal静态类ThreadLocal.Values

ThreadLocal内部类Value是被设计用来保存线程的变量的一个类，它相当于一个容器，存储保存进来的变量。

###### 3.1 ThreadLocal.Values的结构

![img](https:////upload-images.jianshu.io/upload_images/5713484-11cd724b883f4bb4.png?imageMogr2/auto-orient/strip|imageView2/2/w/687/format/webp)

结构.png

主要原理是Values将数据存储在数组table(Object[] table)中，那么键值对如何存放？就是table被设计为下标为0，2，4...2n的位置存放key，而1，3，5...(2n+1)的位子存放value。取数据的时候可以直接通过下标存取线程变量。

###### 3.2 ThreadLocal.Values的具体实现

**(1) ThreadLocal.Values的成员变量**

成员变量

```cpp
        /**
         * Size must always be a power of 2.
         */
        private static final int INITIAL_SIZE = 16;

        /**
         * Placeholder for deleted entries.
         */
        private static final Object TOMBSTONE = new Object();

        /**
         * Map entries. Contains alternating keys (ThreadLocal) and values.
         * The length is always a power of 2.
         */
        private Object[] table;

        /** Used to turn hashes into indices. */
        private int mask;

        /** Number of live entries. */
        private int size;

        /** Number of tombstones. */
        private int tombstones;

        /** Maximum number of live entries and tombstones. */
        private int maximumLoad;

        /** Points to the next cell to clean up. */
        private int clean;
```

我们就来一次看一下：

> - INITIAL_SIZE：默认的初始化容量，所以初始容量为16
> - TOMBSTONE: 表示被删除的实体标记，移除变量时它是把对应的key的位置赋值为TOMBSTONE
> - table：保存变量的地方，长度必须是2的N次方的值
> - mask：计算下标的掩码，它的值是table的长度-1
> - size：存放及拿来的实体的数量
> - tombstones：表示被删除的实体的数量
> - maximumLoad：阀值，用来判断是否需要进行rehash
> - clean：表示下一个要进行清理的位置点

**(2) ThreadLocal.Values的无参构造函数**

```kotlin
        /**
         * Constructs a new, empty instance.
         */
        Values() {
            //创建32为容量的容器，为什么是32不是16
            initializeTable(INITIAL_SIZE);
            this.size = 0;
                        this.tombstones = 0;
        }
```

我们知道INITIAL_SIZE=16，那为什么是创建容量是32而不是16，那我们就来看一下initializeTable(int) 方法

**(3) initializeTable(int) 方法**

```cpp
        /**
         * Creates a new, empty table with the given capacity.
         */
        private void initializeTable(int capacity) {
            //通过capacity创建table容器
            this.table = new Object[capacity * 2];
            //下标的掩码
            this.mask = table.length - 1;
            this.clean = 0;
            // 2/3的 最大负载因子
            this.maximumLoad = capacity * 2 / 3; // 2/3
        }
```

**(4) ThreadLocal.Values的有参构造函数**

```kotlin
        /**
         * Used for InheritableThreadLocals.
         */
        Values(Values fromParent) {
            this.table = fromParent.table.clone();
            this.mask = fromParent.mask;
            this.size = fromParent.size;
            this.tombstones = fromParent.tombstones;
            this.maximumLoad = fromParent.maximumLoad;
            this.clean = fromParent.clean;
            inheritValues(fromParent);
        }
```

通过代码我们知道，就是传入一个Value，然后根据这个传入的Value进行相应的值复制而已。

结合上面的无参的构造函数我们得出如下结论：ThreadLocal.Values有两个构造函数，一个是普通的构造函数，一个是类似于继承的那种，从一个父Values对象来生成新的Values。

**(5) add(ThreadLocal,Object)方法**

```dart
        /**
         * Adds an entry during rehashing. Compared to put(), this method
         * doesn't have to clean up, check for existing entries, account for
         * tombstones, etc.
         */
        void add(ThreadLocal<?> key, Object value) {
            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];
                if (k == null) {
                     //在index的位置上放入一个"键"
                    table[index] = key.reference;
                    //在 index+1的位置上放入一个"值"
                    table[index + 1] = value;
                    return;
                }
            }
        }
```

上面的代码向我们展示了table存储数据的方法，它是以一种类似于map的方法来存储的，在index处存入map的键，在index的下一位存入键对应的值，而这个键则是ThreadLocal的引用，这里毫无问题。但是有一个地方问题则是大大的有：int index=key.hash&mask。大家都明白这行代码的作用是获得可用的索引，可是到底是怎么获得的？为什么要通过这这种方式来获得？要解决这个问题，我们要先知道何为"可用的索引"，通过分析观察，我总结出了一些条件：

> - 要是偶数
> - 不能越界
> - 尽可能能分散(尽可能的新产生的所以不要是已经出现过的数，不然table的空间不能充分的利用，而且观察上面的代码，会发现如果新产生的索引是已经出现过的数的话数据根本存不进去)

好的，现在我们来看看这个int index=key.hash&mask 究竟能不能搞定这些个问题。先来看下mask：



```kotlin
this.mask = table.length - 1;
```

> mask的值是table的长度减一，而我前面说过，table的长度是2N，也就说mask总是等于2N-1，这意味着mask的二进制表示总是N个1，那么这有说明什么？key.hash&mask的结果其实就是key.hash后面的n位！这样一来，首先上面的第二个条件已经满足了，因为N位无符号的二进制的范围是0~(2N-1)，刚好在table的范围之内。

这时候我们再来看下key.hash：



```dart
//1.5之后加入的一个类，它所持有的Integer可以原子的增加，其本身是线程安全的
private static AtomicInteger hashCounter = new AtomicInteger(0);
/**
 * Internal hash. We deliberately don't bother with #hashCode().
 * Hashes must be even. This ensures that the result of
 * (hash & (table.length - 1)) points to a key and not a value.
 *
 * We increment by Doug Lea's Magic Number(TM) (*2 since keys are in
 * every other bucket) to help prevent clustering.
 */
private final int hash = hashCounter.getAndAdd(0x61c88647 * 2);
```

hash的初始值是0，然后每次调用hash的时候，它都会先返回给你它的当前值，然后将当前值加上一个(0x61c88647 * 2)。别的先不看，这样一来这个hash肯定是满足上面的第一个条件的：乘以2相当于二进制里左移一位，那么最后一位就肯定是0了，这样的话它与mask的与运算的结果肯定最后一位是0，也就是换成十进制之后肯定是偶数。这现在就剩第三个条件，其实肯定是满足的。

> 这里面的关键就是这个奇怪的数字**0x61c88647**，根源就在它的身上。它转换成为有符号32位二进制数是**0110 0001 1100 1000 1000 0110 0100 0111**，那么它的复数就是**1110 0001 1100 1000 1000 0110 0100 0111**（为什么这里可以用它的负数？因为其实它和它的负数在运算的时候结果是一样的，已面访根本到不了符号就已经内存溢出的，一方面在ThreadLocal里运算的时候有一个*2，所以它的符号其实已经没有了），运用计算的一些知识，我们知道在底层运算的时候其实用的就是它的补码，即
>  1001 1110 0011 0111 0111 1001 1011 1001**，这个数据十进制是2654435769，而2654435769这个数就有意思了。
>
> ![img](https:////upload-images.jianshu.io/upload_images/5713484-66f95728a49d494b.png?imageMogr2/auto-orient/strip|imageView2/2/w/292/format/webp)
>
> 公式
>
> 通过这个数，我们可以得到斐波拉契散列——这个散列中的数是绝对分散且不重复的——也就是说上面条件的第三点也已经满足了，如果有想进一步探索的同学可以自行研究。

**(6) put(ThreadLocal,Object)方法**

```dart
        /**
         * Cleans up after garbage-collected thread locals.
         */
        private void cleanUp() {
            if (rehash()) {
                // 如果rehash()方法返回的是true，就不需要继续clean up
                // If we rehashed, we needn't clean up (clean up happens as
                // a side effect).
                return;
            }

            //如果table的size是0的话，也不需要clean up ，因为都没有什么键值在里面好clean的
            if (size == 0) {
                // No live entries == nothing to clean.
                return;
            }

            // Clean log(table.length) entries picking up where we left off
            // last time.

             // 默认的clean是0，下一次在clean的时候是从上一次clean的地方继续。
            int index = clean;
            Object[] table = this.table;
            for (int counter = table.length; counter > 0; counter >>= 1,
                    index = next(index)) {
                Object k = table[index];
                //如果已经删除，则删除下一个
                if (k == TOMBSTONE || k == null) {
                    continue; // on to next entry
                }

                // The table can only contain null, tombstones and references.
                // 这个table只可能存储null，tombstones和references键
                @SuppressWarnings("unchecked")
                Reference<ThreadLocal<?>> reference
                        = (Reference<ThreadLocal<?>>) k;
                //如果ThreadLocal已经不存在了，则释放Value数据
                if (reference.get() == null) {
                    // This thread local was reclaimed by the garbage collector.
                    // 已经被释放
                    table[index] = TOMBSTONE;
                    table[index + 1] = null;
                    //被删除的数量
                    tombstones++;
                    size--;
                }
            }

            // Point cursor to next index.
            clean = index;
        }
```

cleanUp()方法只做了一件事，就是把失效的键放上TOMBSTONE去占位，饭后释放它的值。那么rehash()是干什么的其实已经显而易见了：

> - 从字面意思来也知道是重新规划table的大小。
>    -联想clenUp()的作用，它都已经把失效键放上TOMBSTONE，然后肯定是想办法干掉这些TOMBSTONE。

cleanUp()方法只做了一件事，就是把失效的键放上TOMBSTONE去占位，然后释放它的值。那么rehash()是干什么的其实已经很显而易见了：

- 从字面意思来也知道是重新规划table的大小。
- 联想cleanUp()的作用，它都已经把失效键放上TOMBSTONE，接下来呢？显然是想办法干掉这些TOMBSTONE，还我内存一个朗朗乾坤喽。

**(7) put(ThreadLocal,Object)方法**

```csharp
        /**
         * Sets entry for given ThreadLocal to given value, creating an
         * entry if necessary.
         */
        void put(ThreadLocal<?> key, Object value) {
             // 清理废弃的元素
            cleanUp();

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            // 通过key获取索引index
            for (int index = key.hash & mask;; index = next(index)) {
                // 通过索引获取ThreadLocal
                Object k = table[index];
                // 如果ThreadLocal是存在，直接返回
                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                // 找到firstTombstone这个索引值，然后赋值对应的key和value
                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }
```

> 可以看到，put()方法里面的逻辑其实很简单，就是在想方设法的把传进来的兼职对给存进去——其中对获得的index的值进行了一些判断，以决定如何进行存储——总之是想要尽可能的节省空间。另外，值得注意的是，在遇到相同索引处存放着同一个键的时候，其采取的方式是新值换旧值。

**(8) remove(ThreadLocal)方法**

```csharp
       /**
         * Removes entry for the given ThreadLocal.
         */
        void remove(ThreadLocal<?> key) {
            // 先把table清理一下
            cleanUp();

            for (int index = key.hash & mask;; index = next(index)) {
                Object reference = table[index];
                //把那个引用的用TOMBSTONE占用 
                if (reference == key.reference) {
                    // Success!
                    table[index] = TOMBSTONE;
                    table[index + 1] = null;
                    tombstones++;
                    size--;
                    return;
                }

                if (reference == null) {
                    // No entry found.
                    return;
                }
            }
        }
```

> 这个方法很简答， 就是把传进来的那个键对应的值给清理掉

**(9) getAfterMiss(ThreadLocal)方法**

```csharp
        /**
         * Gets value for given ThreadLocal after not finding it in the first
         * slot.
         */
        Object getAfterMiss(ThreadLocal<?> key) {
            Object[] table = this.table;
            //通过散列算法得到ThreadLocal的first slot的索引值
            int index = key.hash & mask;

            // If the first slot is empty, the search is over.
            if (table[index] == null) {
                // 如果 first slot 上没有存储 则将ThreadLocal的弱引用和本地数据存储到table数组的相邻位置并返回本地数据对象的引用。
                Object value = key.initialValue();

                // If the table is still the same and the slot is still empty...
                if (this.table == table && table[index] == null) {
                    table[index] = key.reference;
                    table[index + 1] = value;
                    size++;

                    cleanUp();
                    return value;
                }
                // The table changed during initialValue().
                // 遍历table数组，根据不同判断将ThreadLocal的弱引用和本地数据对象引用存储到数组的相应位置
                put(key, value);
                return value;
            }

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            // Continue search.
            for (index = next(index);; index = next(index)) {
                Object reference = table[index];
                if (reference == key.reference) {
                    return table[index + 1];
                }

                // If no entry was found...
                if (reference == null) {
                    Object value = key.initialValue();

                    // If the table is still the same...
                    if (this.table == table) {
                        // If we passed a tombstone and that slot still
                        // contains a tombstone...
                        if (firstTombstone > -1
                                && table[firstTombstone] == TOMBSTONE) {
                            table[firstTombstone] = key.reference;
                            table[firstTombstone + 1] = value;
                            tombstones--;
                            size++;

                            // No need to clean up here. We aren't filling
                            // in a null slot.
                            return value;
                        }

                        // If this slot is still empty...
                        if (table[index] == null) {
                            table[index] = key.reference;
                            table[index + 1] = value;
                            size++;

                            cleanUp();
                            return value;
                        }
                    }

                    // The table changed during initialValue().
                    put(key, value);
                    return value;
                }

                if (firstTombstone == -1 && reference == TOMBSTONE) {
                    // Keep track of this tombstone so we can overwrite it.
                    firstTombstone = index;
                }
            }
        }
```

> getAfterMiss函数根据不同的判断将ThreadLocal的弱引用和当前线程的本地对象以类似map的方式，存储在table数据的相邻位置，其中散列的所以hash值是通过hashCounter.getAndAdd(0x61c88647 * 2) 算法来得到。

## 三、 Android 面试中的关于ThreadLocal的问题

> - 问题1  ThreadLocal内部实现原理，怎么保证数据中仅被当前线程持有？
> - 问题2 ThreadLocal修饰的变量一定不能被其他线程访问吗？
> - 问题3 ThreadLocal的对象存放在哪里？
> - 问题4 ThreadLocal会存在内存泄露吗？

#### (一) ThreadLocal内部实现原理，怎么保证数据中仅被当前线程持有？

> ThreadLocal在进行放值时的代码如下：



```csharp
    /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

> ThreadLocal的值放入了当前线程的一个Values实例中，所以只能在本线程访问，其他线程无法访问。

#### (二)  ThreadLocal修饰的变量一定不能被其他线程访问吗？

> 不是，对于子线程是可以访问父线程中的ThreadLocal修饰的变量的。如果在主线程中创建一个InheritableThreadLocal实例，那么在子线程中就可以得到InheritableThreadLocal实例，并获取相应的值。在ThreadLocal中的**inheritValues(Values fromParent)**方法获取父线程中的值

#### (三)  ThreadLocal的对象存放在哪里？

> 是在堆上，在Java中，线程都会有一个栈内存，栈内存属于单个线程，其存储的变量只能在其所属线程中可见。但是ThreadLocal的值是被线程实例所有，而线程是由其创建的类型所持有，所以ThreadLocal实例实际上也是被其他创建的类所持有的，故它们都存在于堆上。

#### (四)  ThreadLocal会存在内存泄露吗？

> 是不会，虽然ThreadLocal实例被线程中的Values实例所持有，也可以被看成是线程所持有，若使用线程池，之前的线程实例处理完后出于复用的目的依然存在，但Values在选择key时，并不是直接选择ThreadLocal实例，而是ThreadLocal实例的弱引用：



```php
Reference<ThreadLocal<?>> reference = (Reference<ThreadLocal<?>>) k;
ThreadLocal<?> key = reference.get();
```

在get()方法中也是采用弱引用：



```kotlin
private final Reference<ThreadLocal<T>> reference
= new WeakReference<ThreadLocal<T>>(this);
if (this.reference == table[index]) {
return (T) table[index + 1];
```

这样能保存如查到当前thread被销毁时，ThreadLocal也会随着销毁被GC回收。

## 四、 ThreadLocal的总结

分析到这里，整个ThreadLocal的源码就分析的差不多了。在这里我们简单的总结一下这个类：

> - 这个类之所以能够存储每个thread的信息，是因为它的内部有一个Values内部类，而Values中有一个Object组。
> - Objec数组是以一种近似于map的形式来存储数据的，其中偶数位存ThreadLocal的弱引用，它的下一位存值。
> - 在寻址的时候，Values采用一种很神奇的方式——斐波拉契散列寻址Values里面的getAfterMiss()方式让人觉得很奇怪

## 五、思考

这里大家思考一下，谷歌的Android团队为什么要重写ThreadLocal，而不是直接使用Java层面的ThreadLocal？



# Handler机制之SystemClock类



官网位置在[https://developer.android.com/reference/android/os/SystemClock.html](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/SystemClock.html)

平时看源码的是[SystemClock.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/SystemClock.java)

老规矩先看下类的注释

## 一、类注释



```dart
/**
 * Core timekeeping facilities.
 *
 * <p> Three different clocks are available, and they should not be confused:
 *
 * <ul>
 *     <li> <p> {@link System#currentTimeMillis System.currentTimeMillis()}
 *     is the standard "wall" clock (time and date) expressing milliseconds
 *     since the epoch.  The wall clock can be set by the user or the phone
 *     network (see {@link #setCurrentTimeMillis}), so the time may jump
 *     backwards or forwards unpredictably.  This clock should only be used
 *     when correspondence with real-world dates and times is important, such
 *     as in a calendar or alarm clock application.  Interval or elapsed
 *     time measurements should use a different clock.  If you are using
 *     System.currentTimeMillis(), consider listening to the
 *     {@link android.content.Intent#ACTION_TIME_TICK ACTION_TIME_TICK},
 *     {@link android.content.Intent#ACTION_TIME_CHANGED ACTION_TIME_CHANGED}
 *     and {@link android.content.Intent#ACTION_TIMEZONE_CHANGED
 *     ACTION_TIMEZONE_CHANGED} {@link android.content.Intent Intent}
 *     broadcasts to find out when the time changes.
 *
 *     <li> <p> {@link #uptimeMillis} is counted in milliseconds since the
 *     system was booted.  This clock stops when the system enters deep
 *     sleep (CPU off, display dark, device waiting for external input),
 *     but is not affected by clock scaling, idle, or other power saving
 *     mechanisms.  This is the basis for most interval timing
 *     such as {@link Thread#sleep(long) Thread.sleep(millls)},
 *     {@link Object#wait(long) Object.wait(millis)}, and
 *     {@link System#nanoTime System.nanoTime()}.  This clock is guaranteed
 *     to be monotonic, and is suitable for interval timing when the
 *     interval does not span device sleep.  Most methods that accept a
 *     timestamp value currently expect the {@link #uptimeMillis} clock.
 *
 *     <li> <p> {@link #elapsedRealtime} and {@link #elapsedRealtimeNanos}
 *     return the time since the system was booted, and include deep sleep.
 *     This clock is guaranteed to be monotonic, and continues to tick even
 *     when the CPU is in power saving modes, so is the recommend basis
 *     for general purpose interval timing.
 *
 * </ul>
 *
 * There are several mechanisms for controlling the timing of events:
 *
 * <ul>
 *     <li> <p> Standard functions like {@link Thread#sleep(long)
 *     Thread.sleep(millis)} and {@link Object#wait(long) Object.wait(millis)}
 *     are always available.  These functions use the {@link #uptimeMillis}
 *     clock; if the device enters sleep, the remainder of the time will be
 *     postponed until the device wakes up.  These synchronous functions may
 *     be interrupted with {@link Thread#interrupt Thread.interrupt()}, and
 *     you must handle {@link InterruptedException}.
 *
 *     <li> <p> {@link #sleep SystemClock.sleep(millis)} is a utility function
 *     very similar to {@link Thread#sleep(long) Thread.sleep(millis)}, but it
 *     ignores {@link InterruptedException}.  Use this function for delays if
 *     you do not use {@link Thread#interrupt Thread.interrupt()}, as it will
 *     preserve the interrupted state of the thread.
 *
 *     <li> <p> The {@link android.os.Handler} class can schedule asynchronous
 *     callbacks at an absolute or relative time.  Handler objects also use the
 *     {@link #uptimeMillis} clock, and require an {@link android.os.Looper
 *     event loop} (normally present in any GUI application).
 *
 *     <li> <p> The {@link android.app.AlarmManager} can trigger one-time or
 *     recurring events which occur even when the device is in deep sleep
 *     or your application is not running.  Events may be scheduled with your
 *     choice of {@link java.lang.System#currentTimeMillis} (RTC) or
 *     {@link #elapsedRealtime} (ELAPSED_REALTIME), and cause an
 *     {@link android.content.Intent} broadcast when they occur.
 * </ul>
 */
```

我翻译一下

> - 核心计时设施
> - 这里面有三种不同的时钟可用，不应该混淆
> - [System.currentTimeMillis()](https://link.jianshu.com?t=https://developer.android.com/reference/java/lang/System.html#currentTimeMillis())是标准的"wall"钟(日期和时间)以来表示毫秒。这个时钟可以由用户或者手机网络设置(见setCurrentTimeMillis(long))，所以时间可能不可预知向前或向后跳。这个时钟只应使用符合真实世界的日期和时间和你重要的，比如在一个日历或闹钟应用程序。时间间隔测量应该使用不同的时钟。如果你打算使用[System.currentTimeMillis()](https://link.jianshu.com?t=https://developer.android.com/reference/java/lang/System.html#currentTimeMillis())，则需要留意**ACTION_TIME_TICK ACTION_TIME_TICK**、**ACTION_TIME_CHANGED ACTION_TIME_CHANGED**、**ACTION_TIMEZONE_CHANGED**、**ACTION_TIMEZONE_CHANGED**的Intent广播从而了解时间的变化。
> - uptimeMillis()表示自系统启动时开始计数，以毫秒为单位。返回的是从系统启动到现在这个过程中的处于非休眠期的时间。当系统进入深度睡眠(CPU关闭，屏幕显示器不显示，设备等待外部输入)时，或者空闲或其他省电机制的影响，此时时钟停止，但是该时钟不会被时钟调整。这个方法是大多数间隔时间的基础，例如Thread.sleep(millls)方法、Object.wait(millis)方法、System.nanoTime()都是基于此方法的。该时钟是被保证为单调的，并且适用当间隔不跨越设备睡眠时间间隔定时。大多数方法接受时间戳的值就像uptimeMillis()方法。
> - elapsedRealtime()和elapsedRealtimeNanos()则是返回系统启动后到现在的的时间，并且包括深度睡眠时间。该时钟保证是单调的，即使CPU在省电模式下，该事件也会继续计时。该时钟可以被使用在当测量事件可能跨越系统睡眠的时间段。
> - 有几种控制事件时间机制
>   - 标准的方法像Thread.sleep(millis)
>      和 Object.wait(millis)总是可用的，这些方法使用的是uptimeMillis()时钟，如果设备进入深度休眠，剩余的时间将被推迟直到系统唤醒。这些同步方法可能被Thread.interrupt()中断，所以你必须处理InterruptedException异常。
>   - SystemClock.sleep(millis)是一个类似于Thread.sleep(millis)的实用方法，但是它忽略InterruptedException异常。使用该函数产生的延迟如果你不使用Thread.interrupt()，因为它会保存线程的中断状态。
>   - 在android.os.Handler类中执行异步调用的使用会用到一个绝对的时间或者相对时间的概念。所以Handler使用uptimeMillis()方法获取一个时钟，并且需要调用android.os.Looper来进行事件循环)(通常存在于任何GUI应用程序中)。
>   - 即使设备或者应用程序处于深度休眠或者未运行， android.app.AlarmManage仍然可以发出一次或者重复事件。事件可以根据你的选择来的，事件可以是java.lang.System.currentTimeMilli或者是elapsedRealtime()，并且会导致产生一个Intent的广播。

上面提到了一个概念"关于Android的深度睡眠"，这里就简单介绍下：

### 1、Android的深度睡眠

> 所以Android的深度睡眠，即屏幕关闭后，一段时间不做任何操作，不连接USB，然后在一定的时间后，手机很多服务、进程都睡眠了，不再运行了。

### 2、关于时间间隔

通过上面的注释可以解决我们之前Android一直困扰我们的一个问题：

Android中计算时间间隔的方法：

> 记录开始时间startTime，然后每次回调，获取当前时间 currentTime，计算差值=currentTime - startTime。

而获取当前时间，Android提供两个方法：

> - SystemClock.uptimeMillis()
> - System.currentTimeMillis()

这两个方法的区别：

> - SystemClock.uptimeMillis()：从开机到现在的毫秒数(手机睡眠时间不包括在内)
> - System.currentTimeMillis()：从1970年1月1日 UTC到现在的毫秒数

存在的问题：

> System.currentTimeMillis()获取的时间，可以通过System.setCurrentTimeMillis(long)方法，进行修改，那么在某些情况下，一旦被修改，时间间隔就不准了，所以推荐使用SystemClock.uptimeMillis()

这里大家可以参考**AnimationUtils**类的**currentAnimationTimeMillis()**方法。

## 二、源码解析

```java
public final class SystemClock {
    private static final String TAG = "SystemClock";

    /**
     * This class is uninstantiable.
     */
    private SystemClock() {
        // This space intentionally left blank.
    }

    /**
     * Waits a given number of milliseconds (of uptimeMillis) before returning.
     * Similar to {@link java.lang.Thread#sleep(long)}, but does not throw
     * {@link InterruptedException}; {@link Thread#interrupt()} events are
     * deferred until the next interruptible operation.  Does not return until
     * at least the specified number of milliseconds has elapsed.
     *
     * @param ms to sleep before returning, in milliseconds of uptime.
     */
    public static void sleep(long ms)
    {
        long start = uptimeMillis();
        long duration = ms;
        boolean interrupted = false;
        do {
            try {
                Thread.sleep(duration);
            }
            catch (InterruptedException e) {
                interrupted = true;
            }
            duration = start + ms - uptimeMillis();
        } while (duration > 0);
        
        if (interrupted) {
            // Important: we don't want to quietly eat an interrupt() event,
            // so we make sure to re-interrupt the thread so that the next
            // call to Thread.sleep() or Object.wait() will be interrupted.
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * Sets the current wall time, in milliseconds.  Requires the calling
     * process to have appropriate permissions.
     *
     * @return if the clock was successfully set to the specified time.
     */
    public static boolean setCurrentTimeMillis(long millis) {
        IBinder b = ServiceManager.getService(Context.ALARM_SERVICE);
        IAlarmManager mgr = IAlarmManager.Stub.asInterface(b);
        if (mgr == null) {
            return false;
        }

        try {
            return mgr.setTime(millis);
        } catch (RemoteException e) {
            Slog.e(TAG, "Unable to set RTC", e);
        } catch (SecurityException e) {
            Slog.e(TAG, "Unable to set RTC", e);
        }

        return false;
    }

    /**
     * Returns milliseconds since boot, not counting time spent in deep sleep.
     *
     * @return milliseconds of non-sleep uptime since boot.
     */
    native public static long uptimeMillis();

    /**
     * Returns milliseconds since boot, including time spent in sleep.
     *
     * @return elapsed milliseconds since boot.
     */
    native public static long elapsedRealtime();

    /**
     * Returns nanoseconds since boot, including time spent in sleep.
     *
     * @return elapsed nanoseconds since boot.
     */
    public static native long elapsedRealtimeNanos();

    /**
     * Returns milliseconds running in the current thread.
     * 
     * @return elapsed milliseconds in the thread
     */
    public static native long currentThreadTimeMillis();

    /**
     * Returns microseconds running in the current thread.
     * 
     * @return elapsed microseconds in the thread
     * 
     * @hide
     */
    public static native long currentThreadTimeMicro();

    /**
     * Returns current wall time in  microseconds.
     * 
     * @return elapsed microseconds in wall time
     * 
     * @hide
     */
    public static native long currentTimeMicro();
}
```

通过上面源代码，我们得出两个结论

> - 构造函数是私有的，是不允许外部new的
> - 所有方法都是static的，所以可以直接通过类直接调用。

## 三、方法解析

### (一)、sleep(long)方法



```java
/**
     * Waits a given number of milliseconds (of uptimeMillis) before returning.
     * Similar to {@link java.lang.Thread#sleep(long)}, but does not throw
     * {@link InterruptedException}; {@link Thread#interrupt()} events are
     * deferred until the next interruptible operation.  Does not return until
     * at least the specified number of milliseconds has elapsed.
     *
     * @param ms to sleep before returning, in milliseconds of uptime.
     */
    public static void sleep(long ms)
    {
        long start = uptimeMillis();
        long duration = ms;
        boolean interrupted = false;
        do {
            try {
                Thread.sleep(duration);
            }
            catch (InterruptedException e) {
                interrupted = true;
            }
            duration = start + ms - uptimeMillis();
        } while (duration > 0);
        
        if (interrupted) {
            // Important: we don't want to quietly eat an interrupt() event,
            // so we make sure to re-interrupt the thread so that the next
            // call to Thread.sleep() or Object.wait() will be interrupted.
            Thread.currentThread().interrupt();
        }
    }
```

**1、先来看下注释：**

> 等待一个给定的毫秒数(对uptimemillis())方法返回之前。类似于Thread.sleep(long)方法，却不会抛出InterruptedException。Thread的interrupt()引起的事件将被推迟到下一次中断操作。至少在指定的毫秒数之后才返回。

**2、简析**

我们看了一下代码，其实这个方法本质上就是封装了Thread.sleep()方法，但是，不会抛出InterruptedException

### (二)、setCurrentTimeMillis(long)方法

```java
    /**
     * Sets the current wall time, in milliseconds.  Requires the calling
     * process to have appropriate permissions.
     *
     * @return if the clock was successfully set to the specified time.
     */
    public static boolean setCurrentTimeMillis(long millis) {
        IBinder b = ServiceManager.getService(Context.ALARM_SERVICE);
        IAlarmManager mgr = IAlarmManager.Stub.asInterface(b);
        if (mgr == null) {
            return false;
        }

        try {
            return mgr.setTime(millis);
        } catch (RemoteException e) {
            Slog.e(TAG, "Unable to set RTC", e);
        } catch (SecurityException e) {
            Slog.e(TAG, "Unable to set RTC", e);
        }

        return false;
    }
```

**1、先来看下注释：**

> 设置系统时针，以毫秒为单位。要求调用过程具有适当的权限。

**2、简析**

> 通过代码我们知道，利用AIDL获取IAlarmManager的客户端，然后调用IAlarmManager的setTime()方法

**所以这个方法就是用来设置系统指针的。**

### (三)、uptimeMillis()方法



```java
    /**
     * Returns milliseconds since boot, not counting time spent in deep sleep.
     *
     * @return milliseconds of non-sleep uptime since boot.
     */
    native public static long uptimeMillis();
```

> 返回的是系统从启动到当前的处于非休眠期的时间。

### (四)、elapsedRealtime()方法



```java
    /**
     * Returns milliseconds since boot, including time spent in sleep.
     *
     * @return elapsed milliseconds since boot.
     */
    native public static long elapsedRealtime();
```

> 返回的是系统从启动到现在的时间

### (五)、elapsedRealtimeNanos()方法



```java
    /**
     * Returns nanoseconds since boot, including time spent in sleep.
     *
     * @return elapsed nanoseconds since boot.
     */
    public static native long elapsedRealtimeNanos();
```

> 返回系统启动到现在的纳秒数，包含休眠时间

### (六)、elapsedRealtimeNanos()方法



```java
    /**
     * Returns milliseconds running in the current thread.
     * 
     * @return elapsed milliseconds in the thread
     */
    public static native long currentThreadTimeMillis();
```

> 返回当前线程运行的毫秒数

### (七)、currentThreadTimeMicro()方法



```java
    /**
     * Returns microseconds running in the current thread.
     * 
     * @return elapsed microseconds in the thread
     * 
     * @hide
     */
    public static native long currentThreadTimeMicro();
```

> 返回当前线程中已经运行的时间(单位是milliseconds毫秒)

### (八)、currentThreadTimeMicro()方法



```java
    /**
     * Returns current wall time in  microseconds.
     * 
     * @return elapsed microseconds in wall time
     * 
     * @hide
     */
    public static native long currentTimeMicro();
```

> 返回当前线程中已经运行的时间(单位是microseconds微秒)

## 四、JNI和Native对应的代码

通过我们之前的文章[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)我们知道，SystemClock对应的JNI文件是[android_os_SystemClock.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_SystemClock.cpp)

代码很简单，我就不说了
 代码如下：



```cpp
/*
 * System clock functions.
 */

#include <sys/time.h>
#include <limits.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>

#include "JNIHelp.h"
#include "jni.h"
#include "core_jni_helpers.h"

#include <sys/time.h>
#include <time.h>

#include <utils/SystemClock.h>

namespace android {

/*
 * native public static long uptimeMillis();
 */
static jlong android_os_SystemClock_uptimeMillis(JNIEnv* env,
        jobject clazz)
{
    return (jlong)uptimeMillis();
}

/*
 * native public static long elapsedRealtime();
 */
static jlong android_os_SystemClock_elapsedRealtime(JNIEnv* env,
        jobject clazz)
{
    return (jlong)elapsedRealtime();
}

/*
 * native public static long currentThreadTimeMillis();
 */
static jlong android_os_SystemClock_currentThreadTimeMillis(JNIEnv* env,
        jobject clazz)
{
    struct timespec tm;

    clock_gettime(CLOCK_THREAD_CPUTIME_ID, &tm);

    return tm.tv_sec * 1000LL + tm.tv_nsec / 1000000;
}

/*
 * native public static long currentThreadTimeMicro();
 */
static jlong android_os_SystemClock_currentThreadTimeMicro(JNIEnv* env,
        jobject clazz)
{
    struct timespec tm;

    clock_gettime(CLOCK_THREAD_CPUTIME_ID, &tm);

    return tm.tv_sec * 1000000LL + tm.tv_nsec / 1000;
}

/*
 * native public static long currentTimeMicro();
 */
static jlong android_os_SystemClock_currentTimeMicro(JNIEnv* env,
        jobject clazz)
{
    struct timeval tv;

    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000LL + tv.tv_usec;
}

/*
 * public static native long elapsedRealtimeNano();
 */
static jlong android_os_SystemClock_elapsedRealtimeNano(JNIEnv* env,
        jobject clazz)
{
    return (jlong)elapsedRealtimeNano();
}

/*
 * JNI registration.
 */
static JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "uptimeMillis",      "()J",
            (void*) android_os_SystemClock_uptimeMillis },
    { "elapsedRealtime",      "()J",
            (void*) android_os_SystemClock_elapsedRealtime },
    { "currentThreadTimeMillis",      "()J",
            (void*) android_os_SystemClock_currentThreadTimeMillis },
    { "currentThreadTimeMicro",       "()J",
            (void*) android_os_SystemClock_currentThreadTimeMicro },
    { "currentTimeMicro",             "()J",
            (void*) android_os_SystemClock_currentTimeMicro },
    { "elapsedRealtimeNanos",      "()J",
            (void*) android_os_SystemClock_elapsedRealtimeNano },
};
int register_android_os_SystemClock(JNIEnv* env)
{
    return RegisterMethodsOrDie(env, "android/os/SystemClock", gMethods, NELEM(gMethods));
}

}; // namespace android
```



# Handler机制之Looper与Handler简介

## 一、 概述

[Android Handler官网](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Handler.html)

> 工欲善其事必先利其器，在Android的源码世界里面，有两大利器，一是Binder IPC机制，关于Binder的内容上面已经讲过了。另一个便是Handler消息机制(由Handler/Looper/MessageQueue等构成)。Android有大量的消息驱动方法来进行交互，就像Android的四大组件(Activity、Service、Broadcast、ContentProvider)的启动过程交互，都离不开Handler的消息机制，所以Android系统某种意义上说也是一种以消息驱动的系统。

那Android为什么要采用Handler的消息机制那？

> 答： 在Android中，只有主线程才能更新UI，但是主线程不能进行耗时操作，否则会产生ANR异常，所以常常把耗时操作放到其他子线程进程。如果在子线程中需要更新UI，一般都是通过Handler发送消息，主线接受消息并进行相应的逻辑处理。当然除了直接使用Handler，还可以用View的post()方法、Activity的runOnUIThread()方法来更新UI和AsyncTask其实他们本质也是使用了Handler来实现的。后面我们会单独说他们

要理解Handler的消息机制，就不得不说Handler/Looper/Message/MessageQueue/Message这四4个类，下面我们先大概了解下这几个类

## 二、 相应类的理解

关于类的理解，还是以官网上的类介绍为主

### (一) Handler

[Handler](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Handler.html)

**1、什么是Handler**

> A Handler allows you to send and process [Message](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Message.html)
>  and Runnable objects associated with a thread's [MessageQueue](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/MessageQueue.html)
>  . Each Handler instance is associated with a single thread and that thread's message queue. When you create a new Handler, it is bound to the thread / message queue of the thread that is creating it -- from that point on, it will deliver messages and runnables to that message queue and execute them as they come out of the message queue.

简单翻一下：

> Handler是一个可以通过关联一个消息队列来发送和处理消息，发送或处理Runnable对象的一个处理程序，每个Handler都关联一个单个的线程和消息队列，当你创建一个新的Handler的时候它就将绑定到一个线程或线程上的消息队列，从那时起，这个Handler就将为这个消息队列提供消息或Runnable对象，处理消息队列释放出来的消息或Runnable对象。

**2、Handler有什么用**

> There are two main uses for a Handler: (1) to schedule messages and runnables to be executed as some point in the future; and (2) to enqueue an action to be performed on a different thread than your own.

简单翻译 一下：

> Handler有两个主要的用途
>
> - 1 安排消息和Runnable对象在未来之星
> - 2 将你的一个动作放在不同的线程上执行

**3、怎么使用Handler**

> Scheduling messages is accomplished with the post(Runnable), postAtTime(Runnable, long), postDelayed(Runnable, long), sendEmptyMessage(int), sendMessage(Message), sendMessageAtTime(Message, long), and sendMessageDelayed(Message, long) methods. The post versions allow you to enqueue Runnable objects to be called by the message queue when they are received; the sendMessage versions allow you to enqueue a Message object containing a bundle of data that will be processed by the Handler's handleMessage(Message) method (requiring that you implement a subclass of Handler).

翻译一下：

> 通过重写**post(Runnable)**、**postAtTime(Runnable, long)**、**postDelayed(Runnable, long)**、**sendEmptyMessage(int)**、**sendMessage(Message)**、**sendMessageAtTime(Message, long)**、**sendMessageDelayed(Message, long)**来完成发送消息。当前版本允许你通过消息队列接受一个Runnable对象，sendMessage方法当前版本允许你将一个包的数据通过消息队列的方式处理，但是你需要重写Handler的handleMessage方法

> When posting or sending to a Handler, you can either allow the item to be processed as soon as the message queue is ready to do so, or specify a delay before it gets processed or absolute time for it to be processed. The latter two allow you to implement timeouts, ticks, and other timing-based behavior.

翻译一下：

> 当你发送一个Handler时，你可以使消息队列(MessageQueue)尽快去处理已经准备好的条目，或者指定一个延迟处理的时间或者指定时间处理，后两者允许你实现超时，Ticks(系统相对的时间单位)和其他时间段为基础的行为。

> When a process is created for your application, its main thread is dedicated to running a message queue that takes care of managing the top-level application objects (activities, broadcast receivers, etc) and any windows they create. You can create your own threads, and communicate back with the main application thread through a Handler. This is done by calling the same post or sendMessage methods as before, but from your new thread. The given Runnable or Message will then be scheduled in the Handler's message queue and processed when appropriate.

> 即当一个进程被应用程序创建时，它的主线程会运行一个消息队列负责管理它创建的高层应用程序对象(如Activity、Broadcast Receiver等)和任何它的窗口创建的对象，你可以通过一个Handler，创建自己的线程来实现与主线程之间的交互，但前提是你得在你的线程重写sendMessage方法并写上一行行代码，这样你给定的Runnable或者Message将被MessageQueue(消息队列)预定，并在合适的时间处理

注意：有心的同学会注意，我们在进行Handler源码分析的时候，可以知道Handler中的Message是分2中类型的，一种是Message，就是Message对象，一种是CallMessage，就是Runnable对象，但是MessageQueue中只支持DataMessage，再插入到MessageQueue的时候，会把Runnable对象封装到Message对象中。

### (二) Looper

[Looper](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Looper.html)

> Class used to run a message loop for a thread. Threads by default do not have a message loop associated with them; to create one, call prepare() in the thread that is to run the loop, and then loop() to have it process messages until the loop is stopped.
>  Most interaction with a message loop is through the Handler class.
>  This is a typical example of the implementation of a Looper thread, using the separation of prepare() and loop() to create an initial Handler to communicate with the Looper.

```java
  class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```

翻译一下：

> 用于为线程运行消息循环的类。默认线程没有与它们相关联的消息喜欢；所以要在运行循环的线程中调用prepare()，然后调用loop()让它循环处理消息，直到循环停止。
>  与消息循环的交互是通过Handler类
>  下面这个是一个典型的Looper线程实例，它使用prepar()和loop()的分割来创建一个初始的Handler与Looper进行通信。

## 三、 Handle原理详解

### (一)、模型

Handler的消息机制主要包含：

> - [Message](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java)：消息
> - [MessageQueue](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)：消息队列的
> - [Handler](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)：消息管理类向消息池发送各种消息事件
> - [Looper](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)：不断的循环执行(Looper.loop)，按分发机制将消息分发给目标处理者

### (二)、架构图

![img](https:////upload-images.jianshu.io/upload_images/5713484-8f23829636674617.png?imageMogr2/auto-orient/strip|imageView2/2/w/731/format/webp)

Handle机制架构图.png

### (三)、举例说明

我们先用一个典型的关于Handler/Looper的线程为例展示
 代码如下：



```java
class LooperThread extends Thread {
    public Handler mHandler;

    public void run() {
        //情景1
        Looper.prepare();   
        //情景3
        mHandler = new Handler() {  
            public void handleMessage(Message msg) {
                //TODO    定义消息处理逻辑. 
                //情景4
                Message msg=Message.obtain();
            }
        };
       // 情景2
        Looper.loop();  
    }
}
```

OK，那我们便依照上面的的四个情景依次展开

#### 1、情景1   Looper.prepare();

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的82行



```dart
     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }
```

看下prepare()方法的注释，翻译一下：

> 初始化当前线程和Looper，这样可以在实际开始启动循环(loop())之前创建一个Handler并且关联一个looper。确保在先调用这个方法，然后调用loop()方法，并且通过调用quit()结束。

我们发现prepare()方法里面什么都没有调用，就是调用了prepare(true)，那我们继续跟踪下

**1.1、 prepare(boolean)方法**

这里面的入参boolean表示Looper是否允许退出，true就表示允许退出，对于false则表示Looper不允许退出。
 代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的86行

```csharp
    private static void prepare(boolean quitAllowed) {
         //每个线程只允许执行一次该方法，第二次执行的线程的TLS已有数据，则会抛出异常。
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //创建Looper对象，并且保存到当前线程的TLS区域。
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

那sThreadLocal是什么？我们来看下

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的86行

```dart
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

原来是ThreadLocal啊，关于ThreadLocal的内容请参考[Android Handler机制2之ThreadLocal](https://www.jianshu.com/p/c2a482b48d17)

我们看到上面调用了new Looper(quitAllowed)，那我们就来看下这个Looper的构造函

**1.2、 Looper(boolean)构造函数**

```java
   private Looper(boolean quitAllowed) {
        // 创建MessageQueue对象
        mQueue = new MessageQueue(quitAllowed);
        // 记录当前线程
        mThread = Thread.currentThread();
    }
```

通过上面代码，我们发现就是创建了一个MessageQueue，并且把当前线程赋值给本地变量的mThread。

> - 1 这样就实现了Looper和MessageQueue的关联
> - 2 这样就实现了Thread和Looper的关联

**1.3、 Looper类的思考**

因为是构造函数，我们看下Looper这个类的结构，如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-741a4f7884310da1.png?imageMogr2/auto-orient/strip|imageView2/2/w/920/format/webp)



> 咦，大家发现没，Loop类就一个构造函数，也就是我们上面的说的Looper(boolean) 这个构造函数。而且这个构造函数也是private。我们又发现在这个私有的Looper(boolean)的构造函数，只有在**private static void prepare(boolean quitAllowed)**调用，而这个静态方法也是private的，我们看下这个方法在哪里被调用

![img](https:////upload-images.jianshu.io/upload_images/5713484-802dfd9b56cf7566.png?imageMogr2/auto-orient/strip|imageView2/2/w/1094/format/webp)

prepare(boolean)调用处.png

所以我们得出结论

私有的构造函数 => 私有的静态方法prepare(boolean) => static prepare()/prepareMainLooper()

所以我们知道，Looper这个类的对象不能直接创建，必须通过Looper来的两个静态方法prepare()/prepareMainLooper()来间接创建

**1.4、prepareMainLooper()方法**

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的99行

```dart
    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
         // 设置不允许退出的Looper
        prepare(false);
        synchronized (Looper.class) {
            //将当前的Looper保存为Looper。每个线程只允许执行一次
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

① 注释

首先翻译一下注释

> 初始化当前当前线程的looper。并且标记为一个程序的主Looper。由Android环境来创建应用程序的主Looper。因此这个方法不能由咱们来调用。另请参阅prepare()

② 方法解析

> - 首先 通过方法我们看到调用了prepare(false);**注意这里的入参是false**
> - 其次 做了sMainLooper的非空判断，如果是有值的，直接抛异常，因为这个sMainLooper必须是空，因为主线程有且只能调用一次prepareMainLooper()，如果sMainLooper有值，怎说说明prepareMainLooper()已经被调用了，而sMainLooper的赋值是由myLooper来执行，
> - 最后调用myLooper()方法来给sMainLooper进行赋值。

③ sMainLooper

上面提到了一个变量时sMainLooper，那sMainLooper到底是什么？我们来看下源代码.
 代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的 69行



```cpp
  private static Looper sMainLooper;  // guarded by Looper.class
```

通道源码我们知道原来就是Looper对象啊。但是它是静态的。而在Java7之前，静态变量存在**永久代\*(PermGen)*。在Hospot JVM上，PermGen 就是方法区；在Java7之后，将变量的存储转移到了堆。所以说这个sMainLooper就是主线程的Looper。所以只有通过prepareMainLooper()就可以给主线程Looper赋值了。

**1.5、myLooper()方法**

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的173行



```csharp
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

> 这里的**sThreadLocal.get()**是和**prepare(boolean)**方法里面的**sThreadLocal.set(new Looper(quitAllowed));**一一对应的。而在prepareMainLooper()方法里面，

提出一个问题，在prepareMainLooper里面调用myLooper()，那么myLooper()方法的返回有没有可能为null?

> 答：第一步就是调用**prepare(false);**，所以说**myLooper()**这个方法的返回值是一定有值的。

#### 2、情景2   Looper.loop();

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的 122行



```java
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
         // 获取TLS存储的Looper对象
        final Looper me = myLooper();
        //没有Looper 对象，直接抛异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //获取当前Looper对应的消息队列
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        // 确保权限检查基于本地进程，而不是基于最初调用进程
        final long ident = Binder.clearCallingIdentity();
        // 进入 loop的主循环方法
       // 一个死循环，不停的处理消息队列中的消息，消息的获取是通过MessageQueue的next()方法实现
        for (;;) {
             // 可能会阻塞
            Message msg = queue.next(); // might block
             // 如果没有消息，则退出循环
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            // 默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
           // 用于分发消息，调用Message的target变量(也就是Handler了)的dispatchMessage方法来处理消息
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            // 确保分发过程中identity不会损坏
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                // 打印identiy改变的log，在分发消息过程中是不希望身份被改变
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            // 将Message放入消息池
            msg.recycleUnchecked();
        }
    }
```

> loop进入循环模式，不断重复下面的操作，直到没有消息时退出循环
>
> - 读取MessageQueue的下一条Message
> - 把Message分发给相应的target
> - 再把分发后的Message回到消息池，以便重复利用

**这里有几个重要方法，我们分别会在后面的文章中说明**

> - MessageQueue.next()方法： 读取下一条Message，有阻塞
>   - 会在[Android Handler机制6之MessageQueue简介](https://www.jianshu.com/p/14ba1cb98b08)中讲解**MessageQueue.next()**方法
> - Message.Handler.dispatchMessage(Message msg)方法：消息分发
>   - 会在[Android Handler机制7之消息传递流程](https://www.jianshu.com/p/9a7e85878727)中讲解 **Message.Handler.dispatchMessage(Message msg)**方法
> - Message.recycleUnchecked()方法：消息放入消息池
>   - 会在[Android Handler机制5之Message简介与消息对象对象池](https://www.jianshu.com/p/e271ee639b68)中讲解**Message.recycleUnchecked()**方法

#### 3 Looper的退出循环方法

> 上面了开启循环的方法，那么怎么退出循环？Looper里面退出循环有两个方法分别是**quit()**和**quitSafely()**方法

**3.1、Looper.quit()方法**

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的 227行

```dart
    /**
     * Quits the looper.
     * <p>
     * Causes the {@link #loop} method to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @see #quitSafely
     */
    public void quit() {
        mQueue.quit(false);
    }
```

**① 注释**

简单的翻译一下：

> - 退出循环
> - 将终止(loop()方法)而不处理消息队列中的任何更多消息。在调用quit()后，任何尝试去发送消息都是失败的。例如Handler.sendMessage(Message)方法将返回false。因为循环终止之后一些message可能会被无法传递，所以这个方法是不安全的。可以考虑使用quitSafely()方法来确保所有的工作有序地完成。

**② 方法内容：**

Looper的quit()方法内部的本质是调用mQueue的quit()的方法，入参是false

**3.2、Looper.quitSafely()方法**

代码在[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java)的 227行



```dart
    /**
     * Quits the looper safely.
     * <p>
     * Causes the {@link #loop} method to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * However pending delayed messages with due times in the future will not be
     * delivered before the loop terminates.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p>
     */
    public void quitSafely() {
        mQueue.quit(true);
    }
```

① 注释

简单的翻译一下：

> - 安全退出循环
> - 调用quitSafely()方法会使循环结束，只要消息队列中已经被传递的所有消息都将被处理。然而，在循环结束之前，将来不会提交处理延迟消息。
> - 调用退出后，所有尝试去发送消息都将失败。就像调用Handler.sendMessage(Message)将返回false。

② 方法内容：

Looper的quit()方法内部的本质是调用mQueue的quit()的方法，入参是true

**3.3、MessageQueue.quit(boolean)方法**

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)的 413行

```java
    void quit(boolean safe) {
        //当mQuitAllowed为false，表示不运行退出，强行调用quit()会超出异常
        //mQuitAllowed 是在Looper构造函数里面构造MessageQueue()以参数参进去的
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            // 防止多次执行退出操作
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                //移除尚未触发的所有消息
                removeAllFutureMessagesLocked();
            } else {
                //移除所有消息
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
             //mQuitting=false，那么认定mPtr!=0
            nativeWake(mPtr);
        }
    }
```

消息退出的方式：

> - 当safe=true，只移除尚未触发的所有消息，对于正在触发的消息并不移除
> - 当safe=false，移除所有消息

#### 4、情景3:创建Handler对象

看到Handler对象，首先要看下Handler的构造函数



![img](https:////upload-images.jianshu.io/upload_images/5713484-4d8e0f3529b7a338.png?imageMogr2/auto-orient/strip|imageView2/2/w/774/format/webp)

Handler的构造函数.png

好多的构造函数啊，一共7个，其中6个有参的构造函数，1个有参的构造函数，所有的构造函数都是public的，为了大家更好的理解，我将这些构造函数编号如下：

> - ① public Handler()
> - ② public Handler(Callback callback)
> - ③ public Handler(Looper looper)
> - ④ public Handler(Looper looper, Callback callback)
> - ⑤ public Handler(boolean async)
> - ⑥ public Handler(Callback callback, boolean async)
> - ⑦ public Handler(Looper looper, Callback callback, boolean async)

上面的7个Handler的构造函数大体上可以分为两大类:

> - 两个参数的Handler构造函数
> - 三个参数的Handler构造函数

##### 4.1 两个参数的Handler构造函数

这里说一下 ① ② ⑤ ⑥ 这四个是"两个参数的Handler构造函数"，为什么这么说，大家可以看下源码

**4.1.1构造函数①，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  113行**



```dart
    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }
```

> 他内部其实是调用的**构造函数⑥**，只不过第一个入参为null，第二个参数为false

**4.1.2 构造函数②，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  127行**



```dart
    /**
     * Constructor associates this handler with the {@link Looper} for the
     * current thread and takes a callback interface in which you can handle
     * messages.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     *
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Callback callback) {
        this(callback, false);
    }
```

> 他内部其实是调用的**构造函数⑥**，只不过第一个入参为callback，第二个参数为false

**4.1.3 构造函数⑤，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  167行**



```dart
    /**
     * Use the {@link Looper} for the current thread
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(boolean async) {
        this(null, async);
    }
```

> 他内部其实是调用的**构造函数⑥**，只不过第一个入参为null，第二个参数为async

综上所述我们知道 构造函数① ② ⑤ ，归根结底都是调用的是构造函数⑥，那我们就来看下构造函数⑥

**4.1.4 构造函数⑥，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  188行**



```dart
    /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            //如果是匿名类、内部类、局不类(方法内部)，且没有声明为STATIC，则存在内存泄露风险，所以要打印日志提醒开发者
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        //获取当前线程的Looper。
        mLooper = Looper.myLooper();
        //如果当前线程没有Looper，则说明没有调用Looper.prepare()，抛异常
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //把Looper的Queue赋值给Handler的mQueue
        mQueue = mLooper.mQueue;
         // mCallback 赋值
        mCallback = callback;
         // mAsynchronous 赋值
        mAsynchronous = async;
    }
```

老规矩，先来看下注释，简单翻译如下：

> - 使用当前线程的Looper，使用这个指定的Callback接口，并且设置这个handler是否是异步的。
> - 除非在构造函数设置为异步的，否则默认的的Handler是同步的。
> - 异步消息相对于同步消息的而言的，表示消息不会受到中断或者事件的影响其全局顺序。异步消息是不受到MessageQueue.enqueueSyncBarrier(long)的同步障碍影响。

通过上面代码我们我们知道

> - Handler默认采用的是当前线程TLS中的Looper对象，只要执行了Looper.prepare()方法，就可以可以获取有效的Looper对象，同时我们上面说了Looper的构造函数关联了MessageQueue和Thread。而在Handler的构造函数里面Handler里面的MessageQueue即是Looper的MessageQueue。
>    **所以在同一个线程里面(已经调用了Looper.prepare())，Handler里面的MessageQueue和的Looper的MessageQueue指向了同一个对象**。

所以我们也说这这个方法让Handler对象和Looper对象绑定在一起了。

上面提到了Callback，是个什么？让我们来看下
 代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  80行



```java
    /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
```

简单翻译一下：

> 当你实例化Handler的时候可以使用Callback接口，这样可以避免你自己去去实现Handler的子类

有点复杂，大家怎么理解？大家平时写Handler不是一般是写一个匿名内部类居多吧，如下：



```java
    private Handler handler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
```

如果存在一个回调的需求，就可以直接使用Callback了，就不用去实现Handler的子类了。

##### 4.2 三个参数的Handler构造函数

这里说一下③、④、⑦ 这三个是"三个参数的Handler构造函数"，为什么这么说，依旧看下源码

**4.2.1构造函数③，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  136行**



```java
    /**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }
```

> 他内部其实是调用的**构造函数⑦**，只不过第一个入参为looper，第二个参数为null，第三个参数为false。

**4.2.2构造函数④，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  147行**



```php
    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     */
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }
```

> 他内部其实是调用的**构造函数⑦**，只不过第一个入参为looper，第二个参数为callback，第三个参数为false。

**4.2.3构造函数⑦，代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)的  227行**

```dart
    /**
     * Use the provided {@link Looper} instead of the default one and take a callback
     * interface in which to handle messages.  Also set whether the handler
     * should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param looper The looper, must not be null.
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

老规矩，先来看下注释，简单翻译如下：

> - 不使用默认的Looper，使用由第一个入参提供的Looper，并采取回调接口Callback来处理消息，同时也可以设置是否是异步的。
> - Handler默认是同步的，如果在构造函数里面设置了异步，才会变成异步的。
> - 异步消息相对于同步消息的而言的，表示消息不会受到中断或者事件的影响其全局顺序。异步消息是不受到MessageQueue.enqueueSyncBarrier(long)的同步障碍影响。

通过上面代码我们我们知道

> - 首先根据给定的Looper赋值本地变量mLooper和mQueue。这样实现了Handler和Looper的关联

##### 4.3 Handler构造函数总结

综合7个构造函数我们发现其实构造函数的本质都是给下面四个本地变量赋值mLooper、mQueue、mCallback、mAsynchronous。那我们就一次分析下：

> - 1、如果在构造函数中没有给出Looper，则使用默认的Looper，通过Looper.mQueue来给mQueue赋值，实现Handler、Looper、MessageQueue三者绑定。
>    -通过构造函数来设置Callback的回调接口，不设置则为null
>    -通告mAsynchronous设控制是同步的还是异步的，而mAsynchronous的值默认是false的，这个mAsynchronous可以通过构造函数来设置。

总结如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-a4c9a5a5bc0c6129.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Handler构造总结.png

有同学说图片看不清，按我就弄个表格

| Hander本地变量             |    ① Handler()    | ② Handler(Callback) | ③ Handler(Looper) | ④ Handler(Looper, Callback) | ⑤ Handler(boolean) | ⑥ Handler(Callback, boolean) | ⑦ Handler(Looper,Callback, boolean) |
| -------------------------- | :---------------: | ------------------: | ----------------: | --------------------------: | -----------------: | ---------------------------: | ----------------------------------: |
| mLooper(Looper对象)        | Looper.myLooper() |   Looper.myLooper() |            looper |                      looper |  Looper.myLooper() |            Looper.myLooper() |                              looper |
| mQueue(MessageQueue对象)   |       null        |            callback |              null |                    callback |               null |                     callback |                            callback |
| mAsynchronous(boolean类型) |       false       |               false |             false |                       false |              async |                        async |                               async |

## 四、 主线程的Looper的初始化

> 上面说了很多，那么主线程的Looper是什么时候初始化的那？是在系统启动的时候，初始化的。

代码在[ActivityThread.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/app/ActivityThread.java)  5401行



```csharp
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        /*** 重点 */
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }


        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        /*** 重点 */
        Looper.loop();
       throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

PS:这里补充一个概念ActivityThread不是一个thread，ActivityThread是一个类，大家仔细看他的源代码ActivityThread并没有实现Thread，大家千万不要跑偏了。

通过上面的代码，我们发现

> - 首先调用了Looper的静态方法prepareMainLooper()给主线程绑定一个Looper，同时设置Looper对应的MessageQueue对象的mQuitAllowed为false，则该messageQueue是不能退出的。
> - 其次调用Looper.loop();开启循环

通过上面两个步骤开启了主线程的Hanlder机制



# Handler机制之Message简介与消息对象对象池

## 一、 Message和MessageQueue类注释

> 为了让大家更好的理解谷歌团队设计这个两个类Message和MessageQueue的意图，我们还是从这个两个类的类注释开始

### (一) Message.java

[Message](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/Message.html)

> Defines a message containing a description and arbitrary data object that can be sent to a Handler. This object contains two extra int fields and an extra object field that allow you to not do allocations in many cases.
>  While the constructor of Message is public, the best way to get one of these is to call Message.obtain() or one of the Handler.obtainMessage() methods, which will pull them from a pool of recycled objects.

翻译一下：

> 定义一个可以发送给Handler的描述和任意数据对象的消息。此对象包含两个额外的int字段和一个额外的对象字段，这样就可以使用在很多情况下不用做分配工作。
>  尽管Message的构造器是公开的，但是获取Message对象的最好方法是调用Message.obtain()或者Handler.obtainMessage()，这样是从一个可回收的对象池中获取Message对象。

至此Java层面Handler机制中最重要的四个类大家有了一个初步印象。下面咱们源码跟踪一下

## 二、获取Message成员变量解析

### (一) 成员变量 what

> Message用一个标志来区分不同消息的身份，不同的Handler使用相同的消息不会弄混，一般使用16进制形式来表示，阅读起来比较容易

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 39行



```dart
    /**
     * User-defined message code so that the recipient can identify 
     * what this message is about. Each {@link Handler} has its own name-space
     * for message codes, so you do not need to worry about yours conflicting
     * with other handlers.
     */
    public int what;
```

注释翻译：

> 用户定义的Message的标识符用以分辨消息的内容。Hander拥有自己的消息代码的命名空间，因此你不用担心与其他的Handler冲突。

### (二) 成员变量 arg1和arg2

> arg1和arg2都是Message类的可选变量，可以用来存放两个整数值，不用访问obj对象就能读取的变量。

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 46行



```dart
    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg1; 

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg2;
```

因为两个注释一样，我就不重复翻译了，翻译如下：

> 如果你仅仅是保存几个整形的数值，相对于使用setData()方法，使用arg1和arg2是较低成本的替代方案。

### (三) 成员变量 obj

> obj 用来保存对象，接受纤细后取出获得传送的对象。

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 65行



```dart
    /**
     * An arbitrary object to send to the recipient.  When using
     * {@link Messenger} to send the message across processes this can only
     * be non-null if it contains a Parcelable of a framework class (not one
     * implemented by the application).   For other data transfer use
     * {@link #setData}.
     * 
     * <p>Note that Parcelable objects here are not supported prior to
     * the {@link android.os.Build.VERSION_CODES#FROYO} release.
     */
    public Object obj;
```

注释翻译：

> - 将一个独立的对象发送给接收者。当使用Messenger去送法消息，并且这个对象包含Parcelable类的时候，它必须是非空的。对于其他数据的传输，建议使用setData()方法
> - 请注意，在Android系统版本FROYO(2.2)之前不支持Parcelable对象。

### (四)  其他成员变量

其他变量我就不一一解释了，大家就直接看注释吧



```java
    // 回复跨进程的Messenger 
    public Messenger replyTo;

    // Messager发送这的Uid
    public int sendingUid = -1;

    // 正在使用的标志值 表示当前Message 正处于使用状态，当Message处于消息队列中、处于消息池中或者Handler正在处理Message的时候，它就处于使用状态。
    /*package*/ static final int FLAG_IN_USE = 1 << 0;

    // 异步标志值 表示当前Message是异步的。
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;

    // 消息标志值 在调用copyFrom()方法时，该常量将会被设置，其值其实和FLAG_IN_USE一样
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

    // 消息标志，上面三个常量 FLAG 用在这里
    /*package*/ int flags;

    // 用于存储发送消息的时间点，以毫秒为单位
    /*package*/ long when;

    // 用于存储比较复杂的数据
    /*package*/ Bundle data;
    
    // 用于存储发送当前Message的Handler对象，前面提到过Handler其实和Message相互持有引用的
    /*package*/ Handler target;
    
    // 用于存储将会执行的Runnable对象，前面提到过除了handlerMessage(Message msg)方法，你也可以使用Runnable执行操作，要注意的是这种方法并不会创建新的线程。
    /*package*/ Runnable callback;
 
    // 指向下一个Message，也就是线程池其实是一个链表结构
    /*package*/ Message next;

    // 该静态变量仅仅是为了给同步块提供一个锁而已
    private static final Object sPoolSync = new Object();

    //该静态的Message是整个线程池链表的头部，通过它才能够逐个取出对象池的Message
    private static Message sPool;

    // 该静态变量用于记录对象池中的Message的数量，也就是链表的长度
    private static int sPoolSize = 0;
  
    // 设置了对象池中的Message的最大数量，也就是链表的最大长度
    private static final int MAX_POOL_SIZE = 50;

     //该版本系统是否支持回收标志位
    private static boolean gCheckRecycle = true;
```

## 三、获取Message对象

### (一)、Message构造函数

如果想获取Message对象，大家第一印象肯定是找Message的构造函数，那我们就来看下Message的构造函数。代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 475行



```cpp
    /** Constructor (but the preferred way to get a Message is to call {@link #obtain() Message.obtain()}).
    */
    public Message() {
    }
```

发现代码里面什么都没有

那我们看下注释，简单翻译一下：

> 构造函数，但是获取Message的首选方法是通过Message.obtain()来调用

其实在上面解释Message的注释时也是这样说的，说明Android官方团队是推荐使用Message.obtain()方法来获取Message对象的，那我们就来看下Message.obtain()

### (二)、Message.obtain()方法

我们来看下[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 类的结构图

![img](https:////upload-images.jianshu.io/upload_images/5713484-952c64fcc0bf394f.png?imageMogr2/auto-orient/strip|imageView2/2/w/880/format/webp)



Message.居然有8个obtain函数
 为了方便后续的跟踪，也将这8个方法编号，分别如下

> - ① public static Message obtain()
> - ② public static Message obtain(Message orig)
> - ③ public static Message obtain(Handler h)
> - ④ public static Message obtain(Handler h, Runnable callback)
> - ⑤ public static Message obtain(Handler h, int what)
> - ⑥ public static Message obtain(Handler h, int what, Object obj)
> - ⑦ public static Message obtain(Handler h, int what, int arg1, int arg2)
> - ⑧ public static Message obtain(Handler h, int what, int arg1, int arg2, Object obj)

这里我们也像Handler一样，分为两大类

> - **无参的obtain()方法**
> - **有参的obtain()方法**

在讲解无参的obtain()的时候很有必要先了会涉及一个概念**“Message对象池”**，所以我们就合并一起讲解了

## 四、Message的消息对象池和无参的obtain()方法

先来看一下下面 无参的obtain()方法的代码

### 1、① public static Message obtain(Message orig)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 122行



```csharp
    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
       // 保证线程安全
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                 // flags为移除使用标志
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

老规矩，先看下注释，翻译如下：

> 从全局的pool返回一个实例化的Message对象。这样可以避免我们重新创建冗余的对象。

等等上面提到了一个名词pool，我们通常理解为"池"，我们看到源代码的里面有一个变量是"sPool"，那么"sPool"，这里面就涉及到了Message的设计原理了，在Message里面是有一个"对象池"，下面我们就详细了解下

### 2、sPool

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 111行



```cpp
   private static Message sPool;
```

> - 好像也没什么嘛？就是一个Message对象而已，所以sPool默认是null。
> - 这时候我们再来看上面**public static Message obtain()**方法，我们发现**public static Message obtain()**就是直接new了Message 直接返回而已。好像很简单的样子，大家心里肯定感觉"咱们是不是忽略了什么？"
> - 是的，既然官方是不推荐使用new Message的，因为这样可能会重新创建冗余的对象。所以我们推测大部分情况下sPool是不为null的。那我们就反过来看下，来全局找下sPool什么时候被赋值的

我们发现除了**public static Message obtain()**里面的**if (sPool != null) {}**里面外，还有**recycleUnchecked**给sPool赋值了。那我们就来看下这个方法

### 3、recycleUnchecked()

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 291行



```csharp
   /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        // 添加正在使用标志位，其他情况就除掉
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;
        //拿到同步锁，以避免线程不安全
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

### 4、消息对象池的理解

为了让大家更好的理解，我把recycleUnchecked()和obtain()合在一起，省略一些不重要的代码
 代码如下：



```java
void recycleUnchecked() {
                ...
        if (sPoolSize < MAX_POOL_SIZE) {
                // 第一步
                next = sPool;
                 // 第二步
                sPool = this;
                 // 第三步
                sPoolSize++;
                 ...
         }
    }

public static Message obtain() {
    synchronized (sPoolSync) {
        //第一步
        if (sPool != null) {
            // 第二步
            Message m = sPool;
            // 第三步
            sPool = m.next;
            // 第四步
            m.next = null;
            // 第五步
            m.flags = 0; 
            // 第六步
            sPoolSize--;
            return m;
        }
    }
}
```

#### 4.1、recycleUnchecked()的理解

假设消息对象池为空，从new message开始，到这个message被取出使用后，准备回收 先来看**recycleUnchecked()**方法

> - 第一步，**next=sPool**，因为消息对象池为空，所以此时sPool为null，同时next也为null。
> - 第二步，**spool = this**，将当前这个message作为消息对象池中下一个被复用的对象。
> - 第三步，**sPoolSize++**，默认为0，此时为1，将消息对象池的数量+1，这个数量依然是全系统共共享的。

这时候假设又调用了，这个方法，之前的原来的第一个Message对象我假设定位以为**message1**，依旧走到上面的循环。

> - 第一步，**next=sPool**，因为消息对象池为message1，所以此时sPool为message1，同时next也为message1。
> - 第二步，**sPool = this**，将当前这个message作为消息对象池中下一个被复用的对象。
> - 第三步，**sPoolSize++**，此时为1，将消息对象池的数量+1，sPoolSize为2，这个数量依然是全系统共共享的。

以此类推，直到sPoolSize=50(MAX_POOL_SIZE = 50)

#### 4.2、obtain()的理解

假设上面已经回收了一个Message对象，又从这里获取一个message，看看会发生什么？

> - 第一步，判断**sPool**是否为空，如果消息对象池为空，则直接new Message并返回
> - 第二步，**Message m = sPool**，将消息对象池中的对象取出来，为m。
> - 第三步，**sPool = m.next**，将消息对象池中的下一个可以复用的Message对象(m.next)赋值为消息对象池中的当前对象。(如果消息对象池就之前就一个，则此时sPool=null)
> - 第四步，将m.next置为null，因为之前已经把这个对象取出来了，所以无所谓了。
> - 第五步，**m.flags = 0**，设置m的标记位，标记位正在被使用
> - 第六步，**sPoolSize--**，因为已经把m取出了，这时候要把消息对象池的容量减一。

#### 4.3、深入理解消息对象池

> 上面的过程主要讨论只有一个message的情况，详细解释一下sPool和next，将sPool看成一个指针，通过next来将对象组成一个链表，因为每次只需要从池子里拿出一个对象，所以不需要关心池子里具体有多少个对象，而是拿出当前这个sPool所指向的这个对象就可以了，sPool从思路上理解就是通过左右移动来完成复用和回收

**4.3.1、obtain()复用**

如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-92a3baed67adab5e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

obtain()复用1.png

> 当移动Obtain()的时候，让sPool=next，因此第一个message.next就等于第二个message，从上图看相当于指针向后移动了一位，随后会将第一个message.next的值置为空。如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-6519bd8864b9ab57.png?imageMogr2/auto-orient/strip|imageView2/2/w/1188/format/webp)

obtain()复用2.png

现在这个链表看上去就断了，如果in-use这个message使用完毕了，怎么回到链表中？这就是recycleUnchecked() – 回收了

**4.3.2、recycleUnchecked()回收**

> 这时候在看下recycleUnchecked()里面的代码，next ＝ sPool，将当前sPool所指向的message对象赋值给in－use的next，然后sPool=this，将sPool指向第一个message对象。如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-87acab2ca912fddf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1188/format/webp)

ecycleUnchecked()回收.png

这样，就将链表恢复了，而且不管是复用还是回收大欧式保证线程同步的，所以始终会形成一条链式结构。

如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-b2cc776bf0dad2bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1024/format/webp)

回收.png

## 五、obtain()有参函数解析

### (一)、② public static Message obtain(Message orig)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 142行



```kotlin
   /**
     * Same as {@link #obtain()}, but copies the values of an existing
     * message (including its target) into the new one.
     * @param orig Original message to copy.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Message orig) {
        Message m = obtain();
        m.what = orig.what;
        m.arg1 = orig.arg1;
        m.arg2 = orig.arg2;
        m.obj = orig.obj;
        m.replyTo = orig.replyTo;
        m.sendingUid = orig.sendingUid;
        if (orig.data != null) {
            m.data = new Bundle(orig.data);
        }
        m.target = orig.target;
        m.callback = orig.callback;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是将message的所有内容复制一份到新的消息中。

看代码我们知道首先调用obtain()从消息对象池中获取一个Message对象m，然后把orig中的所有属性赋值给m。

### (二)、③ public static Message obtain(Handler h)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 164行



```php
     /**
     * Same as {@link #obtain()}, but sets the value for the <em>target</em> member on the Message returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是成员变量中的target的值用以指定的值(入参)来替换。

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target重新赋值而已。

### (三)、④ public static Message obtain(Handler h, Runnable callback)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 178行



```php
    /**
     * Same as {@link #obtain(Handler)}, but assigns a callback Runnable on
     * the Message that is returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @param callback Runnable that will execute when the message is handled.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是成员变量中的target的值用以指定的值(入参)来替换，并且添加一个回调的Runnable

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target和m的callback重新赋值而已。

### (四)、⑤ public static Message obtain(Handler h, int what)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 193行



```java
    /**
     * Same as {@link #obtain()}, but sets the values for both <em>target</em> and
     * <em>what</em> members on the Message.
     * @param h  Value to assign to the <em>target</em> member.
     * @param what  Value to assign to the <em>what</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是重置了成员变量target和what的值。

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target和m的what重新赋值而已。

### (五)、public static Message obtain(Handler h, int what, Object obj)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 209行



```php
    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, and <em>obj</em>
     * members.
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param obj  The <em>object</em> method to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是重置了成员变量target、what、obj的值。

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target、what和obj这三个成员变量重新赋值而已。

### (六)、⑦ public static Message obtain(Handler h, int what, int arg1, int arg2)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 228行



```php
    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, 
     * <em>arg1</em>, and <em>arg2</em> members.
     * 
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是重置了成员变量target、what、arg1、arg2的值。

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target、what、arg1、arg2这四个成员变量重新赋值而已。

### (七)、⑧ public static Message obtain(Handler h, int what, int arg1, int arg2, Object obj)

代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 249行



```php
    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, 
     * <em>arg1</em>, <em>arg2</em>, and <em>obj</em> members.
     * 
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @param obj  The <em>obj</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, 
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;

        return m;
    }
```

先翻译注释：

> 和obtain()一样，但是重置了成员变量target、what、arg1、arg2、obj的值。

代码很简单就是调用obtain()从消息对象池中获取一个Message对象m，然后将m的target、what、arg1、arg2、obj这五个成员变量重新赋值而已。

### (八)、总结

> 我们发现 上面有参的obtain()方法里面第一行代码都是  Message m = obtain();，所以有参的obtain()的方法的本质都是调用无参的obtain()方法，只不过有参的obtain()可以通过入参来重置一些成员变量的值而已

## 六、Message的 浅拷贝

Message的浅拷贝 就是copyFrom(Message o)函数
 代码在[Message.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Message.java) 320行



```kotlin
    /**
     * Make this message like o.  Performs a shallow copy of the data field.
     * Does not copy the linked list fields, nor the timestamp or
     * target/callback of the original message.
     */
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }
```

先翻译注释：

> 把这个Message做成像o一样。执行数据字段的浅拷贝，不复制字段连接，也不赋值目标的Handler和callback回调。

其实从本质上看就是从一个消息体复制到另一个消息体。有时候你可能需要用到Message的拷贝功能，也就是说拷贝一个和Message 一模一样的B，这时候你可以使用copyFrom(Message o) 拷贝一个Message对象。

有时候你可能需要用到Message的拷贝功能，也就是说拷贝一个和Message A一模一样的B，这时候你可以使用copyFrom(Message o)方法来浅拷贝一个Message对象。从代码中可以看出大部分数据确实都是浅拷贝，但是对于data这个Bundle类型的成员变量却进行了深拷贝，所以说该方法是一个浅拷贝方法感觉也不是很贴切。



# Handler机制之MessageQueue简介

## 一、MessageQueue简介

> MessageQueue即消息队列，这个消息队列和上篇文章里面的[Android Handler机制5之Message简介与消息对象对象池](https://www.jianshu.com/p/e271ee639b68)里面的 ***消息对象池\***  可**不是**同一个东西。MessageQueue是一个消息队列，Handler将Message发送到消息队列中，消息队列会按照一定的规则取出要执行的Message。需要注意的是Java层的MessageQueue负责处理Java的消息，native也有一个MessageQueue负责处理native的消息，本文重点是Java层，所以暂时不分析native源码。

## 二、MessageQueue类注释

[MessageQueue.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)

> Low-level class holding the list of messages to be dispatched by a Looper. Messages are not added directly to a MessageQueue, but rather through Handler objects associated with the Looper.
>  You can retrieve the MessageQueue for the current thread with Looper.myQueue().

翻译一下：

> 它是一个被Looper分发、低等级的持有Message集合的类。Message并不是直接加到MessageQueue的，而是通过Handler对象和Looper关联到一起。
>  我们可以通过Looper.myQueue()方法来检索当前线程的MessageQueue
>  它是一个低等级的持有Messages集合的类，被Looper分发。Messages并不是直接加到MessageQueue的，而是通过Handler对象和Looper关联到一起。我们可以通过Looper.myQueue()方法来检索当前线程的

## 三、MessageQueue成员变量



```java
    // True if the message queue can be quit.
    //用于标示消息队列是否可以被关闭，主线程的消息队列不可关闭
    private final boolean mQuitAllowed;

    @SuppressWarnings("unused")
    // 该变量用于保存native代码中的MessageQueue的指针
    private long mPtr; // used by native code

    //在MessageQueue中，所有的Message是以链表的形式组织在一起的，该变量保存了链表的第一个元素，也可以说它就是链表的本身
    Message mMessages;

    //当Handler线程处于空闲状态的时候(MessageQueue没有其他Message时)，可以利用它来处理一些事物，该变量就是用于保存这些空闲时候要处理的事务
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();

    // 注册FileDescriptor以及感兴趣的Events，例如文件输入、输出和错误，设置回调函数，最后
    // 调用nativeSetFileDescriptorEvent注册到C++层中，
    // 当产生相应事件时，由C++层调用Java的DispathEvents，激活相应的回调函数
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;

     // 用于保存将要被执行的IdleHandler
    private IdleHandler[] mPendingIdleHandlers;

    //标示MessageQueue是否正在关闭。
    private boolean mQuitting;

    // Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
    // 标示 MessageQueue是否阻塞
    private boolean mBlocked;

    // The next barrier token.
    // Barriers are indicated by messages with a null target whose arg1 field carries the token.
    // 在MessageQueue里面有一个概念叫做障栅，它用于拦截同步的Message，阻止这些消息被执行，
    // 只有异步Message才会放行。障栅本身也是一个Message，只是它的target为null并且arg1用于区分不同的障栅，
     // 所以该变量就是用于不断累加生成不同的障栅。
    private int mNextBarrierToken;
```

## 四、MessageQueue的构造函数

通过分析下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-d67060a3b5e56a84.png?imageMogr2/auto-orient/strip|imageView2/2/w/882/format/webp)

MessageQueue构造函数.png

我们知道MessageQueue就一个构造函数
 代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 68行



```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

> MessageQueue只是有一个构造函数，该构造函数是包内可见的，其内部就两行代码，分别是设置了MessageQueue是否可以退出和native层代码的相关初始化

## 五、native层代码的初始化

在MessageQueue的构造函数里面调用 nativeInit()，我们来看下
 代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 61行



```java
    private native static long nativeInit();
```

根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeInit这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativeInit(JNIEnv* , jclass )函数

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp) 172 行



```cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    // 初始化native消息队列
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

后面在讲解流程时候的会详细讲解，这里就不如深入了。

## 六、IdleHandler简介

> 作为Android开发者我们知道，Handler除了用于发送Message，其本身也承载着执行具体业务逻辑的责任handlerMessage(Message msg)，而IdleHandler在处理业务逻辑方面和Handler一样，不过它只会在线程空闲的时候才执行业务逻辑的处理，这些业务经常是哪些不是很紧要或者不可预期的，比如GC。

### (一) IdleHandler接口

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 777行



```dart
    /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

老规矩 先来翻译一下接口的注释：

> 回调的接口，当线程空闲的时候可以利用它来处理一些业务员

这个IdleHandler接口就一个抽象方法queueIdle，我也看一下抽象方法的注释

> 当消息队内所有的Message都执行完之后，该方法会被调用。该返回值为True的时候，IdleHandler会一直保持在消息队列中，False则会执行完该方法后移除IdleHandler。需要注意的是，当消息队列中还有其他Delay Message并且这些Message还没到被执行的时间的时候，由于线程是空闲的，所以IdleHandler也可能会被执行，

从源码可以看出IdleHandler其实就是一个简单的回调接口，内部就一个带返回值的方法**boolean queueIdle()**，在使用的时候只需要实现该接口并加入到MessageQueue中就可以了，例如

从源码可以看出IdleHandler其实就是一个简单的回调接口，内部就一个带返回值的方法boolean queueIdle()，在使用的时候只需要实现该接口并加入到MessageQueue中就可以了，例如下面简答的代码所示



```java
    MessageQueue messageQueue = Looper.myQueue();
    messageQueue.addIdleHandler(new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            // do something.
            return false;
        }
    });
```

### (二) 添加IdleHandler：addIdleHandler(IdleHandler handler)

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 115行



```dart
    /**
     * Add a new {@link IdleHandler} to this message queue.  This may be
     * removed automatically for you by returning false from
     * {@link IdleHandler#queueIdle IdleHandler.queueIdle()} when it is
     * invoked, or explicitly removing it with {@link #removeIdleHandler}.
     *
     * <p>This method is safe to call from any thread.
     *
     * @param handler The IdleHandler to be added.
     */
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
```

看先注释：

> - 添加一个新的IdleHanlder到消息队列中，当IdleHandler的回调方法返回False的时候，该IdleHanlder在被执行后会被立即移除，你也可以通过调用removeIdleHandler(IdleHandler handler)方法来移除指定的IdleHandler。
> - 在任何线程中调用该方法都是安全的。

方法内部很简单，就是三步

> - 第一步，做非空判断
> - 第二步，加一个同步锁
> - 第三步，调用mIdleHandlers.add(handler);添加 (PS:mIdleHandlers是一个ArrayList)

### (三) 删除IdleHandler：removeIdleHandler(IdleHandler handler)

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
 133行



```dart
    /**
     * Remove an {@link IdleHandler} from the queue that was previously added
     * with {@link #addIdleHandler}.  If the given object is not currently
     * in the idle list, nothing is done.
     *
     * <p>This method is safe to call from any thread.
     *
     * @param handler The IdleHandler to be removed.
     */
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }
```

看先注释：

> - 从消息队列中移除一个之前添加的IdleHandler。如果该IdleHandler不存在，则什么也不做。

移除IdleHandler的方法同样很简单，下一步同步处理然后直接mIdleHandlers.reomve(handler)就可以了。

## 七、MessageQueue中的Message分类

在MessageQueue中，Message被分成3类，分别是

> - 同步消息
> - 异步消息
> - 障栅

那我们就一次来看下：

### (一) 同步消息：

> 正常情况下我们通过Handler发送的Message都属于同步消息，除非我们在发送的时候执行该消息是一个异步消息。同步消息会按顺序排列在队列中，除非指定Message的执行时间，否咋Message会按顺序执行。

### (二) 异步消息：

> 想要往消息队列中发送异步消息，我们必须在初始化Handler的时候通过构造函数public Handler(boolean async)中指定Handler是异步的，这样Handler在讲Message加入消息队列的时候就会将Message设置为异步的。

### (三) 障栅(Barrier)：

> 障栅(Barrier) 是一种特殊的Message，它的target为null(只有障栅的target可以为null，如果我们自己视图设置Message的target为null的话会报异常)，并且arg1属性被用作障栅的标识符来区别不同的障栅。障栅的作用是用于拦截队列中同步消息，放行异步消息。就好像交警一样，在道路拥挤的时候会决定哪些车辆可以先通过，这些可以通过的车辆就是异步消息。

![img](https:////upload-images.jianshu.io/upload_images/5713484-5720414dcc7f45f3.png?imageMogr2/auto-orient/strip|imageView2/2/w/576/format/webp)

同步和异步.png

#### **1、添加障栅：postSyncBarrier()**

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
 458行



```dart
    /**
     * Posts a synchronization barrier to the Looper's message queue.
     *
     * Message processing occurs as usual until the message queue encounters the
     * synchronization barrier that has been posted.  When the barrier is encountered,
     * later synchronous messages in the queue are stalled (prevented from being executed)
     * until the barrier is released by calling {@link #removeSyncBarrier} and specifying
     * the token that identifies the synchronization barrier.
     *
     * This method is used to immediately postpone execution of all subsequently posted
     * synchronous messages until a condition is met that releases the barrier.
     * Asynchronous messages (see {@link Message#isAsynchronous} are exempt from the barrier
     * and continue to be processed as usual.
     *
     * This call must be always matched by a call to {@link #removeSyncBarrier} with
     * the same token to ensure that the message queue resumes normal operation.
     * Otherwise the application will probably hang!
     *
     * @return A token that uniquely identifies the barrier.  This token must be
     * passed to {@link #removeSyncBarrier} to release the barrier.
     *
     * @hide
     */
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }
```

看先注释：

> - 向Looper的消息队列中发送一个同步的障栅(barrier)
> - 如果没有发送同步的障栅(barrier)，消息处理像往常一样该怎么处理就怎么处理。当发现遇到障栅(barrier)后，队列中后续的同步消息会被阻塞，直到通过调用removeSyncBarrier()释放指定的障栅(barrier)。
> - 这个方法会导致立即推迟所有后续发布的同步消息，知道满足释放指定的障栅(barrier)。而异步消息则不受障栅(barrier)的影响，并按照之前的流程继续处理。
> - 必须使用相同的token去调用removeSyncBarrier()，来保证插入的障栅(barrier)和移除的是一个同一个，这样可以确保消息队列可以正常运行，否则应用程序可能会挂起。
> - 返回值是障栅(barrier)的唯一标识符，持有个token去调用removeSyncBarrier()方法才能达到真正的释放障栅(barrier)

方法内部很简单就是调用了postSyncBarrier(SystemClock.uptimeMillis())，通过[Android Handler机制3之SystemClock类](https://www.jianshu.com/p/e6877401a0e2)，我们知道SystemClock.uptimeMillis()是手机开机到现在的时间。那我们来看下这个postSyncBarrier(long)方法

**1.1、postSyncBarrier(long when)**

在讲解之前先补充一个知识点：MessageQueue里面的所有Message是按照时间从前往后有序排列的。后面我们讲解Handler机制流程的时候会详细说明

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
 462行



```csharp
 private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
              /** 第一步 */
            final int token = mNextBarrierToken++;
             /** 第二步 */
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

              /** 第三步 */
            Message prev = null;
            //把消息队列的第一个元素指向p
            Message p = mMessages;
            if (when != 0) {
             /** 第四步 */
                while (p != null && p.when <= when) {
                    //通过p的时间点和障栅的时间点的比较，如果比障栅的小，就把消息队列中的消息向后移动一位(因为消息队列中所有元素是按照时间排序的)
                    prev = p;
                    p = p.next;
                }
            }
              /** 第五步 */
             //prev != null 代表不是消息队列的头部，则需要考虑前面一个消息和后面的一个消息
            if (prev != null) { // invariant: p == prev.next
                //msg的下一个消息是p 
                msg.next = p;
                 //msg的上一个消息是msg
                prev.next = msg;
            } else {
                //prev == null 代表是消息队列的头部，则只需要负责下一个消息即可
                msg.next = p;
                //设置自己是消息队列的头部
                mMessages = msg;
            }
            /** 第六步 */
            return token;
        }
    }
```

方法详解

> - **第一步** 获取障栅的唯一标示，然后自增该变量作为下一个障栅的标示，通过mNextBarrierToken ++，我们得知，这些唯一标示是从0开始，自加1的。
> - **第二步** 从Message消息对象池中获取一个Message，并重置它的when和arg1。并且arg1为token的值，通过msg.markInUse()标示msg正在被使用。这里并没有给tareget赋值。
> - **第三步** 创建变量pre和p为第四步做准备，其中p被赋值为mMessages，而mMessages未消息队列的第一个元素，所以p此时就是消息队列的第一个元素。
> - **第四步** 通过对 队列中的第一个Message的when和障栅的when作比较，决定障栅在整个消息队列中的位置，比如是放在队列的头部，还是队列中第二个位置，如果障栅在头部，则拦截后面所有的同步消息，如果在第二的位置，则会放过第一个，然后拦截剩下的消息，以此类推。
> - **第五步** 把msg插入到消息队列中
> - **第六步** 返回token

从源码中我们可以看出，在把障栅插入队列的时候先通过when的比较，根据不同的情况把障栅插入到不同的位置，具体情况如下图所示：
 ps:蓝色的为Message、红色的为Barrier

当Message.when<Barrier.when，也就是第一个Message的执行时间点在障栅之前。



![img](https:////upload-images.jianshu.io/upload_images/5713484-16f22f1ce2877b4b.png?imageMogr2/auto-orient/strip|imageView2/2/w/817/format/webp)

障栅插入队列1.png

当Message.when>=Barrier.when，也就是第一个Message的执行时间点在障栅之后。



![img](https:////upload-images.jianshu.io/upload_images/5713484-924d5b8fc7e08dbf.png?imageMogr2/auto-orient/strip|imageView2/2/w/844/format/webp)

障栅插入队列2.png

**大家在看上面的代码的时候有没有注意到一个事情**，就是msg这个对象的target是null，因为从始至终就没有赋值过，这也是后面在移除障栅的时候通过判断条件之一：是target是否为null来判断的

#### 2、移除障栅

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
 501行



```java
   /**
     * Removes a synchronization barrier.
     *
     * @param token The synchronization barrier token that was returned by
     * {@link #postSyncBarrier}.
     *
     * @throws IllegalStateException if the barrier was not found.
     *
     * @hide
     */
    public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            // 获取消息队列的第一个元素
            Message p = mMessages;
            //遍历消息队列的所有元素，直到p.targe==null并且 p.arg1==token才是我们想要的障栅
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            //是否需要唤醒
            final boolean needWake;
            //如果是障栅是不是第一个圆度
            if (prev != null) {
                //跳过障栅，将障栅的上一个元素的next指向障栅的next
                prev.next = p.next;
                //因为有元素，所以不需要唤醒
                needWake = false;
            } else {
                //如果是第一个元素，则直接下消息队列中的第一个元素指向障栅的下一个即可
                mMessages = p.next;
                 //如果消息队列中的第一个元素是null则说明消息队列中消息，所以需要唤醒
                 //
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```

> 删除障栅(barrier)的方法也很简单，就是不断地遍历消息队列(链表结构)，直到倒找与指定的token相匹配的障栅，然后把它从队列中移除。



# Handler机制之消息发送

## 一 、Handler发送消息

大家平时发送消息主要是调用的两大类方法
 如下两图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-8685a4b0ec7a80a4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

send方案.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-b4b22f9056fd9df5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

send方案.png

光看上面这些API你可能会觉得handler能法两种消息，一种是Runnable对象，一种是message对象，这是直观的理解，但其实post发出的Runnable对象最后都封装成message对象了。

> - send方案发送消息(需要回调才能接收消息)
>   - 1、sendMessage(Message) 立即发送Message到消息队列
>   - 2、sendMessageAtFrontOfQueue(Message) 立即发送Message到队列，而且是放在队列的最前面
>   - 3、sendMessageAtTime(Message,long) 设置时间，发送Message到队列
>   - 4、sendMessageDelayed(Message,long) 延时若干毫秒后，发送Message到队列
> - post方案  立即发送Message到消息队列
>   - 1、post(Runnable)  立即发送Message到消息队列
>   - 2、postAtFrontOfQueue(Runnable) 立即发送Message到队列，而且是放在队列的最前面
>   - 3、postAtTime(Runnable,long) 设置时间，发送Message到队列
>   - 4、postDelayed(Runnable,long) 在延时若干毫秒后，发送Message到队列

下面我们就先从send方案中的第一个sendMessage() 开始源码跟踪下：

## 二、 Handler的send方案

我们以Handler的sendMessage(Message msg)为例子。

### (一)、boolean sendMessage(Message msg)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  505行



```dart
    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

老规矩先翻译一下注释：

> 在当前时间，在所有待处理消息之后，将消息推送到消息队列的末尾。在和当前线程关联的的Handler里面的handleMessage将收到这条消息，

我们看到sendMessage(Message)里面代码很简单，就是调用了sendMessageDelayed(msg,0)

#### 1、boolean sendMessageDelayed(Message msg, long delayMillis)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  565行



```dart
    /**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

注释和boolean sendMessage(Message msg)方法差不多，我就不翻译了

> 该方法内部就做了两件事
>
> - 1、判断delayMillis是否小于0
> - 2、调用了public boolean sendMessageAtTime(Message msg, long uptimeMillis)方法

#### 2、boolean sendMessageAtTime(Message msg, long uptimeMillis)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  592行



```dart
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

老规矩先翻译一下注释：

> 以android系统的SystemClock的uptimeMillis()为基准，以毫秒为基本单位的绝对时间下，在所有待处理消息后，将消息放到消息队列中。深度睡眠中的时间将会延迟执行的时间，你将在和当前线程办的规定的Handler中的handleMessage中收到该消息。

这里顺便提一下异步的作用，因为通常我们理解的异步是指新开一个线程，但是这里不是，因为异步的也是发送到looper所绑定的消息队列中，这里的异步主要是针对Message中的障栅(Barrier)而言的，当出现障栅(Barrier)的时候，同步的会被阻塞，而异步的则不会。所以这个异步仅仅是一个标记而已。

> 该方法内部就做了两件事
>
> - 1、获取消息队列，并对该消息队列做非空判断，如果为null，直接返回false，表示消息发送失败
> - 2、调用了boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)方法

#### 3、boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)  626行



```cpp
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

本方法内部做了三件事

> - 1、设置msg的target变量，并将target指向自己
> - 2、如果Handler的mAsynchronous值为true(默认为false，即不设置)，则设置msg的flags值，让是否异步在Handler和Message达成统一。
> - 3、调用MessageQueue的enqueueMessage()方法

#### 4、boolean enqueueMessage(Message msg, long when)方法

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  533行



```csharp
    boolean enqueueMessage(Message msg, long when) {
        //第一步
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        // 第二步
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        // 第三步
        synchronized (this) {
             // 第四步
            //判断消息队列是否正在关闭
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
             // 第五步
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
             // 第六步
            //根据when的比较来判断要添加的Message是否应该放在队列头部，当第一个添加消息的时候，
            // 测试队列为空，所以该Message也应该位于头部。
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                // 把msg的下一个元素设置为p
                msg.next = p;
                // 把msg设置为链表的头部元素
                mMessages = msg;
                 // 如果有阻塞，则需要唤醒
                needWake = mBlocked;
            } else {
                 // 第七步
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                //除非消息队列的头部是障栅(barrier)，或者消息队列的第一个消息是异步消息，
                //否则如果是插入到中间位置，我们通常不唤醒消息队列，
                 // 第八步
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                  // 第九步
                 // 不断遍历消息队列，根据when的比较找到合适的插入Message的位置。
                for (;;) {
                    prev = p;
                    p = p.next;
                    // 第十步
                    if (p == null || when < p.when) {
                        break;
                    }
                    // 第十一 步
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                // 第十二步
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
              // 第十三步
            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
         // 第十四步
        return true;
    }
```

因为这是消息入队的流程，为了让大家更好的理解，我将上面的流程大致分为12步骤

> - **第1步骤、**  判断msg的target变量是否为null，如果为null，则为障栅(barrier)，而障栅(barrier)入队则是通过postSyncBarrier()方法入队，所以msg的target一定有值
> - **第2步骤、**  判断msg的标志位，因为此时的msg应该是要入队，意味着msg的标志位应该显示还未被使用。如果显示已使用，明显有问题，直接抛异常。
> - **第3步骤、**  加入同步锁。
> - **第4步骤、**  判断消息队列是否正在被关闭，如果是正在被关闭，则return false告诉消息入队是失败，并且回收消息
> - **第5步骤、**  设置msg的when并且修改msg的标志位，msg标志位显示为已使用
> - **第6步骤、**  如果p==null则说明消息队列中的链表的头部元素为null；when == 0 表示立即执行；when< p.when 表示 msg的执行时间早与链表中的头部元素的时间，所以上面三个条件，那个条件成立，都要把msg设置成消息队列中链表的头部是元素
> - **第7步骤、**  如果上面三个条件都不满足则说明要把msg插入到中间的位置，不需要插入到头部
> - **第8步骤、**  如果头部元素不是障栅(barrier)或者异步消息，而且还是插入中间的位置，我们是不唤醒消息队列的。
> - **第9步骤、**  进入一个死循环，将p的值赋值给prev，前面的带我们知道，p指向的是mMessage，所以这里是将prev指向了mMessage，在下一次循环的时候，prev则指向了第一个message，一次类推。接着讲p指向了p.next也就是mMessage.next，也就是消息队列链表中的第二个元素。这一步骤实现了消息指针的移动，此时p表示的消息队列中第二个元素。
> - **第10步骤、**  p==null，则说明没有下一个元素，即消息队列到头了，跳出循环；p!=null&&when < p.when 则说明当前需要入队的这个message的执行时间是小于队列中这个任务的执行时间的，也就是说这个需要入队的message需要比队列中这个message先执行，则说明这个位置刚刚是适合这个message的，所以跳出循环。 如果上面的两个条件都不满足，则说明这个位置还不是放置这个需要入队的message，则继续跟链表中后面的元素，也就是继续跟消息队列中的下一个消息进行对比，直到满足条件或者到达队列的末尾。
> - **第11步骤、**  因为没有满足条件，说明队列中还有消息，不需要唤醒。
> - **第12步骤、**  跳出循环后主要做了两件事：事件A，将入队的这个消息的next指向循环中获取到的应该排在这个消息之后message。事件B，将msg前面的message.next指向了msg。这样就将一个message完成了入队。
> - **第13步骤、** 如果需要唤醒，则唤醒，具体请看后面的Handler中的Native详解。
> - **第14步骤、** 返回true，告知入队成功。

提供两张图，让大家更好的理解入队



![img](https:////upload-images.jianshu.io/upload_images/5713484-30aa581f3e9375d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/992/format/webp)

入队1.png

![img](https:////upload-images.jianshu.io/upload_images/5713484-031079e74451aec4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1044/format/webp)

入队2.png

总结一句话就是：就是遍历消息队列中所有的消息，根据when的比较找到合适添加Message的位置。

上面是sendMessage(Message msg)发送消息机制，这样再来看下其他的send方案

### (二)  boolean sendMessageAtFrontOfQueue(Message msg)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 615行



```dart
    /**
     * Enqueue a message at the front of the message queue, to be processed on
     * the next iteration of the message loop.  You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
```

老规矩先翻译一下注释：

> 在消息队列的最前面插入一个消息，在消息循环的下一次迭代中进行处理。你将在当前线程关联的Handler的handleMessage()中收到这个消息。由于它可以轻松的解决消息队列的排序问题和其他的意外副作用。

方法内部的实现和boolean sendMessageAtTime(Message msg, long uptimeMillis)大体上一致，唯一的区别就是该方法在调用enqueueMessage(MessageQueue, Message, long)方法的时候，最后一个参数是0而已。

### (三)  boolean sendEmptyMessage(int what)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 517行



```dart
   /**
     * Sends a Message containing only the what value.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }
```

老规矩先翻译一下注释：

> 发送一个仅有what的Message

内容很简单，也就是调用sendEmptyMessageDelayed(int,long)而已，那么下面我们来看下sendEmptyMessageDelayed(int,long)的具体实现。

**1、boolean sendEmptyMessageDelayed(int what, long delayMillis)**

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 531行



```dart
    /**
     * Sends a Message containing only the what value, to be delivered
     * after the specified amount of time elapses.
     * @see #sendMessageDelayed(android.os.Message, long) 
     * 
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
```

老规矩先翻译一下注释：

> 发送一个仅有what的Message，并且延迟特定的时间发送

这个方法内部主要就是做了3件事

> - 1、调用Message.obtain();从消息对象池中获取一个空的Message。
> - 2、设置这个Message的what值
> - 3、调用sendMessageDelayed(Message,long) 将这个消息方法

sendMessageDelayed(Message,long) 这个方法上面有讲解过，这里就不详细说了

### (四)  boolean sendEmptyMessageAtTime(int what, long uptimeMillis)

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 547行



```dart
    /**
     * Sends a Message containing only the what value, to be delivered 
     * at a specific time.
     * @see #sendMessageAtTime(android.os.Message, long)
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }
```

老规矩先翻译一下注释：

> 发送一个仅有what的Message，并且在特定的时间发送

这个方法内部主要就是做了3件事

> - 1、调用Message.obtain();从消息对象池中获取一个空的Message。
> - 2、设置这个Message的what值
> - 3、调用sendMessageAtTime(Message,long) 将这个消息方法

### (五)  小结

综上所述

> - public final boolean sendMessage(Message msg)
> - public final boolean sendEmptyMessage(int what)
> - public final boolean sendEmptyMessageDelayed(int what, long delayMillis)
> - public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis)
> - public final boolean sendMessageDelayed(Message msg, long delayMillis)
> - public boolean sendMessageAtTime(Message msg, long uptimeMillis)
> - public boolean sendMessageAtTime(Message msg, long uptimeMillis)
> - public final boolean sendMessageAtFrontOfQueue(Message msg)

> 以上这些send方案都会从这里或者那里最终走到**boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)**如下图

![img](https:////upload-images.jianshu.io/upload_images/5713484-e56c82c4e36768f5.png?imageMogr2/auto-orient/strip|imageView2/2/w/554/format/webp)

send小结.png

## 三、 Handler的post方案

### (一)、boolean post(Runnable r)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 324行



```dart
    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is 
     * attached. 
     *  
     * @param r The Runnable that will be executed.
     * 
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```

老规矩先翻译一下注释：

> 将一个Runnable添加到消息队列中，这个runnable将会在和当前Handler关联的线程中被执行。

这个方法内部很简单，就是调用了sendMessageDelayed(Message, long);这个方法，所以可见**boolean post(Runnable r)**这个方法最终还是走到上面说到的send的流程中，最终调用**boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)**。

这里面调用了**Message getPostMessage(Runnable r)**，我们来看一下。

**1、Message getPostMessage(Runnable r)方法**

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 725行



```cpp
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

代码很简单，主要是做了两件事

> - 通过Message.obtain()从消息对象池中获取一个空的Message
> - 将这空的Message的callback变量指向Runnable

最后返回这个Message m。

**2、小结**

> 所以我们知道**boolean post(Runnable r)方法**的内置也是通过Message.obtain()来获取一个Message对象m，然后仅仅把m的callback指向参数r而已。最后最终通过调用send方案的某个流程最终调用到**boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)**

这时候聪明的同学一定会想，那么post方案的其他方法是不是也是这样的？**是的，的确都是这样**，这都是“套路”，那我们就用一一去检验。

### (二)、boolean postAtTime(Runnable r, long uptimeMillis)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 347行



```dart
    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     *  
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postAtTime(Runnable r, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }
```

老规矩先翻译一下注释：

> 将一个Runnable添加到消息队列中，这个runnable将会一个特定的时间被执行，这个时间是以android.os.SystemClock.uptimeMillis()为基准。如果在深度睡眠下，会推迟执行的时间，这个Runnable将会在和当前Hander关联的线程中被执行。

方法内部也是先是调用getPostMessage(Runnable)来获取一个Message，这个Message的callback字段指向了这个Runnable，然后调用sendMessageAtTime(Message,long)。

### (三)、boolean postAtTime(Runnable r, Object token, long uptimeMillis)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 372行



```dart
    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * at a specific time given by <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * The runnable will be run on the thread to which this handler is attached.
     *
     * @param r The Runnable that will be executed.
     * @param uptimeMillis The absolute time at which the callback should run,
     *         using the {@link android.os.SystemClock#uptimeMillis} time-base.
     * 
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     *         
     * @see android.os.SystemClock#uptimeMillis
     */
    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
```

这个方法的翻译同上，这个方法和上个方法唯一不同就是多了Object参数，而这个参数仅仅是把Message.obtain();获取的Message的obj字段的指向第二个入参token而已。最后也是调用sendMessageAtTime(Message,long)。

### (四)、boolean postDelayed(Runnable r, long delayMillis)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 396行



```dart
    /**
     * Causes the Runnable r to be added to the message queue, to be run
     * after the specified amount of time elapses.
     * The runnable will be run on the thread to which this handler
     * is attached.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     *  
     * @param r The Runnable that will be executed.
     * @param delayMillis The delay (in milliseconds) until the Runnable
     *        will be executed.
     *        
     * @return Returns true if the Runnable was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the Runnable will be processed --
     *         if the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean postDelayed(Runnable r, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
```

这个方法也很简单，就是依旧是通过getPostMessage(Runnable)来获取一个Message，最后调用sendMessageDelayed(Message,long)而已。

### (五)、boolean postDelayed(Runnable r, long delayMillis)方法

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 416行



```dart
    /**
     * Posts a message to an object that implements Runnable.
     * Causes the Runnable r to executed on the next iteration through the
     * message queue. The runnable will be run on the thread to which this
     * handler is attached.
     * <b>This method is only for use in very special circumstances -- it
     * can easily starve the message queue, cause ordering problems, or have
     * other unexpected side-effects.</b>
     *  
     * @param r The Runnable that will be executed.
     * 
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean postAtFrontOfQueue(Runnable r)
    {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
```

这个方法也很简单，就是依旧是通过getPostMessage(Runnable)来获取一个Message，最后调用sendMessageAtFrontOfQueue(Message)而已。

### (六)、小结

Handler的post方案的如下方法

> - boolean post(Runnable r)
> - postAtTime(Runnable r, long uptimeMillis)
> - boolean postAtTime(Runnable r, Object token, long uptimeMillis)
> - boolean postDelayed(Runnable r, long delayMillis)
> - boolean postAtFrontOfQueue(Runnable r)
>    都是通过**Message getPostMessage(Runnable )**中调用Message m = Message.obtain();来获取一个空的Message，然后把这个Message的callback变量指向了Runnable，最终调用相应的send方案而已。

所以我们可以这样说:

> ###### Handler的发送消息(障栅除外)，无论是通过send方案还是pos方案最终都会做走到 ***boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)\*** 中

![img](https:////upload-images.jianshu.io/upload_images/5713484-5d5c06c67b925a3d.png?imageMogr2/auto-orient/strip|imageView2/2/w/822/format/webp)



# Handler机制之消息的取出与消息的其他操作

## 一、消息的取出

### (一)、消息的取出主要是通过Looper的loop方法

代码如下[Looper.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Looper.java) 122行



```java
  /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
         //第1步
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //第2步
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

         //第3步
        for (;;) {
            //第四步
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                // 第5步
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            // 第6步
            msg.recycleUnchecked();
        }
    }
```

这个方法已经在[Android Handler机制4之Looper与Handler简介](https://www.jianshu.com/p/e635c3f851ea)中说过了，我就重点说下流程，大体上分为6步

> - **第1步** 获取Looper对象
> - **第2步** 获取MessageQueue消息队列对象
> - **第3步** while()死循环遍历
> - **第4步** 通过queue.next()来从MessageQueue的消息队列中获取一个Message msg对象
> - **第5步** 通过msg.target. dispatchMessage(msg)来处理消息
> - **第6步** 通过msg.recycleUnchecked()方来回收Message到消息对象池中

由于**第1步**、**第2步**和**第3步**比较简单就不讲解了，而**第6步**在y已经讲解过，也不讲解了，下面我们来重点说下**第4步**和**第5步**

### (二)、Message next()方法

> 从消息队列中提取Message交给Looper来处，这个步骤应该是MessageQueue乃至整个线程消息机制的核心了，所以我们将这部分放到最后来将，因为其内部的代码逻辑比较复杂，涉及到了障栅如何拦截同步消息、如何阻塞线程、如何在空闲的时候执行IdleHandler以及如何关闭Looper等内容，在源码已经做了详细的注释，不过由于逻辑比较复杂所以想要看明白，大家还要花费一定时间的。

PS:在Looper.loop()中获取消息的方式就是调用**next()**方法。

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)
 307行



```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        // 如果消息循环已经退出了。则直接在这里return。因为调用disposed()方法后mPtr=0
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        //记录空闲时处理的IdlerHandler的数量
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        // native层用到的变量 ，如果消息尚未到达处理时间，则表示为距离该消息处理事件的总时长，
        // 表明Native Looper只需要block到消息需要处理的时间就行了。 所以nextPollTimeoutMillis>0表示还有消息待处理
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                //刷新下Binder命令，一般在阻塞前调用
                Binder.flushPendingCommands();
            }
            // 调用native层进行消息标示，nextPollTimeoutMillis 为0立即返回，为-1则阻塞等待。
            nativePollOnce(ptr, nextPollTimeoutMillis);
            //加上同步锁
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                // 获取开机到现在的时间
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                // 获取MessageQueue的链表表头的第一个元素
                Message msg = mMessages;
                 // 判断Message是否是障栅，如果是则执行循环，拦截所有同步消息，直到取到第一个异步消息为止
                if (msg != null && msg.target == null) {
                     // 如果能进入这个if，则表面MessageQueue的第一个元素就是障栅(barrier)
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    // 循环遍历出第一个异步消息，这段代码可以看出障栅会拦截所有同步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                       //如果msg==null或者msg是异步消息则退出循环，msg==null则意味着已经循环结束
                    } while (msg != null && !msg.isAsynchronous());
                }
                 // 判断是否有可执行的Message
                if (msg != null) {  
                    // 判断该Mesage是否到了被执行的时间。
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        // 当Message还没有到被执行时间的时候，记录下一次要执行的Message的时间点
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Message的被执行时间已到
                        // Got a message.
                        // 从队列中取出该Message，并重新构建原来队列的链接
                        // 刺客说明说有消息，所以不能阻塞
                        mBlocked = false;
                        // 如果还有上一个元素
                        if (prevMsg != null) {
                            //上一个元素的next(越过自己)直接指向下一个元素
                            prevMsg.next = msg.next;
                        } else {
                           //如果没有上一个元素，则说明是消息队列中的头元素，直接让第二个元素变成头元素
                            mMessages = msg.next;
                        }
                        // 因为要取出msg，所以msg的next不能指向链表的任何元素，所以next要置为null
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        // 标记该Message为正处于使用状态，然后返回Message
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    // 没有任何可执行的Message，重置时间
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                // 关闭消息队列，返回null，通知Looper停止循环
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                // 当第一次循环的时候才会在空闲的时候去执行IdleHandler，从代码可以看出所谓的空闲状态
                // 指的就是当队列中没有任何可执行的Message，这里的可执行有两要求，
                // 即该Message不会被障栅拦截，且Message.when到达了执行时间点
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                
                // 这里是消息队列阻塞( 死循环) 的重点，消息队列在阻塞的标示是消息队列中没有任何消息，
                // 并且所有的 IdleHandler 都已经执行过一次了
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
    
                // 初始化要被执行的IdleHandler，最少4个
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            // 开始循环执行所有的IdleHandler，并且根据返回值判断是否保留IdleHandler
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            // 重点代码，IdleHandler只会在消息队列阻塞之前执行一次，执行之后改标示设置为0，
            // 之后就不会再执行，一直到下一次调用MessageQueue.next() 方法。
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            // 当执行了IdleHandler 的 处理之后，会消耗一段时间，这时候消息队列里的可能有消息已经到达 
             // 可执行时间，所以重置该变量回去重新检查消息队列。
            nextPollTimeoutMillis = 0;
        }
    }
```

总的来说当我们试图产品从MessageQueue中获取一个Message的时候，会分为以下几步

> - **首先**、MessageQueue会先判断队列中是否有障栅的存在，如果有的话，只会返回异步消息，否则就逐个返回。
> - **其次**、当MessageQueue没有任何消息可以处理的时候，它会进度阻塞状态等待新的消息到来(无线循环)，在阻塞之前它会执行以便 IdleHandler，所谓的阻塞其实就是不断的循环查看是否有新的消息进入队列中。
> - **再次**、当MessageQueue被关闭的时候，其成员变量mQuitting会被标记为true，然后在Looper视图从队列中取出Message的时候返回null，而Message==null就是告诉Looper消息队列已经关闭，应该停止循环了，这一点可以在Looper.loop()房源中看出。
> - **最后**、如果大家细心一定会发现，Handler线程里面实际上有两个无线循环体，Looper循环体和MessageQueue循环体，真正阻塞的地方是MessageQueue的next()方法里。

这里有个难点，我简单说下



```csharp
//****************  第一部分  ***************
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }

//============分割线==============


//****************  第二部分  ***************
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
```

> - 第一种情况：**第一部分**主要是判断链表的第一个元素是否是障栅，如果是障栅，则进入if，内部区域，然后进行while循环，如果在链表中有一个元素是异步的，则跳出循环，然后进入**第二部分**，其中**第二部分**就是取出这个异步消息
> - 第二种情况：没进入进入**第一部分**的if，则说明头部元素不是障栅(barrier)，则直接进入**第二部分**，这时候取出的就是当前的头部元素。

PS:

> nativePollOnce是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，需要等待的时长，当nextPollTimeoutMillis=-1时，表示消息队列无消息，会一直等待下去。nativePollOnce()是在native做了大量的工作。

### (三)、msg.target.dispatchMessage(msg);方法

我们知道这个方法其实是Handler的dispatchMessage(Message)方法，那我们就来详细看下
 代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 93行



```csharp
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            //当Message存在回调方法，回调msg.callback.run()方法；
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                //当Handler存在Callback成员变量时，回调方法handleMessage()；
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //Handler自身的回调方法handleMessage()
            handleMessage(msg);
        }
    }
```

这个方法很简单就是二个条件，三种情况

> - 情况1：如果msg.callback 不为空，则执行handleCallback(Message)，而handleCallback(Message)的内部最终调用的是message.callback.run();，所以最终是msg.callback.run()。
> - 情况2：如果msg.callback 为空，且mCallback不为空，则执行mCallback.handleMessage(msg)。
> - 情况3：如果msg.callback 为空，且mCallback也为空，则执行handleMessage()方法

这里我们可以看到，在分发消息时三个方法的优先级分别如下：

> - Message的回调方法优先级最高，即message.callback.run()；
> - Handler的回调方法优先级次之，即Handler.mCallback.handleMessage(msg)；
> - Handler的默认方法优先级最低，即Handler.handleMessage(msg)。

对于很多情况下，消息分发后的处理情况是第3种情况，即Handler.handleMessage()，一般地往往是通过覆写该方法从而实现自己的业务逻辑。

## 二、消息(Message)的移除

### (一) Handler的消息移除

> 消息(Message)的移除，其实就是根据身份what、消息Runnable或msg.obj移除队列中对应的消息。例如发送msg，用同一个msg.what作为参数。所有方法最终调用MessageQueue.removeMessages，来进行时机操作的。

代码如下，因为不复杂，我就合并在一起了



```java
// Handler.java
public final void removeCallbacks(Runnable r) {
    mQueue.removeMessages(this, r, null);
}

public final void removeCallbacks(Runnable r, Object token) {
    mQueue.removeMessages(this, r, token);
}

public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}

public final void removeMessages(int what, Object object) {
    mQueue.removeMessages(this, what, object);
}

public final void removeCallbacksAndMessages(Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}
```

所以我们知道，Handler里面的删除工作，其实本地都是调用MessageQueue来操作的。

下面我们就来看下MessageQueue是怎么操作的？

### (二) MessageQueue的消息移除

MessageQueue的消息移除在其类类的方法如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-834f223bde4df3bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1018/format/webp)

MessageQueue的消息移除.png

一共有5个方法如下：

> - 移除方法1：void removeMessages(Handler , int , Object )
> - 移除方法2：void removeMessages(Handler, Runnable,Object)
> - 移除方法3：void removeCallbacksAndMessages(Handler, Object)
> - 移除方法4：void removeAllMessagesLocked()
> - 移除方法5：void removeAllFutureMessagesLocked()

那我们就依次讲解下:

**移除方法1：void removeMessages(Handler , int , Object )方法**

> 从消息队列中删除所有符合指定条件的Message

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  587行



```csharp
  void removeMessages(Handler h, int what, Object object) {
        // 第1步
        if (h == null) {
            return;
        }
         // 第2步
        synchronized (this) {
           // 第3步
            Message p = mMessages;

            // Remove all messages at front.

          
            //第4步
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            //第5步
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

上面的代码大体可以分为5个步骤如下：

> - **第1步、**，对传递进来的Handler做非空判断，如果传递进来的Handler为空，则直接返回
> - **第2步、**，加同步锁
> - **第3步、**，获取消息队列链表的头元素
> - **第4步、**，如果从消息队列的头部就有符合删除条件的Message，就从头开始遍历删除所有符合条件的Message,并不端更新mMessages指向的Message。
> - **第5步、**，因为有了**第4步、**，前面的的情况不会发生，也就是我们不需要关心指向的问题，现在处理的问题就是删除剩下的符合删除条件的Message。

总结一下：

> 从消息队列中删除Message的操作也是遍历消息队列然后删除所有符合条件的Message，但是这里有连个小细节需要注意，从代码中可以看出删除Message分为两次操作，第一次是先判断符合删除条件的Message是不是从消息队列的头部就开始有了，这时候会设计修改mMessage指向的问题，而mMessage代表的就是整个消息队列，在排除了第一种情况之后，剩下的就是继续遍历队列删除剩余的符合删除条件的Message。其他重载方法也是同样的操作，唯一条件就是条件不同而已，

**移除方法2：void removeMessages(Handler, Runnable,Object)方法**

> 从消息队列中删除所有符合指定条件的Message

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  604行



```kotlin
    void removeMessages(Handler h, Runnable r, Object object) {
        if (h == null || r == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.callback == r
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

里面代码和移除方法1：void removeMessages(Handler , int , Object )基本一致，唯一不同就是筛选条件不同而已。

**移除方法3：void removeMessages(Handler, Runnable,Object)方法**

> 从消息队列中删除所有符合指定条件的Message

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  689行



```kotlin
    void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

里面代码和移除方法1：void removeMessages(Handler , int , Object )基本一致，唯一不同就是筛选条件不同而已。

**移除方法4：void removeAllMessagesLocked()方法**

> 删除所有的消息

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  722行



```csharp
   private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }
```

这个方法很简单，就是删除所有的消息

**移除方法5：void removeAllFutureMessagesLocked()**

> 删除所有未来消息

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  732行



```csharp
    private void removeAllFutureMessagesLocked() {
         // 第1步
        final long now = SystemClock.uptimeMillis();
         // 第2步
        Message p = mMessages;
        if (p != null) {
            // 第3步
            if (p.when > now) {
                removeAllMessagesLocked();
            } else {
                // 第4步
                Message n;
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
               // 第5步
                p.next = null;
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }
```

这个方法大体上分为5个步骤，具体解释如下：

> - 第1步：获取当前时间(其实从手机开机到现在的时间)
> - 第2步：获取消息队列链表的的头元素
> - 第3步：如果头元素的执行的时间就大于当前时间，因为我们知道链表的排序其实有从当前到未来的顺序排列的，所以但如果头元素大于当前时间，意味着这个链表的所有元素的执行时间都大于当前，则删除链表中的全部元素。
> - 第4步：如果消息队列中的头元素小于或等于当前时间，则说明要从消息队列中截取，从中间的某个未知的位置截取到消息队列链表的队尾。这个时候就需要找到这个具体的位置，这个步骤主要就是做这个事情。通过对比时间，找到合适的位置
> - 第5步：找到合适的位置后，就开始删除这个位置到消息队列队尾的所有元素

## 三、关闭消息队列

> 通过前面的文章，我们知道Handler消息机制的停止，本质上是停止Looper的循环，在[Android Handler机制4之Looper与Handler简介](https://www.jianshu.com/p/e635c3f851ea)文章中我们知道Looper的停止实际上是关闭消息队列的关闭，现在我们来揭示MessageQueue是如何关闭的

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  413行



```java
    void quit(boolean safe) {
         // 第1步
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        // 第2步
        synchronized (this) {
            // 第3步
            if (mQuitting) {
                return;
            }
            mQuitting = true;
            // 第4步
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            // 第5步
            nativeWake(mPtr);
        }
    }
```

这个方法内部大概分为5个步骤

> - 第1步：判断是否允许退出，因为在构造MessageQueue对象的时候传入了一个boolean参数，来表示该MessageQueue是否允许退出。而这个boolean参数在Looper里面设置，Loooper.prepare()方法里面是true，在Looper.prepareMainLooper()是false，由此可见我们知道：**主线程的MessageQueue是不能退出**。其他工作线程的MessageQueue是可以退出的。
> - 第2步：加上同步锁
> - 第3步：主要防止重复退出，加入一个mQuitting变量表示是否退出
> - 第4步：如果该方法的变量safe为true，则删除以当前时间为分界线，删除未来的所有消息，如果该方法的变量safe为false，则删除当前消息队列的所有消息。
> - 第5步：删除小时后nativeWake函数，以触发nativePollOnce函数，结束等待，这个块内容请在[Android Handler机制9之Native的实现]()中，这里就不详细描述了

## 四、查看消息是否存在

Handler机制也存在查找是否存在某条消息的机制，代码如下：



```java
// Handler.java
public final boolean hasMessages(int what) {
    return mQueue.hasMessages(this, what, null);
}

public final boolean hasMessages(int what, Object object) {
    return mQueue.hasMessages(this, what, object);
}

public final boolean hasCallbacks(Runnable r) {
    return mQueue.hasMessages(this, r, null);
}
```

我们发现其内部都是调用MessageQueue的hasMessages函数，那我们就来看下

### boolean hasMessages(Handler h, int what, Object object) 方法

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  587行



```kotlin
    boolean hasMessages(Handler h, int what, Object object) {
        //第1步
        if (h == null) {
            return false;
        }
        //第2步
        synchronized (this) {
            //第3步
            Message p = mMessages;
            //第4步
            while (p != null) {
                if (p.target == h && p.what == what && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
```

该方法的主要内容可以分为4个步骤

> - 第1步：判断传入进来的Handler是否为空，如果传入的Handler为空，直接返回false，表示没有找到
> - 第2步：加上同步锁
> - 第3步：取出消息队列链表中的头部元素
> - 第4步：遍历消息队里链表中的所有元素，如果有元素消息符合指定条件则return false，如果遍历完毕还没有则返回false

boolean hasMessages(Handler h, Runnable r, Object object)方法和本方法基本一致，唯一不同就是筛选条件不同而已。我就说讲解了。

## 五、阻塞非安全执行

> 如果当前执行线程是Handler的线程，Runnable会被立刻执行。否则把它放在消息队列中一直等待执行完毕或者超时，超时后这个任务还在队列中，在后面的某个时刻它仍然会执行，很有可能造成死锁，所以尽量不要用它。

这个方法使用场景是Android初始化一个WindowManagerService，应为WindowManagerService不成功，其他组件就不允许继续，所以使用阻塞的方式直到完成。
 代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java) 461行



```dart
    /**
     * Runs the specified task synchronously.
     * <p>
     * If the current thread is the same as the handler thread, then the runnable
     * runs immediately without being enqueued.  Otherwise, posts the runnable
     * to the handler and waits for it to complete before returning.
     * </p><p>
     * This method is dangerous!  Improper use can result in deadlocks.
     * Never call this method while any locks are held or use it in a
     * possibly re-entrant manner.
     * </p><p>
     * This method is occasionally useful in situations where a background thread
     * must synchronously await completion of a task that must run on the
     * handler's thread.  However, this problem is often a symptom of bad design.
     * Consider improving the design (if possible) before resorting to this method.
     * </p><p>
     * One example of where you might want to use this method is when you just
     * set up a Handler thread and need to perform some initialization steps on
     * it before continuing execution.
     * </p><p>
     * If timeout occurs then this method returns <code>false</code> but the runnable
     * will remain posted on the handler and may already be in progress or
     * complete at a later time.
     * </p><p>
     * When using this method, be sure to use {@link Looper#quitSafely} when
     * quitting the looper.  Otherwise {@link #runWithScissors} may hang indefinitely.
     * (TODO: We should fix this by making MessageQueue aware of blocking runnables.)
     * </p>
     *
     * @param r The Runnable that will be executed synchronously.
     * @param timeout The timeout in milliseconds, or 0 to wait indefinitely.
     *
     * @return Returns true if the Runnable was successfully executed.
     *         Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     *
     * @hide This method is prone to abuse and should probably not be in the API.
     * If we ever do make it part of the API, we might want to rename it to something
     * less funny like runUnsafe().
     */
    public final boolean runWithScissors(final Runnable r, long timeout) {
         // 第1步
        if (r == null) {
            throw new IllegalArgumentException("runnable must not be null");
        }
         // 第2步
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout must be non-negative");
        }

          // 第3步
         // 如果为同一个线程，则直接执行runnable，而不需要加入到消息队列。
        if (Looper.myLooper() == mLooper) {
            r.run();
            return true;
        }
 
        // 第4步
        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }
```

首先先简单翻译一下注释：

> - 同步运行指定的任务。
> - 如果当前线程就是Handler的处理线程，则可以不用排队，直接运行这个runnable。否则如果当前线程和Handler的处理编程不是同一个线程则需要发送这个runnable到Handler线程，并且等待它完成后再返回。
> - 使用这个方法是有风险的，使用不当可能会导致死锁。不要在有锁或者可能有锁的代码区域调用这个方法。
> - 这个方法的使用场景通常是，一个后台线程必须等待Handler线程中的一个任务的完成。但是，这往往是不优雅设计才会出现的问题。所以在使用这个方法的时候，请首先考虑改进设计方案。
> - 这个方法的使用场景是：在你建立Handler线程之前，你需要执行一些初始化操作。
> - 如果发生超时，虽然该方法还是会返回false，但是该
>    如果超时发生，那么该方法返回<code> false </ code>，但是runnable仍是会保留在Handler中，并且在一段时间以后会在被执行。
> - 在使用这个方法的时候，并且要退出一个Looper的时候，请一定要调用quitSafely()这个方法。否则runWithScissors()这个方法可能会无限期挂起。(TODO：我们应该通知MessageQueue去阻止runnable来解决这个问题)

该方法内部的执行流程主要分为4个步骤，如下：

> - **第1步、**：Runnable非空判断
> - **第2步、**：timeout是否小于0判断
> - **第3步、**：如果Looper的线程和Handler的线程是同一个线程
> - **第4步、**，构造一个BlockingRunnable对象，并调用该对象的postAndWait(Handler,long)方法

上面涉及到一个咱们之前没有讲解过的类：BlockingRunnable，他是Handler的静态内部类，我们来研究下

### Handler的静态内部类BlockingRunnable

BlockingRunnable是Handler的一个私有内部静态类，利用Object的wait和notifyAll方法实现。

代码在[Handler.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/Handler.java)



```java
  private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                     // runnable 执行完之后，会通知wait的线程不再wait
                    notifyAll();
                }
            }
        }

        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {
                return false;
            }

            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                       // post runnable 之后，将调用线程变为wait状态
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                         // post runnable 之后，将调用线程变为wait状态
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
```

通过分析源码我们获取的了如下信息：

> - 1、该类实现了Runnable接口
> - 2、构造函数：接受一个Runnable作为参数的构造函数，包含了真正要执行的Task。
> - 3、run函数很简单，直接调用mTask.run()，一个finally内会同步对象本身(因为mDone涉及到多线程，而notifyAll()则需要synchronized配合)
> - 4、postAndWait(Handler, long)：首先尝试将BlockingRunnable自己post到handler上，如果post失败，则直接返回false；其次如果上一步的post成功，就需要同步对象本身(为了使用wait())；此时，如果timeout>0，那么就一个while循环+wait(long)，中间有任何的interrupt都直接catch重新结算wait的时间，只有在任务完成(mDone=true，另外线程的run函数会设置此值)或者任何超时才会返回(true/false)；如果imeout <=0，也就无限等待了





# Handler机制之Native的实现

## 一、简述

> 前面的文章讲解了Java层的消息处理机制，其中MessageQueue类里面涉及到的多个Native方法，除了MessageQueue的native方法，native本身也有一套完整的消息机制，处理native消息。在整个消息机制中，MessageQueue是连接Java层和Native层的纽带，换而言之，Java层可以向MessageQueue消息队列中添加消息，Native层也可以向MessageQueue消息队列中添加消息。

Native的层关系图:

![img](https:////upload-images.jianshu.io/upload_images/5713484-89187a1ae0061e39.png?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)

Native关系图.png

## 二、MessageQueue

在MessageQueue的native方法如下：



```java
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

在[Android Handler机制6之MessageQueue简介]()中的**五、native层代码的初始化**中 说了MessaegQueue构造函数调用了nativeInit()，为了更好的理解，我们便从MessageQueue构造函数开始说起

### (一)、  nativeInit() 函数

> nativeInit() 的主要作用是初始化，是在MessageQueue的构造函数中调用

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 68行



```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        // 通过JNI调用了Native层的相关函数，导致了NativeMessageQueue的创建
        mPtr = nativeInit();
    }
```

> MessageQueue只是有一个构造函数，该构造函数是包内可见的，其内部就两行代码，分别是设置了MessageQueue是否可以退出和native层代码的相关初始化

在MessageQueue的构造函数里面调用 nativeInit()，我们来看下
 代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 61行



```java
    private native static long nativeInit();
```

> 根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeInit这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativeInit(JNIEnv* , jclass )函数

#### 1、jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz)方法

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp) 172 行



```cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    // 在Native层又创建了NativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
     // 这里返回值是Java层的mPtr，因此mPtr实际上是Java层MessageQueue与NativeMessesageQueue的桥梁
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

> 此时Java层和Native层的MessageQueue被mPtr连接起来了，NativeMessageQueue只是Java层MessageQueue在Native层的体现，其本身并没有实现Queue的数据结构，而是从其父类MessageQueue中继承mLooper变量。与Java层类型，这个Looper也是线程级别的单列。

代码中是直接new 的NativeMessageQueue无参构造函数，那我们那就来看下

#### 2、NativeMessageQueue无参构造函数

NativeMessageQueue是android_os_MessageQueue.cpp的内部类
 代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)  78行



```php
NativeMessageQueue::NativeMessageQueue() : mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    // 获取TLS中的Looper对象
    mLooper = Looper::getForThread(); 
    if (mLooper == NULL) {
        // 创建native层的Looper对象
        mLooper = new Looper(false); 
         // 保存native 层的Looper到TLS中(线程级单例)
        Looper::setForThread(mLooper); 
    }
}
```

- Looper::getForThread()：功能类比于Java层的Looper.myLooper();
- Looper::setForThread(mLooper)：功能类比于Java层的ThreadLocal.set()

> 通过上述代码我们知道：
>
> - 1、Java层的Looper的创建导致了MessageQueue的创建，而在Native层则刚刚相反，NativeMessageQueue的创建导致了Looper的创建
> - 2、MessageQueue是在Java层与Native层有着紧密的联系，但是此次Native层的Looper与Java层的Looper没有任何关系。
> - 3、Native层的Looper创建和Java层的也完全不一样，它利用了Linux的epoll机制检测了Input的fd和唤醒fd。从功能上来讲，这个唤醒fd才是真正处理Java Message和Native Message的钥匙。

PS:5.0以上的版本Loooer定义在System/core下

上面说了半天，那我们就来看下Native的Looper的构造函数

#### 3、 Native层的Looper的构造函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)  71行



```cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
     /**  这才是Linux后来才有的东西，负责线程通信，替换了老版本的pipe */
    //构造唤醒时间的fd
    mWakeEventFd = eventfd(0, EFD_NONBLOCK); 
    AutoMutex _l(mLock);
    rebuildEpollLocked();  
}
```

这个方法重点就是调用了rebuildEpollLocked();  函数

PS：这里说一下eventfd，event具体与pipe有点像，用来完成两个线程之间(现在也支持进程级别)，能够用来作为线程之间通讯，类似于pthread_cond_t。

#### 4、 Looper::rebuildEpollLocked() 函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)  140行



```cpp
void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    if (mEpollFd >= 0) {
#if DEBUG_CALLBACKS
        ALOGD("%p ~ rebuildEpollLocked - rebuilding epoll set", this);
#endif
        // 关闭老的epoll实例
        close(mEpollFd);
    }

    // Allocate the new epoll instance and register the wake pipe.
    // 创建一个epoll实例，并注册wake管道。
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    struct epoll_event eventItem;
    // 清空，把未使用的数据区域进行置0操作
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
     //关注EPOLLIN事件，也就是可读
    eventItem.events = EPOLLIN;
     //设置Fd
    eventItem.data.fd = mWakeEventFd;
    //将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)，其实只是为epoll放置一个唤醒机制
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance.  errno=%d",
            errno);
    // 这里主要添加的是Input事件，如键盘、传感器输入，这里基本上是由系统负责。
    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
         // 将request的队列事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set, errno=%d",
                    request.fd, errno);
        }
    }
```

这里一定要明白的是，添加这些fd除了mWakeEventFd负责解除阻塞让程序继续运行，从而处理Native Message和Java Message外，其他fd与Message的处理其实毫无关系。此时Java层与Native联系如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-6686a918de234d47.png?imageMogr2/auto-orient/strip|imageView2/2/w/417/format/webp)

Java与Native.png

这时候大家可能有点蒙，所以我下面补充1个知识点，希望能帮助大家

#### 5、 小结

所有整个流程整理如下图：



![img](https:////upload-images.jianshu.io/upload_images/5713484-00d37c498065b163.png?imageMogr2/auto-orient/strip|imageView2/2/w/710/format/webp)

流程.png

### (二) nativeDestroy()

> nativeDestroy是在MessageQueue的dispose()方法中调用，主要用于清空回收

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 84行



```cpp
    // Disposes of the underlying message queue.
    // Must only be called on the looper thread or the finalizer.
    private void dispose() {
        if (mPtr != 0) {
            // native方法
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }
```

> 根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeDestroy()这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativeDestroy()函数

#### 1、android_os_MessageQueue_nativeDestroy()函数

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 183行



```cpp
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    // 强制类型转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    //调用nativeMessageQueue的decStrong()函数
    nativeMessageQueue->decStrong(env);
}
```

我们看到上面代码是

> - 首先，将Java层传递下来的mPtr转换为nativeMessageQueue
> - 其次，nativeMessageQueue调用decStrong(env)

nativeMessageQueue继承自RefBase类，所以decStrong最终调用的是RefBase.decStrong()。

[Android跨进程通信IPC之4——AndroidIPC基础2](https://www.jianshu.com/p/28406f85e266)的第五部分**五、智能指针**，中对智能指针有详细描述，这里就不过多介绍了

#### 2、总体流程图

![img](https:////upload-images.jianshu.io/upload_images/5713484-9260340af6d00374.png?imageMogr2/auto-orient/strip|imageView2/2/w/703/format/webp)

nativeDestroy()流程.png

### (三) nativePollOnce()

> nativePollOnce()是在MessageQueue的next()方法中调用，用于提取消息的调用链

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)   323行



```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    for (;;) {
        ...
        //阻塞操作
        nativePollOnce(ptr, nextPollTimeoutMillis);
        ...
    }
```

> 根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeDestroy()这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativePollOnce()函数

#### 1、nativePollOnce()

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java) 188行



```cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    //将Java层传递下来的mPtr转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis); 
}
```

我们看到上面代码是

> - 首先，将Java层传递下来的mPtr转换为nativeMessageQueue
> - 其次，nativeMessageQueue调用pollOnce(env, obj, timeoutMillis)

那我们就来看下pollOnce(env, obj, timeoutMillis)方法

#### 2、 NativeMessageQueue::pollOnce(JNIEnv*, jobject, int)函数



```php
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    // 重点函数
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

> 这个函数内容很简答， 主要就是进行赋值，并调用pollOnce(timeoutMillis)

那我们再来看一下pollOnce(timeoutMillis)函数

#### 3、Looper::pollOnce()函数

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h) 264 行



```cpp
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, NULL, NULL, NULL); 
}
```

> 这个函数里面主要是调用的是ollOnce(timeoutMillis, NULL, NULL, NULL);

#### 4、Looper::pollOnce(int, int*, int*, void**)函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 264 行



```cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    // 对fd对应的Responses进行处理，后面发现Response里都是活动fd
    for (;;) {
        // 先处理没有Callback的Response事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                // ident>=0则表示没有callback，因为POLL_CALLBACK=-2
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
         // 注意这里处于循环内部，改变result的值在后面的pollInner
        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        // 再处理内部轮训
        result = pollInner(timeoutMillis);
    }
}
```

参数说明：

> - timeoutMillis:超时时长
> - outFd:发生事件的文件描述符
> - outEvents：当前outFd上发生的事件，包含以下4类事件
>   - EVENT_INPUT：可读
>   - EVENT_OUTPUT：可写
>   - EVENT_ERROR：错误
>   - EVENT_HANGUP：中断
> - outData：上下文数据

这个函数内部最后调用了pollInner(int)，让我们来看一下

#### 5、Looper::pollInner()函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 220 行



```cpp
int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif

    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %" PRId64 "ns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }

    // Poll.
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
     // 即将处于idle状态
    mPolling = true;
    // fd最大的个数是16
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    // 等待时间发生或者超时，在nativeWake()方法，向管道写端写入字符，则方法会返回。
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    // 不再处于idle状态
    mPolling = false;
     // 请求锁 ，因为在Native Message的处理和添加逻辑上需要同步
    // Acquire lock.
    mLock.lock();

    // Rebuild epoll set if needed.
    // 如果需要，重建epoll
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        // epoll重建，直接跳转到Done
        rebuildEpollLocked();
        goto Done;
    }

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        // epoll事件个数小于0，发生错误，直接跳转Done
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    //如果需要，重建epoll
    if (eventCount == 0) {
    //epoll事件个数等于0，发生超时，直接跳转Done
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif
   // 循环处理所有的事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        //首先处理mWakeEventFd
        if (fd == mWakeEventFd) {
            //如果是唤醒mWakeEventFd有反应
            if (epollEvents & EPOLLIN) {
                /**重点代码*/
                // 已经唤醒了，则读取并清空管道数据
                awoken();  // 该函数内部就是read，从而使FD可读状态被清除
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            // 其他input fd处理，其实就是将活动放入response队列，等待处理
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                 // 处理request，生成对应的response对象，push到响应数组
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;
    // Invoke pending message callbacks.
    // 再处理Native的Message，调用相应回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                 // 释放锁
                mLock.unlock();

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                // 处理消息事件
                handler->handleMessage(message);
            } // release handler
            // 请求锁
            mLock.lock();
            mSendingMessage = false;
             // 发生回调
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    // 释放锁
    mLock.unlock();

    // Invoke all response callbacks.
    // 处理带有Callback()方法的response事件，执行Response相应的回调方法
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            // 处理请求的回调方法
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                // 移除fd
                removeFd(fd, response.request.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
             // 清除response引用的回调方法
            response.request.callback.clear();
             // 发生回调
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

pollOnce返回值说明：

> - POLL_WAKE： 表示由wake()出发，即pipe写端的write事件触发
> - POLL_CALLBACK：表示某个被监听fd被触发
> - POLL_TIMEOUT：表示等待超时
> - POLL_ERROR：表示等待期间发生错误

pollInner()方法的处理流程：

> - 1、先调用epoll_wait()，这是阻塞方法，用于等待事件发生或者超时。
> - 2、对于epoll_wait()返回，当且仅当以下3种情况出现
>   - POLL_ERROR：发生错误，直接跳转Done
>   - POLL_TIMEOUT：发生超时，直接跳转到Done
>   - 检测到管道有事情发生，则再根据情况做相应处理：
>     - 如果检测到管道产生事件，则直接读取管道的数据
>     - 如果是其他事件，则处理request，生成对应的response对象，push到response数组
> - 3、进入Done标记位的代码：
>   - 先处理Native的Message，调用Native的Handler来处理该Message
>   - 再处理Resposne数组，POLL_CALLBACK类型的事件

从上面的流程，可以发现对于Request先收集，一并放入response数组，而不是马上执行。真正在Done开始执行的时候，先处理Native Message，再处理Request，说明Native Message优先级高于Request请求的优先级。

PS:在polOnce()方法中，先处理Response数组不带Callback的事件，再调用了再调用了pollInner()函数。

#### 6、Looper::awoken()函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 418行



```cpp
void Looper::awoken() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ awoken", this);
#endif
    uint64_t counter;
    // 不断的读取管道数据，目的就是为了清空管道内容
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}
```

#### 7、小结

整体的流程图如下：



![img](https:////upload-images.jianshu.io/upload_images/5713484-f255be313ceb8898.png?imageMogr2/auto-orient/strip|imageView2/2/w/725/format/webp)

流程图.png

### (四)、nativeDestroy()

nativeWake用于唤醒功能，在添加消息到消息队列enqueueMessage()，或者把消息从消息队列中全部移除quit()，再有需要时会调用nativeWake方法。包含唤醒过程的添加消息的调用链
 下面来进一步来看看调用链的过程：

#### 1、enqueueMessage(Message, long)

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)  533行



```csharp
    boolean enqueueMessage(Message msg, long when) {
          ....
          //将Message按按时间插入MessageQueue
            if (needWake) {
                nativeWake(mPtr);
            }
         ....
    }
```

在向消息队列添加Message时，需要根据mBlocked情况来就决定是否需要调用nativeWake。

根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeDestroy()这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativeWake(JNIEnv*, jclass, jlong ) 函数

#### 2、android_os_MessageQueue_nativeWake()

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)  194行



```cpp
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    // 将Java层传递下来的mPtr转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    //调用wake函数
    nativeMessageQueue->wake();
}
```

我们看到上面代码是

> - 首先，将Java层传递下来的mPtr转换为nativeMessageQueue
> - 其次，nativeMessageQueue调用wake()函数

#### 3、NativeMessageQueue::wake()函数

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)  121行



```cpp
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

这个方法很简单，就是直接调用Looper的wake()函数，

#### 4、Looper::wake()函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)  404行



```cpp
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    uint64_t inc = 1;
    // 向管道mWakeEventFd写入字符1
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

> Looper类的 wake()函数只是往mWakeEventfd中写了一些内容，这个fd只是通知而已，类似于pipi，最后会把epoll_wai唤醒，线程就不阻塞了继续发送
>  Native层的消息，然后处理之前的addFd事件，然后处理Java层的消息。

PS:其中TEMP_FAILURE_RETRY 是一个宏定义，当执行write失败后，会不断重复执行，直到执行成功为止。

#### 5、小结

总结一下流程图如下:

![img](https:////upload-images.jianshu.io/upload_images/5713484-cd211827ac547ced.png?imageMogr2/auto-orient/strip|imageView2/2/w/728/format/webp)

流程图.png

### (五)、sendMessage()

> 前面几篇文章讲述了Java层如何向MessageQueue类添加消息，那么接下来讲讲Native层如何向MessageQueue发送消息。

#### 1、Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) 函数

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 583行



```cpp
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now, handler, message);
}
```

> 我们看到方法里面调用了sendMessageAtTime(now, handler, message) 函数

#### 2、 Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,



```cpp
    const Message& message)函数
```

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 588行



```cpp
void Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
        const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now + uptimeDelay, handler, message);
}
```

> 我们看到方法里面调用了sendMessageAtTime(now, handler, message) 函数

所以我们说：

> **sendMessage()**和**sendMessageDelayed()**都是调用**sendMessageAtTime()**来完成消息插入。

那我们就来看一下sendMessageAtTime()

#### 3、 Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,



```cpp
    const Message& message)函数
```



```cpp
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ sendMessageAtTime - uptime=%" PRId64 ", handler=%p, what=%d",
            this, uptime, handler.get(), message.what);
#endif

    size_t i = 0;
    { // acquire lock
       // 请求锁
        AutoMutex _l(mLock);

        size_t messageCount = mMessageEnvelopes.size();
       // 找到message应该插入的位置i
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }

        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

        // Optimization: If the Looper is currently sending a message, then we can skip
        // the call to wake() because the next thing the Looper will do after processing
        // messages is to decide when the next wakeup time should be.  In fact, it does
        // not even matter whether this code is running on the Looper thread.
        // 如果当前正在发送消息，那么不再调用wake()，直接返回
        if (mSendingMessage) {
            return;
        }
    } // release lock
    // 释放锁
    // Wake the poll loop only when we enqueue a new message at the head.
    // 当消息加入到消息队列的头部时，需要唤醒poll循环
    if (i == 0) {
        wake();
    }
}
```

### (六)、sendMessage()

本节介绍了MessageQueue的native()方法，经过层层调用：

> - nativeInit()方法，最终实现由epoll机制中的epoll_create()/epoll_ctl()完成
> - nativeDestory()方法，最终实现由RefBase::decStrong()完成
> - nativePollOnce()方法，最终实现由Looper::pollOnce()完成
> - nativeWake()方法，最终实现由Looper::wake()调用write方法，向管道写入字符
> - nativeIsPolling()，nativeSetFileDescriptorEvents()这两个方法类似，此处就不一一列举了。

## 三、Native结构体和类

> [Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h)/[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)文件中定义了Message结构体，消息处理类，回调类，Looper类

### (一)、Message结构体

代码在([http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h)) 50行



```cpp
struct Message {
    Message() : what(0) { }
    Message(int what) : what(what) { }

    /* The message type. (interpretation is left up to the handler) */
    // 消息类型
    int what;
};
```

### (二)、消息处理类

#### 1、MessageHandler类

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h) 67行



```cpp
/**
 * Interface for a Looper message handler.
 *
 * The Looper holds a strong reference to the message handler whenever it has
 * a message to deliver to it.  Make sure to call Looper::removeMessages
 * to remove any pending messages destined for the handler so that the handler
 * can be destroyed.
 */
class MessageHandler : public virtual RefBase {
protected:
    virtual ~MessageHandler() { }

public:
    /**
     * Handles a message.
     */
    virtual void handleMessage(const Message& message) = 0;
};
```

这个类很简单，就不多说了，这里说下注释:

> - 处理Looper消息程序的接口。
> - 当一个消息要传递给其对应的Handler时候，Looper持有一个消息Handler的强引用。在这个Handler销毁之前，请确保调用Looper :: removeMessages来删除待处理的消息。

#### 2、WeakMessageHandler类

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h) 82行



```cpp
/**
 * A simple proxy that holds a weak reference to a message handler.
 */
class WeakMessageHandler : public MessageHandler {
protected:
    virtual ~WeakMessageHandler();

public:
    WeakMessageHandler(const wp<MessageHandler>& handler);
    virtual void handleMessage(const Message& message);

private:
    wp<MessageHandler> mHandler;
};
```

这里并没有handleMessage的代码，我们是不是忽略了什么？再找一下，果然这块的代码在
 [Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 38行



```cpp
void WeakMessageHandler::handleMessage(const Message& message) {
    sp<MessageHandler> handler = mHandler.promote();
    if (handler != NULL) {
        调用Mes
        handler->handleMessage(message); 
    }
}
```

### (三)、回调类

#### 1、LooperCallback类

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h) 98行



```cpp
/**
 * A looper callback.
 */
class LooperCallback : public virtual RefBase {
protected:
    virtual ~LooperCallback() { }

public:
    /**
     * Handles a poll event for the given file descriptor.
     * It is given the file descriptor it is associated with,
     * a bitmask of the poll events that were triggered (typically EVENT_INPUT),
     * and the data pointer that was originally supplied.
     *
     * Implementations should return 1 to continue receiving callbacks, or 0
     * to have this file descriptor and callback unregistered from the looper.
     */
    // 用于处理指定的文件描述符poll事件
    virtual int handleEvent(int fd, int events, void* data) = 0;
};
```

简单翻译一下handleEvent方法的注释：

> - 处理给定文件描述符的轮训事件。
> - 用来 将 最初提供的数据指针和轮训事件的掩码(通常为EVENT_INPUT)来关联的文件描述符。
> - 实现子类如果想继续接收回调则返回1，如果未注册文件描述符和回调则返回0

#### 2、SimpleLooperCallback类

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp) 118行



```cpp
class SimpleLooperCallback : public LooperCallback {
protected:
    virtual ~SimpleLooperCallback();
public:
    SimpleLooperCallback(Looper_callbackFunc callback);
    virtual int handleEvent(int fd, int events, void* data);
private:
    Looper_callbackFunc mCallback;
};
```

它和WeakMessageHandler类一样handleEvent的方法在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)  55行



```cpp
int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    // 调用回调方法
    return mCallback(fd, events, data); 
}
```

### (四)、Looper类

#### 1 、 Native层的Looper类简介

[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)

#### 2 、 Native层的Looper类常量



```cpp
// 每个epoll实例默认的文件描述符个数
static const int EPOLL_SIZE_HINT = 8; 
 // 轮训事件的文件描述符个数上限
static const int EPOLL_MAX_EVENTS = 16; 
```

#### 3、Native Looper类的常用方法：

| 方法                                                         |                        解释                         |
| ------------------------------------------------------------ | :-------------------------------------------------: |
| Looper（bool）                                               |                  Looper的构造函数                   |
| static sp<Looper>  prepar(int)                               | 如果该线程没有绑定Looper，才创建Loopr，否则直接返回 |
| int pollOnec(int ,int* int*,void*)                           |                 轮训，等待事件发生                  |
| void wake()                                                  |                     唤醒Looper                      |
| void sendMessage(const sp<MessageHandler>&handler,const Message&message) |                      发送消息                       |
| int addFd(int,int,int,Looper_callbackFunc,void*)             |              添加要监听的文件描述符fd               |

#### 4、Request、Resposne、MessageEvent 三个结构体

Looper类的内部定义了Request、Resposne、MessageEnvelope这三个结构体
 关系图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-1c05ba12a1c93152.png?imageMogr2/auto-orient/strip|imageView2/2/w/677/format/webp)

结构体关系图.png

4.1、**Request**  **结构体**

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h)  420行



```cpp
// 请求结构体
struct Request { 
    int fd;
    int ident;
    int events;
    int seq;
    sp<LooperCallback> callback;
    void* data;
    void initEventItem(struct epoll_event* eventItem) const;
};
```

**4.2、Resposne  结构体**

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h)  431行



```cpp
// 响应结构体
struct Response { 
    int events;
    Request request;
};
```

**4.3、MessageEnvelope  结构体**

代码在[Looper.h](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/include/utils/Looper.h)  436行



```cpp
// 信封结构体
struct MessageEnvelope { 
    MessageEnvelope() : uptime(0) { }
    MessageEnvelope(nsecs_t uptime, const sp<MessageHandler> handler,
            const Message& message) : uptime(uptime), handler(handler), message(message) {
    }
    nsecs_t uptime;
    sp<MessageHandler> handler;
    Message message;
};
```

> MessageEnvelope正如其名字，信封。MessageEnvelope里面记录着收信人(handler)，发信时间(uptime)，信件内容(message)。

#### 5、Native Looper类的类图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-c47e2d01b6154d78.png?imageMogr2/auto-orient/strip|imageView2/2/w/1027/format/webp)

类图.png

#### 6 Native Looper的监听文件描述符

> Native Looper除了提供message机制外，还提供监听文件描述符的方式。通过**addFd()**接口加入需要被监听的文件描述符。

代码在[Looper.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/system/core/libutils/Looper.cpp)  434行



```cpp
    int addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data);
    int addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data);  
```

其中：

> - fd:为所需要监听的文件描述符
> - ident:表示为当前发生时间的标识符，必须>=0,或者为POLL_CALLBACK(-2)如果指定了callback
> - events:表示为要监听的文件类型，默认是EVENT_INPUT。
> - callback:当有事件发生时，会回调该callback函数。
> - data:两种使用方式：
>   - 指定callback来处理事件：当该文件描述符上有事件来时，该callback会被执行，然后从fd读取数据。这个时候ident是被忽略的。
>   - 通过指定的ident来处理事件：当该文件描述符有数据来到时，pollOnce()会返回一个ident，调用者会判断该ident是否等于自己需要处理事件ident，如果是的话，则开始处理事件。

### (五)、Java层的addFd

> 我之前一直以为只能在C层的Looper中才能addFd，原来在Java层也通过JNI做了这个功能。我们可以在MessageQueue中的addOnFileDescriptorEventListener来实现这个功能。
>  代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)    186行



```dart
    /**
     * Adds a file descriptor listener to receive notification when file descriptor
     * related events occur.
     * <p>
     * If the file descriptor has already been registered, the specified events
     * and listener will replace any that were previously associated with it.
     * It is not possible to set more than one listener per file descriptor.
     * </p><p>
     * It is important to always unregister the listener when the file descriptor
     * is no longer of use.
     * </p>
     *
     * @param fd The file descriptor for which a listener will be registered.
     * @param events The set of events to receive: a combination of the
     * {@link OnFileDescriptorEventListener#EVENT_INPUT},
     * {@link OnFileDescriptorEventListener#EVENT_OUTPUT}, and
     * {@link OnFileDescriptorEventListener#EVENT_ERROR} event masks.  If the requested
     * set of events is zero, then the listener is unregistered.
     * @param listener The listener to invoke when file descriptor events occur.
     *
     * @see OnFileDescriptorEventListener
     * @see #removeOnFileDescriptorEventListener
     */
    public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, events, listener);
        }
    }
```

通过上面代码分析，我们知道这里面有两个重点

> - 1 onFileDescriptorEventListener 这个回调
> - 2 updateOnFileDescriptorEventListenerLocked()方法

**1、OnFileDescriptorEventListener**

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)    186行



```dart
   /**
     * A listener which is invoked when file descriptor related events occur.
     */
    public interface OnFileDescriptorEventListener {
        /**
         * File descriptor event: Indicates that the file descriptor is ready for input
         * operations, such as reading.
         * <p>
         * The listener should read all available data from the file descriptor
         * then return <code>true</code> to keep the listener active or <code>false</code>
         * to remove the listener.
         * </p><p>
         * In the case of a socket, this event may be generated to indicate
         * that there is at least one incoming connection that the listener
         * should accept.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_INPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_INPUT = 1 << 0;

        /**
         * File descriptor event: Indicates that the file descriptor is ready for output
         * operations, such as writing.
         * <p>
         * The listener should write as much data as it needs.  If it could not
         * write everything at once, then it should return <code>true</code> to
         * keep the listener active.  Otherwise, it should return <code>false</code>
         * to remove the listener then re-register it later when it needs to write
         * something else.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_OUTPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_OUTPUT = 1 << 1;

        /**
         * File descriptor event: Indicates that the file descriptor encountered a
         * fatal error.
         * <p>
         * File descriptor errors can occur for various reasons.  One common error
         * is when the remote peer of a socket or pipe closes its end of the connection.
         * </p><p>
         * This event may be generated at any time regardless of whether the
         * {@link #EVENT_ERROR} event mask was specified when the listener was added.
         * </p>
         */
        public static final int EVENT_ERROR = 1 << 2;

        /** @hide */
        @Retention(RetentionPolicy.SOURCE)
        @IntDef(flag=true, value={EVENT_INPUT, EVENT_OUTPUT, EVENT_ERROR})
        public @interface Events {}

        /**
         * Called when a file descriptor receives events.
         *
         * @param fd The file descriptor.
         * @param events The set of events that occurred: a combination of the
         * {@link #EVENT_INPUT}, {@link #EVENT_OUTPUT}, and {@link #EVENT_ERROR} event masks.
         * @return The new set of events to watch, or 0 to unregister the listener.
         *
         * @see #EVENT_INPUT
         * @see #EVENT_OUTPUT
         * @see #EVENT_ERROR
         */
        @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
    }

    private static final class FileDescriptorRecord {
        public final FileDescriptor mDescriptor;
        public int mEvents;
        public OnFileDescriptorEventListener mListener;
        public int mSeq;

        public FileDescriptorRecord(FileDescriptor descriptor,
                int events, OnFileDescriptorEventListener listener) {
            mDescriptor = descriptor;
            mEvents = events;
            mListener = listener;
        }
    }
```

**2、updateOnFileDescriptorEventListenerLocked()方法**

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)    222行



```java
    private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
            OnFileDescriptorEventListener listener) {
        final int fdNum = fd.getInt$();

        int index = -1;
        FileDescriptorRecord record = null;
        if (mFileDescriptorRecords != null) {
            index = mFileDescriptorRecords.indexOfKey(fdNum);
            if (index >= 0) {
                record = mFileDescriptorRecords.valueAt(index);
                if (record != null && record.mEvents == events) {
                    return;
                }
            }
        }

        if (events != 0) {
            events |= OnFileDescriptorEventListener.EVENT_ERROR;
            if (record == null) {
                if (mFileDescriptorRecords == null) {
                    mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
                }
                //fd保存在FileDescriptorRecord对象
                record = new FileDescriptorRecord(fd, events, listener);
                // mFileDescriptorRecords 保存
                mFileDescriptorRecords.put(fdNum, record);
            } else {
                record.mListener = listener;
                record.mEvents = events;
                record.mSeq += 1;
            }
            // 调用native函数
            nativeSetFileDescriptorEvents(mPtr, fdNum, events);
        } else if (record != null) {
            record.mEvents = 0;
            mFileDescriptorRecords.removeAt(index);
        }
    }
```

**2.1、android_os_MessageQueue_nativeSetFileDescriptorEvents()函数**

> 根据[Android跨进程通信IPC之3——关于"JNI"的那些事](https://www.jianshu.com/p/cd038167d896)中知道，nativeInit这个native方法对应的是[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)里面的android_os_MessageQueue_nativeSetFileDescriptorEvents(JNIEnv* env, jclass clazz,  jlong ptr, jint fd, jint events)函数

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)    204行



```cpp
static void android_os_MessageQueue_nativeSetFileDescriptorEvents(JNIEnv* env, jclass clazz,
        jlong ptr, jint fd, jint events) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->setFileDescriptorEvents(fd, events);
}
```

我们看到这个函数里面调用了nativeMessageQueue的setFileDescriptorEvents(fd, events);函数。

**2.2、NativeMessageQueue::setFileDescriptorEvents(int fd, int events)函数**

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)   125行



```cpp
void NativeMessageQueue::setFileDescriptorEvents(int fd, int events) {
    if (events) {
        int looperEvents = 0;
        if (events & CALLBACK_EVENT_INPUT) {
            looperEvents |= Looper::EVENT_INPUT;
        }
        if (events & CALLBACK_EVENT_OUTPUT) {
            looperEvents |= Looper::EVENT_OUTPUT;
        }
        // 重点代码
        mLooper->addFd(fd, Looper::POLL_CALLBACK, looperEvents, this,
                reinterpret_cast<void*>(events));
    } else {
        mLooper->removeFd(fd);
    }
}
```

我们看到了在这个函数内部调用了mLooper的addFd函数。

> 大家注意一下Looper的addFd函数，中的倒数二个参数是this，侧面说明了NativeMessageQueue继承了LooperCallback。

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)   41行



```cpp
class NativeMessageQueue : public MessageQueue, public LooperCallback {
public:
    NativeMessageQueue();
    virtual ~NativeMessageQueue();

    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj);

    void pollOnce(JNIEnv* env, jobject obj, int timeoutMillis);
    void wake();
    void setFileDescriptorEvents(int fd, int events);

    virtual int handleEvent(int fd, int events, void* data);
   ...
}
```

所以说，需要实现handleEvent()函数。handleEvent()函数就是在looper中epoll_wait之后，当我们增加的fd有数据就会调用这个函数。

代码在[android_os_MessageQueue.cpp](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)   141行



```cpp
int NativeMessageQueue::handleEvent(int fd, int looperEvents, void* data) {
    int events = 0;
    if (looperEvents & Looper::EVENT_INPUT) {
        events |= CALLBACK_EVENT_INPUT;
    }
    if (looperEvents & Looper::EVENT_OUTPUT) {
        events |= CALLBACK_EVENT_OUTPUT;
    }
    if (looperEvents & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP | Looper::EVENT_INVALID)) {
        events |= CALLBACK_EVENT_ERROR;
    }
    int oldWatchedEvents = reinterpret_cast<intptr_t>(data);
    // 调用回调
    int newWatchedEvents = mPollEnv->CallIntMethod(mPollObj,
            gMessageQueueClassInfo.dispatchEvents, fd, events); /
    if (!newWatchedEvents) {
        return 0; // unregister the fd
    }
    if (newWatchedEvents != oldWatchedEvents) {
        setFileDescriptorEvents(fd, newWatchedEvents);
    }
    return 1;
}
```

最后在java的MessageQueue中的dispatchEvent就是在jni层反调过来的，然后调用之前注册的回调函数

代码在[MessageQueue.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/MessageQueue.java)259行



```java
    // Called from native code.
    private int dispatchEvents(int fd, int events) {
        // Get the file descriptor record and any state that might change.
        final FileDescriptorRecord record;
        final int oldWatchedEvents;
        final OnFileDescriptorEventListener listener;
        final int seq;
        synchronized (this) {
            record = mFileDescriptorRecords.get(fd);
            if (record == null) {
                return 0; // spurious, no listener registered
            }

            oldWatchedEvents = record.mEvents;
            events &= oldWatchedEvents; // filter events based on current watched set
            if (events == 0) {
                return oldWatchedEvents; // spurious, watched events changed
            }

            listener = record.mListener;
            seq = record.mSeq;
        }

        // Invoke the listener outside of the lock.
        int newWatchedEvents = listener.onFileDescriptorEvents(
                record.mDescriptor, events);
        if (newWatchedEvents != 0) {
            newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
        }

        // Update the file descriptor record if the listener changed the set of
        // events to watch and the listener itself hasn't been updated since.
        if (newWatchedEvents != oldWatchedEvents) {
            synchronized (this) {
                int index = mFileDescriptorRecords.indexOfKey(fd);
                if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                        && record.mSeq == seq) {
                    record.mEvents = newWatchedEvents;
                    if (newWatchedEvents == 0) {
                        mFileDescriptorRecords.removeAt(index);
                    }
                }
            }
        }

        // Return the new set of events to watch for native code to take care of.
        return newWatchedEvents;
    }
```

### (六)、总结

#### (一)、Native与Java的对应关系

> MessageQueue通过mPtr变量保存了NativeMessageQueue对象，从而使得MessageQueue成为Java层和Native层的枢纽，既能处理上层消息，也能处理Native消息，下图列举了Java层与Native层的对应图

![img](https:////upload-images.jianshu.io/upload_images/5713484-d799896f90a6c209.png?imageMogr2/auto-orient/strip|imageView2/2/w/947/format/webp)

对应图.png

图解：

> - 1、红色虚线关系：Java层和Native层的MessageQueue通过JNI建立关联，彼此之间能相互调用，搞明白这个互调关系，也就搞明白Java如何调用C++代码，C++代码如何调用Java代码
> - 2、蓝色虚线关系：Handler/Looper/Message这三大类Java层与Native层并没有任何真正的关系，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。
> - 3、WeakMessageHandler继承与MessageHandler类，NativeMessageQueue继承于MessageQueue类。

另外，消息处理流程是先处理NativeMessage，再处理Native Request，最后处理Java Message。理解了该流程也就明白了有时上层消息很少，但响应时间却比较长的真正原因。

#### (二)、Native的流程

整体流程如下：

![img](https://upload-images.jianshu.io/upload_images/5713484-8b73101169262093.png)

整体流程.png

## 四、总结

Handler机制中Native的实现主要涉及了两个类

> - 1、**NativeMessageQueue**：在MessageQueue.java的构造函数中，调用了nativeInit创建了NativeMessageQueue对象，并且把指针变量返回给Java层的mPtr。而在NativeMessageQueue的构造函数中，会在当前线程中创建C++的Looper对象。
> - 2、Looper：控制eventfd的读写，通过epoll监听eventfd的变化，来阻塞调用pollOnce和恢复调用wake当前线程
>   - 通过 epoll监听其他文件描述符的变化
>   - 通过 epoll处理C++层的消息机制，当调用Looper::sendMessageAtTime后，调用wake触发epoll
>   - Looper的构造函数，创建一个eventfd(以前版本是pipe)，eventfd它的主要用于进程或者线程间的通信，然后创建epoll来监听该eventfd的变化
>   - Looper::pollOnce(int timeoutMillis) 内部调用了pollInner，再调用epoll_wait(mEpollFd, ..., timeoutMillis)阻塞timeoutMills时间，并监听文件描述符mEpollFd的变化，当时间到了或者消息到了，即eventfd被写入内容后，从epoll_wait继续往下执行，处理epoll_wait返回的消息，该消息既有可能是eventfd产生的，也可能是其他文件描述符产生的。处理顺序是，先处理普通的C++消息队列mMessageEnvelopes，然后处理之前addFd的事件，最后从pollOnce返回，会继续MessageQueue.java的next()函数，取得Java层的消息来处理；
>   - Looper类的wake，函数只是往mWakeEventfd中写了一些内容，这个fd只是通知而已，类似pipe，最后会把epoll_wait唤醒，线程就不阻塞了，继续先发送C层消息，然后处理之前addFd事件，然后处理Java层消息



# Handler机制之Handler机制总结



## 一、Handler机制的思考

- 先提一个问题哈，如果让你设计一个操作系统，你会怎么设计？

> 我们一般操作系统都会有一个消息系统，里面有个死循环，不断的轮训处理其他各种输入设备输入的信息，比如你在在键盘的输入，鼠标的移动等。这些输入信息最终都会进入你的操作系统，然后由操作系统的内部轮询机制挨个处理这些信息。

- 那Android系统那？它内部是怎么实现的? 如果让你设计，你会怎么设计？

> 答：如果让我设计，肯定和上面一样：
>
> - 1 设计一个类，里面有一个死循环去做循环操作；
> - 2 用一个类来抽象代表各种输入信息/消息；这个信息/消息应该还有一个唯一标识符字段；如果这个信息里面有个对象来保存对应的键值对；方便其他人往这个信息/消息 存放信息；这个信息/消息应该有个字段标明消息产生的时间；
> - 3 而上面的这些 信息/消息 又组成一个集合。常用集合很多，那是用ArrayList好还是LinkedList或者Map好那？因为前面说了是一个死循环去处理，所以这个集合最好是"线性和排序的"比较好，因为输入有先后，一般都是按照输入的时间先后来构成。既然这样就排除了Map，那么就剩下来了ArrayList和LinkedList。我们知道一个操作系统的事件是很多的，也就是说对应的信息/消息很多，所以这个集合肯定会面临大量的"插入"操作，而在"插入"效能这块，LinkedList有着明显的优势，所以这个集合应该是一个链表，但是链表又可以分为很多种，因为是线性排序的，所以只剩下"双向链表"和"单向链表”，但是由于考虑下手机的性能问题，大部分人肯定会倾向于选择"单向链表"，因为"单项链表"在增加和删除上面的复杂度明显低于"双向链表"。
> - 4、最后还应该有两个类，一个负责生产这个输入信息，一个负责消费这些信息。因为涉及到消费端，所以上面2中说的信息/消息应该有一个字段负责指向消费端。

**经过上面的思考，大家是不是发现和其实我们Handler的机制基本上一致。Looper负责轮询；Message代表消息，为了区别对待，用what来做为标识符，when表示时间，data负责存放键值对；MessageQueue则代表Message的集合，Message内部同时也是单项链表的。通过上面的分析，希望大家对Handler机制的总体设计有不一样的感悟。**

## 二、Handler消息机制

> 如果你想要让一个Android的应用程序反应灵敏，那么你必须防止它的UI线程被阻塞。同样地，将这些阻塞的或者计算密集型的任务转到工作线程去执行也会提高程序的响应灵敏性。然而，这些任务的执行结果通常需要重新更新UI组件的显示，但该操作只能在UI线程中去执行。有一些方法解决了UI线程的阻塞问题，例如阻塞对象，共享内存以及管道技术。Android为了解决这个问题，提供了一种自有的消息传递机制——Handler。Handler是Android Framework架构中的一个基础组件，它实现了一种非阻塞的消息传递机制，在消息转换的过程中，消息的生产者和消费者都不会阻塞。

Handler由以下部分组成：

> - Handler
> - Message
> - MessageQueue
> - Looper

下面我们来了解下它们及它们之间的交互。

### (一)、Handler

> Handler 是线程间传递消息的即时接口，生产线程和消费线程用以下操作来使用Handler
>
> - 生产线程：在消息队列中创建、插入或移除消息
> - 消费线程：处理消息

![img](https:////upload-images.jianshu.io/upload_images/5713484-3874332a0b06fd42.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Handler.png

> 每个Handler都有一个与之关联的Looper和消息队列。有两种创建Handler方式(这里不是说只有两个构造函数，而是说把它的构造函数分为两类)
>
> - 通过默认的构造方法，使用当前线程中关联的Looper
> - 显式地指定使用Looper

如果没有指定Looper的Handler是无法工作的，因为它无法将消息放到消息队列中。同样地，它无法获取要处理的消息。



```tsx
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

如果是使用上面Handler的构造函数，它会检查当前线程有没有可用的Looper对象，如果没有，它会抛出一个运行时的异常，如果正常的话，Handler会持有Looper中的消息队列对象的引用。

> PS：同一个线程中多的Handler分享一个同样的消息队列，因为他们分享的是同一个Looper对象

Callback参数是一个可选的参数，如果提供的话，它将会处理由Looper分发的过来的消息。

### (二)、Message

> Message 是容纳任意数据的容器。生产线程发送消息给Handler，Handler将消息加入到消息队列中。消息提供了三种额外的信息，以供Handler和消息队列处理时使用：
>
> - what：一种标识符，Handler能使用它来区分不同的消息，从而采取不同的处理方法
> - time：告诉消息队列合适处理消息
> - target：表示那一个Handler应该处理消息

android.os.Message 消息一般是通过Handler中以下方法来创建的



```java
public final Message obtainMessage()
public final Message obtainMessage(int what)
public final Message obtainMessage(int what, Object obj)
public final Message obtainMessage(int what, int arg1, int arg2)
public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
```

消息从消息池中获取得到，方法中提供的参数会放到消息体对应的字段中。Handler同样可以设置消息的目标为其自身，这允许我们进行链式调用，比如：



```css
mHandler.obtainMessage(MSG_SHOW_IMAGE, mBitmap).sendToTarget();
```

消息池是一个消息对象的单项链表集合，它的最大长度是50。在Handler处理完这条消息之后，消息队列把这个对象返回到消息池中，并且重置其所有字段。

当使用Handler调用post方法来执行一个Runnable时，Handler隐式地创建了一个新的消息，并且设置callback参数来存储这个Runnable。



```undefined
Message m = Message.obtain();
m.callback = r;
```

![img](https:////upload-images.jianshu.io/upload_images/5713484-ca47b9954eb1c6ef.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Handler与Message.png

生产线程发送消息给 Handler 的交互

> 在上图中，我们能看到生产线程和 Handler 的交互。生产者创建了一个消息，并且发送给Handler，随后Handler 将这个消息加入消息队列中，在未来某个时间，Handler 会在消费小城中处理这个消息。

### (三)、MessageQueue

> MessageQueue是一个消息体对象的无界的单向链表集合，它按照时序将消息插入队列，最小的时间戳将会被首先处理。

![img](https:////upload-images.jianshu.io/upload_images/5713484-58e392cf3e3d74cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

MessageQueue.png

消息队列也通过SystemClock.uptimeMillis获取当前时间，维护一个阻塞阀值(dispatch barrier)。当一个消息体的时间戳低于这个值的时候，消息就会分发给Handler进行处理

Handler 提供了三种方式来发送消息：



```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
public final boolean sendMessageAtFrontOfQueue(Message msg)
public boolean sendMessageAtTime(Message msg, long uptimeMillis)
```

以延迟的方式发送消息，是设置了消息体的time字段为SystemClock.uptimeMillis()+delayMillis。然而，通过sendMessageAtFontOfQueue方法是把消息插入到队首，会将其时间字段设置为0，消息会在下一次轮训时被处理。需要谨慎使用这个方法，因为它可能会英系那个消息队列，造成顺序问题，或是其他不可预料的副作用。

### (四)、消息队列、Handler、生产线程的交互

现在我们可以概括消息队列、Handler、生产线程的交互：

![img](https:////upload-images.jianshu.io/upload_images/5713484-7bd73f7bc0b3c31d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

消息队列、Handler、生产线程的交互.png

> 上图中，多个生产线程提交消息到不同的Handler中，然而，不同的Handler都与同一个Looper对象关联，因此所有的消息都加入到同一个消息队列中。这一点非常重要，Android中创建的许多不同的Handler都关联到主线程的Looper。

比如：

- The Choreographer：处理垂直同步与帧更新
- The ViewRoot：处理输入和窗口时间，配置修改等等
- The InputMethodManager不会大量生成消息，因为这可能会抑制处理系统

### (五)、Looper

> Looper 从消息队列中读取消息，然后分发给对应的Handler处理。一旦消息超过阻塞阀，那么Looper就会在下一轮读取过程中读取到它。Looper在没有消息分发的时候变成阻塞状态，当有消息可用时会继续轮询。

每个线程只能关联一个Looper，给线程附加的另外的Looper会导致运行时的异常。通过使用Looper的Threadlocal对象可以保证线程只关联一个Looper对象。

调用Looper.quit()方法会立即终止Looper，并且丢弃消息队列中的已经通过阻塞阀的所有消息。调用Looper.quitSafely()方法能够保证所有待分发的消息在队列中等待的消息被丢弃前得到处理。

![img](https:////upload-images.jianshu.io/upload_images/5713484-e6bc447b4f0a75a8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Looper.png

> Handler 与消息队列和Looper 直接交互的整体流程
>  Looper 应在线程的run方法中初始化。调用静态方法Looper.prepare()会检查线程是否与一个已存在的Looper关联。这个过程的实现是通过Looper类中的ThreadLocal对象来检查Looper对象是否存在。如果Looper不存在，将会创建一个新的Looper对象和一个新的消息队列。如下代码展示了这个过程

**PS: 公有的prepare方法会默认调用prepare(true)**



```csharp
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException(“Only one Looper may be created per thread”);
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Handler 现在能接收到消息并加入到消息队列中，执行静态方法Looper.loop()方法会开始将消息从消息队列中出队。每次轮训迭代器指向下一条消息，接着分发消息对应目标地的Handler，然后回收消息到消息池中。Looper.looper()方法循环执行这个过程，直到Looper终止。下面代码片段展示这个过程：



```java
public static void loop() {
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
    }
}
```

### (六)、整体流程图

![img](https:////upload-images.jianshu.io/upload_images/5713484-45b9074f48b990c8.gif?imageMogr2/auto-orient/strip|imageView2/2/w/1024/format/webp)

整体流程图.gif

## 三、享元模式

> 享元模式是对象池的一种实现，尽可能减少内存的使用，使用缓存来共享可用的对象，避免创建过多的对象。Android中Message使用的设计模式就是享元模式，将获取Message通过obtain方法从对象池获取，Message使用结束通过recyle将Message归还给对象池，达到循环利用对象，避免重复创建的目的

### (一)概念

> **享元模式(Flywight Pattern)** 是一种软件设计模式。它使用共享物件，用来尽可能减少内存使用量以及分享咨询给尽可能多的相似物件；它适用于只是因重复而导致无法使用无法令人接受的大量内存的大量物件。通常物件中的部分状态时可以分享的。常见的做法是把他们放到外部数据结构，当需要使用时将他们传递给享元。

### (二) 为什么要用享元模式

> - 当一个软件系统在运行时产生的对象数量太多，将导致运行代价过高，带来系统性能下降的问题。所以需要采用一个共享来避免大量拥有相同内容对象的开销。在Java中，String类型就是使用享元模式。String对象是final类型，对象一旦创建就不可改变。在Java字符串常量都是存在常量池中的，Java会确保一个字符串常量在常量池只有一个拷贝。
> - 是对对象池的一种实现，共享对象，避免重复的创建，采用一个共享来避免大量拥有相同内容对象的开销。使用享元模式可以有效支持大量的细粒度对象。

Flyweight，如果很多很小的对象它们有很多相同的东西，并且在很多地方用到，那就可以把它们抽取成一个对象，把不同的东西变成外部属性，作为方法的参数传入。

String类型的对象创建后就不可改变，如果两个String对象所包含的内容相同时，JVM只创建一个String对象对应这两个不同的对象引用。字符串常量池。

### (三)  核心思想

1、**概念**

> 运行共享技术有效地支持大量细粒度对象的复用。系统只使用少量的对象，而这些对象都很相似，状态变化很小，可以实现对象的多次复用。由于享元模式要求能够共享对象必须是细颗粒对象，因此它又称为轻量级模式，它是一种对象结构模式。

享元对象共享的关键是区分了内部状态(Intrinsic State)和外部状态(Extrinsic State)。

**2、内部状态(Intrinsic State)**

> 存储在享元对象内部并且不会随环境改变而改变的状态，内部状态可以共享。

**3、外部状态(Extrinsic State)**

> 享元对象的外部状态通常由客户端保存，并在享元对象被创建之后，需要使用的时候再传入到享元对象内部。随环境改变而改变的、不可以共享的状态。一个外部状态与另一个外状态是相互独立的。

由于区分了内部状态和外部状态，我们可以将具有相同内部状态的对象存储在享元池中，享元池中的对象是可以实现共享的，需要的时候将对象从享元池中取出，实现对象的复用。通过向取出的对象注入不同的外部状态，可以得到一系列相似的对象，而这些对象在内存中实际上只存储一份。

### (四) 享元模式分类

> - 单纯享元模式
> - 复合享元模式

**1、单纯享元模式结构重要核心模块**

- **抽象享元角色**：为具体享元角色规定了必须实现的方法，而外部状态时以参数的行贿通过此方法传入。在Java中可以由抽象类、接口担当
- **具体享元角色**：实现抽象橘色规定的方法。如果存在内部状态，就负责为内部状态提供存储空间。
- **享元端角色***：负责创建和管理享元角色。想要达到共享目的，这个角色的实现是关键。
- **客户端角色**：维护对所有享元对象的引用，而且还需要存储对应的外部状态。

> 单纯享元模式和创新型的简单工厂模式实际上非常相似，但是它的重点或者用意却和工厂模式截然不同。工厂模式的使用主要是为了使用系统不依赖于实现的细节；而在享元模式的主要目的是避免大量拥有相同内容对象的开销。

**2、复合享元模式**

- **抽象享元角色**：为了具体享元角色规定了必须实现的方法，而外部状态就是以参数的形式听过此方法传入。在Java中可以由抽象类、接口来担当。
- **具体享元角色**：实现抽象角色规定的方法。如果存在内部状态，就负责为内部状态提供存储空间。
- **复合享元角色**：它所代表的对象是不可以共享的，并且可以分解为多个单纯享元对象的组合。
- **享元工厂角色**：负责创建和管理享元角色。想要达到共享的目的，这个角色的实现是关键！
- **客户端角色**：维护对所有享元对象的引用，而且还需要存储对应的外部状态。

### (五) 享元模式的使用场景

一般在如下场景中使用享元模式

> - 1 一个系统有大量相同或相似的对象，造成内存大量耗费。
> - 2 对象大部分状态都可以外部化，可以将这些外部状态传入对象中。
> - 3 再使用享元模式时需要维护一个存储享元对象的享元池，而这需要耗费一定的系统资源，因此，应该在需要多次重复使用享元对象时才值得使用享元模式。

## 四、HandlerThread

[HandlerThread官网](https://link.jianshu.com?t=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2Fos%2FHandlerThread.html)

### (一)、HandlerThread 简介

> 我们看到HandlerThread很快就会联想到Handler。Android中Handler的使用，一般都在UI线程中执行，因此在Handler接受消息后，处理消息时，不能做一些很耗时的操作，否则将出现ANR错误。Android中专门提供了HandlerThread类，来解决该类问题。HandlerThread是一个线程专门处理Handler的消息，依次从Handler的队列中获取信息，逐个进行处理，保证安全，不会出现混乱引发的异常。HandlerThread继承于Thread，所以它卑职就是一个Thread。与普通Thread的差别就在于，它有个Looper成员变量。

我们看下官方对它的简介：

> Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called.

翻译一下:

> HandlerThread可以创建一个带有looper的线程。Looper对象可以用于创建Handler类来进行调度。

### (二)、类源码解析

代码在[HandlerThread.java](https://link.jianshu.com?t=http%3A%2F%2Fandroidxref.com%2F6.0.1_r10%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fos%2FHandlerThread.java)



```dart
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
public class HandlerThread extends Thread {
    // 线程的优先级 
    int mPriority;

    // 线程id
    int mTid = -1;

    // Looper对象，消息对象以及循环
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        //设置默认的线程优先级
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
    // 自定义设置线程优先级
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    //如果有需要这个方法可以重写，例如可以在这里声明这个Handler关联此线程
    protected void onLooperPrepared() {
    }
    //Thread线程的run方法
    @Override
    public void run() {
        //获取当前线程的id
        mTid = Process.myTid();

        // 一旦调用这句代码，就在此线程中创建了Looper对象，这就是为什么我们要在调用线程start()方法后，才能得到Looper对象，即当调用Looper.myLooper()时不为null
        Looper.prepare();

        // 同步代码块，意思就是当获取mLooper对象后对象后，唤醒所有线程
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
   
        // 设置线程优先级
        Process.setThreadPriority(mPriority);

        //调用上面的方法，需要用户重写
        onLooperPrepared();
        
        // 开启消息循环
        Looper.loop();
        mTid = -1;
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason is isAlive() returns false, this method will return null. If this thread 
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
        // 如果线程已经死了，所以返回null
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        // 同步代码块，正好和上面(run()方法里面的)形成对应，就是说，只要线程活着并且我的looper为null，那么我就让你一直等
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    /**
     * Quits the handler thread's looper.
     * <p>
     * Causes the handler thread's looper to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     *
     * @see #quitSafely
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            // 退出消息循环
            looper.quit();
            return true;
        }
        return false;
    }

    /**
     * Quits the handler thread's looper safely.
     * <p>
     * Causes the handler thread's looper to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * Pending delayed messages with due times in the future will not be delivered.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p>
     * If the thread has not been started or has finished (that is if
     * {@link #getLooper} returns null), then false is returned.
     * Otherwise the looper is asked to quit and true is returned.
     * </p>
     *
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    // 返回线程ID
    public int getThreadId() {
        return mTid;
    }
}
```

> 整体来说上面代码还是比较浅显易懂的。主要作用是建立了一个线程，并且创建了消息队列，有自己的Looper，可以让我们在自己线程中分发和处理消息。

quit和quitSafely都是退出HandlerThread的消息循环，其分别调用Looper的quit和quitSafely方法。我们在这里简单说下区别:

> - 1 quit方法会将消息队列中的所有消息移除(延迟消息和非延迟消息)。
> - 2 quitSafely 会将消息队列所有延迟消息移除。非延迟消息则派发出去让Handler去处理。
> - quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。

### (三)、HandlerThread的使用

通过上面的源码，我们大概也能推测出HandlerThread的使用步骤，它的使用步骤如下：

> - 第一步：创建一个HandlerThread实例，本质是创建一个包含Looper的线程。比如



```cpp
  HandlerThread handlerThread=new HandlerThread("name")
```

> - 第二步：开启线程



```css
  handlerThread.start()
```

> - 第三步：获得线程Looper



```undefined
 Looper looper=handlerThread.getLooper();
```

> - 第四步：创建Handler，并用looper初始化



```cpp
Handler  handler=new Handler(looper);
```

> - 第五步：利用handle进行一些操作
> - 第六步：调用quit()或者quitSafely()来终止它的循环

所以说整体流程如下：

> 当我们使用HandlerThrad创建一个线程，它start()之后会在它的线程创建一个Looper对象且初始化一个MessageQueue，通过Looper对象在他的线程构建一个Handler对象，然后我们通过Handler发送消息的形式将任务发送到MessaegQueue中，因为Looper是顺序处理消息的，所以当有多个任务存在时会顺序的排队执行。但我们不使用的时候我们应该调用它的quit()或者quitSafely()来终止它的循环。

### (四)、HandlerThread和普通Thread的比较

> HandlerThread继承自Thread，当线程开启时，也就是它run方法运行起来后，线程同时创建了了一个含有消息队列的Looper，并对外提供自己这个Looper对象的get方法。

### (五)、HandlerThread的使用场景

> - 1、开发中如果多次使用类似new Thread(){...}.start()。这种方式开启一个子线程，会创建多个匿名线程，使得程序运行起来越来越慢，而HandlerThread自带Looper使他可以通过消息来多次重复使用当前线程，节省开支。
> - 2、android系统提供的Handler类内部的Looper默认绑定的是UI线程的消息队列，对于非UI线程又想使用消息机制，那么HandlerThread内部的Looper是最合适的，它不会干扰或阻塞UI线程。
> - 3、HandlerThread适合处理本地I/O读写操作(比如数据库)，因为本地I/O操作大多数的耗时属于毫秒级别的，对于单线程+异步队列的形式不会产生较大的阻塞。而网络操作相对于比较耗时，容易阻塞后面的请求，因此在这个HandlerThread中不合适加入网络操作。

### (五) 小结：

> - 1、HandlerThread将loop转到子线程中去处理，说白了就是分担MainLooper的工作量，降低了主线程压力，使主界面更流程。
> - 2、开启一个线程起到多个线程的作用。处理任务是串行执行，按消息发送顺序进行处理。HandlerThread本质是一个线程，在线程内部，代码是串行处理的。但是由于每一个任务都将以队列的方式逐个被执行到，一旦队列中某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。HandlerThread拥有自己的消息队列，它不会干扰或阻塞UI线程。
> - 3、对于网络I/O操作，HandlerThread并不合适，因为它只有一个线程，还得排队一个一个等着。

## 五、Handler的内存泄露

### (一)、概述

android使用Java作为开发环境，Java的跨平台和垃圾回收机制已经帮助我们解决了底层的一些问题。但是尽管有了垃圾回收机制，在开发android的时候仍然时不时遇到out of memory的问题，这个时候我们不禁要问，垃圾回收器去哪里了？这里我们主要讲解handler引起的泄露，并且给出了几种解决方案，并且最后提供一个第三方库WeakHandler库。

可能导致泄漏问题的handler一般会被提示 Lint警告如下：



```kotlin
This Handler class should be static or leaks might occur 
```

意思是Handler class应该使用静态声明，否则可能会出现内存泄露
 下面是更详细的说明(Android Studio，现在应该没人用Eclipse了吧)

> Since this Handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler; Make all references to members of the outer class using the WeakReference object.

大概意思是：

> 一旦Handler 被声明为内部类，那么可能导致它的外部类不能够被垃圾回收，如果Handler在其他线程(我们通常称为工作线程(worker thread))使用Looper或MessageQueue(消息队列)，而不是main线程(UI线程)，那么久没有有这个问题。如果Handler使用Looper或MessageQueue在主线程(main thread)，你需要对Handler的声明做如下修改：
>  声明Handler为static类；在外部类实例化一个外部类的WeakReferernce(弱引用)并且在Handler初始化时传入这个对象给你的Handler；将所有引用的外部类成员使用WeakReference对象。

### (二)、什么是内存泄露

> Java使用有向图机制，通过GC自动检查内存中的对象(什么时候检查由虚拟机决定)，如果GC发现一个或一组对象为不可到达状态，则将该对象从内存中回收。也就是说，一个对象不被任何应用所指向，则该对象会在被GC发现的时候被回收；另外，如果一组对象只包含相互的引用，没没有来自他们外部的引用(例如有两个对象A和B相互持有引用，但没有任何外部对象持有指向A或B的引用)，这让然属于不可叨叨，同样会被GC回收。

### (三)、为什么会内存泄露

原因：

> - 当Android应用启动的时候，会先创建一个应用主线程的Looper对象，Looper实现了一个简单的消息队列，一个一个的处理里面的Message对象。主线程Looper对象在整个应用生命周期中存在。
> - 当在主线程中初始化Handler时，该Handler和Looper的消息队列关联。发送到消息的队列的Message会应用发送该消息的Handler对象，这样系统可以调用Handler.handleMessage(Message)来分发处理该消息。
> - 在Java中，非静态(匿名)内部类会引用外部类对象。而静态内部类不会引用外部类对象。
>    -垃圾回收机制中约定，当内存中的一个对象的引用计数为0时，将会被回收。
> - 如果外部类是Activity，则会引起Activity泄露。当Acitivity finish后，延时消息会继续存在主线程消息队列中1分钟，然后处理消息。而该消息引用了Activity的Handler对象，然后这个Handler又引用了这个Activity。这些引用对象会保持到该消息被处理完，这样就导致了该Activity对象无法被回收，从而导致了上面所说的Activity泄露。

所以说如果要修改该问题，只需要按照Lint提示那样，把Handler类定义为静态即可，然后通过WeakReference拉埃保持外部的Activity对象。

### (四)、什么是WeakReference？

> WeakReference弱引用，与强引用(即我们常说的引用)相对，它的特点是，GC在回收时会忽略掉弱引用，即就算有弱引用指向某对象，但只要该对象没有被强引用所指向(实际上多数时候还要求没有软引用，但此处软件用的概念可以忽略)，该对象就会在被GC检查到时回收掉。对于上面的代码，用户在关闭Activity之后，就算后台线程还没有结束，但由于仅有一条来自Handler的弱引用指向Activity，所以GC仍然会在检查的时候把Activity回收掉。这样内存泄露的问题就不会出现了。

### (五)、Handler内存泄露使用弱引用的补充

> 一般将Handler声明为static就不必造成内存泄露，声明成弱引用Activity的话，虽然也不会造成内存泄露，但是需要等到handler中没有执行任务后才会回收，因此性能不高。

所以说使用弱引用可以解决内存泄露，但是需要等到Handler中任务都执行完，才会释放activity内存，不如直接static释放的快。所以说单独使用弱引用性能不是太高。

### (六)、WeakHandler

> **WeakHandler**使用起来和handler一样，但它是线程安全的，WeakHandler使用如下：

**1、WeakHandler的使用**



```java
public class ExampleActivity extends Activity {

    private WeakHandler mHandler; // We still need at least one hard reference to WeakHandler
    
    protected void onCreate(Bundle savedInstanceState) {
        mHandler = new WeakHandler();
        ...
    }
    
    private void onClick(View view) {
        mHandler.postDelayed(new Runnable() {
            view.setVisibility(View.INVISIBLE);
        }, 5000);
    }
}
```

你只需要将在以前的Handler替换成WeakHandler就行了。

**2、WeakHandler的原理**

> WeakHandler的思想是将Handler和Runnable做一次封装，我们使用的是封装后的WeakHandler，但其实真正起到Handler作用的是封装的内部，而封装的内部对handler和runnable都是用的弱引用。

![img](https:////upload-images.jianshu.io/upload_images/5713484-02d9d02c4e8d2d87.png?imageMogr2/auto-orient/strip|imageView2/2/w/585/format/webp)

weakhandler.png

> - 第一幅图是普通handler的引用关系图
> - 第二幅图是使用WeakHandler的引用关系

其实原文有对WeakHandler跟多的解释，但是表述起来也是挺复杂的。

原文地址：[https://techblog.badoo.com/blog/2014/10/09/calabash-android-query/](https://link.jianshu.com?t=https%3A%2F%2Ftechblog.badoo.com%2Fblog%2F2014%2F10%2F09%2Fcalabash-android-query%2F)
 github项目地址：[https://github.com/badoo/android-weak-handler](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fbadoo%2Fandroid-weak-handler)

## 六、Handler的面试题

**1、为什么安卓要使用Handler?**

> 因为android更新UI只能在UI线程。为什么只能在UI线程更新UI？因为Android是单线程模型。为什么Android是单线程模型？那是因为如果任一线程都可以更新UI的话，线程安全处理起来相当麻烦，所以就规定了Android是单线程模型，只允许在UI线程更新UI

**2、消息机制的原理:**

> 这个请参考  **本篇文章 二、Handler消息机制**

**3、MessageQueue是什么时候创建的？**

> MessageQueue是在Looper的构造函数里面创建的，所以一个线程对应一个Looper，一个Looper对应一个MessageQueue。

**4、ThreadLocal在Handler机制中的作用**

> ThreadLocal是一个线程内部的数据存储类，通过它可以在制定的线程中存储数据，数据存储以后，只有在指定线程中可以获取的存储的数据，对于其他线程就获取不到数据。一般来说，当某些数据是以线程为作用域而且不同线程需要有不同的数据副本的时候，可以考虑用ThreadLocal。比如对于Handler，它要获取当前线程的Looper，很显然Looper的作用域就是线程，所以不同线程有不同的Looper。

**5、Looper,Handler,MessageQueue的引用关系?**

> 一个Handler对象持有一个MessageQueue和它构造时所属的线程的Looper引用。也就是说一个Handler必须顶有它对应的消息队列和Looper。一个线程可能有多个Handler，但是至多有只能有一个Looper和一个消息队列。
>  在主线程中new了一个Handler对象后，这个Handler对象自动和主线程生成的Looper以及消息队列关联上了。子线程中拿到主线程中Handler的引用，发送消息后，消息对象就会发送到target属性对应的的那个Handler对应的消息队列中去，由对应Looper来处理(子线程msg->主线程handler->主线程messageQueue->主线程Looper->主线程Handler的handlerMessage)。而消息发送到主线程Handler，那么也就是发送到主线程的消息队列，用主线程的Looper轮询。

**6、MessageQueue里面的数据结构是什么类型的，为什么不是Map或者其他什么类型的?**

> 这个请参考  **本片文章 一、Handler机制的思考**

**7、Handler引起的内存泄漏以及解决办法**

> 这个请参考  **本片文章 五、Handler的内存泄露**



# Handler机制之Callable、Future和FutureTask

## 一、概述

> - Java从发布的一个版本开始就可以很方便地编写多线程的应用程序，并在设计中引入异步处理。创建线程有两种方式，一种是直接继承Thread，另外一种就是实现Runnable接口。Thread类、Runnable接口和Java内存管理模型使得多线程编程简单直接。但是大家知道Thread和Runnable接口都不允许声明检查性异常，也不能返定义返回值。没有返回值这点稍微有点麻烦。
> - 不能声明抛出检查型异常则更麻烦一些。public void run()方法契约意味着你必须捕获并处理检查型异常。即使你小心地保存了异常信息(在捕获异常时)以便稍后检查， 但也不能保证这个类(Runnnable)的所有使用者都读取这个异常信息。你也可以修改Runnable实现的getter，让他们都能抛出任务执行中的异常。但这种方法除了繁琐也不是十分安全可靠，你不能强迫使用者调用这些方法，程序员很可能会调用join()方法等待线程结束后就不管了。
> - 但是现在不用担心了，以上问题终于在1.5中解决了。Callable接口和Future接口接口的引入以及他们对线程池的支持优雅地解决了这个两个问题。

## 二、Callable

[Callable.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/util/concurrent/Callable.java)

### (一) Runnable

说到Callable就不能不说下java.lang.Runnable，它是一个接口，它只声明了一个run()方法，由于这个run()方法的返回值是void的，所以在执行完任务之后无法返回任何结果。



```dart
public  interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```

### (二) Callable

> Callable位于java.util.concurrent包下，它也是一个接口，在它里面也只声明了一个方法，只不过这个方法叫做call()



```php
/**
 * A task that returns a result and may throw an exception.
 * Implementors define a single method with no arguments called
 * {@code call}.
 *
 * <p>The {@code Callable} interface is similar to {@link
 * java.lang.Runnable}, in that both are designed for classes whose
 * instances are potentially executed by another thread.  A
 * {@code Runnable}, however, does not return a result and cannot
 * throw a checked exception.
 *
 * <p>The {@link Executors} class contains utility methods to
 * convert from other common forms to {@code Callable} classes.
 *
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> the result type of method {@code call}
 */
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
     //
    V call() throws Exception;
}
```

> 可以看到，这一个泛型的接口，call()函数返回的类型就是传递进来的V类型。

### (三) Callable的类注释

为了让我们更好的理解Callable，还是从类的注释开始，翻译如下：

> - 有执行结果的，并且可以引发异常的任务
>    它的实现类需要去实现没有参数的的方法，即call()方法
> - Callable有点类似于Runnable接口，因为它们都是设计成被线程去执行的可执行代码的实例。由于Runnable是没有返回值的，并且不能抛出一个检查出的异常。
> - Executors类包含方法可以使得Callable转化成其他普通形式。

### (四)、 Runnable和Callable的区别：

> - 1、Runnable是Java 1.1有的，而Callable是1.5之后才加上去的
> - 2、Callable规定的方法是call()，Runnable规定的方法是run()
> - 3、Callable的任务执行后可返回值，而Runnable的任务是不能返回(因为是void)
> - 4、call()方法是可以抛出异常的，而run()方法不可以
> - 5、运行Callable任务可以拿到一个Future对象，表示异步计算的结果，它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果，通过Future对象可以了解任务的执行情况，可取消任务的执行，还可获取执行结果
> - 6、加入线程池运行，Runnable使用ExecutorService的execute()方法，Callable使用submit()方法

## 三、Future

[Future.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/util/concurrent/Future.java)

### (一) Future类详解

> Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞知道任务返回结果。Future类位于java.util.concurrent包下，它是一个接口：



```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

在Future接口中声明了5个方法，下面依次解释下每个方法的作用：

> -  **boolean cancel(boolean mayInterruptIfRunning)**：方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置为true，则表示可以取消正在执行过程中任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
> -  **boolean isCancelled()**：表示任务是否被取消成功，如果在任务正常完成之前取消成功则返回true.
> -  **isDone()**：方法表示任务是否已经完成，若任务完成，则返回true。
> -  **V get()**：方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回
> -  **V get(long timeout, TimeUnit unit)**：用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

总结一下，Future提供了三种功能:

> - 判断任务是否完成
> - 能够中断任务
> - 能够获取任务的执行结果

### (二) Future类注释

> - Future可以表示异步计算的结果。Future提供一个方法用来检查这个计算是否已经完成，还提供一个方法用来检索计算结果。get()方法可以获取计算结果，这个方法里面可能产生阻塞，如果产生了阻塞了，就阻塞到计算结束。cancel()方法可以取消执行。还有一些方法可以用来确定任务是否已经完成、是否已经取消成功了。如果任务已经执行完毕，则是不能取消的。如果你想使用Future并且，希望它是不可撤销的，同时不关心执行的结果，可以声明Future的泛型，并且基础任务返回值结果为null。
> - 使用示例
>
> 
>
> ```dart
>    class App {
>        ExecutorService executor = ...
>       ArchiveSearcher searcher = ...
> 
>         void showSearch(final String target)
>                 throws InterruptedException {
>             Future<String> future
>                     = executor.submit(new Callable<String>() {
>                 public String call() {
>                     return searcher.search(target);
>                 }
>             });
>             displayOtherThings(); // do other things while searching
>             try {
>                 displayText(future.get()); // use future
>             } catch (ExecutionException ex) {
>                 cleanup();
>                 return;
>             }
>         }
>    }
> ```
>
> FutureTask是Future的具体实现类，同时也实现了Runnable。所以FutureTask可以被Executor执行。例如Executor的submit方法可以换成如下写法
>
> 
>
> ```tsx
> FutureTask<String> future =
>   new FutureTask<>(new Callable<String>() {
>     public String call() {
>       return searcher.search(target);
>   }});
> executor.execute(future);
> ```
>
> 内存一致性效应：如果想在另一个线程调用 Future.get()方法，则在调用该方法之前应该先执行其自己的操作。

### (三)  boolean cancel(boolean mayInterruptIfRunning)方法注释

翻译如下：

> 尝试去关闭正在执行的任务，如果任务已经完成，或者任务已经被取消，或者任务因为某种原因而无法被取消，则关闭事失败。当这个任务还没有被执行，则调用此方法会成功，并且这个任务将来不会被执行。如果任务已经开始了，mayInterruptIfRunning这个入参决定是否应该中断该任务。这个方法被执行返回后，再去调用isDone()方法，将一直返回true。如果调用这个方法后返回true，再去调用isCancelled()方法，则isCancelled()方法一直返回true。mayInterruptIfRunning这个参数表示的是该任务的线程是否可以被中断，true表示可以中断，如果这个任务不能被取消，则返回false，而大多数这种情况是任务已完成。

上面已经提到了Future只是一个接口，所以是无法直接用来创建对象使用的，在注释里面推荐使用FutureTask，那我们就来看下FutureTask

## 四、FutureTask

[FutureTask.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/util/concurrent/FutureTask.java)
 我们先来看一下FutureTask的实现：

### (一)、RunnableFuture

通过代码我们知道



```ruby
public class FutureTask<V> implements RunnableFuture<V>
```

说明FutureTask类实现了RunnableFuture接口，我们看一下RunnableFuture接口

[RunnableFuture.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/util/concurrent/RunnableFuture.java)



```php
/**
 * A {@link Future} that is {@link Runnable}. Successful execution of
 * the {@code run} method causes completion of the {@code Future}
 * and allows access to its results.
 * @see FutureTask
 * @see Executor
 * @since 1.6
 * @author Doug Lea
 * @param <V> The result type returned by this Future's {@code get} method
 */
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

> 通过代码我知道RunnableFuture继承了Runnable接口和Future接口，而FutureTask实现了RunnableFuture接口。所以它既可以作为Runnable被线程执行，也可以作为Future得到Callable的返回值

### (二)、FutureTask的类注释



```php
/**
 * A cancellable asynchronous computation.  This class provides a base
 * implementation of {@link Future}, with methods to start and cancel
 * a computation, query to see if the computation is complete, and
 * retrieve the result of the computation.  The result can only be
 * retrieved when the computation has completed; the {@code get}
 * methods will block if the computation has not yet completed.  Once
 * the computation has completed, the computation cannot be restarted
 * or cancelled (unless the computation is invoked using
 * {@link #runAndReset}).
 *
 * <p>A {@code FutureTask} can be used to wrap a {@link Callable} or
 * {@link Runnable} object.  Because {@code FutureTask} implements
 * {@code Runnable}, a {@code FutureTask} can be submitted to an
 * {@link Executor} for execution.
 *
 * <p>In addition to serving as a standalone class, this class provides
 * {@code protected} functionality that may be useful when creating
 * customized task classes.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <V> The result type returned by this FutureTask's {@code get} methods
 */
```

为了更好的理解作者设计，先来看下类注释，翻译如下：

> - 一个可以取消的异步执行，这个类是Future的基础实现类，提供一下方法，比如可以去开启和关闭执行，查询是否已经执行完毕，以及检索计算的结果。这个执行结果只能等执行完毕才能获取，如果还未执行完毕则处于阻塞状态。除非调用runAndReset()方法，否则一旦计算完毕后则无法重启启动或者取消。
> - Callable或者Runnable可以包装FutureTask，由于FutureTask实现Runnable，所以在Executor可以执行FutureTask
> - 除了可以作为一个独立的类外，FutureTask还提供一些protected方法，这样在自定义任务类是，就会很方便。

### (三)、FutureTask的结构

结构如下图



![img](https:////upload-images.jianshu.io/upload_images/5713484-845992814b025912.png?imageMogr2/auto-orient/strip|imageView2/2/w/1026/format/webp)

FutureTask结构图.png

继承关系如下:

![img](https:////upload-images.jianshu.io/upload_images/5713484-f8e05e959876a4da.png?imageMogr2/auto-orient/strip|imageView2/2/w/695/format/webp)

继承关系.png

### (四)、静态final类WaitNode



```dart
    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

先翻译一下注释

> 精简版的链表结构，其中每一个节点代表堆栈中等待的线程。如果想了解更多的详细说明，请参考其他类比如Phaser和SynchronousQueue

结构如下图

   ![img](https://upload-images.jianshu.io/upload_images/5713484-16f69c36488e3809.png?imageMogr2/auto-orient/strip|imageView2/2/w/494/format/webp) 

WaitNode.png

再来看下构造函数和成员变量

> - thread：代表等待的线程
> - next：代表下一个节点，通过这个节点我们也能退出这个链表是单向链表
> - 构造函数：无参的构造函数里面将thread设置为当前线程。
>    总结：
>    WaitNode就是一个链表结构，用于记录等待当前FutureTask结果的线程。

### (五)、FutureTask的状态

FutureTask一共有7种状态，代码如下：



```dart
    /*
     * Revision notes: This differs from previous versions of this
     * class that relied on AbstractQueuedSynchronizer, mainly to
     * avoid surprising users about retaining interrupt status during
     * cancellation races. Sync control in the current design relies
     * on a "state" field updated via CAS to track completion, along
     * with a simple Treiber stack to hold waiting threads.
     *
     * Style note: As usual, we bypass overhead of using
     * AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
     */

    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

老规矩先来翻译一下注释，我们看到注释有两部分，我们依次翻译如下：

> - 上半部分注释：
>    修订说明：和之前版本的AbstractQueuedSynchronizer不同，主要是为了避免令人惊讶的用户在取消竞争迁建保留中断的状态。在当前设计中，同步的控制是通过CAS中的更新字段——"state"来完成跟踪的。并且通过一个Treiber堆栈来保存这些等待的线程。风格笔记(笔者注：这个真心不知道怎么翻译，谁知道请在下面留言)：和往常一样，我们直接使用不安全的内在函数， 并且忽略使用AtomicXFieldUpdaters的开销。

简单的说就是，FutureTask中使用state表示任务状态，state变更由CAS操作保证原子性。

> - 下半部分注释：
>    这个任务的运行状态，最初是NEW状态，在调用set()方法或者setException()方法或者cancel()方法后，运行的状态就变为终端的状态。在运行期间，如果计算出结果后，状态变更为COMPLETING，如果通过调用cancel(true)安全的中断运行，则状态变更为INTERRUPTING。由于值是唯一且不能被进一步修改，所以从中间状态到最终状态的转化是有序的。
> - 可能的状态变更流程
> - NEW -> COMPLETING -> NORMAL
> - NEW -> COMPLETING -> EXCEPTIONAL
> - NEW -> CANCELLED
> - NEW -> INTERRUPTING -> INTERRUPTED

上面翻译我感觉不是很好，用白话解释一下：

> FutureTask对象初始化时，在构造器把state设置为NEW，之后状态变更依据具体执行情况来定。
>
> - 任务执行正常，并且还没结束，state为COMPLETING，代表任务正在执行即将完成，接下来很快会被设置为NORMAL或者EXCEPTIONAL，这取决于调用Runnable中的call()方法是否抛出异常，没有异常则是NORMAL，抛出异常是EXCEPTIONAL。
> - 任务提交后、任务结束前取消任务，都有可能变为CANCELLED或者INTERRUPTED。在调用cancel(boolean) 是，如果传入false表示不中断线程，state会变成CANCELLED，如果传入true，则state先变为
>    INTERRUPTING，中断完成后，变为INTERRUPTED。

总结一下就是：FutureTask的状态变化过程为，以下4种情况：

> - 任务正常执行并返回： NEW -> COMPLETING -> NORMAL
> - 任务执行中出现异常：NEW -> COMPLETING -> EXCEPTIONAL
> - 任务执行过程中被取消，并且不中断线程：NEW -> CANCELLED
> - 任务执行过程中被取消，并且中断线程：NEW -> INTERRUPTING -> INTERRUPTED

那我们就简单的解释下几种状态

> - NEW：任务初始化状态
> - COMPLETING：任务正在完成状态(任务已经执行完成，但是结果还没有赋值给outcome)
> - NORMAL：任务完成(结果已经赋值给outcome)
> - EXCEPTIONAL：任务执行异常
> - CANCELLED：任务被取消
> - INTERRUPTING：任务被中断中
> - INTERRUPTED：任务被中断

### (六)、FutureTask的成员变量



```php
    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
```

> - callable：任务具体执行体，具体要做的事
> - outcome：任务的执行结果，get()方法的返回值
> - runner：任务的执行线程
> - waiters：获取任务结果的等待线程(是一个链式列表)

### (七)、FutureTask的构造函数

FutureTask有两个构造函数
 分别是FutureTask(Callable<V> callable)和FutureTask(Runnable runnable, V result)，那我们来依次分析

#### 1、构造函数 FutureTask(Callable<V> callable)



```java
    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Callable}.
     *
     * @param  callable the callable task
     * @throws NullPointerException if the callable is null
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

老规矩先翻译一下注释：

> 创建一个FutureTask，并在将来执行的时候，运行传入的Callable。

看下代码，我们知道：

> - 1、通过代码我们知道如果传入的Callable为空直接抛出异常，说明构造时传入的Callable不能为空。
> - 2、设置当前状态为NEW。

所以总结一句话就是，通过传入Callable来构造一个FutureTask。

#### 2、构造函数 FutureTask(Runnable runnable, V result)



```kotlin
    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```

老规矩先翻译一下注释：

> 创建一个FutureTask，并在将来执行的时候，运行传入的Runnable，并且将成功完成后的结果返给传入的result。

看下代码，我们知道：

> - 1、先通过调用Executors的callable(Runnable, T)方法返回的Callable
> - 2、将上面返回的Callable指向本地变量callable
> - 3、设置当前状态为NEW。

所以总结一句话就是，通过传入Runnable来构造一个任务

这里顺带说下Executors.callable(runnable, result)方法的内部实现

**2.1、Executors.callable(Runnable, T)**



```dart
    /**
     * Returns a {@link Callable} object that, when
     * called, runs the given task and returns the given result.  This
     * can be useful when applying methods requiring a
     * {@code Callable} to an otherwise resultless action.
     * @param task the task to run
     * @param result the result to return
     * @param <T> the type of the result
     * @return a callable object
     * @throws NullPointerException if task null
     */
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
```

方法内部很简单，就是new了一个RunnableAdapter并返回，那我们来看下RunnableAdapterr适配器



```java
    /**
     * A callable that runs given task and returns given result.
     */
    private static final class RunnableAdapter<T> implements Callable<T> {
        private final Runnable task;
        private final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

通过上面代码我们知道：

> - RunnableAdapter是FutureTask的一个静态内部类并且实现了Callable，也就是说RunnableAdapter是Callable子类。
> - call方法实现代码是，执行Runnable的run方法，并返回构造函数传入的result参数。

这里实际上是将一个Runnable对象伪装成一个Callable对象，是适配器对象。

**3、构造函数总结**

通过分析上面两个构造函数，我们知道无论采用第一个构造函数，还是第二个构造函数，其结果都是给本地变量callable初始化赋值，所以说FutureTask最终都是执行Callable类型的任务。然后设置状态为NEW。

### (八)、FutureTask的几个核心方法

FutureTask有几个核心方法：

> - public void run()：表示任务的执行
> - public V get()和public V get(long timeout, TimeUnit unit)：表示获取任务的结果
> - public boolean cancel(boolean mayInterruptIfRunning)：表示取消任务

那我们就依次来看下

#### 1、run()方法



```csharp
    /**
     * 判断任务的状态是否是初始化的状态
     * 判断执行任务的线程对象runner是否为null，为空就将当前执行线程赋值给runner属性
     *  不为空说明应有线程准备执行这个任务了
     */
    public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
           // 任务状态时NEW，并且callable不为空，则执行任务
           // 如果认为被cancel了，callable会被置空
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                
                V result;  // 结果的变量
                
                boolean ran;  // 执行完毕的变量
                try {
                    // 执行任务并返回结果
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    // 执行异常
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    //任务执行完毕就设置结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            // 将执行任务的执行线程清空 
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                //判断线程的状态
                handlePossibleCancellationInterrupt(s);
        }
    }
```

通过上面代码和注释我们知道run方法内部的流程如下：

> -  **第一步**：检查当前任务是否是NEW以及runner是否为空，这一步是防止任务被取消
> -  **第二步**：double-check任务状态和state
> -  **第三步**：执行业务逻辑，也就是c.call()方法被执行
> -  **第四步**：如果业务逻辑异常，则调用setException方法将异常对象赋值给outcome，并更新state的值
> -  **第五步**：如果业务正常，则调用set方法将执行结果赋给outcome，并更新state值。

**1.1、Unsafe类**

Java不能够直接访问操作系统底层，而是通过本地方法来访问。Unsafe提供了硬件级别的原子访问，主要提供以下功能：

> - 分配释放内存
> - 定位某个字段的内存位置
> - 挂起一个线程和恢复，更多的是通过LockSupport来访问。park和unpark
> - CAS操作，比较一个对象的某个位置的内存值是否与期望值一致。

主要方法是compareAndSwap()

**1.1.1UNSAFE.compareAndSwapObject(this,runnerOffset,null, Thread.currentThread())方法**

UNSAFE.compareAndSwapObject(this,RUNNER,null, Thread.currentThread())这行代码什么意思？

> compareAndSwapObject可以通过反射，根据偏移量去修改对象，第一个参数表示要修改的对象，第二个表示偏移量，第三个参数用于和偏移量对应的值进行比较，第四个参数表示如何偏移量对应的值和第三个参数一样时要把偏移量设置成的值。

翻译成白话文就是：

> 如果this对象的RUNNER偏移地址的值是null，那就把它设置为Thread.currentThread()。

上面提到了一个概念是**RUNNER**，那这个**RUNNER** 是什么东东？
 我们在源码中找到



```kotlin
    // Unsafe mechanics
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    private static final long STATE;
    private static final long RUNNER;
    private static final long WAITERS;
    static {
        try {
            STATE = U.objectFieldOffset
                (FutureTask.class.getDeclaredField("state"));
            RUNNER = U.objectFieldOffset
                (FutureTask.class.getDeclaredField("runner"));
            WAITERS = U.objectFieldOffset
                (FutureTask.class.getDeclaredField("waiters"));
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }

        // Reduce the risk of rare disastrous classloading in first call to
        // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
        Class<?> ensureLoaded = LockSupport.class;
    }
```

它对应的就是runner的成员变量，也就是说如果状态不是NEW或者runner不是null，run方法直接返回。

> 所以我们知道
>
> - private static final long STATE：表示state这个成员变量
> - private static final long RUNNER：表示的是runner这个成员变量
> - rivate static final long WAITERS：表示的是waiters这个成员变量

在这个run方法里面分别调用了setException(Throwable )和set(V)方法，那我们就来详细看下

**1.2、setException(Throwable t)方法**



```dart
    /**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
         // state状态 NEW-> COMPLETING
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = t;
             // // COMPLETING -> EXCEPTIONAL 到达稳定状态
            U.putOrderedInt(this, STATE, EXCEPTIONAL); // final state
            // 一些 结束工作
            finishCompletion();
        }
    }
```

简单翻译一下方法的注释:

> - 除非这个Future已经设置过了，或者被取消了，否则这个产生的异常将会汇报到ExecutionException里面
> - 如果在run()方法里面产生了异常，则会调用这个方法

所以总结一下就是：当任务执行过程中出现异常时候，对异常的处理方式

> PS：这个方法是protected，所以可以重写

**1.3、set(V v)方法**

这个方法主要是：执行结果的赋值操作



```dart
    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        // state 状态  NEW->COMPLETING
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = v;
            // COMPLETING -> NORMAL 到达稳定状态
            U.putOrderedInt(this, STATE, NORMAL); // final state
            // 一些结束工作
            finishCompletion();
        }
    }
```

通过上面我们知道这个方法内部的流程如下：

> - 首先 将任务的状态改变
> - 其次 将结果赋值
> - 再次 改变任务状态
> - 最后 处理等待线程队列(将线程阻塞状态改为唤醒，这样等待线程就拿到结果了)

PS：这里使用的是 UNSAFE的putOrderedInt方法，其实就是原子量的LazySet内部使用的方法，为什么要用这个方法？首先LazySet相对于Volatile-Write来说更加廉价，因为它没有昂贵的Store/Load屏障，其次后续线程不会及时的看到state从COMPLETING变为NORMAL，但这没有什么关系，而且NORMAL是state最终的状态，不会再变化了。

在这个方法里面调用了finishCompletion()方法，那我们就来看下这个方法

**1.4、finishCompletion()方法**

这个方法是：

> 在任务执行完成(包括取消、正常结束、发生异常)，将等待线程队列唤醒，同时让任务执行体清空。

代码如下：



```csharp
    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        // 遍历等待节点
        for (WaitNode q; (q = waiters) != null;) {
            if (U.compareAndSwapObject(this, WAITERS, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        // 唤醒等待线程
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        // 这里可以自定义实现任务完成后要做的事情(在子类重写done()方法)
        done();
        // 清空callable
        callable = null;        // to reduce footprint
    }
```

> 由代码和注释可以看出来，这里就是遍历WaitNode链表，对每一个WaitNode对应的线程依次进行LockSupport.unpark(t)，使其结束阻塞。WaitNode通知完毕后，调用done方法。目前该方法是空实现，所以如果你想在任务完成后执行一些业务逻辑可以重写这个方法。所以这个方法主要是在于唤醒等待线程。由前面知道，当任务正常结束或者异常结束时，都会调用finishCompletion()去唤醒等待线程。这时候等待线程就可以醒来，可以获取结果了。

·

**1.4.1、LockSupport简介**

这里首先说下LockSupport，很多新手对这个东西，不是很熟悉，我先简单说下，这里就不详细说明了。

> LockSupport是构建concurrent包的基础之一

\####### ① 操作对象
 LockSupport调用Unsafe的natvie代码：



```java
public native void unpark(Thread jthread); 
public native void park(boolean isAbsolute, long time); 
```

这两个函数声明清楚地说明了操作对象：park函数是将当前Thread阻塞，而unPark函数则是将另一个Thread唤醒。

与Object类的wait/notify 机制相比，park/unpark有两个优点：

> - 1、以thread为操作对象更符合阻塞线程的直观定义
> - 2、操作更精准，可以准确地唤醒某一个线程(notify随机唤醒一个线程，notifyAll唤醒所有等待的线程)，增加了灵活性

\####### ② 关于许可
 在上面的文件，使用了阻塞和唤醒，是为了和wait/notify做对比。其实park/unpark的设计原理核心是"许可"。park是等待一个许可。unpark是为某线程提供一个"许可"。如果说某线程A调用park，那么除非另外一个线程unpark(A)给A一个许可，否则线程A将阻塞在park操作上。

**1.5、handlePossibleCancellationInterrupt(int) 方法**



```csharp
    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        // 如果当前正在中断过程中，自等待，等中断完成
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }
```

先来看下注释：

> 执行计算而不设置其结果，然后重新设置future为初始化状态，如果执行遇到异常或者任务被取消，则不再执行此操作。这样设计的目的是：执行多次任务。

看代码我们知道他主要就是: 如果其他线程正在终止该任务，那么运行该任务的线程就暂时让出CPU时间一直到state= INTERRUPTED为止。

所以它的作用就是:

> 确保cancel(true) 产生的中断发生在run()或者 runAndReset()方法过程中

#### 2、get()与get(long, TimeUnit)方法

任务是由线程池提供的线程执行，那么这时候主线程则会阻塞，直到任务线程唤醒它们。我们看看get()是怎么做的？

**2.1 get()方法**

代码如下：



```java
    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        // state 小于 COMPLETING 则说明任务仍然在执行，且没有被取消
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```

通过代码我们知道

> - 首先 校验参数
> - 然后 判断是否正常执行且没有被取消，如果没有则调用awaitDone(boolean,long)方法
> - 最后 调用report(int) 方法

这里面涉及两个方法分别是awaitDone(boolean,long)方法和report(int) 方法，那让我们依次来看下。

**2.1.1  awaitDone(boolean,long)方法**

这个方法主要是等待任务执行完毕，如果任务取消或者超时则停止
 代码如下：



```java
    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion or at timeout
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        // The code below is very delicate, to achieve these goals:
        // - call nanoTime exactly once for each call to park
        // - if nanos <= 0L, return promptly without allocation or nanoTime
        // - if nanos == Long.MIN_VALUE, don't underflow
        // - if nanos == Long.MAX_VALUE, and nanoTime is non-monotonic
        //   and we suffer a spurious wakeup, we will do no worse than
        //   to park-spin for a while
        // 起始时间
        long startTime = 0L;    // Special value 0L means not yet parked
        // 当前等待线程的节点
        WaitNode q = null;
         // 是否将节点放在了等待列表中
        boolean queued = false;
        // 通过死循环来实现线程阻塞等待
        for (;;) {
            int s = state;
            if (s > COMPLETING) {
                 // 任务可能已经完成或者被取消了
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING)
                // We may have already promised (via isDone) that we are done
                // so never return empty-handed or throw InterruptedException
                // 任务线程可能被阻塞了，让出cpu
                Thread.yield();
            else if (Thread.interrupted()) {
                // 线程中断则移除等待线程并抛出异常
                removeWaiter(q);
                throw new InterruptedException();
            }
            else if (q == null) {
                // 等待节点为空，则初始化新节点并关联当前线程
                if (timed && nanos <= 0L)
                   // 如果需要等待，并且等待时间小于0表示立即，则直接返回
                    return s;
                // 如果不需要等待，或者需要等待但是等待时间大于0。
                q = new WaitNode();
            }
            else if (!queued)
                // 等待线程入队，因为如果入队成功则queued=true
                queued = U.compareAndSwapObject(this, WAITERS,
                                                q.next = waiters, q);
            else if (timed) {
                //如果有超时设置
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        // 已经超时，则移除等待节点
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    // 任务还在执行，且没有被取消，所以继续等待
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this);
        }
    }
```

两个入参：

> - timed 为true 表示设置超时时间，false表示不设置超时间
> - nanos 表示超时的状态

for死循环里面的逻辑如下：

> -  **第一步** 判断任务是否已经完处于完成或者取消了，如果直接返回转状态值，如果不是，则走第二步
> -  **第二步**，如果状态值是COMPLETING，则说明当前是在set()方法时被阻塞了，所以只需要让出当前线程的CPU资源。
> -  **第三步**，如果线程已经中断了，则移除线程并抛出异常
> -  **第四步**，如果能走到这一步，状态值只剩下NEW了，如果状态值是NEW，并且q==null，则说明这是第一次，所以初始化一个当前线程的等待节点。
> -  **第五步**，此时queued=false，说明如果还没入队，则它是在等待入队
> -  **第六步**，能走到这一步，说明queued=true，这时候判断timed是否为true，如果为true则设置了超时时间，然后看一下startTime是否为0，如果为0，则说明是第一次，因为startTime默认值为0，如果是第一此，则设置startTime=1。保证startTime==0是第一次。如果startTime!=0，则说明不是第一次，如果不是第一次，则需要计算时差elapsed，如果elapsed大于nanos，则说明超时，如果小于则没有超时，还有时差，然等待这个时差阻塞。
> -  **第七步**，如果timed为false，则说明没有超时设置。

所以总结一下：

> waitDone就是将当前线程加入等待队列(waitNode有当前Thread的Thread变量)，然后用LockSupport将自己阻塞，等待超时或者被解除阻塞后，判断是否已经完成(state为>= COMPLETING)，如果未完成(state< COMPLETING)抛出超时异常，如果已完成则稍等或者直接返回结果。

这个方法里面调用了removeWaiter(WaitNode) 这个方法，所以我们来先看这个这个removeWaiter(WaitNode) 里面是怎么实现的

**2.1.2  removeWaiter(WaitNode)**



```dart
    /**
     * Tries to unlink a timed-out or interrupted wait node to avoid
     * accumulating garbage.  Internal nodes are simply unspliced
     * without CAS since it is harmless if they are traversed anyway
     * by releasers.  To avoid effects of unsplicing from already
     * removed nodes, the list is retraversed in case of an apparent
     * race.  This is slow when there are a lot of nodes, but we don't
     * expect lists to be long enough to outweigh higher-overhead
     * schemes.
     */
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            // 将node 的thread 域置空
            node.thread = null;
            /**
             * 下面过程中会将node从等待队列中移除，以thread为null为依据
             * 如果过程中发生了竞争，重试
             */
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!U.compareAndSwapObject(this, WAITERS, q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```

首先来看下类的注释

> 为了防止累积的内存垃圾，所以需要去取消超时或者已经被中断的等待节点。内部节点因为没有CAS所以很简单，所以他们可以被无害的释放。为了避免已删除节点的影响，如果存在竞争的情况下，需要重新排列。所以当节点很多是，速度会很慢，因此我们不建议列表太长而导致效率降低。

这个方法主要就是将线程节点从等待队列中移除

**2.1.2  report(int) 方法**



```java
    /**
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
         // 如果任务正常执行完成，返回任务执行结果
        if (s == NORMAL)
            return (V)x;

        // 如果任务被取消，抛出异常
        if (s >= CANCELLED)
            throw new CancellationException();
        
        // 其他状态 抛出执行异常 ExecutionException 
        throw new ExecutionException((Throwable)x);
    }
```

 ![img](https://upload-images.jianshu.io/upload_images/5713484-9bd43f3d62437fba.png?imageMogr2/auto-orient/strip|imageView2/2/w/495/format/webp) 

report.png

> 如果任务处于NEW、COMPLETING和INTERRUPTING 这三种状态的时候是执行不到report方法的，所以没有对这三种状态尽心转换。

**2.2 get(long, TimeUnit)方法**

最多等待为计算完成所给的时间之后，获取其结果



```java
    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
```

有参get方法源码很简洁，首先校验参数，然后根据state状态判断是否超时，如果超时则异常，不超时则调用report去获取最终结果。
 当 s <= COMPLETING 时，表明任务仍然在执行且没有被取消，如果它为true，那么走到awaitDone方法。关于awaitDone方法上面已经讲解了，这里就不过阐述了。

#### 3、cancel(boolean)方法

只能取消还没有被执行的任务(任务状态为NEW的任务)



```kotlin
    public boolean cancel(boolean mayInterruptIfRunning) {
        // 如果任务状态不是初始化状态，则取消任务
         //如果此时任务已经执行了，并且可能执行完成，但是状态改变还没有来得及修改，也就是在run()方法中的set()方法还没来得及调用
         //   继续判断任务的当前状态时否为NEW，因为此时执行任务线程可能再度获得处理了，任务状态可能已发生改变
        if (!(state == NEW &&
              U.compareAndSwapInt(this, STATE, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
                return false;

        // 如果任务状态依然是NEW，也就是执行线程没有改变任务的状态，
        // 则让执行线程中断(在这个过程中执行线程可能会改变任务的状态)
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    // 将任务状态设置为中断
                    U.putOrderedInt(this, STATE, INTERRUPTED);
                }
            }
        } finally {
            // 处理任务完成的结果
            finishCompletion();
        }
        return true;
    }
```

通过上述代码，我们知道这个取消不一定起作用的。

上面的代码逻辑如下：

> - 第一步：state不等于NEW，则表示任务即将进入最终状态 ，则state == NEW为false，导致if成立直接返回false
> - 第二步：如果mayInterruptIfRunning为true在，表示中断线程，则设置状态为INTERRUPTING，中断之后设置为INTERRUPTED。如果mayInterruptIfRunning为false，表示不中断线程，把state设置为CANCELLED
> - 第三步：state状态为NEW，任务可能已经开始执行，也可能还未开始，所以用Unsafe查看下，如果不是，则直接返回false
> - 第四步：移除等待线程
> - 第五步：唤醒

所以，cancel()方法改变了futureTask的状态为，如果传入的是false，并且业务逻辑已经开始执行，当前任务是不会被终止的，而是会继续执行，知道异常或者执行完毕。如果传入的是true，会调用当前线程的interrupt()方法，把中断标志位设为true。

> 事实上，除非线程自己停止自己的任务，或者退出JVM，是没有其他方法完全终止一个线程任务的。mayInterruptIfRunning=true，通过希望当前线程可以响应中断的方式来结束任务。当任务被取消后，会被封装为CancellationException抛出。

#### 4、runAndReset() 方法

任务可以被多次执行



```dart
    /**
     * Executes the computation without setting its result, and then
     * resets this future to initial state, failing to do so if the
     * computation encounters an exception or is cancelled.  This is
     * designed for use with tasks that intrinsically execute more
     * than once.
     *
     * @return {@code true} if successfully run and reset
     */
    protected boolean runAndReset() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```

先来看下注释：

> 执行计算而不设置其结果，然后重新设置future为初始化状态，如果执行遇到异常或者任务被取消，则不再执行此操作。这样设计的目的是：执行多次任务。

我们可以对比一下runAndReset与run方法，其实两者相差不大，主要就是有两点区别

> - 1 run()方法里面设置了result的值，而runAndReset()则移除了这段代码

下面我们就来看下handlePossibleCancellationInterrupt(int) 这个方法

## 五、总结

> FutureTask大部分就简单分析完了，其他的自己看下就行了。FutureTask中的任务状态由变量state表示，任务状态都是基于state判断。而FutureTask的阻塞则是通过自旋+挂起线程实现的。





# Handler机制之AsyncTask源码解析

[AsyncTask官网](https://link.jianshu.com?t=https://developer.android.com/reference/android/os/AsyncTask.html)
 [AsyncTask源码](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)

## 一、AsyncTask概述

> 我们都知道，Android UI线程是不安全的，如果想要在子线程里面进行UI操作，就需要直接Android的异步消息处理机制，前面我写了很多文章从源码层面分析了Android异步消息Handler的处理机制。感兴趣的可以去了解下。不过为了更方便我们在子线程中更新UI元素，Android1.5版本就引入了一个AsyncTask类，使用它就可以非常灵活方便地从子线程切换到UI线程。

## 二、基本用法

AsyncTask是一个抽象类，我们需要创建子类去继承它，并且重写一些方法。AsyncTask接受三个泛型的参数：

> - Params：指定传给任务执行时的参数的类型
> - Progress：指定后台任务执行时将任务进度返回给UI线程的参数类型
> - Result：指定任务完成后返回的结果类型

除了指定泛型参数，还需要根据重写一些方法，常用的如下：

> - onPreExecute()：这个方法在UI线程调用，用于在任务执行器那做一些初始化操作，如在界面上显示加载进度空间
> - onInBackground：在onPreExecute()结束之后立刻在后台线程调用，用于耗时操作。在这个方法中可调用publishProgress方法返回任务的执行进度。
> - onProgressUpdate：在doInBackground调用publishProgress后被调用，工作在UI线程
> - onPostExecute：后台任务结束后被调用，工作在UI线程。

## 三、AsyncTask类源码解析

[AsyncTask.java源码地址](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)

### (一)、类注释翻译

源码注释如下：



```dart
/**
 * <p>AsyncTask enables proper and easy use of the UI thread. This class allows you
 * to perform background operations and publish results on the UI thread without
 * having to manipulate threads and/or handlers.</p>
 *
 * <p>AsyncTask is designed to be a helper class around {@link Thread} and {@link Handler}
 * and does not constitute a generic threading framework. AsyncTasks should ideally be
 * used for short operations (a few seconds at the most.) If you need to keep threads
 * running for long periods of time, it is highly recommended you use the various APIs
 * provided by the <code>java.util.concurrent</code> package such as {@link Executor},
 * {@link ThreadPoolExecutor} and {@link FutureTask}.</p>
 *
 * <p>An asynchronous task is defined by a computation that runs on a background thread and
 * whose result is published on the UI thread. An asynchronous task is defined by 3 generic
 * types, called <code>Params</code>, <code>Progress</code> and <code>Result</code>,
 * and 4 steps, called <code>onPreExecute</code>, <code>doInBackground</code>,
 * <code>onProgressUpdate</code> and <code>onPostExecute</code>.</p>
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 * <p>For more information about using tasks and threads, read the
 * <a href="{@docRoot}guide/components/processes-and-threads.html">Processes and
 * Threads</a> developer guide.</p>
 * </div>
 *
 * <h2>Usage</h2>
 * <p>AsyncTask must be subclassed to be used. The subclass will override at least
 * one method ({@link #doInBackground}), and most often will override a
 * second one ({@link #onPostExecute}.)</p>
 *
 * <p>Here is an example of subclassing:</p>
 * <pre class="prettyprint">
 * private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
 *     protected Long doInBackground(URL... urls) {
 *         int count = urls.length;
 *         long totalSize = 0;
 *         for (int i = 0; i < count; i++) {
 *             totalSize += Downloader.downloadFile(urls[i]);
 *             publishProgress((int) ((i / (float) count) * 100));
 *             // Escape early if cancel() is called
 *             if (isCancelled()) break;
 *         }
 *         return totalSize;
 *     }
 *
 *     protected void onProgressUpdate(Integer... progress) {
 *         setProgressPercent(progress[0]);
 *     }
 *
 *     protected void onPostExecute(Long result) {
 *         showDialog("Downloaded " + result + " bytes");
 *     }
 * }
 * </pre>
 *
 * <p>Once created, a task is executed very simply:</p>
 * <pre class="prettyprint">
 * new DownloadFilesTask().execute(url1, url2, url3);
 * </pre>
 *
 * <h2>AsyncTask's generic types</h2>
 * <p>The three types used by an asynchronous task are the following:</p>
 * <ol>
 *     <li><code>Params</code>, the type of the parameters sent to the task upon
 *     execution.</li>
 *     <li><code>Progress</code>, the type of the progress units published during
 *     the background computation.</li>
 *     <li><code>Result</code>, the type of the result of the background
 *     computation.</li>
 * </ol>
 * <p>Not all types are always used by an asynchronous task. To mark a type as unused,
 * simply use the type {@link Void}:</p>
 * <pre>
 * private class MyTask extends AsyncTask<Void, Void, Void> { ... }
 * </pre>
 *
 * <h2>The 4 steps</h2>
 * <p>When an asynchronous task is executed, the task goes through 4 steps:</p>
 * <ol>
 *     <li>{@link #onPreExecute()}, invoked on the UI thread before the task
 *     is executed. This step is normally used to setup the task, for instance by
 *     showing a progress bar in the user interface.</li>
 *     <li>{@link #doInBackground}, invoked on the background thread
 *     immediately after {@link #onPreExecute()} finishes executing. This step is used
 *     to perform background computation that can take a long time. The parameters
 *     of the asynchronous task are passed to this step. The result of the computation must
 *     be returned by this step and will be passed back to the last step. This step
 *     can also use {@link #publishProgress} to publish one or more units
 *     of progress. These values are published on the UI thread, in the
 *     {@link #onProgressUpdate} step.</li>
 *     <li>{@link #onProgressUpdate}, invoked on the UI thread after a
 *     call to {@link #publishProgress}. The timing of the execution is
 *     undefined. This method is used to display any form of progress in the user
 *     interface while the background computation is still executing. For instance,
 *     it can be used to animate a progress bar or show logs in a text field.</li>
 *     <li>{@link #onPostExecute}, invoked on the UI thread after the background
 *     computation finishes. The result of the background computation is passed to
 *     this step as a parameter.</li>
 * </ol>
 * 
 * <h2>Cancelling a task</h2>
 * <p>A task can be cancelled at any time by invoking {@link #cancel(boolean)}. Invoking
 * this method will cause subsequent calls to {@link #isCancelled()} to return true.
 * After invoking this method, {@link #onCancelled(Object)}, instead of
 * {@link #onPostExecute(Object)} will be invoked after {@link #doInBackground(Object[])}
 * returns. To ensure that a task is cancelled as quickly as possible, you should always
 * check the return value of {@link #isCancelled()} periodically from
 * {@link #doInBackground(Object[])}, if possible (inside a loop for instance.)</p>
 *
 * <h2>Threading rules</h2>
 * <p>There are a few threading rules that must be followed for this class to
 * work properly:</p>
 * <ul>
 *     <li>The AsyncTask class must be loaded on the UI thread. This is done
 *     automatically as of {@link android.os.Build.VERSION_CODES#JELLY_BEAN}.</li>
 *     <li>The task instance must be created on the UI thread.</li>
 *     <li>{@link #execute} must be invoked on the UI thread.</li>
 *     <li>Do not call {@link #onPreExecute()}, {@link #onPostExecute},
 *     {@link #doInBackground}, {@link #onProgressUpdate} manually.</li>
 *     <li>The task can be executed only once (an exception will be thrown if
 *     a second execution is attempted.)</li>
 * </ul>
 *
 * <h2>Memory observability</h2>
 * <p>AsyncTask guarantees that all callback calls are synchronized in such a way that the following
 * operations are safe without explicit synchronizations.</p>
 * <ul>
 *     <li>Set member fields in the constructor or {@link #onPreExecute}, and refer to them
 *     in {@link #doInBackground}.
 *     <li>Set member fields in {@link #doInBackground}, and refer to them in
 *     {@link #onProgressUpdate} and {@link #onPostExecute}.
 * </ul>
 *
 * <h2>Order of execution</h2>
 * <p>When first introduced, AsyncTasks were executed serially on a single background
 * thread. Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
 * to a pool of threads allowing multiple tasks to operate in parallel. Starting with
 * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are executed on a single
 * thread to avoid common application errors caused by parallel execution.</p>
 * <p>If you truly want parallel execution, you can invoke
 * {@link #executeOnExecutor(java.util.concurrent.Executor, Object[])} with
 * {@link #THREAD_POOL_EXECUTOR}.</p>
 */
```

简单翻译一下：

> - AsyncTask可以轻松的正确使用UI线程，该类允许你执行后台操作并在UI线程更新结果，从而避免了无需使用Handler来操作线程
> - AsyncTask是围绕Thread与Handler而设计的辅助类，所以它并不是一个通用的线程框架。理想情况下，AsyncTask应该用于短操作(最多几秒)。如果你的需求是在长时间保持线程运行，强烈建议您使用由
>    java.util.concurrent提供的各种API包，比如Executor、ThreadPoolExecutor或者FutureTask。
> - 一个异步任务被定义为：在后台线程上运行，并在UI线程更新结果。一个异步任务通常由三个类型：Params、Progress和Result。以及4个步骤：onPreExecute、doInBackground、onProgressUpdate、onPostExecute。
> - 用法：AsyncTask必须由子类实现后才能使用，它的子类至少重写doInBackground()这个方法，并且通常也会重写onPostExecute()这个方法
> - 下面是一个继承AsyncTask类的一个子类的例子
>
> 
>
> ```java
>    private class DownloadFilesTask extends AsyncTask<URL, >Integer, Long>
>    {
>        protected Long doInBackground(URL... urls) {
>            int count = urls.length;
>            long totalSize = 0;
>            for (int i = 0; i < count; i++) {
>                totalSize += Downloader.downloadFile(urls[i]);
>                publishProgress((int) ((i / (float) count) * 100));
>                // Escape early if cancel() is called
>                if (isCancelled()) break;
>            }
>            return totalSize;
>        }
> 
>        protected void onProgressUpdate(Integer... progress) {
>            setProgressPercent(progress[0]);
>        }
> 
>        protected void onPostExecute(Long result) {
>            showDialog("Downloaded " + result + " bytes");
>        }
>    }
> ```
>
> 一旦创建，任务会被很轻松的执行。就像下面这块代码一样
>
> 
>
> ```cpp
> new DownloadFilesTask().execute(url1,url2,url3);
> ```
>
> - AsyncTask泛型，在异步任务中通常有三个泛型 
>   - Params：发送给任务的参数类型，这个类型会被任务执行
>   - Progress：后台线程进度的计算的基本单位。
>   - Result：后台线程执行的结果类型。
> - 如果异步任务不需要上面类型，则可以需要声明类型未使用，通过使用Void来表示类型未使用。就像下面一杨
>
> 
>
> ```java
> private class MyTask extends AsyncTask<Void, Void, Void>{...}
> ```
>
> - 四个步骤 
>   - onPreExecute() 在执行任务之前在UI线程上调用，此步骤通常用于初始化任务，例如在用户界面显示进度条。
>   - doInBackground() 方法在 onPreExecute()执行完成后调用的。doInBackground()这个方法用于执行可能需要很长时间的首台计算。异步任务的参数被传递到这个步骤中。计算结果必须经由这个步骤返回。这个步骤内还可以调用publishProgress()方法发布一个或者多个进度单位。在调用onProgressUpdate()函数后，这个结果将在UI线程上更新。
>   - onProgressUpdate()方法：在调用publishProgress()之后在UI线程上调用。具体的执行时间不确定，该方法用于在后台计算的异步任务，把具体的进度显示在用户界面。例如，它可以用于对进度条进行动画处理或者在文本字段中显示日志。
>   - onPostExecute()方法， 后台任务完成后，在UI线程调用onPostExecute()方法，后台运行的结果作为参数传递给这个方法
> - 取消任务
>    在任何时候都可以通过调用cancel(boolean)来取消任务。调用此方法将导致isCancelled()方法的后续调用返回true。调用此方法后，在执行doInBackground(Object [])方法后，将调用onCancelled(object)，而不是onPostExecute(Object)方法。为了尽可能快的取消任务，如果可能的话，你应该在调用doInBackground(Object[])之前检查isCancelled()的返回值。
> - 线程规则，这个类必须遵循有关线程的一些规则才能正常使用，规则如下： 
>   - 必须在UI线程上加载AsyncTask类，android4.1上自动完成
>   - 任务实例必须在UI线程上实例化
>   - execute()方法必须在主线程上调用
>   - 不要手动去调用onPreExecute()、onPostExecute()、doInBackground()、onProgressUpdate()方法。
>   - 这个任务只能执行一次(如果尝试第二次执行，将会抛出异常)。
>      该任务只能执行一次（如果尝试第二次执行，将抛出异常）。
> - 内存的观察AsyncTask。保证所有回调调用都是同步的，使得以下操作在没有显示同步情况下是安全的。 
>   - 在构造函数或者onPreExecute设置成员变量，并且在doInBackground()方法中引用它们。
>   - 在doInBackground()设置成员字段，并在onProgressUpdate()和onPostExecute()方法中引用他们。
> - 执行顺序。第一引入AsyncTask时，AsyncTasks是在单个后台线程串行执行的。在android1.6以后，这被更改为允许多个任务并行操作的线程池。从android 3.0开始，每个任务都是执行在一个独立的线程上，这样可以避免一些并行执行引起的常见的应用程序错误。如果你想要并行执行，可以使用THREAD_POOL_EXECUTOR来调用executeOnExecutor()方法。

至此这个类的注释翻译完毕，好长啊，大家看完翻译，是不是发现了很多之前没有考虑到的问题。

### (二)、AsyncTask的结构

AsyncTask的结构如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-029800b63aa30167.png?imageMogr2/auto-orient/strip|imageView2/2/w/1036/format/webp)

AsyncTask的结构.png

我们看到在AsyncTask有4个自定义类，一个枚举类，一个静态块，然后才是这个类的具体变量和属性，那我们就依次讲解

### (三)、枚举Status

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   256行



```swift
    /**
     * Indicates the current status of the task. Each status will be set only once
     * during the lifetime of a task.
     */
   
    public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         */
        PENDING,
        /**
         * Indicates that the task is running.
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         */
        FINISHED,
    }
```

枚举Status上的注释翻译一下就是：

> Status表示当前任务的状态，每种状态只能在任务的生命周期内设置一次。

所以任务有三种状态

> - PENDING：表示任务尚未执行的状态
> - RUNNING：表示任务正在执行
> - FINISHED：任务已完成

### (四)、私有的静态类InternalHandler

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   656行



```java
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            // 这个handler是关联到主线程的
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

通过上面的代码我们知道：

> - InternalHandler继承自Handler，并且在它的构造函数里面调用了Looper.getMainLooper()，所以我们知道这个Handler是关联主线程的。
> - 重写了handleMessage(Message)方法，其中这个Message的obj这个类型是AsyncTaskResult(AsyncTaskResult我将在下面讲解)，然后根据msg.what的来区别。
> - 我们知道这个Message只有两个标示，一个是**MESSAGE_POST_RESULT**代表消息的结果，一个是**MESSAGE_POST_PROGRESS**代表要执行onProgressUpdate()方法。

通过这段代码我们可以推测AsyncTask内部实现线程切换，即切换到主线程是通过Handler来实现的。

### (五)、私有的静态类AsyncTaskResult

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   682行



```kotlin
    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```

通过类名，我们大概可以推测出这一个负责AsyncTask结果的类

> AsyncTaskResult这个类 有两个成员变量，一个是AsyncTask一个是泛型的数组。
>
> - mTask参数：是为了AsyncTask是方便在handler的handlerMessage回调中方便调用AsyncTask的本身回调函数，比如onPostExecute()函数、onPreogressUpdata()函数，所以在AsyncTaskResult需要持有AsyncTask。
> - mData参数：既然是代表结果，那么肯定要有一个变量持有这个计算结果

### (六)、私有静态抽象类WorkerRunnable

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   677行



```java
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
```

> 这个抽象类很简答，首先是实现了Callable接口，然后里面有个变量
>  mParams，类型是泛型传进来的数组

### (七)、局部变量详解

AsyncTask的局部变量如下：



```java
    private static final String LOG_TAG = "AsyncTask";

    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    // We want at least 2 threads and at most 4 threads in the core pool,
    // preferring to have 1 less than the CPU count to avoid saturating
    // the CPU with background work
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE_SECONDS = 30;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;


    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static InternalHandler sHandler;

    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;

    private volatile Status mStatus = Status.PENDING;
    
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
```

那我们就来一一解答

> - String LOG_TAG = "AsyncTask"：打印专用
> - CPU_COUNT = Runtime.getRuntime().availableProcessors()：获取当前CPU的核心数
> - CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4))：线程池的核心容量，通过代码我们知道是一个大于等于2小于等于4的数
> - MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1：线程池的最大容量是2倍的CPU核心数+1
> - KEEP_ALIVE_SECONDS = 30：过剩的空闲线程的存活时间，一般是30秒
> - sThreadFactory：线程工厂，通过new Thread来获取新线程，里面通过使用AtomicInteger原子整数保证超高并发下可以正常工作。
> - sPoolWorkQueue：静态阻塞式队列，用来存放待执行的任务，初始容量：128个
> - THREAD_POOL_EXECUTOR：线程池
> - SERIAL_EXECUTOR = new SerialExecutor()：静态串行任务执行器，其内部实现了串行控制，循环的取出一个个任务交给上述的并发线程池去执行。
> - MESSAGE_POST_RESULT = 0x1：消息类型，代表发送结果
> - MESSAGE_POST_PROGRESS = 0x2：消息类型，代表进度
> - sDefaultExecutor = SERIAL_EXECUTOR：默认任务执行器，被赋值为串行任务执行器，就是因为它，AsyncTask变成了串行的了。
> - sHandler：静态Handler，用来发送上面的两种通知，采用UI线程的Looper来处理消息，这就是为什么AnsyncTask可以在UI线程更新UI
> - WorkerRunnable<Params, Result> mWorke：是一个实现Callback的抽象类，扩展了Callable多了一个Params参数。
> - mFuture：FutureTask对象
> - mStatus = Status.PENDING：任务的状态默认为挂起，即等待执行，其类型标示为volatile
> - mCancelled = new AtomicBoolean()：原子布尔类型，支持高并发访问，标示任务是否被取消
> - mTaskInvoked = new AtomicBoolean()：原子布尔类型，支持高并发访问，标示任务是否被执行过

### (八)、静态代码块

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   226行



```cpp
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

> 通过上面的**(七)、局部变量详解**，我们知道在静态代码块中创建了一个线程池threadPoolExecutor，并设置了核心线程会超时关闭，最后并把这个线程池指向THREAD_POOL_EXECUTOR。

### (九)、私有的静态类SerialExecutor

代码在[AsyncTask.java](https://link.jianshu.com?t=http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/os/AsyncTask.java)   226行



```java
    private static class SerialExecutor implements Executor {
        // 循环数组实现的双向Queue，大小是2的倍数，默认是16，有队头和队尾巴两个下标
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        // 正在运行runnable
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            // 添加到双向队列中去
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                         //执行run方法
                        r.run();
                    } finally {
                        //无论执行结果如何都会取出下一个任务执行
                        scheduleNext();
                    }
                }
            });
           // 如果没有活动的runnable，则从双端队列里面取出一个runnable放到线程池中运行
           // 第一个请求任务过来的时候mActive是空的
            if (mActive == null) {
                 //取出下一个任务来
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            //从双端队列中取出一个任务
            if ((mActive = mTasks.poll()) != null) {
                //线线程池执行取出来的任务，真正的执行任务
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

> - 首先，注意SerialExecutor的execute是synchronized的，所以无论多个少任务调用execute()都是同步的。
> - 其次，SerialExecutor里面一个ArrayDeque队列，通过代码我们知道，SerialExecutor是通过ArrayDeque来管理Runnable对象的。通过上面我们知道execute()是同步的，所以如果你有10个任务同时调用SerialExecutor的execute()方法，就会把10个Runnable先后放入到mTasks中去，可见mTasks缓存了将要执行的Runnable。
> - 再次1，如果我们第一个执行execute()方法时，会调用ArrayDeque的offer()方法将传入的Runnable对象添加到队列的尾部，然后判断mActive是不是null，因为是第一次调用，此时mActive还没有赋值，所以mActive等于null。所以此时mActive == null成立，所以会调用scheduleNext()方法。
> - 再次2，我们在调用scheduleNext()里面，看到会调用mTasks.poll()，我们知道这是从队列中取出头部元素，然后把这个头部元素赋值给mActive，然后让THREAD_POOL_EXECUTOR这个线程池去执行这个mActive的Runnable对象。
> - 再次3，如果这时候有第二个任务入队，但是此时mActive!=null，不会执行scheduleNext()，所以如果第一个任务比较耗时，后面的任务都会进入队列等待。
> - 再次4，上面知道由于第二个任务入队后，由于mActive!=null，所以不会执行scheduleNext()，那样这样后面的任务岂不是永远得不到处理，当然不是，因为在offer()方法里面传入一个Runnable的匿名类，并且在此使用了finnally代码，意味着无论发生什么情况，这个finnally里面的代码一定会执行，而finnally代码块里面就是调用了scheduleNext()方法，所以说每当一个任务执行完毕后，下一个任务才会执行。
> - 最后，SerialExecutor其实模仿的是单一线程池的效果，如果我们快速地启动了很多任务，同一时刻只会有一个线程正在执行，其余的均处于等待状态。

PS：scheduleNext()方法是synchronized，所以也是同步的

重点补充：

> **在Android 3.0 之前是并没有SerialExecutor这个类的，那个时候是直接在AsyncTask中构建一个sExecutor常量，并对线程池总大小，同一时刻能够运行的线程数做了规定，代码如下：**



```java
private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  
        MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory); 
```

## 四、AsyncTask类的构造函数

代码如下：



```csharp
    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {

                // 设置方法已经被调用
                mTaskInvoked.set(true);
                 // 设定结果变量
                Result result = null;
                try {
                    //设置线程优先级
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //执行任务
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    // 产生异常则设置失败
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    // 无论执行成功还是出现异常，最后都会调用PostResult
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    // 就算没有调用让然去设置结果
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

通过注释我们知道，这个方法创建一个异步任务，构造函数必须在UI线程调用

这里面设计了两个概念Callable和FutureTask，如果大家对这两个类有疑问，可以看我上一篇文章[Android Handler机制12之Callable、Future和FutureTask]()

> 构造函数也比较简单，主要就是给mWorker和mFuture初始化，其中WorkerRunnable实现了Callable接口，

在构造函数里面调用了postResult(Result)和postResultIfNotInvoked(Result)，那我们就来分别看下

### 1、postResult(Result)方法



```kotlin
     // doInBackground执行完毕，发送消息
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        // 获取一个Message对象
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        // 发送给线程
        message.sendToTarget();
        return result;
    }
```

通过代码我们知道

> - 生成一个Message
> - 把这个Message发送出

这里面调用了 getHandler()，那我们来看下这个方法是怎么写的

### 2、getHandler()方法



```java
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }
```

我们看到返回的是InternalHandler对象，上面说过了InternalHandler其实是关联主线程的，所以上面方法 message.sendToTarget(); 其实是把消息发送给主线程。

> 大家注意一下 这里的Message的what值为MESSAGE_POST_RESULT，我们来看下InternalHandler遇到InternalHandler这种消息是怎么处理的



```java
    private static class InternalHandler extends Handler {
        ... 
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
               ...
            }
        }
    }
```

我们看到MESSAGE_POST_RESULT对应的是指是执行AsyncTask的finish(Result)方法，所以我们可以这样说，无论AsyncTask是成功了还是失败了，最后都会执行finish(Result)方法。那我们来看下finish(Result)方法里面都干了什么？

**2.1、finish(Result result) 方法**



```cpp
    private void finish(Result result) {
        if (isCancelled()) {
            // 如果消息取消了，执行onCancelled方法
            onCancelled(result);
        } else {
            // 如果消息没有取消，则执行onPostExecute方法
            onPostExecute(result);
        }
        // 设置状态值
        mStatus = Status.FINISHED;
    }
```

注释写的很清楚了，我这里就不说明了，通过上面的代码和finish方法的分析，我们知道无论成功还是失败，最后一定会调用finish(Result)方法，所以最后状态的值为FINISHED。

### 3、postResultIfNotInvoked(Result)方法



```java
    private void postResultIfNotInvoked(Result result) {
        // 获取mTaskInvoked的值
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```

通过上面代码我们知道，如果mTaskInvoked不为true，则执行postResult，但是在mWorker初始化的时候为true，除非在没有执行call方法时候，如果没有执行call，说明这个异步线程还没有开始执行，这个时候mTaskInvoked为false。而这时候调用postResultIfNotInvoked则还是会执行postResult(Result)，这样保证了AsyncTask一定有返回值。

## 五、AsyncTask类核心方法解析

### (一)、void onPreExecute()



```java
    /**
     * Runs on the UI thread before {@link #doInBackground}.
     *
     * @see #onPostExecute
     * @see #doInBackground
     */
     // 在调用doInBackground()方法之前，跑在主线程上
    @MainThread
    protected void onPreExecute() {
    }
```

其实注释很清楚了，在task任务开始执行的时候在主线程调用，在doInBackground(Params… params) 方法之前调用。

### (二)、AsyncTask<Params, Progress, Result> onPreExecute() 方法



```dart
    /**
     * Executes the task with the specified parameters. The task returns
     * itself (this) so that the caller can keep a reference to it.
     * 
     * <p>Note: this function schedules the task on a queue for a single background
     * thread or pool of threads depending on the platform version.  When first
     * introduced, AsyncTasks were executed serially on a single background thread.
     * Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
     * to a pool of threads allowing multiple tasks to operate in parallel. Starting
     * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are back to being
     * executed on a single thread to avoid common application errors caused
     * by parallel execution.  If you truly want parallel execution, you can use
     * the {@link #executeOnExecutor} version of this method
     * with {@link #THREAD_POOL_EXECUTOR}; however, see commentary there for warnings
     * on its use.
     *
     * <p>This method must be invoked on the UI thread.
     *
     * @param params The parameters of the task.
     *
     * @return This instance of AsyncTask.
     *
     * @throws IllegalStateException If {@link #getStatus()} returns either
     *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
     *
     * @see #executeOnExecutor(java.util.concurrent.Executor, Object[])
     * @see #execute(Runnable)
     */
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

首先来翻译一下注释

> - 使用指定的参数来执行任务，这个方法的返回值是this，也就是它自己，因为这样设计的目的是可以保持对它的引用。
> - 注意：它的调度模式是不同的，一种是单个后台线程，一种是通过线程池来实现，具体那种模式是根据android版本的不同而不同，当最开始引入AsyncTask的时候，AsyncTask是单个后台线程上串行执行，从Android DONUT 开始，模式变更为通过线程池多任务并行执行。在Android HONEYCOMB开始，又变回了在单个线程上执行，这样可以避免并行执行的错误。如果你还是想并行执行，你可以使用executeOnExecutor()方法并且第一个参数是THREAD_POOL_EXECUTOR就可以了，不过，请注意有关使用警告。
> - 必须在UI主线程上调用此方法。

通过代码我们看到，它的内部其实是调用executeOnExecutor(Executor exec, Params... params)方法，只不过第一个参数传入的是sDefaultExecutor，而sDefaultExecutor是SerialExecutor的对象。上面我们提到了SerialExecutor里面利用ArrayDeque来实现串行的，所以我们可以推测出如果在executeOnExecutor(Executor exec, Params... params)方法里面如果第一个参数是自定义的Executor，AsyncTask就可以实现并发执行。

### (三)、executeOnExecutor(Executor exec, Params... params) 方法



```dart
    /**
     * Executes the task with the specified parameters. The task returns
     * itself (this) so that the caller can keep a reference to it.
     * 
     * <p>This method is typically used with {@link #THREAD_POOL_EXECUTOR} to
     * allow multiple tasks to run in parallel on a pool of threads managed by
     * AsyncTask, however you can also use your own {@link Executor} for custom
     * behavior.
     * 
     * <p><em>Warning:</em> Allowing multiple tasks to run in parallel from
     * a thread pool is generally <em>not</em> what one wants, because the order
     * of their operation is not defined.  For example, if these tasks are used
     * to modify any state in common (such as writing a file due to a button click),
     * there are no guarantees on the order of the modifications.
     * Without careful work it is possible in rare cases for the newer version
     * of the data to be over-written by an older one, leading to obscure data
     * loss and stability issues.  Such changes are best
     * executed in serial; to guarantee such work is serialized regardless of
     * platform version you can use this function with {@link #SERIAL_EXECUTOR}.
     *
     * <p>This method must be invoked on the UI thread.
     *
     * @param exec The executor to use.  {@link #THREAD_POOL_EXECUTOR} is available as a
     *              convenient process-wide thread pool for tasks that are loosely coupled.
     * @param params The parameters of the task.
     *
     * @return This instance of AsyncTask.
     *
     * @throws IllegalStateException If {@link #getStatus()} returns either
     *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
     *
     * @see #execute(Object[])
     */
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }
        //设置状态
        mStatus = Status.RUNNING;
        从这里我们看出onPreExecute是先执行，并且在UI线程
        onPreExecute();
        // 设置参数
        mWorker.mParams = params;
        // 开启了后台线程去计算，这是真正调用doInBackground的地方
        exec.execute(mFuture);
        // 接着会有onProgressUpdate会被调用，最后是onPostExecute
        return this;
    }
```

老规矩 先翻译一下注释：

> - 使用指定的参数来执行任务，这个方法的返回值是this，也就是它自己，因为这样设计的目的是可以保持对它的引用。
> - 这个方法通常与THREAD_POOL_EXECUTOR一起使用，这样可以让多个人物在AsyncTask管理的线程池上并行运行，但你也可以使用自定义的Executor。
> - 警告：由于大多数情况并没有定义任务的操作顺序，所以在线程池中多任务并行并不常见。例如：如果修改共同状态的任务(就像点击按钮就可以编写文件)，对修改的顺讯没有保证。在很少的情况下，如果没有仔细工作，较新版本的数据可能会被较旧的数据覆盖，从而导致数据丢失和稳定性问题。而这些变更最好是连续执行，因为这样可以保证工作的有序化，无论平台版本如何，你可以使用SERIAL_EXECUTOR。
> - 必须在UI主线程上调用此方法。
> -  **参数exec**：为了实现轻松解耦，我们可以使用THREAD_POOL_EXECUTOR这个线程池可以作为合适的进程范围的线程池
> -  **参数params**：任务的参数

那我们来看下一下代码，代码里面的逻辑如下：

> - 该方法首先是判断mStatus状态，如果是正在运行(RUNNING)或者已经结束(FINISHED)，就会抛出异常。
> - 接着设置状态为RUNNING，即运行，执行onPreExecute()方法，并把参数的值赋给mWorker.mParams
> - 于是Executor去执行execute的方法，学过Java多线程的都知道，这里方法是开启一个线程去执行mFuture的run()方法(由于mFuture用Callable构造，所以其实是执行的Callable的call()方法，而mWorker是Callable的是实现类，所以最终执行的是mWorker的call()方法)

PS：mFuture和mWorker都是在AsyncTask的构造方法中初始化过的。

### (四)、 publishProgress(Progress... values) 方法

主要是设置后台进度，onProgressUpdate会被调用



```dart
    /**
     * This method can be invoked from {@link #doInBackground} to
     * publish updates on the UI thread while the background computation is
     * still running. Each call to this method will trigger the execution of
     * {@link #onProgressUpdate} on the UI thread.
     *
     * {@link #onProgressUpdate} will not be called if the task has been
     * canceled.
     *
     * @param values The progress values to update the UI with.
     *
     * @see #onProgressUpdate
     * @see #doInBackground
     */
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

这个方法内部实现很简单

> - 首先判断 任务是否已经被取消，如果已经被取消了，则什么也不做
> - 如果任务没有被取消，则通过InternalHandler发送一个what为MESSAGE_POST_PROGRESS的Message

这样就进入了InternalHandler的handleMessage(Message)里面了，而我们知道InternalHandler的Looper是Looper.getMainLooper()，所以处理Message是在主线程中，我们来看下代码



```java
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

通过代码，我们看到如果what为MESSAGE_POST_PROGRESS，则会在主线程中调用onProgressUpdate(result.mData)，这也就是为什么我们平时在异步线程调用publishProgress(Progress...)方法后，可以在主线程中的onProgressUpdate(rogress... values)接受数据了。

## 六、AsyncTask核心流程

其实在上面讲解过程中，我基本上已经把整体流程讲解过了，我这里补上一张图，比较全面的阐述了AsyncTask的执行流程如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-43d82410d5349943.png?imageMogr2/auto-orient/strip|imageView2/2/w/1008/format/webp)

asynctask执行流程.png

对应的时序图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-a74bbe56e3165851.png?imageMogr2/auto-orient/strip|imageView2/2/w/876/format/webp)

时序图.png

大家如果手机上看不清，我建议down下来在电脑上看。

如果结合AsyncTask的状态值，流程图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-32c8cb777b071897.png?imageMogr2/auto-orient/strip|imageView2/2/w/993/format/webp)

流程.png

如果把AsyncTask和Handler分开则流程图如下：

![img](https:////upload-images.jianshu.io/upload_images/5713484-97aeecfec01adcc4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1123/format/webp)

AsyncTask和Handler分开.png

最后如果把AsyncTask里面所有类的涉及关系整理如下图：

![img](https:////upload-images.jianshu.io/upload_images/5713484-7e7b76b4de893a97.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

20140513095959437.jpeg

## 七、AsyncTask与Handler

> - AsyncTask： 
>   - 优点：AsyncTask是一个轻量级的异步任务处理类，轻量级体现在，使用方便，代码简洁，而且整个异步任务的过程可以通过cancel()进行控制
>   - 缺点：不适用处理长时间的异步任务，一般这个异步任务的过程最好控制在几秒以内，如果是长时间的异步任务就需要考虑多线程的控制问题；当处理多个异步任务时，UI更新变得困难。
> - Handler： 
>   - 优点：代码结构清晰，容易处理多个异步任务
>   - 缺点：当有多个异步任务时，由于要配合Thread或Runnable，代码可能会稍显冗余。

总之，AsyncTask不失为一个非常好用的异步任务处理类。不过我从事Android开发5年多了，很少会用到AsyncTask，一般异步任务都是Handler。



