本篇博客将介绍zmq应答模式，所谓应答模式，就是一问一答，规则有这么几条

        1、 必须先提问，后回答

        2、 对于一个提问，只能回答一次

        3、 在没有收到回答前不能再次提问



        上代码，服务端：

[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-10 
回复请求 
@author: kwsy2015 
'''  
import zmq  
import time  
  
context = zmq.Context()  
socket = context.socket(zmq.REP)  
socket.bind('tcp://*:5555')  
  
while True:  
    message = socket.recv()  
    print 'received request: ' ,message  
      
    time.sleep(1)  
    socket.send('World')  

        客户端：
[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-10 
你无法连续向服务器发送数据，必须发送一次，接收一次 
REQ和REP模式中，客户端必须先发起请求 
@author: kwsy2015 
'''  
import zmq  
  
context = zmq.Context()  
print 'connect to hello world server'  
socket =  context.socket(zmq.REQ)  
socket.connect('tcp://localhost:5555')  
  
for request in range(1,10):  
    print 'send ',request,'...'  
    socket.send('hello')  
    message = socket.recv()  
    print 'received reply ',request,'[',message,']'  

        为了满足条件1，所以，客户端必须先接收问题，客户端总是必须先提问，客户端提问后，必须等待回答，在收到回答前如果又发出提问，那么会报错。
        zmq.REP是应答方，zmq.REQ是提问方，显然，我们可以有多个提问方，但只能有一个提问方，这就好比正在学生正在上课，只能有一个老师来回答问题，可以有多个学生提问，老师只能一个问题一个问题的来回答，所以喽，必须是学生先提问，老师来回答，然后回答下一个学生的提问。



       自问自答环节：

       问题1： 应答方和提问方谁先启动呢？（服务端和客户端谁先启动呢？）

            答： 谁先启动都可以，和pub/sub模式一样

       问题2： 如果服务端断掉或者客户端断掉会产生怎样的影响？

            答：  如果是客户端断掉，对服务端没有任何影响，如果客户端随后又重新启动，那么两方继续一问一答，但是如果是服务端断掉了，就可能会产生一些问题，这要看服务端是在什么情况下断掉的，如果服务端收是在回答完问题后断掉的，那么没影响，重启服务端后，双发继续一问一答，但如果服务端是在收到问题后断掉了，还没来得及回答问题，这就有问题了，那个提问的客户端迟迟得不到答案，就会一直等待答案，因此不会再发送新的提问，服务端重启后，客户端迟迟不发问题，所以也就一直等待提问。

       问题3： 看代码，服务端根本就没去区分提问者是谁，如果有两个提问题的人，如何保证服务端的答案准确的发给那个提问的客户端呢？

            答：  关于这一点，大家不必担心，zmq的内部机制已经做了保证，提问者必然只收到属于自己的答案，我们不必去操心zmq是怎么做到的，你只需关于业务本身即可。

             现在，我们把服务端代码做修改

      

[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-15 
 
@author: kwsy2015 
'''  
import zmq  
import time  
  
context = zmq.Context()  
socket = context.socket(zmq.REP)  
socket.bind('tcp://*:5555')  
  
while True:  
    message = socket.recv()  
    print 'received request: ' ,message  
    time.sleep(1)  
    if message == 'hello':  
        socket.send('World')  
    else:  
        socket.send('success')  
           写两个客户端
           客户端1：

[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-15 
 
@author: kwsy2015 
'''  
import zmq  
  
context = zmq.Context()  
print 'connect to hello world server'  
socket =  context.socket(zmq.REQ)  
socket.connect('tcp://localhost:5555')  
  
for request in range(1,10):  
    print 'send ',request,'...'  
    socket.send('hello')  
    message = socket.recv()  
    print 'received reply ',request,'[',message,']'  
            客户端2：
[python] view plain copy
#coding=utf-8  
''''' 
Created on 2015-10-15 
 
@author: kwsy2015 
'''  
import zmq  
  
context = zmq.Context()  
print 'connect to hello world server'  
socket =  context.socket(zmq.REQ)  
socket.connect('tcp://localhost:5555')  
  
for request in range(1,10):  
    print 'send ',request,'...'  
    socket.send('ok')  
    message = socket.recv()  
    print 'received reply ',request,'[',message,']'  

          实际的运行结果如图：


             不难看出，每个客户端都收到了只属于自己的答案