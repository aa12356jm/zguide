我们已经了解REQ/REP,PUB/SUB,PULL/PUSH这三种模式，也曾提到过，一个上下文可以创建多个socket套接字，那么如何管理这些套接字呢？



假设我们的一个客户端既有pull又有sub，他们两个都需要接收消息，该如何协调呢，毕竟，当一个socket要收消息的时候，函数recv是阻塞的，所以，我们第一个思路是不让它阻塞，看示例代码：



    #coding=utf-8  
    ''''' 
    Created on 2015-10-13 
    在这里，同时处理多个套接字，那么接收消息的时候，就需要设置noblock 
    不然会在第一个接收消息的地方堵塞 
    @author: kwsy2015 
    '''  
    import zmq  
    import time  
    
    # Prepare our context and sockets  
    context = zmq.Context()  
    
    # Connect to task ventilator  
    receiver = context.socket(zmq.PULL)  
    receiver.connect("tcp://localhost:5557")  
    
    # Connect to weather server  
    subscriber = context.socket(zmq.SUB)  
    subscriber.connect("tcp://localhost:5556")  
    subscriber.setsockopt(zmq.SUBSCRIBE, b"10001")  
    
    # Process messages from both sockets  
    # We prioritize traffic from the task ventilator  
    while True:  
    
        # Process any waiting tasks  
        while True:  
            try:  
                #用了NOBLOCK，就意味着得不到消息时不会堵塞在这里  
                msg = receiver.recv(zmq.NOBLOCK)  
            except zmq.ZMQError:  
                break  
            # process task  
    
        # Process any waiting weather updates  
        while True:  
            try:  
                msg = subscriber.recv(zmq.NOBLOCK)  
            except zmq.ZMQError:  
                break  
            # process weather update  
    
        # No activity, so sleep for 1 msec  
        time.sleep(0.001)  

        通过设置zmq.NOBLOCK，我们可以让recv不再阻塞，但是呢，要捕捉zmq.ZMQError这个异常，这样一来，两个套接字就可以不发生冲突了

        但是明显，你可以感受得到，这种做法的丑陋，看起来不是那么的优雅，所以我们换一种做法

       

    #coding=utf-8  
    ''''' 
    Created on 2015-10-13 
    这种方式比msreader要更好一些 
    @author: kwsy2015 
    '''  
    import zmq  
    
    # Prepare our context and sockets  
    context = zmq.Context()  
    
    # Connect to task ventilator  
    receiver = context.socket(zmq.PULL)  
    receiver.connect("tcp://localhost:5557")  
    
    # Connect to weather server  
    subscriber = context.socket(zmq.SUB)  
    subscriber.connect("tcp://localhost:5556")  
    subscriber.setsockopt(zmq.SUBSCRIBE, b"10001")  
    
    # Initialize poll set  
    poller = zmq.Poller()  
    poller.register(receiver, zmq.POLLIN)  
    poller.register(subscriber, zmq.POLLIN)  
    
    # Process messages from both sockets  
    while True:  
        try:  
            socks = dict(poller.poll())  
        except KeyboardInterrupt:  
            break  
    
        if receiver in socks:  
            message = receiver.recv()  
            # process task  
    
        if subscriber in socks:  
            message = subscriber.recv()  
            # process weather update  

这种做法就很想socket的select模式，大家谁也别争，谁也别抢，只要有消息达到，我就通知你们，然后你们各自检查是不是自己的消息。我们在客户端创建多个socket套接字可能是合理的，但是服务端就最好别这么做了，REQ,PUSH,PUB，道理其实也很简单，服务就是服务，多个员工可以挤在一个办公司里办公，哪有多个老板挤在一起办公的。