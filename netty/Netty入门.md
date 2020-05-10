# Netty入门

IO模型就是说用什么样的通道进行数据的发送和接收,Java共支持3种网络编程IO模式:BIO,NIO,AIO

BIO(BlockingIO)

同步阻塞模型,一个客户端链接对应一个处理线程

缺点:

1. IO代码里read操作是阻塞操作,如果链接不做数据读写操作会导致线程阻塞,浪费资源
2. 如果线程很多,会导致服务器线程太多,压力太大.

应用场景:

BIO方式使用于链接数目笔记哦啊小且固定的架构,这种方式对服务器资源要求比较高,但程序简单易理解.

![image-20200510155103833](/Users/sjy/Library/Application Support/typora-user-images/image-20200510155103833.png)

BIO代码示例:

```java
    // 服务端代码
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9000);
        while (true) {
            System.out.println("等待连接。。");
            //阻塞方法
            Socket socket = serverSocket.accept();
            System.out.println("有客户端连接了。。");
            // 有请求,就启动线程
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        handler(socket);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
            //handler(socket);
        }
    }

    // 多线程处理业务
    private static void handler(Socket socket) throws IOException {
        System.out.println("thread id = " + Thread.currentThread().getId());
        byte[] bytes = new byte[1024];

        System.out.println("准备read。。");
        //接收客户端的数据，阻塞方法，没有数据可读时就阻塞
        int read = socket.getInputStream().read(bytes);
        System.out.println("read完毕。。");
        if (read != -1) {
            System.out.println("接收到客户端的数据：" + new String(bytes, 0, read));
            System.out.println("thread id = " + Thread.currentThread().getId());
        }
        socket.getOutputStream().write("HelloClient".getBytes());
        socket.getOutputStream().flush();
    }
```

```java
    // 客户端代码
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 9000);
        //向服务端发送数据
        socket.getOutputStream().write("HelloServer".getBytes());
        socket.getOutputStream().flush();
        System.out.println("向服务端发送数据结束");
        byte[] bytes = new byte[1024];
        //接收服务端回传的数据
        socket.getInputStream().read(bytes);
        System.out.println("接收到服务端的数据：" + new String(bytes));
        socket.close();
    }
```

**NIO(Non Blocking IO)**

同步非阻塞,服务器实现模式为一个线程可以处理多个请求(链接),客户端发送的链接请求都会注册到**多路复用器selector**上,多路复用器轮询到链接有IO请求就进行处理.

I/O多路复用底层一般用的Linux API(select,poll,epoll)来实现,他们的区别如下表:

|              | select                                  | poll                                    | epoll(jdk1.5以上)                                            |
| ------------ | --------------------------------------- | --------------------------------------- | ------------------------------------------------------------ |
| **操作方式** | 遍历                                    | 遍历                                    | 回调                                                         |
| **底层实现** | 数组                                    | 链表                                    | 哈希表                                                       |
| **IO效率**   | 每次调用都进行线性便利,时间复杂度为O(n) | 每次调用都进行线性便利,时间复杂度为O(n) | 事件通知方式,每当有IO事件就绪,系统注册的回调函数就会被调用,时间复杂度O(1) |
| **量大链接** | 有上限                                  | 无上限                                  | 无上限                                                       |

应用场景:

NIO方式使用于链接数目多且链接比较短(轻操作)的架构,比如聊天服务器,弹幕系统,服务器间通讯,编程比较复杂,JDK1.4开始支持

![image-20200510160909958](/Users/sjy/Library/Application Support/typora-user-images/image-20200510160909958.png)



![image-20200510161457179](/Users/sjy/Library/Application Support/typora-user-images/image-20200510161457179.png)































