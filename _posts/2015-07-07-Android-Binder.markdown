---
layout: post
title:  "Android 之 Binder 学习记录"
date:   2015-03-01 14:34:57
tags: Android
---


# Android 之 Binder 学习记录

标签（空格分隔）： Android

---


- [x] **简介**
- [ ] **进程间通信是如何实现的**
- [ ] **Binder 和 Socket 的比较**
    - [ ] Binder 的特点
- [ ] **MediaService的分析**
    - [x] MediaService 的诞生
    - [x] ProcessState 分析
    - [x] IPCThreadState分析
    - [ ] 和Binder设备交换数据 
    - [ ] 获取服务管理器 
    - [ ] MediaService服务的创建和注册
- [ ] **Binder 设备驱动**
- [ ] **总结**
     

---
##简介
  媒体播放服务是一个应用程序，通过一种进程间通信机制为其他媒体播放客户端提供功能，这种进程间通信机制可以使媒体播放客户端调用媒体播放服务中的函数、过程等，就像是在调用API，执行动作的过程是在另一个进程中，这种进程间通信机制对用户来说是透明的，所以用户感觉不到这个机制的存在，这个机制就是Binder。
  
##MediaService的分析

###  MediaService的诞生 ###

MediaService 是个独立的可执行程序叫： mediaserver，在 init.rc 配置文件中有一项，

```
service media /system/bin/mediaserver
    class main
    user media
    group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm
    ioprio rt 4
```

init 是开机时启动的第一个程序，init 读入配置文件 init.rc 然后执行每一项，由此启动媒体服务
因此 meidiaserver 的父进程id一定是1

> 小提示：知道了程序名，我们可以通过搜索Android.mk文件，查找是哪个模块生成的这个程序
> 在 `frameworks\av\media\mediaserver\Android.mk` 中有一句`LOCAL_MODULE:=  mediaserver`，应该是它没错了

通过`Android.mk`文件我们可以分析出大量信息，比如使用到的动态库、静态库、源文件等.
mediaserver 的源文件只有一个叫 main_mediaserver.cpp， 在 source insight里打开可以快速跳转，比较方便分析

main_mediaserver.cpp 的内容非常简单，首先判断一下是否开启了测试模式，没有的话大段代码就跳过了
主要的只有几句如下:
```c++
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();
        registerExtensions();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
```
这里除了有MediaPlayerService服务，还有另外几个服务先不管，只看MediaPlayerService，继续慢慢分析

首先调用 ProcessState::self() 初始化 proc 变量
```c++
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState;
    return gProcess;
}
```
这里有两个全局变量定义在 Static.cpp 中
gProcessMutex是一个Mutex类型的变量， 在进入self方法时自动上锁，从self方法离开后自动解锁
gProcess 是一个指向ProcessState实例的全局唯一变量，这里用一个单例模式保证只有一个实例。
跟进 ProcessState 的构造器
```c++
ProcessState::ProcessState()
    : mDriverFD(open_driver())
    , mVMStart(MAP_FAILED)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // XXX Ideally, there should be a specific define for whether we
        // have mmap (or whether we could possibly have the kernel module
        // availabla).
#if !defined(HAVE_WIN32_IPC)
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
        }
#else
        mDriverFD = -1;
#endif
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}

```
构造函数主要是对一些成员变量初始化
mDriverFD ：通过调用open_driver 返回一个打开`/dev/binder`设备的文件描述符
mVMStart : 在mDriverFD上映射一段虚拟内存地址空间，用作进程间通信存放数据，这段内存的大小是`#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))`定义的
mThreadPoolStarted: ProcessState 实现了一个线程是，这个变量标识线程池是否已经打开
mThreadPoolSeq: 线程池中线程序数
构造函数执行完成后，打开了binder设备用作进程间通信，并且映射了一段内存空间用来保存进程间通信的数据，初始化线程池标识变量，初始化线程池线程序数从1开始

接着跳过中间的服务管理和服务初始化，先看看和 ProcessState 相关的另外两句话
```c++
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
```

