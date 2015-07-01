## 1. Why Blocking Queue ##

Blocking Queue（阻塞队列）意味着如果队列空了，出队操作线程将阻塞直至有其他线程完成了入队操作；如果队列满，入队动作将阻塞，直到有出队操作线程完成了出队操作。

APR基于POSIX实现了一个thread safe的Blocking Queue，为什么说是thread safe呢？保障thread safe的手段有多种，比如编写可重入的代码，加入某种机制使得序列化访问数据（比如mutex），或者是TLS（Thread Local Storage，即让每个线程有自己的私有拷贝）。APR基于互斥锁和条件变量实现thread safe。  

基于POSIX各种接口的运行环境可以分为四类：   

- 中断处理函数
- 线程处理函数
- 退出处理函数
- 信号处理函数     


## 2. 实现原理 ##

互斥锁和条件变量用于解决同一个进程内各个线程之间的同步问题。如果互斥锁或者条件变量位于共享内存区，就可以同步多个进程。  

- 互斥锁用于保护临界区，上锁分为阻塞和非阻塞， `pthread_mutex_lock`和`pthread_mutex_trylock`，导致封装结构返回的状态也不同(一个等待一个忙)；解锁接口`pthread_mutex_unlock`其实就是在等待线程中选择一个唤醒，而如何选择则留给具体实现，比如加入优先级。
- 条件变量总是关联一个互斥锁，两者往往封装在一起。




`pthread_cond_timedwait()`完成两件事情：一是把调用线程阻塞在条件变量上；二是对关联的互斥锁解锁。这就是意味着**调用线程在调用该函数之前必须已经拿到互斥锁**，而一旦从该函数返回，该互斥锁重新归调用者所有。  
  
那么调用线程何时能够继续执行，而不是阻塞在这儿呢？两种情况：   

- 另外一个线程给针对该条件变量进行信号或者广播操作，即发信号给调用线程，告诉它别等了；
- 调用线程被cancel了（一个线程等在一个条件变量上是可以被取消的）。

无论是哪种方式获得了unblock状态，还是先得重新拿到互斥锁才行。因此在使用是有以下注意点：   

- 千万不要递归使用一个与条件变量相关联的互斥锁，因为如果一个阻塞在某个条件变量上的线程被取消，该线程重新拿到了保护条件变量的互斥锁，以此确保在线程的cleanup处理程序在调用`pthread_cond_timedwait()`前后状态一致；如果其他线程拿到该互斥锁，那么被取消的线程会一直被阻塞，知道能够拿到互斥锁。   

- 确保一个线程的cleanup处理释放互斥锁；
- 超时设置是绝对时间，可以为纳秒级别；
- 除了中断处理函数不能调用外，其余都可以安全地调用该函数

关于返回值，也有一些玄机在里面，比如：

- EOK：意味着成功或者是被收到的信号所中断；
- EAGAIN：没有足够的系统资源等这个条件变量了；
- EFAULT：访问缓冲区时候出错；
- EINVAL：参数无效，比如条件变量、互斥锁或者超时时间，**还有一种是用不同的互斥锁同时等待同一个条件变量**；
- EPERM：当前线程并不是互斥锁的owner；
- ETIMEDOUT:指定的超时时间到了。



这里面提到了解除调用线程阻塞的两个方法之一是利用信号和广播，即`pthread_cond_signal`和`pthread_cond_broadcast`。其中`pthread_cond_signal`的处理细节如下：  

- 其作用是唤醒阻塞在某个条件变量之上的线程，唤醒谁呢？优先级最好的那个，如果有多个线程，优先级也一样，那就唤醒等待时间最长的那个；
- 可以作为信号处理函数和线程处理函数，但是不能作为中断处理函数和取消点，因为中断里面不能阻塞。

该函数的返回值较为简单，要么成功EOK，要么条件变量无效EINVAL，要么EFAULT。

