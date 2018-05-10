---
layout: post
title: 异步日志库 - Muduo 日志库源码解析
subtitle: 他山之石，可以攻玉
tags: [Program]
---

Muduo Logging 的代码由 Logging.h/cc, LogStream.h/cc , LogFile.h/cc , AsyncLogging.h/cc 这四对组成。把整体分为三个模块来考虑：`日志生成`、`日志传递`、`日志打印`。

## 一、日志生成

这一模块要做的工作就是接收用户的信息，将其与日志格式内的其他信息（日期、时间、线程、行号。。）进行拼接组合，形成一条字符串。

这部分由 Logging.h/cc 以及 LogStream.h/cc 组合完成。首先外部代码对日志库所有的访问入口都是通过定义好的宏来：


```C++
#define LOG_TRACE if (muduo::Logger::logLevel() <= muduo::Logger::TRACE) \
  muduo::Logger(__FILE__, __LINE__, muduo::Logger::TRACE, __func__).stream()   // __func__ 预定义符是 gcc 引入的，没什么特别的，见 https://gcc.gnu.org/onlinedocs/gcc/Function-Names.html
#define LOG_DEBUG if (muduo::Logger::logLevel() <= muduo::Logger::DEBUG) \
  muduo::Logger(__FILE__, __LINE__, muduo::Logger::DEBUG, __func__).stream()
#define LOG_INFO if (muduo::Logger::logLevel() <= muduo::Logger::INFO) \
  muduo::Logger(__FILE__, __LINE__).stream()
```

我们以 LOG_INFO 为例，将其展开后为：

```C++
muduo::Logger(__FILE__, __LINE__).stream() << "This is a log."
```

将这句话放在日志打印的地方意味着什么呢？

1. 创建一个 Logger(__FILE__, __LINE__) 的匿名对象；
2. 调用这个匿名对象的 stream() 成员函数；
3. 调用重载后的 << 操作符，输入日志信息；
4. 析构该匿名对象，析构函数内会调用真正的output日志信息函数。

注意，匿名对象的析构发生在该条语句结束后，实际上在实现上，前3步的目的是整合拼接一条完整的日志信息，实际的日志打印动作就发生在第4步析构上。这种手法初看比较新颖，但如果把日志对象看做一种资源的话，可以理解为RAII手法的变种。

### 1. Logger

下面给出 Logger 类的整体结构：

```C++
class Logger {
    // 定义 LogLevel 枚举类型
    enum LogLevel {
        TRACE,
        ...
    }
    // 编译期间从 __FILE__ 中获取 basename
    class SourceFile {
        ...
    }
     
    Logger(...);
    LogStream& stream() {return impl_.stream_;}
 
    private:
    // Impl 类
    class Impl {
        ...
    }
    Impl impl_;
}
```

可以看到，整个 Logger 类主要由 3 个部分组成，分别是枚举类型的LogLevel，非常hack的获取basename的SourceFile类，私有的Impl类。

我们从构造函数入手：

```C++

Logger::Logger(SourceFile file, int line)
  : impl_(INFO, 0, file, line)
{
}
```

初始化一个 Logger 其实是初始化它的 Impl 对象：


```C++
Logger::Impl::Impl(LogLevel level, int savedErrno, const SourceFile& file, int line)
  : time_(Timestamp::now()),
    stream_(),
    level_(level),
    line_(line),
    basename_(file)
{
  formatTime();
  CurrentThread::tid();
  stream_ << T(CurrentThread::tidString(), CurrentThread::tidStringLength()); // T 类把 char*  和对应的 len 整合
  stream_ << T(LogLevelName[level], 6);
  if (savedErrno != 0)
  {
    stream_ << strerror_tl(savedErrno) << " (errno=" << savedErrno << ") ";
  }
}
```

#####TODO. 为什么要使用私有 Impl 类，有什么好处？什么情况下这样用？

在构造过程中，通过 formatTime() 将日期、时间信息传递给 stream对象；这里面要注意的点有：

**Cache:** 区对于一个时间信息： 20180426 14:50:34.345346Z，前面的 20180426 14:50:34 是缓存在 t_time[64] 这个 __thread（线程私有变量）变量里的。同时通过 t_lastSecond 来标识缓存的有消息。也就是说，同1s内打印的日志，只有微秒部分是需要被格式化的。（格式化总是效率低的，想想 Mario 里的 Sprintf）
{: .box-note}

