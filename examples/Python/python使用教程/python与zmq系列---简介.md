本系列将为大家介绍zmq的使用，编程语言使用python,版本2.7，zmq版本是2.1，win8.1操作系统。

        先简单的介绍一下zmq吧，它是一个云时代极速消息通信库，励志要写入linux内核。这东西简单实用，完成同样的功能，如果是用socket，那恐怕要写出一大堆的代码，但用zmq，只需要简单的几行代码就可以了。

         python环境下的zmq安装非常简单，安装包地址：（奇怪，安装包地址不见了）

在此特别声明，本系列所用到的python代码，并非本人原创，而是在https://github.com/imatix/zguide/tree/master/examples/Python 上获取，写此系列的博客，旨在分析这些示例代码，为有意学习zmq的同学开辟一个快速了解学习的通道。


         如果你已经安装好开发环境，那么，我们先来了解如何获知自己zmq的版本和pyzmq的版本，这两个版本必须保持一致才行，好在，我提供的安装包把这两个东东一起安装了，所以你不必为此操心费神。

---
    #coding=utf-8  
    ''''' 
    Created on 2015-10-13 
    获得zmq的版本 
    @author: kwsy2015 
    '''  
    import zmq  
    
    print("Current libzmq version is %s" % zmq.zmq_version())  
    print("Current  pyzmq version is %s" % zmq.pyzmq_version())

