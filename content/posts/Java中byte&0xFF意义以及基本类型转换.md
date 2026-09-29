---
title: java基本类型转换
date: 2025-08-13 12:00:00
---

# 引言：

读JDK17源码中，发现Bits类中有这样一段代码：

```java
    static boolean getBoolean(byte[] b, int off) {
        return b[off] != 0;
    }

    static char getChar(byte[] b, int off) {
        return (char) ((b[off + 1] & 0xFF) +
                       (b[off] << 8));
    }

    static short getShort(byte[] b, int off) {
        return (short) ((b[off + 1] & 0xFF) +
                        (b[off] << 8));
    }

    static int getInt(byte[] b, int off) {
        return ((b[off + 3] & 0xFF)      ) +
               ((b[off + 2] & 0xFF) <<  8) +
               ((b[off + 1] & 0xFF) << 16) +
               ((b[off    ]       ) << 24);
    }

```

这几个方法作用都是从byte\[\]数组的偏移量`off`中取出对应类型的基础数据类型，除了最高位字节其他字节都`&0xFF`后按照大端模式进行移位操作，但是`byte`长度为一个字节，`&0xFF`后仍然为其本身，其中的意义是什么呢？

# byte&0xFF意义

写了个Demo进行对比：

```java
static int getInt(byte[] b, int off) {
    return ((b[off + 3] & 0xFF)) +
            ((b[off + 2] & 0xFF) << 8) +
            ((b[off + 1] & 0xFF) << 16) +
            ((b[off]) << 24);
}

static int getInt2(byte[] b, int off) {
    return ((b[off + 3]) << 0) +
            ((b[off + 2]) << 8) +
            ((b[off + 1]) << 16) +
            ((b[off + 0]) << 24);
}

public static void main(String[] args) {
    byte[] bs = new byte[]{1, 2, 3, -55};
    System.out.println(getInt(bs, 0));
    System.out.println(getInt2(bs, 0));
}

```

```shell
16909257
16909001

```

输出结果不一样😂再去jdk22看，这个类被移除掉了，替换成了native方法VarHandle.get  
简化逻辑为下面这段代码：

```java
public static void main(String[] args) {
    byte b = -55;
    int i1 = b & 0xFF;
    int i2 = (int) b;
    System.out.println(i1);
    System.out.println(i2);
}

```

输出结果:

```shell
201
-55

```

这块结果不一样，涉及到位操作我们需要将`-55`转换成机器码

    原码 1011 0111
    反码 1100 1000
    补码 1100 1001


`int i2 = (int) b`强制转换byte为int时，负数前面会自动补1，正数补0

    补码 1111 1111 1111 1111 1111 1111 1100 1001
    反码 1111 1111 1111 1111 1111 1111 1100 1000
    原码 1000 0000 0000 0000 0000 0000 0011 0111


由原码可知强制转换后仍为-55  
`int i1 = b & 0xFF`首先二元运算&会将结果提升为int，-55的补码`1111 1111 1111 1111 1111 1111 1100 1001`位与0xFF，结果为`0000 0000 0000 0000 0000 0000 1100 1001`，转换成值为201

byte默认为singed，范围\[-127,128\]由此可知此处的byte&0xFF其实是去符号取unsinged的操作。

# 基本类型转换

如果在二元操作中，不存在double，float，long的话，那么byte、short、char类型都会被转化为int类型，具体表现为小容器向大容器转换
 
graph LR  
byte(byte)–>short(short)–>int(int)–>long(long)–>float(float)–>double(double)  
char(char)–>int  

|              |          |      |
| ------------ | -------- | ---- |
| 基本数据类型 | 字节长度 | 位数 |
| byte         | 1        | 8    |
| short        | 2        | 16   |
| char         | 2        | 16   |
| int          | 4        | 32   |
| long         | 8        | 64   |
| float        | 4        | 32   |
| double       | 8        | 64   |

其中需要明确的是float虽然4个字节，但是表示的范围比long大，只是long转float可能会有精度丢失的问题

float的32位，可以分为三段来看，第一段占第一位，表示正负；第二段第2-9位表示指数；第三段第10-32位表示尾数

1.  在10进制中123.456可以被表示成1.23456 × 102 ，在二进制中10100.110可以被表示成1.0100110 × 24
    
2.  浮点数有精度，超过精度的数无法被表示，如0.0000000000000001就是精确到小数点后x位。
    
3.  IEEE浮点数有三个部分组成：符号位（S），指数位（E），尾部（M），任何数都能以二进制表示以为S_1.M_2E