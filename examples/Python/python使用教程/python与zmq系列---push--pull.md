今天为大家介绍push/pull模式，这是一个什么模式呢？战争时期，食物紧缺，实行配给制，大家都排好队，有人专门发放食物，前一个人领取了食物，后一个人跟上继续领取食物，这个push端就是发放食物的，pull端就是领取食物的，所不同的是，现实中，你领取完了食物就不能排队等候了，但zmq的push/pull模式中，一个pull端领取完了食物，可以继续排队等待push端发放食物。

 

push端代码：


    #coding=utf-8  
    ''''' 
    Created on 2015-10-29 
    
    @author: kwsy2015 
    '''  
    import zmq  
    import time  
    
    context = zmq.Context()  
    server = context.socket(zmq.PUSH)  
    server.bind('tcp://*:6666')  
    count = 0  
    while True:  
        server.send('%d' % count)  
        print 'send','count'  
        count +=1  
        time.sleep(0.2)  

pull端代码：
    [python] view plain copy
    #coding=utf-8  
    ''''' 
    Created on 2015-10-29 
    
    @author: kwsy2015 
    '''  
    import zmq  
    context = zmq.Context()  
    client = context.socket(zmq.PULL)  
    client.connect('tcp://localhost:6666')  
    
    while True:  
        msg = client.recv()  
        print msg  

        开启一个push端，开启三个pull端，你可以通过打印的日志感受这种模式

        push端只管向外推送数据，而不关心有多少个pull端连接自己，pull端呢，只管等待消息，至于各个pull端是如何竞争的，我们不需要考虑，一个任务数据，只会被一个pull端接收，这个模式像不像一个小的负载均衡呢。



        在push/pull模式中，服务端和客户端谁都可以先启动，一方断掉不会影响到另一方。

        熟悉redis的同学应该已经想到了，这种模式不就是redis里的任务队列么，没错，其实呢，这个和redis里的任务队列没有区别，但redis的任务队列是不会丢失消息的，也就是说，在redis中，一方不停的向任务队列里放任务，哪怕没有客户端从任务队列里取任务也是没有任何问题的，想要达到redis任务队列容量的最高上限恐怕是不容易的。但在zmq中，我们push的消息却是没有一个数据库来存储的，消息要么在发送缓冲区，要么在接收缓冲区，要么在从发送缓冲区去接收缓冲区的路上，如果一方非常快的push消息，而pull方却动作缓慢，那么消息就会在这三个区域累积，累积的越多也就越危险。zmq中，套接字的连接都有自己的发送或接收的管道和hwm,这个hwm就是管道的容量，在v2.x中，hwm默认是无限的，在v3.x版本中，默认为1000，我查阅了一些资料，却没有弄清楚这个1000的单位是什么，M，K,还是消息的数量？不得而知，但我们务必清楚，累积了太多总是不好的，就算hwm设置为无限，你的内存也总是有限的，这一点一定要在编程时注意