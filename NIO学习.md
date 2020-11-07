---
title:
  NIO学习
tags:
  [Socket]
categories: 
	[网络编程]
cover: http://static.imlgw.top///20190420/zjSkt65Oip3h.png?imageslim
---

### 非阻塞IO

🔸 NIO 全称：Non-blocking I/O

🔸 JDK 1.4 引入的全新的输入输出标准库NIO，也叫New I/O

🔸 在标准的Java代码中提供了高速的，可伸缩的，面向块的，非阻塞色的IO操作

`CSDN` 看到的一个很通俗的解释：

传统的socket IO中，需要为每个连接创建一个线程，当并发的连接数量非常巨大时，线程所占用的栈内存和CPU线程切换的开销将非常巨大。使用NIO，不再需要为每个线程创建单独的线程，可以用一个含有限数量线程的线程池，甚至一个线程来为任意数量的连接服务。由于线程数量小于连接数量，所以每个线程进行IO操作时就不能阻塞，如果阻塞的话，有些连接就得不到处理，NIO提供了这种非阻塞的能力。

小量的线程如何同时为大量连接服务呢，答案就是就绪选择。这就好比到餐厅吃饭，每来一桌客人，都有一个服务员专门为你服务，从你到餐厅到结帐走人，这样方式的好处是服务质量好，一对一的服务，VIP啊，可是缺点也很明显，成本高，如果餐厅生意好，同时来100桌客人，就需要100个服务员，那老板发工资的时候得心痛死了，这就是传统的一个连接一个线程的方式。

老板是什么人啊，精着呢。这老板就得捉摸怎么能用10个服务员同时为100桌客人服务呢，老板就发现，服务员在为客人服务的过程中并不是一直都忙着，客人点完菜，上完菜，吃着的这段时间，服务员就闲下来了，可是这个服务员还是被这桌客人占用着，不能为别的客人服务，用华为领导的话说，就是工作不饱满。那怎么把这段闲着的时间利用起来呢。这餐厅老板就想了一个办法，让一个服务员（前台）专门负责收集客人的需求，登记下来，比如有客人进来了、客人点菜了，客人要结帐了，都先记录下来按顺序排好。每个服务员到这里领一个需求，比如点菜，就拿着菜单帮客人点菜去了。点好菜以后，服务员马上回来，领取下一个需求，继续为别人客人服务去了。这种方式服务质量就不如一对一的服务了，当客人数据很多的时候可能需要等待。但好处也很明显，由于在客人正吃饭着的时候服务员不用闲着了，服务员这个时间内可以为其他客人服务了，原来10个服务员最多同时为10桌客人服务，现在可能为50桌，60客人服务了。

这种服务方式跟传统的区别有两个：

1、增加了一个角色，要有一个专门负责收集客人需求的人。NIO里对应的就是Selector。

2、由阻塞服务方式改为非阻塞服务了，客人吃着的时候服务员不用一直侯在客人旁边了。传统的IO操作，比如read()，当没有数据可读的时候，线程一直阻塞被占用，直到数据到来。NIO中没有数据可读时，read()会立即返回0，线程不会阻塞。

NIO中，客户端创建一个连接后，先要将连接注册到Selector，相当于客人进入餐厅后，告诉前台你要用餐，前台会告诉你你的桌号是几号，然后你就可能到那张桌子坐下了，SelectionKey就是桌号。当某一桌需要服务时，前台就记录哪一桌需要什么服务，比如1号桌要点菜，2号桌要结帐，服务员从前台取一条记录，根据记录提供服务，完了再来取下一条。这样服务的时间就被最有效的利用起来了。

### NIO Family

🔹 Buffer 缓冲区: 用于数据处理的基础单元，客户端发送与接收都需要经过Buffer转发

- 包括: ByteBuffer，CharBuffer，ShortBuffer，IntBuffer，LongBuffer，FloatBuffer，DoubleBuffer
- 写数据时先写到 Buffer --> Channel ，读则相反
- 为NIO块状操作提供基础，数据都按"块"进行传输，一个Buffer代表一"块"数据

🔹 Channel 通道: 类似于流，但不同于 I/O Stream，流具有独占性和单向性，通道则偏向于数据的流通多样性

- 可以同通道中获取数据也可输出数据到通道，按"块"Buffer 进行
- 可并发可异步读写数据，但是读写操作是独占的
- 读数据时读取到Buffer，写数据必须通过Buffer写数据
- 包括: FileChannel，SocketChannel，DatagramChannel等

🔹 Selectors 选择器 : 处理客户端所有事件的分发器

🔹 Charset 字符编码: 加密解密 原生支持的，数据通道级别的数据处理方式，可以用于数据传输级别的数据加密解密操作

### NIO 常见API

![mark](http://static.imlgw.top/image/20190719/kPNTwrb6KKlN.png?imageslim)

#### Selector注册事件

🔸 `SelectionKey.OP_CONNNECT` 连接就绪

🔸 `SelectionKey.OP_ACCEPT` 接受就绪

🔸 `SelectionKey.OP_READ` 读就绪

🔸 `SelectionKey.OP_WRITE` 写就绪

#### Selector使用流程

🔸 `open()` 开启一个选择器，可以给选择器注册需要关注的事件

🔸 `register()` 将一个Channel 注册到选择器，当选择器触发对应关注事件时回调到Channel中，处理相关数据

🔸`select()/selectNow()`当前就绪事件的数量，阻塞方法，可以被wakeUp唤醒，数量为0

🔸 `selectedKeys()` 得到的就绪状态的事件集合

- Interest集合：所有注册的集合
- Ready集合：已经就绪的集合
- Channel通道
- Selector 选择器
- obj 附加值

🔸 `wakeUp()` 唤醒一个处于`select`状态的选择器，就算没有就绪的Channel 也会返回，只不过为空

🔸 `close()` 关闭一个选择器，注销所有的关注的事件

#### Selector注意事项

🔸 注册到选择器的通道必须为非阻塞状态

🔸 `FileChannel` 不能用于`Selector`，因为`FileChannel` 不能切换为非阻塞模式，但是可以使用通道的方式操作文件(以`块状`的方式去和磁盘交互)，`SocketChannel`是可以用于`Selector`的。

### NIO重写服务器

### ByteBuffer

NIO Family里面很重要的一个成员

- **PUT操作**

```java
	final int nextPutIndex() {                          // package-private
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }

    final int nextPutIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }
```

![mark](http://static.imlgw.top/image/20190719/TTrpBiyaRjsl.png?imageslim)

- **GET操作**

```java
    final int nextGetIndex() {                          // package-private
        if (position >= limit)
            throw new BufferUnderflowException();
        return position++;
    }

    final int nextGetIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferUnderflowException();
        int p = position;
        position += nb;
        return p;
    }
```

![mark](http://static.imlgw.top/image/20190719/qzE4xducQ4hm.png?imageslim)

- **clear操作**

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```
![mark](http://static.imlgw.top/image/20190719/cg4YkHcz6jTh.png?imageslim)



- **flip操作**

```java
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

![mark](http://static.imlgw.top/image/20190719/atyDWnqwPTo9.png?imageslim)

- 读写模式

![mark](http://static.imlgw.top/image/20190719/Q0pBAplAU1L1.png?imageslim)

### 小Bug

![mark](http://static.imlgw.top/image/20190719/VKoYWGpkl5uG.png?imageslim)

换行符的问题，readLine实际上是