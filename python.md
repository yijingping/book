# python进程与线程

## 什么时候使用多线程和多进程
__Python多线程只有用在非计算密集型应用（如I/O密集的抓取程序）上才有意义__

因为Python解释器使用了GIL（Global Interpreter Lock，全局解释器锁定），在任意时刻只允许单个Python线程执行。

- 如果程序是费I/O的，可以使用多线程；
- 如果程序是费CPU的，多线程反而会降低速度；（这种情况应该使用c模块扩展实现）

__但Python多进程是可以利用多个CPU的__

所以上面的问题也可以用python多进程来解决。
但凡是多进程的程序都应优先考虑下面的解决方案：

- 将要共享的数据放在redis或者数据库中，分布式执行程序，遇到性能问题，只要加机器就行了。
- 或者使用分布式队列系统，如ZeroMQ、RabbitMQ、Beanstalk、Thrift（这个不是队列）等， 开多个worker。

## 多进程（multiprocessing）

### 创建多进程
两种创建方式：传参和继承

传参 

    import multiprocessing
    import time
    
    def clock(interval):
        while True:
            print("The time is %s" % time.ctime())
            time.sleep(interval)
    
    if  __name__ == '__main__':
        p = multiprocessing.Process(target=clock, args=(3,))
        p.start()

继承 

    import multiprocessing
    import time
    
    class ClockProcess(multiprocessing.Process):
        def __init__(self, interval):
            multiprocessing.Process.__init__(self)
            self.interval = interval
        def run(self):
            while True:
                print("The time is %s" % time.ctime())
                time.sleep(self.interval)
    
    if  __name__ == '__main__':
        p = ClockProcess(3)
        p.start()

__Tips：__ 

如果设置了`p.daemon=True`，应在`main`结束前执行`p.join()`等待`p`进程结束，否则main thread退出，children thread也会退出

### 进程间通信（IPC）
`multiprocessing`模块支持进程间通信的2种主要形式：队列（Queue/JoinableQueue）和管道（Pipe），其他方式（非主流）：共享内存和信号量。

* Queue：
主要方法： `get()`和`put(item [, block [, timeout]])`

* JoinableQueue：
比Queue多2个主要方法：`task_done()`和`join()`

* Pipe：
在进程之间创建一条管道，并返回元组(conn1,conn2)，默认情况下，是双向的。1发的在2收，2发的在1收。
主要方法： `send(obj)` 和 `recv()`， `send_bytes(buffer [, offset [, size]])`和`recv_bytes([maxlength])`

在创建一个进程的时候，元组是不同的，所以在不同的进程执行close只会关闭当前的conn。

* 共享内存和信号量
比队列和管道效率都更高，限制也更多。应为前面两种执行了序列化和反序列化操作。

如何生产者（producer） 想要通知消费者（consumer），它不再生产任何东西，消费者应该关闭。需要编写代码设置标志（sentinel）。如：None。

    import multiprocessing
    
    def consumer(input_q):
        while True:
            item = input_q.get()
            if item is None:
                break
            print(item)
        # 关闭
        print("Consumer done")
    
    def producer(sequence, output_q):
        for item in sequence:
            output_q.put(item)
    
    if __name__ == '__main__':
        q = multiprocessing.Queue()
        cons_p = multiprocessing.Process(target=consumer, args=(q,))
        cons_p.start()
    
        sequence = [1,2,3,4]
        producer(sequence, q)
        # 设置关闭标志
        q.put(None)
    
        # 等待消费者进程关闭
        cons_p.join()

### 进程池（procss pool）

### 多进程处理的一般建议
* 避免使用共享数据，尽量使用消息传递和队列。这样可以更好的扩展。
* 避免混合多进程和多线程

## 多线程（multithreading）

### 创建多线程
与多进程一样，两种创建方式：传参和继承。（Java中也是如此）
只须将`multiprocessing.Process`换成`multiprocessing.Thread`即可

### 使用Lock
涉及使用Lock的时候，一定要用下面的形式

    # 第1种
    try:
        lock.acquire()
        # critical section
        statements
        ...
    finally:
        lock.release()

    # 第2种
    with lock:
        # critical section
        statements
        ...

### 多线程通信
提供三种队列：Queue(FIFO)/LifoQueue/PriorityQueue。

与进程的队列不同，他们都有task_done()和join()方法。

