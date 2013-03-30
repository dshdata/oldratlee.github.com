---
layout: post
title: 【译】Java的通用I/O API
location: Hangzhou
permalink: /474/tech/java/generic-io-api-in-java-and-api-design.html
write-time: 2012-05-11 15:20
tags:
- XX
- YY
---

转自自己在公司发的Blog http://code.alibabatech.com/blog/architecture_1477/generic-io-api-in-java-and-api-design.html

原文：A generic input/output API in Java(by Rickard ?berg)给出了一个通用Java IO API的设计。更重要的一点是，这篇文章给出实现这个API设计本身的步骤和过程，这让API设计实现过程有了条理。文中示范了从 普通简单实现 整理成 正确分解、可以复用、可扩展的API设计 的过程。这个很值得理解和学习！ 

PS: 设计偏向是艺术，一个赏心悦目的设计，尤其是API设计，旁人看来多是妙手偶得的感觉，如果有些章可循真是一件好事。给出***减少艺术的艺术工作量***的方法的人是大师。

上周处理了很多数据搬移，有原始byte形式的，也有String形式的，还有SPI和领域级对象形式。这些活让我觉得，以可伸缩、高性能、正确处理错误的方式把数据从一处搬到另一处，是非常有难度。我要一遍又一遍做一些事，比如从文件中读出String。

这让我有了个想法：一定有个通用模式来处理这些事，可以抽取出来放到库中。“从文本文件中读取lines”这样的事应该只做一遍，然后用在各个需要的场景中。让我们看一个读文件然后写入另一个文件的典型场景，看看能不能从中发现包含了哪几个部分。

```java
1: File source = new File( getClass().getResource( "/iotest.txt" ).getFile() );
1: File destination = File.createTempFile( "test", ".txt" );
1: destination.deleteOnExit();
2: BufferedReader reader = new BufferedReader(new FileReader(source));
3: long count = 0;
2: try
2: {
4:    BufferedWriter writer = new BufferedWriter(new FileWriter(destination));
4:    try
4:    {
2:        String line = null;
2:        while ((line = reader.readLine()) != null)
2:        {
3:            count++;
4:            writer.append( line ).append( '\n' );
2:        }
4:        writer.close();
4:    } catch (IOException e)
4:    {
4:        writer.close();
4:        destination.delete();
4:    }
2: } finally
2: {
2:     reader.close();
2: }
1: System.out.println(count)
```

行左边的数字是我标识的4个部分。

1. 客户代码，初始化了传输，要知道输入和输出的源。
1. 从输入中读的代码。
1. 辅助代码，用于跟踪整个过程。这些代码我希望能够重用，而不管是何种传输的类型。
1. 最后这个部分是接收数据，写数据。这个代码，我要批量读写，可以在第2第4部分修改，改成一次处理多行。

API
=============

一旦明确上面划分的内容，剩下就只是为每个部分整理成一个接口，并保证在各种场景能方便使用。结果如下。 首先要有输入，即Input接口：

```java
public interface Input<T, SenderThrowableType extends Throwable>
{
    <ReceiverThrowableType extends Throwable> void transferTo( Output<T,ReceiverThrowableType> output )
        throws SenderThrowableType, ReceiverThrowableType;
}
```

Input，如Iterables，可以被多次使用，用于初始化一处到另一处的传输。因为我泛化传输的数据类型为T，所以可以是任何类型（byte[]、String、EntityState、MyDomainObject）。为了让发送者和接收者可以抛出各自的异常，接口上把各自己的异常声明成了类型参数。比如：在出错的时，Input抛的可以是SQLException，Output抛的是IOException。异常是强类型的，并且在出错时发送和接收双方都必须知道的，这使的双方做合适的恢复操作，关闭他们打开了的资源。

在接收端的是Output接口：

```java
public interface Output<T, ReceiverThrowableType extends Throwable>
{
    <SenderThrowableType extends Throwable> void receiveFrom(Sender<T, SenderThrowableType> sender)
            throws ReceiverThrowableType, SenderThrowableType;
}
```

当receiveFrom方法被Input调用时（通过调用Input的transferTo方法触发），Output应该打开好了它所需要的资源，然后期望数据从Sender发送过来。Input和Output必须要有类型T，两者对要发送的内容达到一致。后面我们可以看到如何处理不一致的情况。

接下来是Sender接口：

```java
public interface Sender<T, SenderThrowableType extends Throwable>
{
    <ReceiverThrowableType extends Throwable> void sendTo(Receiver<T, ReceiverThrowableType> receiver)
        throws ReceiverThrowableType, SenderThrowableType;
}
```