然后将线程信息输入给 stream。

**Cache** 线程id也是类似的被缓存到了 __thread 变量中。每次只需要简单的拷贝字符串即可。
{: .box-note}

最后将日志级别信息输入到 stream 中。

完成构造之后，就是用户的日志信息同样的输入到 stream 之中。我们先不考虑 steam 的具体情况，看下析构 Logger 这个匿名对象会发生什么事。

```C++
Logger::~Logger()
{
  impl_.finish();
  const LogStream::Buffer& buf(stream().buffer());
  g_output(buf.data(), buf.length());  // 日志输出函数，可设置
  if (impl_.level_ == FATAL) 
  {
    g_flush();  // 日志flush 函数，可设置
    abort();
  }
}
```

finish函数会把log格式后面的信息添加进去，包括文件名、行数信息。

**Compile time calculation** 其中，文件名的剪切工作是通过strrchr函数在编译期间求得的，这个有时间再看吧。。。参考 https://www.zhihu.com/question/65616567
{: .box-note}

析构函数非常清晰明了，在这里会取到之前放在 stream 中的字符串，并调用 g_output() 对这条日志输出。

### 2. LogStream

LogStream 及其相关类的结构为：

```C++
class LogStream {
    typedef detail::FixedBuffer<detail::kSmallBuffer> Buffer;
    Buffer buffer_;
     
    self& operator<<(const string& v);
    self& operator<<(...);
}
 
template<int SIZE>
class FixedBuffer {
    char data_[SIZE];
    char* cur_;   // cur_ 指针指向data_数组中有效数据的尾部。
 
    void append(...);
 
    const char* data() const {return data_;}
    int length() const {return static_cast<int>(cur_-data_);}
}
```

`LogStream`在构造过程中，会初始化一个 `FixedBuffer` 对象，其中又初始化一个`char`数组（大小为预先定义的 `kSmallBuffer : 4k`）。
然后通过重载的操作符，将信息写入到这个数组中。其中`Buffer`对象提供 `data()` 和 `length()` 接口供`Logger`类使用。

可以看到`Buffer`提供的`append()`接口入参是`const char* buf, size_t len`，然后内部通过 `memcpy` 来复制字符串。

**Cache** 线回顾 Logger，日志信息的固定格式都为定长的，通过 T 传入stream，这样就直接通过memcpy来拷贝，避免了每次通过 strlen 来获取字符串长度。
{: .box-note}


## 二、日志传递

我们可以很容易的抽象出来一个异步日志库的模式：N个业务线程通过接口将日志信息推送到一个结构内；1 个日志打印线程从这个结构中顺序地打印日志到文件中。

这是一个很典型的多生产者、单消费者场景。这种场景考虑的点无非就三个方向

1. 生产者吞吐能力；
2. 生产者消费者如何高效传递数据；
3. 消费者吞吐能力。

Muduo这部分的内容在 AsyncLogging.h/cc 中。下面给出 AsyncLogging 类的大体结构。

```C++

class AsyncLogging {
public:
    AsyncLogging(...);
    ~AsyncLogging() { ..stop()..};
 
    void append(const char* logline, int len);  // 供 Logger 注册的接口
 
    void start();  // 启动线程
    void stop();   // 关闭线程
 
private:
    void threadFunc();  // 异步线程处理逻辑
 
    BufferPtr  currentBuffer_;  // 当前写入的 buffer
    BufferPtr  nextBuffer_;     // 下一个备用 buffer
    BufferVector  buffers_;     // 写入完毕，待打印的 buffer 集合
}
```

其中的重点是 append 和 threadFunc 函数，前者负责收集业务线程发来的日志消息，后者负责调控 buffer，并调用真正的日志打印动作。

我们称 append 为前台，threadFunc 后台，Muduo logging 的思想为，前台、后台分别 持有 2 个 buffer 和一个 buffervector。前台写满一个buffer后，放入它的buffervector，并通知后台，后台来处理buffer的交换和填充。

