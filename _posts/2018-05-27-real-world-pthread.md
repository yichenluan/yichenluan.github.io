---
layout: post
title: Real World Pthread
tags: [Linux, C++]
---

Pthread 是标准库提供的 API，实际使用是 tough 的，下面看看实际上，该如何封装 Pthread。

以下代码删减自Muduo，去除了错误检查相关，只保留核心思想。

# 一、Mutex

```c++
class MutexLock : boost::noncopyable
{
public:
    MutexLock(){
        pthread_mutex_init(&mutex_, NULL);
    }
    ~MutexLock(){
        pthread_mutex_destroy(&mutex_);
    }
    void lock(){
        pthread_mutex_lock(&mutex_);
    }
    void unlock(){
        pthread_mutex_unlock(&mutex_);
    }
    pthread_mutex_t* getPthreadMutex() /* non-const */{
        return &mutex_;
    }
private:
    pthread_mutex_t mutex_;
};
 
class MutexLockGuard : boost::noncopyable
{
public:
    explicit MutexLockGuard(MutexLock& mutex): mutex_(mutex){
        mutex_.lock();
    }
    ~MutexLockGuard(){
        mutex_.unlock();
    }
private:
    MutexLock& mutex_;
};
```

代码一看便知，不做过多讲解，通过RAII手法来控制 mutex 的生命周期，再通过 Guard 包装来保证 lock 和 unlock 的对应。

具体使用如下：

```c++
class Foo
{
public:
    int size() const;
 
 
private:
    mutable MutexLock mutex_;
    std::vector<int> data_; // GUARDED BY mutex_
};
 
int Foo::size() const {
    MutexLockGuard lock(mutex_);
    return data_.size();
}
```

需要注意的是别写成了 MutexLockGuard(mutex_)，这样产生的是一个匿名对象，该条语句结束后立即析构，没有锁住临界区。


# 二、Condition

```c++
class Condition : boost::noncopyable
{
public:
    explicit Condition(MutexLock& mutex): mutex_(mutex){
        pthread_cond_init(&pcond_, NULL);
    }
     ~Condition(){
        pthread_cond_destroy(&pcond_);
    }
    void wait(){
        pthread_cond_wait(&pcond_, mutex_.getPthreadMutex());
    }
    // returns true if time out, false otherwise.
    bool waitForSeconds(double seconds);
    void notify(){
        pthread_cond_signal(&pcond_);
    }
    void notifyAll(){
        pthread_cond_broadcast(&pcond_);
    }
 
private:
    MutexLock& mutex_;
    pthread_cond_t pcond_;
};
```

与Mutex的设计是非常一致的，具体如何使用呢？请看下面的代码：

```c++
// 使用 Condition 实现一个等待计数到0的抽象
class CountDownLatch : boost::noncopyable
{
public:
    explicit CountDownLatch(int count);
    void wait();
    void countDown();
    int getCount() const;
private:
    mutable MutexLock mutex_;
    Condition condition_;
    int count_;
};
 
CountDownLatch::CountDownLatch(int count)
    : mutex_(),
    condition_(mutex_),
    count_(count){}
 
void CountDownLatch::wait(){
    MutexLockGuard lock(mutex_);
    while (count_ > 0){
        condition_.wait();
    }
}
void CountDownLatch::countDown(){
    MutexLockGuard lock(mutex_);
    --count_;
    if (count_ == 0){
        condition_.notifyAll();
    }
}
int CountDownLatch::getCount() const{
    MutexLockGuard lock(mutex_);
    return count_;
}
```

需要注意的是，mutex_和conditin_的顺序很重要，需要先初始化 mutex_。

# 三、Thread

Thread 的抽象是利用上面的包裹和API进行包装，形成易于使用的接口，限于篇幅，就不贴出代码了，可见于：[Thread.h](https://github.com/chenshuo/muduo/blob/master/muduo/base/Thread.h)

有几个注意的地方：

- Thread对象的析构并没有销毁线程句柄，实际上如果没有被join，就detach线程，避免资源泄漏
- 通过 CoundDownLatch，包装Thread::start()在线程真正被调度到后才返回。