```c++
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
```

打开线程池，主要是设置 mThreadPoolStarted 标识变量为 true, 然后调用了 `spawnPooledThread(true)`
```c++
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```
调用 `makeBinderThreadName`生成一个线程名，线程名是根据当前线程池中的线程序号`mThreadPoolSeq`加1拼接上`Binder_%X`

PoolThread 类继承自 Thread 类，Thrad类位置在`system\core\libutils\Threads.cpp`文件中

```c++
class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};
```
这个类除了构造器只有一个protected  的方法 virtual bool threadLoop , 猜测肯定是父类定义了这个虚方法，然后子类实现，在父类的某个地方调用，查看Thread 的实现，果然如此，
在 Thraed 的 run 方法中创建了一个线程，在线程中跑 threadLoop 这个方法，在创建线程时根据是否会调用Java层的代码分两种创建方式,为了调用Java层代码需要做额外的工作
```C++
  bool res;
    if (mCanCallJava) {
        res = createThreadEtc(_threadLoop,
                this, name, priority, stack, &mThread);
    } else {
        res = androidCreateRawThreadEtc(_threadLoop,
                this, name, priority, stack, &mThread);
    }
```
无论哪种方式，都是在跑_threadLoop, _threadLoop调用子类的threadLoop，回到子类PoolThread中
```C++
virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
```
可以看到只是调用了`IPCThreadState::self()->joinThreadPool(mIsMain);`
这和Main_mediaserver中的两句中的第二句是相同的
```c++
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
```
但是它们是在不同的线程中调用的，根据名字猜测是把调用者线程加入线程池中。

`ProcessState::self()->startThreadPool();` 这一句创建了一个新的线程，并加入线程池
`IPCThreadState::self()->joinThreadPool();`把当前线程也就是主线程加入线程池

joinThreadPool 的原型如下，可以看出没加参数调用 isMain 默认是 true，可是`spawnPooledThread(true);`传的也是 true 进来，
`void                joinThreadPool(bool isMain = true);`
在 IPCThreadState 的 executeCommand 中，其中有个分支也是创建线程的，这里传入的 isMain 就是 false了, 不同的服务，根据需要可以动态增加线程
```
case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
```

继续看看joinThradPool的实现,首先调用 IPCThreadState的self方法获取实例对象，
````
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;
    }
    
    if (gShutdown) return NULL;
    
    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        if (pthread_key_create(&gTLS, threadDestructor) != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```
这个self使用TLS技术使每个线程调用self时都有一个自己的IPCThreadState实例

> IPCThreadState 和 ProcessState 什么关系？
> IPCThreadState 是每个线程有一个实例，主要用作获取和执行命令
> ProcessState 在整个进程中只有一个实例，持有一个打开Binder设备的文件描述符，这个描述符在所有线程中都可访问，所以在IPCThreadState中可使用这个描述符和Binder设备交互，具体在talkWithDriver方法中，详情见下文。

看看IPCThreadState的构造
```C++
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(androidGetTid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```
设置 mIn 和 mOut 的容量为256，mIn 和 mOut 是Parcel 类型实例，用来封装进程间通信传递的数据，一个输入一个输出

来到关键的joinThreadPool
```C++
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    
    // This thread may have been spawned by a thread that was in the background
    // scheduling group, so first we will make sure it is in the foreground
    // one to avoid performing an initial transaction in the background.
    set_sched_policy(mMyThreadId, SP_FOREGROUND);
        
    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }
        
        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%p\n",
        (void*)pthread_self(), getpid(), (void*)result);
    
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}


