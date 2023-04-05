---
title: "ThreadPool 分析和对比"
date: 2020-03-27T16:00:39+08:00
---
# 线程池分类及对比
- HS/HA（半同步半异步）:主线程处理工作任务并存入工作队列，工作线程从工作队列取出任务进行处理，如果工作队列为空，则取不到任务的工作线程进入挂起状态。
- L/F（领导者追随者）:线程池中的线程可处在3种状态之一：领导者 leader、追随者 follower 或工作者 processor。任何时刻线程池只有一个领导者线程。事件到达时，领导者线程负责消息分离，并从处于追随者线程中选出一个来当继任领导者，然后将自身设置为工作者状态去处置该事件

||HS/HA|L/F|
|---|---|---|
|优势|有缓冲，可以承载大流量|不需要额外的存储队列；cpu cache 友好；降低分派任务产生的延迟|
|劣势|存在线程间的共享数据和额外的拷贝（c++11之后可避免）|实现复杂；无缓冲|

根据这两种线程池的特性可以得到：

如果是对业务逻辑中的某一处一系列的任务进行并行处理，降低总体完成时间的话。选择 HS/HA 比较好，因为一定会存在跨线程的传递数据，L/F 并不占太大的优势。

而L/F则适合某个业务逻辑从开始到结束均在单一线程内完成，多个同样的业务逻辑并行处理
[一篇详细论证的论文](http://www.dre.vanderbilt.edu/~schmidt/PDF/OM-01.pdf)
# 一种 HS/HA 模式的实现
涉及到的部分 c++11 相关知识
- [右值引用](https://zh.cppreference.com/w/cpp/language/reference)
- [std::forward](https://zh.cppreference.com/w/cpp/utility/forward)
- [lambda表达式](https://zh.cppreference.com/w/cpp/language/lambda)
- [std::function](https://zh.cppreference.com/w/cpp/utility/functional/function)
- [模板形参包](https://zh.cppreference.com/w/cpp/language/parameter_pack)
- [std::packaged_task](https://zh.cppreference.com/w/cpp/thread/packaged_task)
- [decltype](https://zh.cppreference.com/w/cpp/language/decltype)
- [std::bind](https://zh.cppreference.com/w/cpp/utility/functional/bind)
```c++
#include <vector>
#include <queue>
#include <memory>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <stdexcept>

class ThreadPool {
public:
	ThreadPool(size_t);
	~ThreadPool();
	template<typename F, typename... Args>
	auto enqueue(F&& f, Args&&... args)->std::future<decltype(f(args...))>;
private:
	// need to keep track of threads so we can join them
	std::vector<std::thread> workers;
	// the task queue
	std::queue<std::function<void()>> tasks;

	// synchronization
	std::mutex queue_mutex;
	std::condition_variable condition;
	bool stop;
};

// the constructor just launches some amount of workers
ThreadPool::ThreadPool(size_t threads)
	: stop(false)
{
	for (size_t i = 0; i < threads; ++i)
		workers.emplace_back([this] {
		while (true)
		{
			std::function<void()> task;
			{
				std::unique_lock<std::mutex> lock(this->queue_mutex);
				this->condition.wait(lock, [this] { return this->stop || !this->tasks.empty(); });
				if (this->stop && this->tasks.empty())
					return;
				task = std::move(this->tasks.front());
				this->tasks.pop();
			}
			task();
		}
	}
	);
}

// the destructor joins all threads
ThreadPool::~ThreadPool()
{
	{
		std::lock_guard<std::mutex> lock(queue_mutex);
		stop = true;
	}
	condition.notify_all();
	for (auto &worker : workers)
		worker.join();
}

// add new work item to the pool
template<typename F, typename... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)->std::future<decltype(f(args...))>
{
	using return_type = decltype(f(args...));

	auto task = std::make_shared<std::packaged_task<return_type()>>(std::bind(std::forward<F>(f), std::forward<Args>(args)...));

	std::future<return_type> res = task->get_future();
	{
		std::lock_guard<std::mutex> lock(queue_mutex);

		// don't allow enqueueing after stopping the pool
		if (stop)
			throw std::runtime_error("enqueue on stopped ThreadPool");

		tasks.emplace([task]() { (*task)(); });
	}
	condition.notify_one();
	return res;
}
```
# 线上某服务的 HS/HA 实现
```c++
// AsyncIoJob.h
#ifndef _ASYNCIOJOB_H_
#define _ASYNCIOJOB_H_


#include <stdint.h>
#include <string>
#include <semaphore.h>

using namespace std;

typedef enum _JobStatus{
    INIT=0,
    RUNNING,
    COMPLETE,
    ERROR
}JobStatus;


typedef enum _JobType{    
    THREAD_JOB = 1,
    ASYIO_JOB = 2,
}JobType;


typedef void* (*JobFunc)(void*);

typedef struct _thread_job_params{
    void* args;
    JobFunc worker_func; 
   
}thread_job_params;


typedef struct _AsyncIoJob{ 
    JobType         type;
    JobStatus       status;
    std::string     msg;
    sem_t*          client_sem;
    unsigned int    timeused_ms;
    uint64_t        ts_in;
    uint64_t log_id;
    union{       
        thread_job_params           p_thread_job;
    }params;
}AsyncIoJob;

#endif
```
```c++
// AsyncIo.cpp
#include <list>
#include <string.h>
#include <unistd.h>
#include <sys/syscall.h>  
#define gettid() syscall(__NR_gettid)
#include "AsyncIo.h"
#include "log.h"
#include "utils.h"


AsyncIo::AsyncIo(int thread_count)
{
    this->thread_count = thread_count;
    this->stop_tag = false;
}

AsyncIo::~AsyncIo()
{
}

void* worker(void *arg)
{
    pthread_t tid = pthread_self();
    pthread_detach(tid);
    AsyncIo * aio = (AsyncIo *)arg;
    int ttid = gettid();
    set_pid( ttid );
    log_notice("worker thread[%lu] start ttid=%d", tid, ttid);
    while(!aio->GetStopTag())
    {
        AsyncIoJob *job = aio->PopJob();
        if(NULL == job) continue;
        set_uuid(job->log_id);
        job->status = RUNNING;
        int ret;
        int cost = Utils::timems() - job->ts_in;
        if( cost > 200 ){
            log_notice("job_delay:%d",cost);        
        }
        switch(job->type)
        {
            case THREAD_JOB:
                {
                    thread_job_params& t_job = job->params.p_thread_job;
                    t_job.worker_func(t_job.args);
                    job->status = COMPLETE;
                    sem_post(job->client_sem);
                    break;               
                }
            case ASYIO_JOB:
                {
                    thread_job_params& t_job = job->params.p_thread_job;
                    t_job.worker_func(t_job.args);
                    job->status = COMPLETE;
                    delete job;
                    break;               
                }


            default:
                log_warn("job type error: %d", job->type);
                sem_post(job->client_sem);
        }
    }
    log_notice("worker thread[%lu] stop", tid);
}

bool AsyncIo::Init()
{
    for(int i = 0; i < this->thread_count; i ++)
    {
        pthread_t *tid = new pthread_t;
        int err = pthread_create(tid, NULL, worker, (void*)this);
        if(err != 0){
            log_warn("pthread_create error[%d]:%s", i, strerror(err));
            return false;
        }
        this->tid_list.push_back(tid);
    }
    return true;
}

void AsyncIo::Uninit()
{
    this->SetStopTag(true);
    if(this->thread_count > 0) {
        std::list<pthread_t*>::iterator it = this->tid_list.begin();
        for(; it != this->tid_list.end(); ++it)
        {
            //pthread_join(it, NULL);
            delete *it;
        }
    }
}

void AsyncIo::SetStopTag(bool stop)
{
    this->stop_tag = stop;
}

bool AsyncIo::GetStopTag()
{
    return this->stop_tag;
}

bool AsyncIo::PushJob(AsyncIoJob* job)
{
    if(NULL == job)
        return false;
    job->ts_in = Utils::timems();
    this->aio_job_queue.push(job);
    return true;
}

int AsyncIo::JobsCount()
{
    return this->aio_job_queue.size();
}

AsyncIoJob* AsyncIo::PopJob()
{
    AsyncIoJob *job = NULL;
    this->aio_job_queue.wait_and_pop(job);
    return job;
}
```

# 两种线程池实现的对比
||ThreadPool|AsyncIo|
|---|---|---|
|优势|使用c++11实现，代码结构较为清晰，使用较为便捷|功能较为丰富，各版本编译器均可以编译使用|
|劣势|仅有核心功能；使用模板实现，可读性较差；不支持c++11的编译器无法使用|c风格实现，代码结构和使用较为复杂，类型不安全|
# Benchmark
gcc 4.8.5，8核cpu，10个线程，100个任务，每个任务耗时200ms情况下的平均值
||ThreadPool(ms)|AsyncIo(ms)|
|---|---|---|
|O0|2225.0|2156.8|
|O1|2162.1|2173.6|

O0 时 ThreadPool 相对AsyncIo 多3%左右的时间

O2 时 ThreadPool 相对 AsyncIo 少0.6%左右的时间

综上，几乎性能上没有什么差别。有时间调整参数多做几次实验