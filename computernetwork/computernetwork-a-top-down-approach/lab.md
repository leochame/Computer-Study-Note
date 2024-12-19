# Lab

## 第二章：

1.创建流式套接字

```python
# import socket module
from socket import *

# AP_INET：IPv4 SOCKET_STREAM 流式socket
serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
########## Begin ##########
serverSocket.bind(('127.0.0.1',6789))
#设置连接队列大小为1，并使套接字处于监听状态。
serverSocket.listen(1);
########## End ##########
print(serverSocket)
serverSocket.close()
```

2.服务端获取连接请求

当服务器中的套接字serverSocket监听到了连接请求之后，内核和客户建立连接，并将连接放入连接队列中。典型的服务器程序是可以同时服务多个客户端的，当有客户端发起连接时，服务器就调用 accept()返回并接收这个连接，如果有大量客户端发起请求，服务器来不及处理，还没有 accept 的客户端就处于连接等待状态。如果服务器调用 accept() 时还没有客户端的连接请求，就阻塞等待直到有客户端连接上来.

```python
#import socket module
from socket import *
serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
serverSocket.bind(("127.0.0.1",6789))
serverSocket.listen(1)

#while True:
#Establish the connection
print('开始WEB服务...')

try:
    ########## Begin ##########
    '''
      返回值： 
      connectionSocket 客户端连接套接字,是 accept()接收到一个客户端连接请求后返回的一个新的套接字，
      它代表了服务端和客户端的连接。后面可以用于读取数据以及关闭连接。
     addr 连接的客户端地址
    '''
    connectionSocket,addr = serverSocket.accept()

    '''
　　　　　功能 ： 接收对应客户端消息
　　　　　参数 ： 一次最多接收多少字节
　　　　　返回值 ： 接收到的内容
　　　* 如果没有消息则会阻塞等待
    '''
    message = connectionSocket.recvmsg(1024)

    ########## End ##########
    print(addr[0])
    print(message)
    connectionSocket.close()
except IOError:

        connectionSocket.close()
serverSocket.close()
```



3. 服务端读取请求文件内容

```python
#import socket module
from socket import *

serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
serverSocket.bind(("127.0.0.1",6789))
serverSocket.listen(1)

#while True:
print('开始WEB服务...')

try:
    connectionSocket, addr = serverSocket.accept()
    message = connectionSocket.recv(1024) # 获取客户发送的报文
        
    #读取文件内容
    ######### Begin #########
    
    fileName = message.split()[1]
    fo = open(fileName[1:])
    outputdata = fo.read()


    ######### End #########
    print(outputdata)
    connectionSocket.close()
    fo.close()
except IOError:
        
    connectionSocket.close()
serverSocket.close()
```

4.web服务器响应请求头部信息

WEB服务器接收到客户端请求后，就会响应该请求。

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

```python
#import socket module
from socket import *

serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
serverSocket.bind(("127.0.0.1",6789))
serverSocket.listen(1)

#while True:
print('开始WEB服务...')

try:
        connectionSocket, addr = serverSocket.accept()
        message = connectionSocket.recv(1024) # 获取客户发送的报文
        
        #读取文件内容
        filename = message.split()[1]       
        f = open(filename[1:])
        outputdata = f.read();
        
        #发送响应的头部信息
        header = ' HTTP/1.1 200 OK\nConnection: close\nContent-Type: text/html\nContent-Length: %d\n\n' % (len(outputdata))
         #########Begin#########
        
        connectionSocket.send(header.encode())
        #########End#########
       
        connectionSocket.close()
except IOError:
        
        connectionSocket.close()
serverSocket.close()
```



5.服务端响应请求的正文

通过循环的方式将数组内容利用套接字发送出去

```python
#import socket module
from socket import *

serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
serverSocket.bind(("127.0.0.1",6789))
serverSocket.listen(1)

#while True:
print('开始WEB服务...')
try:
        connectionSocket, addr = serverSocket.accept()
        message = connectionSocket.recv(1024) # 获取客户发送的报文
        
        #读取文件内容
        filename = message.split()[1]       
        f = open(filename[1:])
        outputdata = f.read();
        
        #向套接字发送头部信息
        header = ' HTTP/1.1 200 OK\nConnection: close\nContent-Type: text/html\nContent-Length: %d\n\n' % (len(outputdata))
        connectionSocket.send(header.encode())
        
        #发送文件内容
        #########Begin#########
        for i in range(0,len(outputdata)):
                connectionSocket.send(outputdata[i].encode())

        #########End######### 
        
        #关闭连接
        connectionSocket.close()
except IOError:             #异常处理
        #发送文件未找到的消息
        
        #关闭连接
        connectionSocket.close()
#关闭套接字
serverSocket.close()
```

6. 服务端异常（文件不存在异常）处理

```python
 #import socket module
from socket import *

serverSocket = socket(AF_INET, SOCK_STREAM) 
#Prepare a sever socket 
serverSocket.bind(("127.0.0.1",6789))
serverSocket.listen(1)

#while True:
print('开始WEB服务...')
try:
		connectionSocket, addr = serverSocket.accept()
		message = connectionSocket.recv(1024) # 获取客户发送的报文
        
        #读取文件内容
        filename = message.split()[1]       
		f = open(filename[1:])
		outputdata = f.read();
		
        #向套接字发送头部信息
		header = ' HTTP/1.1 200 OK\nConnection: close\nContent-Type: text/html\nContent-Length: %d\n\n' % (len(outputdata))
		connectionSocket.send(header.encode())

		#S发送请求文件的内容
		for i in range(0, len(outputdata)):
			connectionSocket.send(outputdata[i].encode())
		
        #关闭连接
        connectionSocket.close()
except IOError:             #异常处理
		#发送文件未找到的消息
		header = ' HTTP/1.1 404 not Found'
		#########Begin#########

        connectionSocket.send(header.encode())
        
        #########End#########
		#关闭连接
		connectionSocket.close()
#关闭套接字
serverSocket.close()
```