```

主体是一个循环，循环开始前写入`mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);`
循环结束后写入```mOut.writeInt32(BC_EXIT_LOOPER);``` 看起来像是发出一个通知，我开始进入循环了，我从循环退出了...

循环体中调用了一个十分关键的函数 `result = getAndExecuteCommand();` 从函数名猜测是获取和执行命令，根据命令执行结果决定是否退出循环，理一理猜测流程：
调用joinThreadPool让线程执行一个死循环，在循环体中获取和执行命令，线程可能会阻塞在获取命令上，可根据命令的执行结果决定是否退出循环

看一看 `getAndExecuteCommand()`的实现
```C++
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }

        result = executeCommand(cmd);

        // After executing the command, ensure that the thread is returned to the
        // foreground cgroup before rejoining the pool.  The driver takes care of
        // restoring the priority, but doesn't do anything with cgroups so we
        // need to take care of that here in userspace.  Note that we do make
        // sure to go in the foreground after executing a transaction, but
        // there are other callbacks into user code that could have changed
        // our group so we want to make absolutely sure it is put back.
        set_sched_policy(mMyThreadId, SP_FOREGROUND);
    }

    return result;
}

```
首先调用 `result = talkWithDriver();` 和 Binder设备通信，通信成功失败的状态放入result，通信的数据放入mIn（应该是从Binder设备读数据，没数据可读时阻塞在talkWithDriver函数中）。
接着对 mIn 中的数据做一些简单的校验，校验通过后从 mIn 中取出 命令 调用` result = executeCommand(cmd);` 执行命令，并返回结果到上层那个循环中，循环根据这个结果决定是否退出循环。

先进入 talkWithDriver 看看是怎么和设备交换数据的，看看是否在读数据时是否确实会阻塞
```C++

