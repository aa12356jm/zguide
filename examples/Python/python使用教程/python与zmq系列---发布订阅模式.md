本系列的内容，参考了电子工业出版社出版的《ZeroMQ云时代极速消息通信库》这本书的内容编排，如果你想阅读书籍，我只告诉你原价108元。

        先通过一个例子来了解zmq吧

server端代码：

        
    #coding=utf-8  
    ''''' 
    Created on 2015-10-13 
    服务端，发布模式 
    @author: kwsy2015 
    '''  
    import zmq  
    from random import randrange  
    
    context = zmq.Context()  
    socket = context.socket(zmq.PUB)  
    socket.bind("tcp://*:5556")  
    
    while True:  
        zipcode = randrange(1, 100000)  
        temperature = randrange(-80, 135)  
        relhumidity = randrange(10, 60)  
    
        socket.send("%i %i %i" % (zipcode,temperature , relhumidity))  

客户端代码：

      


    #coding=utf-8  
    ''''' 
    Created on 2015-10-13 
    订阅模式，如果设置了过滤条件，那么只会接收到以过滤条件开头的消息 
    @author: kwsy2015 
    '''  
    import sys  
    import zmq  
    
    
    #  Socket to talk to server  
    context = zmq.Context()  
    socket = context.socket(zmq.SUB)  
    
    print("Collecting updates from weather server...")  
    socket.connect("tcp://localhost:5556")  
    
    # Subscribe to zipcode, default is NYC, 10001  
    zip_filter = sys.argv[1] if len(sys.argv) > 1 else "10002"  
    
    #此处设置过滤条件，只有以 zip_filter 开头的消息才会被接收  
    socket.setsockopt(zmq.SUBSCRIBE, zip_filter)  
    
    # Process 5 updates  
    total_temp = 0  
    for update_nbr in range(5):  
        string = socket.recv()  
        print string  
        zipcode, temperature, relhumidity = string.split()  
        total_temp += int(temperature)  
    
    print("Average temperature for zipcode '%s' was %dF" % (  
        zip_filter, total_temp / update_nbr)  
    )  

          我这里做一个简单的总结：

          1、 zmq的程序，也是要分清服务端和客户端的，服务端也是要绑定ip和端口的

          2、 有了第1条，你瞬间觉得这和socket没什么两样么，别急，第2条马上震惊你，如果我们先启动客户端，后启动服务端，那么程序是可以正常运行的，换成socket，就不行，socket只能先启动服务端，后启动客户端

          3、 学习zmq的过程，千万别总想着socket，你能用socket传输文件，但是如果用zmq做同样的事情，那你就错误的使用了zmq，记住，这是一个消息通信库，它自己实现了一些协议，使得我们可以非常轻松的在节点间，进程间，线程间传递消息，如果你对我刚才说的节点间，进程间，线程间传递消息没什么兴趣，说明，你平日里写的程序都是单进程，单线程的，只管顺序执行就好了，其他的不用考虑。



          下面，我来分析这两段程序。

          1、 不论是服务端还是客户端，都需要获得zmq上下文

[python] view plain copy
context = zmq.Context()  

          2、 然后哩，我们得获得socket，这个socket不是我们平日里以为的那个socket。zmq里叫socket，我猜可能是为了方便大家学习才这么命名。它的表现，已经远远的超出了我们对以前的那个socket的了解。每一个socket都是有自己的类型的，示例中，服务端的socket的类型是zmq.PUB，客户端的socket的类型是zmq.SUB，pub是发布，sub是订阅。说的通俗点，就是有一个pub节点，可以有多个sub节点，pub节点发出去的消息，如果sub节点没有设置过滤条件，那么就会接收所有的消息，如果有过滤条件，就只接收满足过滤条件的消息。想想看，有没有那么一个时刻，你希望你的程序等待一个命令，收到命令后，你让程序去做一些事情？那么pub与sub模式非常适合这种应用场景。
         3、 设置过滤条件很简单

[python] view plain copy
socket.setsockopt(zmq.SUBSCRIBE, zip_filter)  
          第二个参数就是你期望的过滤条件，只有那些以这个过滤条件开头的消息才会被接收


         最后是自问自答环节

         问题1： 如果想创建多个socket怎么写？

               答： 一个上下文可以创建任意多个socket，完全不受限制

      

         问题2： 明明先启动了客户端，后启动的服务端，为啥有些消息却没有收到呢？

               答： 就算你先启动了客户端，服务端pub出去的一些消息也还是可能没有被收到，因为你启动服务端时，服务端与客户端要建立连接，而这个时候，消息其实已经发出去了，所以你没收到

    

          问题3： 在订阅发布模型中，如果客户端断开连接，或是服务端断开连接会产生什么样的影响

                答： 如果是客户端断开连接，没什么的，就好比一堆人在听收音机，现在离开一个人，收音机继续播放喽。如果是服务端断开了呢，比如程序死掉了，那么请放心，客户端不会发生崩溃，只是阻塞在socket.recv() 这条语句上，更神奇的是，如果你恢复了服务端

           现在，我们修改一下客户端程序

           

[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-13 
订阅模式，如果设置了过滤条件，那么只会接收到以过滤条件开头的消息 
@author: kwsy2015 
'''  
import sys  
import zmq  
import time  
  
#  Socket to talk to server  
context = zmq.Context()  
socket = context.socket(zmq.SUB)  
  
print("Collecting updates from weather server...")  
socket.connect("tcp://localhost:5556")  
  
# Subscribe to zipcode, default is NYC, 10001  
zip_filter = sys.argv[1] if len(sys.argv) > 1 else "10002"  
  
#此处设置过滤条件，只有以 zip_filter 开头的消息才会被接收  
socket.setsockopt(zmq.SUBSCRIBE, zip_filter)  
  
# Process 5 updates  
total_temp = 0  
for update_nbr in range(50):  
    print 'wait recv'  
    string = socket.recv()  
    print 'has recv'  
    time.sleep(1)  
    print string  
    zipcode, temperature, relhumidity = string.split()  
    total_temp += int(temperature)  
  
print("Average temperature for zipcode '%s' was %dF" % (  
      zip_filter, total_temp / update_nbr)  
)  

         服务端
 

[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-13 
服务端，发布模式 
@author: kwsy2015 
'''  
import zmq  
import time  
from random import randrange  
  
context = zmq.Context()  
socket = context.socket(zmq.PUB)  
socket.bind("tcp://*:5556")  
  
while True:  
    zipcode = randrange(1, 100000)  
    temperature = randrange(-80, 135)  
    relhumidity = randrange(10, 60)  
  
    socket.send("%i %i %i" % (10002,temperature , relhumidity))  

        服务端和客户端都启动，这时候，客户端收到一条消息后会睡一秒钟，但是服务端却是一刻不停的在发送消息，那么问题来了，一个发的快，一个收的慢，那么这时候把服务端停掉会怎样呢？

         实际的效果是，服务端停下来了，客户端依然在接收消息，因为有一些消息被缓存起来了，虽然服务端不再发送了，客户端却依然可以接收得到，但这种接收，只是从之前接收的缓冲区里取数据。

         现在，我们在服务端最后加上一条语句，time.sleep(2)，这样，服务端发送一条消息后，睡两秒钟，发的慢，收的快了，我们再次启动服务端和客户端，当客户端收到一些消息后，关掉服务端，这次，客户端很快就停止接收了，因为发的慢，所以缓冲区里没有数据，现在，我们再次启动服务端，你会发现，客户端又开始接收数据了，哈哈，神奇吧！