Output调用sendTo方法，传入一个Receiver，Sender使用这个Receiver来发送一个一个的数据。Sender在这个时候发起传输，把类型数据T传输到Receiver，一次一个。Receiver接口如下： 

```java
public interface Receiver<T, ReceiverThrowableType extends Throwable>
{
    void receive(T item)
        throws ReceiverThrowableType;
}
```

当Receiver从Sender收到数据时，即可以马上写到底层资源中，也可以分批写入。Receiver知道传输什么时候结束（sendTo方法返回了），所以正确写入剩下的分批数据、关闭持有的资源。

这个简单的模式在发送方和接收方各有2个接口，并保持了以可伸缩、高性能和容错的方式传输数据的潜能。

标准化I/O
==============

上文的API定义了数据发送和接收的契约，然后可以制定几个输入输出（I/O）的标准。比如：从文本文件中读取文本行后再写成文本文件。这个操作可以静态方法中，方便的重用。最后，拷贝文本文件可以写成：

```java
File source = ...
File destination = ...
Inputs.text( source ).transferTo( Outputs.text(destination) );
```

一行代码处理了读文件、写文件、资源清理和其它零零碎碎的操作。真心的赞！transferTo方法会抛出IOException，要向用户显示Error可以catch这个异常。但实际处理这些Error往往是，关闭文件，把没有写成功的文件删除，而这些Input、Output已经处理好了。我们再也不需要关心文件读写的细节！

拦截传输过程
============

上面处理了基本的I/O传输，我们常常还要做些其它的事。可能要计数一下传输了多少个数据，过滤一下数据，或者是每1000条数据做一下日志，又或者要看一下正在进行什么操作。既然输入输出已经分离，这些事变成在输入输出的协调代码中简单地插入一些逻辑。大部分协调代码有类似的功能，可以放到标准的工具方法中，更方便使用。

第一个标准修饰器是一个过滤器。实现时我用到了Specification。

```java
public static <T,ReceiverThrowableType extends Throwable> Output<T, ReceiverThrowableType> filter( final Specification<T> specification, final Output<T, ReceiverThrowableType> output)
{
   ... create an Output that filters items based on the Specification<T> ...
}
```

Specification如下： 

```java
interface Specification<T>
{
     boolean test(T item);
}
```

有了这个简单部件，我可以在传输时轻松地过滤掉那些不要出现在接收者端的数据。下面的例子删除文件中的空行： 

```java
File source = ...
File destination = ...
Inputs.text( source ).transferTo( Transforms.filter(new Specification<String>()
{
   public boolean test(String string)
   {
      return string.length() != 0;
   }
}, Outputs.text(destination) );
```

第二个常见的操作是把数据从一个类型映射到另一个类型。就是处理要Input和Output的数据类型不同，要有方法把输入数据类型映射成输出的数据类型。下面例子的把String映射成JSONObject，操作方法会是这个样子： 

```java
public static <From,To,ReceiverThrowableType extends Throwable> Output<From, ReceiverThrowableType> map( final Function<From,To> function, final Output<To, ReceiverThrowableType> output)
```

Function定义是：

```java
interface Function<From, To>
{
    To map(From from);
}
```

通过这些，可以把String的Input连接到JSONObject的Output：  

```java
Input<String,IOException> input = ...;
Output<JSONObject,RuntimeException> output = ...;
input.transferTo(Transforms.map(new String2JSON(), output);
```

String2JSON类实现了Function接口，它的map方法把String转换成JSONObject。

到了现在，我们可以实现前面提到数据计数的例子，可以把计数实现成一个通用的映射，转换前后的类型不变，只是维护了一个计数，在每次调用map方法时更新计数。例子代码如下：

```java
File source = ...
File destination = ...
Counter<String> counter = new Counter<String>();
Inputs.text( source ).transferTo( Transforms.map(counter, Outputs.text(destination) ));
System.out.println("Nr of lines:"+counter.getCount())
```

Usage in the Qi4j SPI
========================

【译者注，这一节说具体库Qi4j，略过】

结论
===============

软件开发时，从一个输入到另一个输出的数据和对象的搬移很常见，可能在中间还要做些转换。通常都是用一些零散代码（scratch）来完成这些事，结果是代码错误和使用不当的模式。通过引入通用I/O API，恰当封闭和隔离，这个任务可以可以更轻松地以伸缩、高性能、无错误的方式完成，并且还可以在在需要额外功能时修饰实现。

这遍文章仅仅勾勒了这种使用方式，API和辅助类可以在Qi4j Core 1.3-SNAPSHOT中有（详见Qi4j的主页）。理想状态是，在整个Qi4j使用中任何使用I/O的地方一开始按这种方式来。

多谢你的阅读，希望你能有所收获 :-)

<EOF>