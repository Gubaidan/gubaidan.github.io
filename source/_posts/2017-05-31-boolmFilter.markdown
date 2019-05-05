---
layout: post
title: "BoolmFilter算法解析"
date: 2017-05-23 16:00
author: "Gubaidan"
header-img: "/Header/lake.jpg"
cdn: 'header-on'
tags:
	- 算法
---
# 介绍

BloomFilter（布隆过滤器）是一种可以高效地判断元素是否在某个集合中的算法。

在很多日常场景中，都大量存在着布隆过滤器的应用。例如：检查单词是否拼写正确、网络爬虫的URL去重、黑名单检验，微博中昵称不能重复的检测。Google Chrome浏览器使用BloomFilter来判断一个网站是否为恶意网站。

对于以上场景，可能很多人会说，用HashSet甚至简单的链表、数组做存储，然后判断是否存在不就可以了吗？

当然，对于少量数据来说，HashSet是很好的选择。但是对于海量数据来说，BloomFilter相比于其他数据结构在空间效率和时间效率方面都有着明显的优势。

但是，布隆过滤器具有一定的误判率，有可能会将本不存在的元素判定为存在。因此，对于那些需要“零错误”的应用场景，布隆过滤器将不太适用。具体的原因将会在第二部分中介绍。  

在本文的第二部分，本文将会介绍BloomFilter的基本算法思想；第三部分将会基于Google开源库Guava来讲解BloomFilter的具体实现；在第四部分中，将会介绍一些开源的BloomFilter的扩展，以解决目前BloomFilter的不足。

# 算法讲述

布隆过滤器是基于Hash来实现的，在学习BloomFilter之前，也需要对Hash的原理有基本的了解。个人认为，BloomFilter的总体思想实际上和bitmap很像，但是比bitmap更节省空间，误判率也更低。

BloomFilter的整体思想并不复杂，主要是使用k个Hash函数将元素映射到位向量的k个位置上面，并将这k个位置全部置为1。当查找某元素是否存在时，查找该元素所对应的k位是否全部为1即可说明该元素是否存在。

# 算法流程

**BloomFilter的整体算法流程可总结为如下步骤：**