`pthread_cond_broadcast`顾名思义广播式通知，解除所有等待在该条件变量上的线程，根据线程的优先级依次解除。


### 2.1. 数据结构 ###
   
队列的同步基于一个互斥锁和两个条件变量（一个非空条件，一个非满条件）。

```
struct apr_queue_t {
    void              **data;
    unsigned int        nelts; 			/**< # elements */
    unsigned int        in;    			/**< next empty location */
    unsigned int        out;   			/**< next filled location */
    unsigned int        bounds;			/**< max size of queue */
    unsigned int        full_waiters;
    unsigned int        empty_waiters;
    apr_thread_mutex_t *one_big_mutex;	/**< 典型的用一个mutex */
    apr_thread_cond_t  *not_empty;		/**< 加一个非空条件变量 */
    apr_thread_cond_t  *not_full;		/**< 加一个非满条件变量 */
    int                 terminated;
};

```


### 2.2. 入队操作 ###




### 2.3. 出队操作 ###


出队操作`queue_pop`考虑了阻塞和非阻塞两种操作，因此该接口的参数有元数据`void **`和超时`timeout`这两个参数。 


```
static apr_status_t queue_pop(apr_queue_t *queue, void **data,
                              apr_interval_time_t timeout)
{
    apr_status_t rv;

    if (queue->terminated) {
        return APR_EOF; /* no more elements ever again */
    }

    rv = apr_thread_mutex_lock(queue->one_big_mutex);
    if (rv != APR_SUCCESS) {
        return rv;
    }

    /* 在1.被唤醒 2.队列非空 两个条件都满足之前一直阻塞 */
    if (apr_queue_empty(queue)) {
        if (!timeout) {
			/* 不设超时则直接返回 EAGAIN*/
            apr_thread_mutex_unlock(queue->one_big_mutex);
            return APR_EAGAIN;
        }
		/* 目前没有收到终止命令 */
        if (!queue->terminated) {
			/* 等待thread的个数累加 */
            queue->empty_waiters++;
            if (timeout > 0) {
				/* 当前thread等待条件变量not_empty timeout这么长时间，看看该条件是否能为真 */
                rv = apr_thread_cond_timedwait(queue->not_empty,
                                               queue->one_big_mutex,
                                               timeout);
            }
            else {
				/* 当前线程直接判断条件变量是否满足，即看看对了是否非空 */
                rv = apr_thread_cond_wait(queue->not_empty,
                                          queue->one_big_mutex);
            }
			/* 等待thread的个数递减 */
            queue->empty_waiters--;
            if (rv != APR_SUCCESS) {
                apr_thread_mutex_unlock(queue->one_big_mutex);
                return rv;
            }
        }
		/* 此时调用线程被唤醒，但是发现队列仍旧是空的，返回 */
        /* If we wake up and it's still empty, then we were interrupted */
        if (apr_queue_empty(queue)) {
            Q_DBG("queue empty (intr)", queue);
            rv = apr_thread_mutex_unlock(queue->one_big_mutex);
            if (rv != APR_SUCCESS) {
                return rv;
            }
            if (queue->terminated) {
                return APR_EOF; /* no more elements ever again */
            }
            else {
                return APR_EINTR;
            }
        }
    } 

	/* 在队列中删除该元数据，并更新各种统计值 */
    *data = queue->data[queue->out];
    queue->nelts--;

    queue->out++;
    if (queue->out >= queue->bounds)
        queue->out -= queue->bounds;

	/* 因为有元素出队，如果有线程在等待非满条件变量，用信号通知下*/
    if (queue->full_waiters) {
        Q_DBG("signal !full", queue);
        rv = apr_thread_cond_signal(queue->not_full);
        if (rv != APR_SUCCESS) {
            apr_thread_mutex_unlock(queue->one_big_mutex);
            return rv;
        }
    }

    rv = apr_thread_mutex_unlock(queue->one_big_mutex);
    return rv;
}   


```