```C++
void AsyncLogging::append(const char* logline, int len)
{
    muduo::MutexLockGuard lock(mutex_);
    if (currentBuffer_->avail() > len){       // 当前buffer空间够
        currentBuffer_->append(logline, len);    // 写入
    } else {
        buffers_.push_back(currentBuffer_.release());       // 放入vector，代表该buffer已满，该打印了
        if (nextBuffer_){       // nextBuffer_ 不为空
        currentBuffer_ = boost::ptr_container::move(nextBuffer_);   // 转移，此时 currBuffer 不为空, nextBuffer 变为 NULL
        } else {  // 没有可用 buffer
        currentBuffer_.reset(new Buffer); // Rarely happens，new 一个
        }
        currentBuffer_->append(logline, len);
        cond_.notify();   // 通知后台事件发生
    }
}
```

append 的代码很好理解，就是当前buffer满了后，交给vector，通知后台，并移动 nextbuffer 或者 new 一个。

那么这里还少了的逻辑就是何时 next_buffer 会被填充。看 threadFunc() 的代码:

```C++
void AsyncLogging::threadFunc() {
    BufferPtr newBuffer1(new Buffer);
    BufferPtr newBuffer2(new Buffer);
    BufferVector buffersToWrite;
 
    while (running_) {
        {
            muduo::MutexLockGuard lock(mutex_);
            if (buffers_.empty())  // unusual usage!
            {
                cond_.waitForSeconds(flushInterval_);  // 条件变量，等 3s 或者前台发来消息
            }
            buffers_.push_back(currentBuffer_.release());  // 把当前在写的 buffer 也放到前台vector中
            currentBuffer_ = boost::ptr_container::move(newBuffer1);  // 把 newbuffer1 移交给 currBuffer
            buffersToWrite.swap(buffers_);   // 交换前后台 buffervector
            if (!nextBuffer_){   // 移交 newbuffer2 给前台 nextBuffer
                nextBuffer_ = boost::ptr_container::move(newBuffer2);
            }
        }
        if (!newBuffer1){
            newBuffer1 = buffersToWrite.pop_back();
            newBuffer1->reset();   // 落地完的 buffer 还给 newBuffer1
        }
        if (!newBuffer2){
            newBuffer2 = buffersToWrite.pop_back();
            newBuffer2->reset();
        }  
    }
}
```

这里只描述最常见的一种情况，更多的在书中 P117。

```C++
0. 前台、后台初始化情况：currBuffer: A; nextBuffer: B; newBuffer1: C; newBuffer2: D
1. 前台写满 A，进行一系列操作后，通知后台，此时：currBuffer: B, nextBuffer: NULL, buffers: [A], newBuffer1: C; newBuffer2: D, buffersToWrite: []
2. Lock()
3. 后台将 B 加入 buffers中，此时：currBuffer: NULL, nextBuffer: NULL, buffers: [A, B], newBuffer1: C; newBuffer2: D, buffersToWrite: []
4. 开始交换前后台buffer结构：此时：currBuffer: C, nextBuffer: D, buffers: [], newBuffer1: NULL; newBuffer2: NULL, buffersToWrite: [A, B]
5. Unlock()
6. 调用接口将 buffersToWrite 内容写入文件
7. 将 buffersToWrite 中的buffer移交回 newbuffer，此时：currBuffer: C, nextBuffer: D, buffers: [], newBuffer1: B; newBuffer2: A, buffersToWrite: []
```

整体看，就是毅种循环，下面分析下前后台锁的代价：

当我们说锁的代价，基本不考虑加锁、解锁的代价，真正影响性能的是临界区的竞争。

前台临界区：

1. append消息（调用 memcpy 复制）到buffer中（无可避免，这是多线程下的一个全局对象，也有优化方法，无非就是hash到各个桶里，每个桶一个mutex，减少锁竞争）
2. buffer交换操作，实际操作的是指针，代价低
3. new buffer（基本不会发生）

后台临界区：

1. buffer 指针的操作，代价低

可以看到，生产者几乎不会阻塞住，所需要的操作就是一次memcpy的代价，通过交换指针来进行信息传递，后台线程得以高效的工作。

## 三、日志打印

这部分没什么好说的，Muduo 使用的是 fwrite_unlocked() 和 fflush() 接口。