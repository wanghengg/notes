# lock.h

# block_queue.h

* **为什么是pthread_cond_wait(cond, mutex)而不是pthread_cond_wait(cond)**	

    `pthread_cond_wait(pthread_cond_t* cond, pthread_mutex_t* mutex)`的功能有3个：

    * 调用者线程首先释放mutex
    * 然后阻塞， 等待被别的线程唤醒
    * 当调用者线程被唤醒后，调用者线程会重新获取mutex

    ```c
    /* Wait for condition variable COND to be signaled or broadcast.
       MUTEX is assumed to be locked before.
    
       This function is a cancellation point and therefore not marked with
       __THROW.  */
    extern int pthread_cond_wait (pthread_cond_t *__restrict __cond,
    			      pthread_mutex_t *__restrict __mutex)
         __nonnull ((1, 2));
    ```

    * 当前线程执行`pthread_cond_wait`之前，已经获得了和临界区相关联的mutex，执行`pthread_cond_wait`会阻塞，但是在进入阻塞状态之前，必须释放已经获得的mutex，让其他线程能够进入临界区。
    * 当前线程执行`pthread_cond_wait`后，阻塞等待的条件满足，等待条件满足时会被唤醒；被唤醒后，仍然处于临界区，因此被唤醒后必须再次获得和临界区相关联的mutex