1. BloomFilter初始化为m位长度的位向量，每一位均初始化为![bf1](http://epoch-night.oss-cn-hangzhou.aliyuncs.com/PostImg/bf1.jpg)

2. 使用k个相互独立的Hash函数，每个Hash函数将元素映射到{1..m}的范围内，并将对应的位置为1。![bf2](http://epoch-night.oss-cn-hangzhou.aliyuncs.com/PostImg/bf2.jpg)如上图所示，元素x分别被三个Hash函数映射到了三个位置8、1、14，并将这三个位置从0变为1。

3. 若检查一个元素y是否存在，首先第一步使用k个Hash函数将元素y映射到k位。分别检测每一位是否为0。若某一位为0，则元素y一定不存在，若全部为1，则有可能存在。

**空间复杂度**

BloomFilter 使用位向量来表示元素，而不存储本身，这样极大压缩了元素的存储空间。其空间复杂度为O(m)，m是位向量的长度。而m与插入总数量n的关系如公式（1）所示：

$$
\begin{equation} \label{euler}  m=-{\frac {n\ln p}{(\ln 2)^{2}}}   \end{equation}
$$
我们可以利用这个公式来算一下需要抓取100万个URL时BloomFilter所占据的空间。

假设要求误判率为1%，因此该公式可转化为m=9.6∗nm=9.6∗n。故此时BloomFilter位向量的大小为100w∗9.6=960wbit100w∗9.6=960wbit，约1.1M内存空间。
只需要1.1M的内存空间，就可满足100万个url的去重需求，这个空间复杂度之低不可谓不惊人。
实际上，哪怕是1亿个URL，也仅需100M左右的内存空间即可满足BloomFilter的空间需求，这对于绝大部分爬虫的体量来说，是完全可行的。

**时间复杂度**
时间复杂度方面 BloomFilter的时间复杂度仅与Hash函数的个数k有关，即O(k)

# 误判率

为什么说，在查找元素时，即使某个元素所映射的k位全部位1，依然无法确定它一定存在？

这是因为当插入的元素很多的情况下，某个元素即使之前不存在，但是它所映射的k位已经被之前其他的元素置为1了，这样就会出现误判，BloomFilter会认为它已经存在了。但是这个概率是非常小的。根据维基百科的推导公式来说，误判率的大小p满足以下公式（2）

$$
\begin{equation}   \ln p = -{\frac {m}{n}}\left(\ln 2\right)^{2} \label{eq:p} \end{equation}
$$
其中m为位向量的长度，n为要插入元素的总数。当误判率为1%时,mn=9.6mn=9.6即每个元素仅需要9.6个字节存储即可

Hash函数的个数k，与误判率大小p的关系为公式（3） 所示

$$
\begin{equation}   k = {-{\ln p} \over {\ln 2}} \label{eq:k1} \end{equation}
$$
当误判率大小为0.1时，k为3。当误判率大小为0.01时，k为7

与位向量的长度m和插入元素的总数n的关系为公式（4）

$$
\begin{equation}   k = {\frac {m}{n}}\ln 2 \label{eq:k2} \end{equation}
$$


# 缺点

BloomFilter 由于并不存储元素，而是用位的01来表示元素是否存在，并且很有可能一个位时被多个元素同时使用。所以无法通过将某元素对应的位置为0来删除元素。

幸运的是，目前学术界和工业界都有很多方法扩展已解决以上问题。具体可以参考本文第三部分 **BloomFilter的优化和扩展**

# Guava’s BloomFilter源码剖析

Guava引入了一个叫做`Funnel`的类，Funnel类定义了如何把一个具体的对象类型分解为原生字段值，从而将值分解为Byte以供后面BloomFilter进行hash运算。通过使用这个类，我们可以自己定义一个属于自己类的Funnel。如下代码

```java
public enum StringFunnel implements Funnel<String> {    
    INSTANCE;   
    @Override    
    public void funnel(String from, PrimitiveSink into) {        		
    	into.putString(from,Charset.defaultCharset());    
    }
}
```



此外，Guava预定义了一些原生类型的Funnel，如String、Long、Integer。具体代码可以在[这里看到](https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/Funnels.java)。当我们的BloomFilter存储的是这些原生类型时，不用再额外自行写Funnel，直接使用Guava预定义的这些即可。

以下是整个代码调用的流程
                                                               

```java
public class BloomFilterSamle {                                                                                             
public static void main(String[] args) {    
	// 创建一个BloomFilter，其预计插入的个数为10，误判率大约为0.01
    BloomFilter<String> bloomFilter = BloomFilter.create(StringFunnel.INSTANCE, 10, 0.01);                              
    // 查询www.google.com是否存在                                                                                                                   
    System.out.println(bloomFilter.mightContain("www.google.com"));
     // 将www.google.com对象放入BloomFilter中
    bloomFilter.put("www.google.com");  
    // 再次查询www.google.com是否存在
    System.out.println(bloomFilter.mightContain("www.google.com"));                                                     
	}    
} 
```



# 源码解读

在例子中，我们通过调用`BloomFilter.create`工厂方法来生成一个BloomFilter

```java
<T> BloomFilter<T> create(Funnel<? super T> funnel, 
                          long expectedInsertions,  //预期会插入多少元素
                          double fpp, //自定义误判率
                          Strategy strategy //Hash策略 
                          ) {  
    if (expectedInsertions == 0) {
      expectedInsertions = 1;
    }
    ...
    //根据插入的数量和误判率来得出位向量应有的长度,这里使用的算法就是公式 2
    long numBits = optimalNumOfBits(expectedInsertions, fpp);
    //根据插入的数量和位向量的长度来得出应该用多少个Hash函数，这里使用的算法是公式 4
    int numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, numBits);
    return new BloomFilter<T>(new BitArray(numBits), numHashFunctions, funnel, strategy);
}
```


整个创建流程非常清晰，如果看懂了本文的算法描述部分，应该不难理解该段代码。整个初始化流程的目的主要有两个：根据参数计算出位向量的长度以及Hash函数的个数

* 根据预期插入的数量expectedInsertions和自定义的误判率fpp来得到位向量的长度numBits，其中optimalNumOfBits的实现如下

    ```java
    // 根据插入的数量和误判率来得出位向量应有的长度
    static long optimalNumOfBits(long n, double p) {
        if (p == 0) {
          p = Double.MIN_VALUE;
        }
    	return (long) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }
    ```

    可以很明显看出，optimalNumOfBits的源码，其实就是对公式(1)的实现

* 根据插入的数量expectedInsertions和位向量的长度numBits来得出应该用多少个Hash函数，其中optimalNumOfHashFunctions的实现如下

     ```java
     static int optimalNumOfHashFunctions(long n, long m) {
         // (m / n) * log(2), but avoid truncation due to division!
         return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
       }
     ```

     同样，optimalNumOfHashFunctions也对应了我们算法中公式(4) 

* 根据numBits生成BitArray

     BitArray是Guava中位向量的表示，具体的实现细节
# 位向量的表示

```java
 static final class BitArray {
    final long[] data;
    long bitCount;

    BitArray(long bits) {
      //对于长度为m的位向量来说，对应的long数组的长度应为m/64向上取整。
      this(new long[Ints.checkedCast(LongMath.divide(bits, 64, RoundingMode.CEILING))]);
    }

    // Used by serialization
    BitArray(long[] data) {
      checkArgument(data.length > 0, "data length is zero!");
      this.data = data;
      long bitCount = 0;
      for (long value : data) {
        bitCount += Long.bitCount(value);
      }
      this.bitCount = bitCount;
    }

    /** Returns true if the bit changed value. */
    boolean set(long index) {
      if (!get(index)) {
        data[(int) (index >>> 6)] |= (1L << index);
        bitCount++;
        return true;
      }
      return false;
    }

    boolean get(long index) {
      return (data[(int) (index >>> 6)] & (1L << index)) != 0;
    }

    /** Number of bits */
    long bitSize() {
      return (long) data.length * Long.SIZE;
    }

    /** Number of set bits (1s) */
    long bitCount() {
      return bitCount;
    }

    BitArray copy() {
      return new BitArray(data.clone());
    }

    /** Combines the two BitArrays using bitwise OR. */
    void putAll(BitArray array) {
      checkArgument(
          data.length == array.data.length,
          "BitArrays must be of equal length (%s != %s)",
          data.length,
          array.data.length);
      bitCount = 0;
      for (int i = 0; i < data.length; i++) {
        data[i] |= array.data[i];
        bitCount += Long.bitCount(data[i]);
      }
    }

    @Override
    public boolean equals(Object o) {
      if (o instanceof BitArray) {
        BitArray bitArray = (BitArray) o;
        return Arrays.equals(data, bitArray.data);
      }
      return false;
    }

    @Override
    public int hashCode() {
      return Arrays.hashCode(data);
    }
  }
```

在BitArray中，使用long数组来表示位向量，一个数组元素对应位向量的64位，所以对于长度为m的位向量来说，对应的long数组的长度应为m/64向上取整。

即

```java
    new long[Ints.checkedCast(LongMath.divide(bits, 64, RoundingMode.CEILING))]
```

在BloomFilter算法讲解部分，我们可以看到，对于位向量的常用操作主要有两个，将位向量某一位置为1以及查看位向量某一位是否为1。分别对应源码中得set操作和get操作 本文只讲下get方法的源码部分，set方法与get方法类似，不再累述。

get方法大致可以分为两部分

* data[(int) (index >>> 6)] 定位到元素

  上面讲到long数组的每一个元素都包含位向量其中的64位，如果想要找出某个位的bit，那么首先第一步就是定位到该bit所在的元素编号。我们一般的做法是`index/64`。 而源码中使用了`index >>> 6`,逻辑右移6位，$2^6 = 64$,其效果与除以64相同。采用位运算的速度比普通的除法要快很多。

* … & (1L << index) 获取位的状态

  源码中直接将要查看的bit以及同一数组元素块的64位bits一起取出，将1L左移index位后求且运算，最终即可得出该位的值。

# BloomFilter的优化和扩展

上文提到布隆过滤器无法支持元素的删除操作,Counting BloomFilter通过存储位元素每一位的置为1的数量，使得布隆过滤器可以支持删除操作。 但是这样会数倍地增加布隆过滤器的存储空间。





# 参考

> [维基百科](<https://en.wikipedia.org/wiki/Bloom_filter>)

> [Dalhousie University](https://www.cs.dal.ca/research/techreports/cs-2002-10)

> [Guava笔记](http://leadtoit.iteye.com/blog/1961751)

> [Counting Bloom Filter](https://blog.csdn.net/jiaomeng/article/details/1498283)

> [http://ifeve.com/google-guava-hashing/](http://ifeve.com/google-guava-hashing/)





