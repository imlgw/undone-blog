---
title:
  Socket网络编程
tags:
  [Socket,网络编程]
categories:
	[Socket]
img:http://static.imlgw.top///20190429/O1leMFc6AcR4.png?imageslim
---

## 起因

在写Socket相关的内容遇到的，在传输一些基本数据类型的时候需要和**byte[]**互相转换，虽然可以用**ByteBuffer**代替但是我们还是要搞清楚为啥要这样写，其实**ByteBuffer**里面也是这样实现的

![mark](http://static.imlgw.top/image/20190712/dky6KpqsDlSp.png?imageslim)

- **int类型**

```java

/**
 * @author imlgw.top
 * @date 2019/7/11 13:23
 */
public class ByteTools {
    public static byte[] int2byte(int a) {
        //1.移位是为了将4个字节分开 分为单独的一个个字节
        //2. & 0xff是为了保证二进制的一致性，因为java(计算机)中存放的数据是采用的补码
        //这里右移会在高位补1,保证了十进制的一致,这样二进制的数据就发生了变化
        // 所以为了保证二进制数据的一致.&上0xff就可以将高位都置为0而不影响低8位
        // 其实也可以理解为截取了低8位作为一个无符号byte
        /*return new byte[]{
                (byte) ((a >> 24) & 0xff),
                (byte) ((a >> 16) & 0xff),
                (byte) ((a >> 8) & 0xff),
                (byte) ((a) & 0xff)
        };*/
        //其实这样也可以,无符号右移,高位补0不管正负
        return new byte[]{
                (byte) (a >>> 24),
                (byte) (a >>> 16),
                (byte) (a >>> 8),
                (byte) (a)
        };
    }
    
    public static int byte2int(byte[] a) {
        //&0xff 变为无符号的int
        return a[3] & 0xff | (a[2] & 0xff) << 8 | (a[1] & 0xff) << 16 | (a[0] & 0xff) << 24;
    }

    public static void main(String[] args) {
        byte a=-127;
        int c=-127;
        // (byte)10000001-->(int) 11111111 11111111 11111111 10000001
        System.out.println(Integer.toBinaryString(a)+":"+a);
        //0xff   1111 1111
        int b=a&0xff;
        System.out.println(Integer.toBinaryString(b)+":"+b);
        System.out.println("---------");
        System.out.println(Integer.toBinaryString(c));
        System.out.println((byte)c);
        System.out.println(c&0xff);
    }
}
```

其实移位的地方好理解，一个**int**4个字节 一个字节8位，把每个字节的8位都对应的移动到最后一个字节，然后取出来放到一个**byte数组**中，

`&`运算是二进制数据的计算方式, 两个操作位都为1，结果才为1，否则结果为0. 



其表示的 int 值为 138, 可见将 byte 类型的 -118 与 `0XFF` 进行与运算后值由 -118 变成了 int 类型的 138, 其中低8位和byte的-118完全一致.

如果b[n]为0或者正数, 其原码, 反码, 补码都是一样的, 和 `0XFF` 进行与运算后的结果不变.

byte 的取值范围为 [-128, 127], 根据上面的转换过程我们可以发现, 只有当 byte 的值为负数的时候才有必要和`0XFF` 进行与运算, 为0或者为正数的时候byte的值和对应int的值完全一致.

