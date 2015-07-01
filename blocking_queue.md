## 1. Why Blocking Queue ##

Blocking Queue（阻塞队列）意味着如果队列空了，出队操作线程将阻塞直至有其他线程完成了入队操作；如果队列满，入队动作将阻塞，直到有出队操作线程完成了出队操作。

APR基于POSIX实现了一个thread safe的Blocking Queue，为什么说是thread safe呢？保障thread safe的手段有多种，比如编写可重入的代码，加入某种机制使得序列化访问数据（比如mutex），或者是TLS（Thread Local Storage，即让每个线程有自己的私有拷贝）。APR基于互斥锁和条件变量实现thread safe。     

## 2. 实现原理 ##

互斥锁和条件变量用于解决同一个进程内各个线程之间的同步问题。如果互斥锁或者条件变量位于共享内存区，就可以同步多个进程。  


- 互斥锁用于保护临界区，上锁分为阻塞和非阻塞， `pthread_mutex_lock`和`pthread_mutex_trylock`，导致封装结构返回的状态也不同(一个等待一个忙)；解锁接口`pthread_mutex_unlock`其实就是在等待线程中选择一个唤醒，而如何选择则留给具体实现，比如加入优先级。
- 条件变量总是关联一个互斥锁，两者往往封装在一起。
  等待条件：  
  `int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);`   
  `int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime);`
  唤醒：即告诉block在当前条件变量上的Thread们不用等了。
  `int pthread_cond_broadcast(pthread_cond_t *cond);`
  `int pthread_cond_signal(pthread_cond_t *cond);`

### 2.1. 数据结构 ###
   
队列的同步基于一个互斥锁和两个条件变量（一个非空条件，一个非满条件）。

{% highlight c %}

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

{% endhighlight %}


### 2.2. 入队操作 ###


### 2.3. 出队操作 ###