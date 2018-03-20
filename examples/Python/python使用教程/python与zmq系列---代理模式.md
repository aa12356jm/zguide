现在，你已经熟练的掌握了REQ/REP模式，它是一个一对多的模式，一个REP对应多个REQ。

        但是现实工作中，我们会遇到这样的难题，一个REP无法满足REQ的提问，因为REQ太多了，虽然可以增加一个REP，但是，这样做会带来很多问题。两个REP的端口不可能是一个，那么就需要将原来的一些REQ与这个新的REQ连接，这里面的工作量可想而知。那么我们能不能采取一种简单方便的方式，使得REQ和REP都可以灵活的增加减少而不对系统造成负面影响导致工作量上升呢？

         面对这样的要求，我们需要一个代理。让REQ和REP都连接这个代理，一切繁杂的事情都由这个代理进行。我从《ZeroMQ 云时代极速消息通信库》这本书里拍下了扩展的请求-应答解析图。

         

         在扩展的请求-应答系统里，REQ和REP并不直接通信，而是通过ROUTER和DEALER，ROUTER是路由器，DEALER是经销商，所有的请求到达路由器以后公平的排队，然后由经销商负载均衡后发送给REP服务器，服务器应答的结果再由经销商和路由器返回给REQ客户端。

         在这个系统里，你可以随意的增加和减少REQ，可以随意的增加和减少REP，而不必为此付出额外的工作量。

         到了这一步，聪明的朋友已经发现了问题了，一个DEALER怎么可以连接多个REP呢？没错，一个DEALER不能连接多个REP，但多个REP可以连接一个DEALER。在一对多的REQ/REP模型中，REP都是等待REQ来连接的，但在我们上图所示的系统中，REP是主动连接DEALER的，而不是等待DEALER来连接它。

         奉上示例代码

         客户端：


        #coding=utf-8  
        ''''' 
        Created on 2015-10-13 
        发起请求 
        @author: kwsy2015 
        '''  
        import zmq  
        
        #  Prepare our context and sockets  
        context = zmq.Context()  
        socket = context.socket(zmq.REQ)  
        #这一次，我们不连接REP，而是连接ROUTER，多个REP连接一个ROUTER  
        socket.connect("tcp://localhost:5559")  
        
        #  发送问题给ROUTER  
        for request in range(1,11):  
            socket.send(b"Hello")  
            message = socket.recv()  
            print("Received reply %s [%s]" % (request, message))  
        socket.close()  
        context.term()  

        中间代理
        [python] view plain copy
        #coding=utf-8  
        ''''' 
        Created on 2015-10-13 
        
        @author: kwsy2015 
        '''  
        import zmq  
        
        # Prepare our context and sockets  
        context = zmq.Context()  
        
        frontend = context.socket(zmq.ROUTER)  
        backend = context.socket(zmq.DEALER)  
        frontend.bind("tcp://*:5559")  
        backend.bind("tcp://*:5560")  
        
        # Initialize poll set  
        poller = zmq.Poller()  
        poller.register(frontend, zmq.POLLIN)  
        poller.register(backend, zmq.POLLIN)  
        
        # Switch messages between sockets  
        while True:  
            socks = dict(poller.poll())  
            
            # frontend 收到了提问后，由backend发送给REP端  
            if socks.get(frontend) == zmq.POLLIN:  
                message = frontend.recv_multipart()  
                backend.send_multipart(message)  
            
            # backend 收到了回答后，由frontend发送给REQ端  
            if socks.get(backend) == zmq.POLLIN:  
                message = backend.recv_multipart()  
                frontend.send_multipart(message)  

##服务端

        #coding=utf-8  
        ''''' 
        Created on 2015-10-13 
        收到请求后回复world 
        @author: kwsy2015 
        '''  
        import zmq  
        
        context = zmq.Context()  
        socket = context.socket(zmq.REP)  
        # REP连接的是DEALER  
        socket.connect("tcp://localhost:5560")  
        
        while True:  
            message = socket.recv()  
            print("Received request: %s" % message)  
            socket.send(b"World")  

    在服务端，REP没有调用bind函数，而是调用了connect函数去连接DEALER。请求的排队和服务端的负载均衡均由中间的代理完成了，事情就是这样的简单。