* #### Thread

  ```
  t = Thread(target=, args=(,), daemon=False)
  t.start() # 启动线程
  # 参数 daemon
  # 如果daemon属性为False，主线程结束时会检测该子线程是否结束，如果该子线程还在运行，
  # 则主线程会等待它完成后再退出；
  # 如果某个子线程的daemon属性为True，主线程运行结束时不对这个子线程进行检查而直接退出，
  # 同时所有daemon值为True的子线程将随主线程一起结束，而不论是否运行完成。
  ```

* 进程间通信

  * ##### Event

    通过修改全局的flag来使等待运行的线程开始运行，clear() 方法将flag置为False,set() 方法将flag 置为True

    ```
    from threading import Thread, Event
    import time
    
    class MyThread(Thread):
        def __init__(self,signal,name):
            Thread.__init__(self)
            self.signal=signal
            self.name=name
        def run(self):
            print("I am %s,I will sleep ..." % self.name)
            self.signal.wait()
            print("I am %s,I will awake ..." % self.name)
    
    
    if __name__ == '__main__':
        signal = Event()
    
        for i in range(3):
            t = MyThread(signal,'thread-%i' % i)
            t.start()
    
        time.sleep(3)
        signal.set()
    ```

    ​	event 对象最好单次使用，就是说，你创建一个 event 对象，让某个线程等待这个对象，一旦这个对象被设置为真，你就应该丢弃它。尽管可以通过 `clear()` 方法来重置 event 对象，但是很难确保安全地清理 event 对象并对它重新赋值。很可能会发生错过事件、死锁或者其他问题（特别是，你无法保证重置 event 对象的代码会在线程再次等待这个 event 对象之前执行）。如果一个线程需要不停地重复使用 event 对象，你最好使用 `Condition` 对象来代替。 下面的代码使用 `Condition` 对象实现了一个周期定时器，每当定时器超时的时候，其他线程都可以监测到 :

    ```
    import threading
    import time
    
    class PeriodicTimer:
        def __init__(self, interval):
            self._interval = interval
            self._flag = 0
            self._cv = threading.Condition()
    
        def start(self):
            t = threading.Thread(target=self.run)
            # 主线程结束时，该线程强制结束
            t.daemon = True
            t.start()
        def run(self):
            '''
            Run the timer and notify waiting threads after each interval
            '''
            while True:
                time.sleep(self._interval)
                with self._cv:
                    # 按位异或运算符 相同为0 不同为1 
                     self._flag ^= 1
                     self._cv.notify_all()
    
        def wait_for_tick(self):
            '''
            相当于一个阻塞线程，等待时钟信号
            Wait for the next tick of the timer
            '''
            with self._cv:
                last_flag = self._flag
                # 等待时钟信号  self._flag ^= 1
                print("loop begin last_flag:{},self._flag:{}".format(last_flag,self._flag))
                while last_flag == self._flag:
                    # 在另一个线程运行该函数 
                    # Condition.wait()  使线程进入Condition的等待池等待通知，并释放锁 (即释放wait_for_tick(self) 进行下面的程序)
                    self._cv.wait()
                print("loop end last_flag:{},self._flag:{}".format(last_flag, self._flag))
    
    # Example use of the timer
    ptimer = PeriodicTimer(3)
    ptimer.start()
    # Two threads that synchronize on the timer
    def countdown(nticks):
        while nticks > 0:
            ptimer.wait_for_tick()
            print('T-minus', nticks)
            nticks -= 1
    
    def countup(last):
        n = 0
        while n < last:
            ptimer.wait_for_tick()
            print('Counting', n)
            n += 1
    
    threading.Thread(target=countdown, args=(10,)).start()
    threading.Thread(target=countup, args=(5,)).start()
    ```

    

  * ##### 信号量 Semaphore

    ```
    # 并发限制为 5   maxconnections = 5
    import threading
    import time
    
    def showfun(n):
        print ("%s start -- %d"%(time.ctime(),n))
        print ("working")
        time.sleep(2)
        print ("%s end -- %d" % (time.ctime(), n) )
        semlock.release()
    
    if __name__ == '__main__':
        maxconnections = 5
        semlock = threading.BoundedSemaphore(maxconnections)
        list=[]
        for i in range(8):
            semlock.acquire()
            t=threading.Thread(target=showfun, args=(i,))
            list.append(t)
            t.start()
    ```

  * ##### Queue 通信

    ```
    # 创建一个线程安全的优先级队列
    import heapq
    import threading
    
    class PriorityQueue:
        def __init__(self):
            self._queue = []
            self._count = 0
            self._cv = threading.Condition()
        def put(self, item, priority):
            with self._cv:
                heapq.heappush(self._queue, (-priority, self._count, item))
                self._count += 1
                self._cv.notify()
    
        def get(self):
            with self._cv:
                while len(self._queue) == 0:
                    self._cv.wait()
                return heapq.heappop(self._queue)[-1]
    ```

    如果一个线程需要在一个“消费者”线程处理完特定的数据项时立即得到通知，你可以把要发送的数据和一个 `Event` 放到一起使用，这样“生产者”就可以通过这个Event对象来监测处理的过程了。示例如下： 

    ```
    from queue import Queue
    from threading import Thread, Event
    
    # A thread that produces data
    def producer(out_q):
        while running:
            # Produce some data
            ...
            # Make an (data, event) pair and hand it to the consumer
            evt = Event()
            out_q.put((data, evt))
            ...
            # Wait for the consumer to process the item
            evt.wait()
    
    # A thread that consumes data
    def consumer(in_q):
        while True:
            # Get some data
            data, evt = in_q.get()
            # Process the data
            ...
            # Indicate completion
            evt.set()
    ```

    `q.qsize()` ， `q.full()` ， `q.empty()` 等实用方法可以获取一个队列的当前大小和状态。但要注意，这些方法都不是线程安全的 

    ```
    Queue.get([block[, timeout]])   # 读队列，timeout等待时间 
    Queue.put(item, [block[, timeout]])   # 写队列，timeout等待时间 
    # 默认为阻塞式的，即block=True，此时调用get()、put()方法不会抛出异常，而是直接进入阻塞，
    # 直至超时，设置 block=False 即为非阻塞式，此时会抛出异常
    ```

* #### 加锁 -- 避免竞争

  

