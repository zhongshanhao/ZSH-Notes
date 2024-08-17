socket是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用以实现进程在网络中通信。通过socket提供的接口，我们可以很方便的使用TCP/IP协议进行网络间进程的通信。

<img src="D:\_temp\网络图片\aHR0cDovL2ltYWdlcy5jbml0YmxvZy5jb20vYmxvZy8zNDkyMTcvMjAxMzEyLzA1MjI1NzIzLTJmZmE4OWFhZDkxZjQ2MDk5YWZhNTMwZWY4NjYwYjIwLmpwZw.jpg" alt="aHR0cDovL2ltYWdlcy5jbml0YmxvZy5jb20vYmxvZy8zNDkyMTcvMjAxMzEyLzA1MjI1NzIzLTJmZmE4OWFhZDkxZjQ2MDk5YWZhNTMwZWY4NjYwYjIwLmpwZw" style="zoom:80%;" />

套接字Socket=（IP地址：端口号），套接字的表示方法是点分十进制的lP地址后面写上端口号，中间用冒号或逗号隔开。

每一个传输层连接唯一地被通信两端的两个端点（即两个套接字）所确定。例如：如果IP地址是210.37.145.1，而端口号是23，那么得到套接字就是(210.37.145.1:23)  。

```
// 两个socket确定一个传输层的连接
// 因此一台主机最大连接数不是端口数，只要socket其中一个参数不相同，就可以是一个不同的连接

socket(ip:端口)   -----   socket(ip:端口)  
```

端口是一个逻辑上的概念，不是物理上真正有一个端口，实际网络传输数据走的都是IO，在linux中，一个进程创建了一个socket套接字，就相当于创建了一个文件，用这个文件表示抽象的socket

[Linux下Socket通信（TCP实现）](https://segmentfault.com/a/1190000010838127)

socket建立连接过程

<img src="D:\_temp\网络图片\20180301101458901.png" alt="20180301101458901" style="zoom:80%;" />