status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }
    
    binder_write_read bwr;
    
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(HAVE_ANDROID_OS)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < (ssize_t)mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }
    return err;
}
```

这里出现一个非常重要的数据结构 `binder_write_read` 用来表示需要读取或写入的数据存放位置、大小等
```
struct binder_write_read {
     signed long write_size;        //需要写入的数据大小，0表示不写入数据
     signed long write_consumed;    //真正写入的数据大小
     unsigned long write_buffer;    //需要写入的数据存放的位置
     signed long read_size;         //需要读取的数据大小，0表示不读取数据
     signed long read_consumed;     //真正读入的数据大小
     unsigned long read_buffer;     //读取的数据存放的位置
};
```
mOut 中存放要写入Binder设备的数据， mIn 中存放从 Binder 设备中读取的数据

Binder 设备 ioctl 系统调用的实现支持同时写入和读取数据，根据   `binder_write_read`  结构中的 write_size 和 read_size 判断是否需要写入或读取数据，

talkWithDriver 函数首先判断是否需要读取，是否需要写入，构造 binder_write_read 结构，然后发起 `ioctl` 系统调用，可以看到这个调用放在了一个 do while 循环中，这里的意思不是要循环写入，而是在处理写入过程中被信号或其他中断打断了当前写入过程时重新发起写入，因为有些中断信号具有更高的优先级，执行流程转去执行中断实现了，当从中断返回后重新启动刚刚被打断的过程， ioctl 被中断会返回错误号 `-EINTR`。

ioctl 成功返回后，判断下真正写入的数据大小和真正读入的数据大小设置 mOut 和 mIn，当有需要写入的数据而这次并没有全部写入时，会在下次调用talkWithDriver时继续写入。

接着回到 `getAndExecuteCommand`, get数据这一步已经完成了，能从 Binder 设备中读到数据，说明肯定有另一方写入数据到 Binder 设备中，这就是进程间通信的另一个进程，这里又出现一个问题，既 Binder 设备只有一个，大家都往同一个设备写入或读取数据，怎么判断数据是发往哪个进程的？肯定有某种方法判断，先不管，以后分析 Binder 设备时在看看这个问题是怎么处理的。

继续往下执行，从 mIn 中取刚刚获取的数据，另一个进程按什么格式写进来的数据，这里就是按同一个格式读取，上来就是一个 `cmd = mIn.readInt32();` 说明头4个字节是命令号，后面的字节根据不同的命令附带上不同的参数。
读取到命名后扔进`executeCommand(cmd)`执行，看看 `executeCommand()`这个函数的实现,太长了，摘抄一段
```C++
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;
    
    switch (cmd) {
    case BR_ERROR:
        result = mIn.readInt32();
        break;
        
    case BR_OK:
        break;
        
    case BR_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readInt32();
        obj = (BBinder*)mIn.readInt32();
        ALOG_ASSERT(refs->refBase() == obj,
                   "BR_ACQUIRE: object %p does not match cookie %p (expected %p)",
                   refs, obj, refs->refBase());
        obj->incStrong(mProcess.get());
        IF_LOG_REMOTEREFS() {
            LOG_REMOTEREFS("BR_ACQUIRE from driver on %p", obj);
            obj->printRefs();
        }
        mOut.writeInt32(BC_ACQUIRE_DONE);
        mOut.writeInt32((int32_t)refs);
        mOut.writeInt32((int32_t)obj);
        break;
        ...
```
可以看到是一个大switch，根据不同的cmd执行不同的动作，这些动作是预先定义的，以`BR`开头的常量
总共有这么多种命令
```
static const char *kReturnStrings[] = {
    "BR_ERROR",
    "BR_OK",
    "BR_TRANSACTION",
    "BR_REPLY",
    "BR_ACQUIRE_RESULT",
    "BR_DEAD_REPLY",
    "BR_TRANSACTION_COMPLETE",
    "BR_INCREFS",
    "BR_ACQUIRE",
    "BR_RELEASE",
    "BR_DECREFS",
    "BR_ATTEMPT_ACQUIRE",
    "BR_NOOP",
    "BR_SPAWN_LOOPER",
    "BR_FINISHED",
    "BR_DEAD_BINDER",
    "BR_CLEAR_DEATH_NOTIFICATION_DONE",
    "BR_FAILED_REPLY"
};
```
在每条命令的执行中，有时需要返回数据给调用者，这时只要把数据写入 mOut 中，并带上相应的命令响应号，在下次循环调用 talkWithDeriver 时，数据就被返回了 ，比如上面的 BR_ACQUIRE
```C++
mOut.writeInt32(BC_ACQUIRE_DONE);
mOut.writeInt32((int32_t)refs);
mOut.writeInt32((int32_t)obj);
```
如此这般，利用 Binder 设备完成了进程间的数据交换，但是还没完。
总结一下，当前，有了ProcessState持有Binder设备的文件描述符可以和设备通信，有了多个IPCThreadState实例，每个实例都在独立的线程，也就是有了多个线程，并且可以动态产生新的线程，这些线程可以利用ProcessState实例中的Binder设备的文件里描述和Binder设备通信，既每条线程都可以进入一个循环从Binder设备读或写数据，这样就完成了基本的进程间通信功能。这里还有一个问题就是进程A要和进程B通信，同时打开Binder设备的进程可能有很多个，怎么找到自己想要通信的进程？无非是进程A要发数据给进程B时，带上进程B的标识信息到Binder设备，Binder设备根据这个标识信息定位具体的进程，具体的实现留在分析Binder设备实现是在看。

现在可以进行进程间交换数据了(IPC)，但是还不够，我们的目标是RPC，说白了基本原理很简单，一层层的封装让基本原理隐藏在很深的地方，扒开了看看，举例子说明

##一个`不要在意细节版音乐播放器`的客户端和服务端例子：
服务端操作声卡、解码音频文件实现真正的音乐播放功能，客户端控制音乐开始播放，停止播放等功能。

|MusicPlayerClient      |MusicPlayerServer|
|---------------        |-----------|
|void play(String uri); |void play(String uri);|
|void stop();           |void stop();|

MusicPlayerClient 可以看作是 MusicPlayerServer 的代理，完成跨进程调用服务端的功能，把实现的接口单独抽取出来

>**下面代码主要描述思想
不要太在意语法细节**

```Java
public interface IMusicPlayer {
    void play(String uri);
    void stop();
}
```

然后客户端和服务端分别在自己的进程中实现 IMusicPlayer

首先给所有方法编号，客户端调用时带上编号，服务端收到时就知道是调用哪个方法了，

```Java
enum MethodId {
    PLAY，
    STOP，
}
```
```Java
class MusicPlayerClient implements IMusicPlayer {
    ProcessState mProcess;
    
    private MusicPlayerClient() {
        mProcess = ProcessState.self();                     //打开Binder设备
    }
    void play(String uri) {
        //这里直接指定发送目标，实际情况下还需要通过一个中介来获取目标的标识
        byte[] target = "MusicPlayerServer".getBytes();     //指定发送远端目标
        byte[] cmd = {PLAY};                                //指定调用方法                  
        byte[] musicUri = = uri.getBytes();                 //方法参数
        byte[] req = byte[] {target, cmd, musicUri};        //拼接在一起
        
        ioctl(mProcess.mDriverFD, BINDER_WRITE_READ,req);   //发送请求到Binder设备
    }
    
    void stop() {
        
        byte[] target = "MusicPlayerServer".getBytes();     //指定发送远端目标
        byte[] cmd = {STOP};                                //指定调用方法                  
        byte[] req = byte[] {target, cmd};                  //拼接在一起
        
        ioctl(mProcess.mDriverFD, BINDER_WRITE,req);        //发送请求到Binder设备
    }
}
```

实现服务端
```Java
class MusicPlayerServer implements IMusicPlayer {
    ProcessState mProcess;
    String mIdentify = "MusicPlayerServer";
    
    private MusicPlayerServer() {
        mProcess = ProcessState.self();                     //打开Binder设备
        
        byte[] req = {mIdentify.getBytes(), this};
        ioctl(mProcess.mDriverFD, ADD_SERVICE,req);         //注册当前服务到Binder设备
        
        loop();                                             //开启循环读取并执行命令
    }
    
    void play(String uri) {
        //打开音频文件
        //解析音频格式，获取属性
        //打开声卡
        //设置声卡属性
        //播放
    }
    
    void stop() {
        //判断当前是否正在播放
        //如果正在播放，则停止播放
    }
    
    void loop() {
        new Thread(new Runnable() {
            public void run() {
                while (true) {
                    byte cmd;
                    byte[] buf;
                    //从Binder设备读取命令，如果没有则阻塞
                    ioctl(mProcess.mDriverFD, BINDER_READ, &buf);    
                    cmd = buf[0];
                    switch (cmd) {
                        case START;
                            String musicUri = new String(buf[1]);
                            play(musicUri);
                        break;
                        case STOP;
                            stop();
                        break;
                    }
                    
                }
            }
        }}.start;
    }
}
```
这样就实现了音乐播放器的服务端和代理，再次强调，不要在意细节。

如果想写一个音乐播放器，则变得很简单

```Java
    class MusicPlayer {
        public void main(String[] args) {
            MusicPlayerClient client = MusicPlayerClietn.instance();
            client.play(getTop1Music());
        }
    }
```
真正的音乐播放功能是发生在另一个进程中的，而这些对用户来说都是透明的，感觉不到，这样就实现了 RPC。

可以看到还有一些问题，在这个`不要在意细节版的音乐播放器`中，可以看到命令读取和执行和Binder设备交互都耦合在一起，而Binder设备交互完全可以抽取出来，命令的实现和命令的读取也可以分离开来，理解了就可以开始加上层层封装把这些原始的东西隐藏起来，为了方便嘛
##是时候引入Android中的Binder框架了##




---
Binder 设备源码
https://android.googlesource.com/kernel/common.git/+/android-3.4/drivers/staging/android/

Android Bander设计与实现 - 设计篇 
http://blog.csdn.net/universus/article/details/6211589

Android Binder设计与实现 - 实现篇（1）
http://www.cnblogs.com/albert1017/p/3849585.html

红茶一杯话Binder（初始篇）
http://my.oschina.net/youranhongcha/blog/149575

从2-3-4树谈到Red-Black Tree（红黑树） 
http://blog.csdn.net/v_JULY_v/article/details/6531399

从B树、B+树、B*树谈到R 树 
http://blog.csdn.net/v_JULY_v/article/details/6530142
