---
title: Java并发编程系列（一）：Singleton血案
date: 2016/11/20
categories:
- 并发编程
tags:
- Java
- 并发编程
- 单例Singleton模式
---

<blockquote class="blockquote-center">很早以前就想认认真真的学习、吃透Java编程方面的各种原理，并且积累一些经验，一是想要技术更上一层楼，二也是想要找到一个自己想要探索的方向。在自己的认知上，一直认为吃透了基础，才可以选择远方。   ——题记</blockquote>
从本文开始将会详细了解一下Java并发编程方法的相关知识，内容的来源可能是自己看书、或者网上的优秀博文，文末都会有相关的参考链接，详细内容可以访问原文。好记性不如烂笔头，自己暂时写不好一篇文章，那么试着整理也是一件不可多得的事情，至少认真思考了，也实践了一遍。本文将从一个单例模式开始，看下如何编写正确的并发代码。

Singleton的习惯使用
====
Singleton设计模式可能是被讨论和使用的最广泛的一个设计模式了，在Java编程中，对于单例模式，程序员一般习惯会写成如下形式：
```java
package edu.sevenvoid.jvm.thread;
/**
 * @author sevenvoid
 *
 * 2016年12月4日
 */
public class SingletonTest {
	
	private static SingletonTest instance = null;
	
	private SingletonTest() {
		//为了说明并发问题，打印对象创建时的线程名
		System.out.println("The current thread is : " + Thread.currentThread().getName()); //
	}
	public static SingletonTest getInstance() {
		if(instance == null) {   //1:A线程执行
			instance = new SingletonTest();  //2:B线程执行
		}
		return instance;
	}
	
	public static void main(String[] args) {
		for(int i=0; i<=5; i++) {
			Thread thread = new Thread(new Runnable() {
				@Override
				public void run() {
					SingletonTest.getInstance();
				}
			});
			thread.start();
		}
	}
}
```
<!-- more-->
上面代码大家应该都知道，所谓的线程不安全的懒汉单例写法，它有如下几个Singleton的特点(都知道的)：
1. 私有（private）的构造函数，表明这个类是不可能形成实例了。这主要是怕这个类会有多个实例
2. 即然这个类是不可能形成实例，那么，我们需要一个静态的方式让其形成实例：getInstance()。注意这个方法是在new自己，因为其可以访问私有的构造函数，所以他是可以保证实例被创建出来的
3. 在getInstance()中，先做判断是否已形成实例，如果已形成则直接返回，否则创建实例
4. 所形成的实例保存在自己类中的私有成员中
5. 我们取实例时，只需要使用Singleton.getInstance()就行了

Singleton的问题
===
## 并发问题
在单线程环境下上面的写法确实正确，但大多数时候我们的程序都是并发执行的，因此上面的写法是有严重的问题的，比如：A线程执行代码1的同时，B线程执行代码2，此时，线程A可能看到instance引用的对象还没有初始化。运行上面的并发代码出现如下结果，说明这个类被实例化了多次，也即不再是Singleton了：
```java
The current thread is : Thread-0
The current thread is : Thread-2
The current thread is : Thread-1
```
## 同步问题
既然涉及到了并发访问，那我们自然而然就想到了同步，比如用synchronized关键字来同步方法或者代码块，如下所示：
```java
public class SingletonTest {
	
	private static SingletonTest instance = null;
	
	private SingletonTest() {
		System.out.println("The current thread is : " + Thread.currentThread().getName());
	}
	//第一种同步方式
	public static SingletonTest getInstance() {
		if(instance == null) {     //C-1-1:A线程执行
			synchronized(SingletonTest.class) {  
				instance = new SingletonTest();  //C-1-2:A线程执行
			}
		}
		return instance;
	}
	//第二种同步方式，当然，这两种方式只能选择一种实现
	public static SingletonTest getInstance() {
		synchronized(SingletonTest.class) {
			if(instance == null) {      //C-2-1:
				instance = new SingletonTest();
			}
		}
		return instance;
	}
}
```
我们可以使用多种方式来同步，比如：
1. 当instance==null时，用synchronized来同步实例的创建
2. 一进入getInstance()方法就使用synchroznied来同步，或者直接同步方法。

这两种情况都有些问题，前者的问题在于假设A，B线程都进入了C-1-1判断中，此时只能有一个线程进入同步块，假设A线程进入同步块完成C-1-2单例实例化之后，B线程进入会再次完成单例实例化；后者的问题在于，这样的写法是保证了线程安全，也通过C-2-1保证了只有一个单例实例化,但是由于getInstance()方法做了同步处理，如果getInstance()方法被多个线程频繁调用，synchronized将会导致程序执行性能的下降。那么，有没有更优雅的方案呢？我们也很容易想到将这两者的优势充分结合起来，形成如下的写法：
```java
public class SingletonTest {
	
	private static SingletonTest instance = null; //2
	
	private SingletonTest() {
		System.out.println("The current thread is : " + Thread.currentThread().getName());
	}
	//double-check
	public static SingletonTest getInstance() {     //3
		if(instance == null) {     //第一次非空判断       //4
			synchronized(SingletonTest.class) {       //5
				if(instance == null) {   //第二次非空判断     //6
					instance = new SingletonTest();  //7问题根源
				}
			}  //8
		} //9
		return instance;  //10
	} //11
}
```
这个版本又叫“双重检查”***Double-Check***：
1. 第一次非空判断是说，如果实例创建了，那就不需要同步了，直接返回就好了，因此，可以大幅降低synchronized带来的性能开销。不然，我们就开始同步线程
2. 第二次非空判断是说，如果被同步的线程中，有一个线程创建了对象，那么别的线程就不用再创建了

双重检查锁定看起来似乎很完美，我们的问题“好像”是成功解决了，但是它仍旧是一个不完美的优化。

## 重排序问题
上面双重检查之后出现的问题主要在于instance = new SingletonTest()这句，这并非是一个原子操作，事实上在 JVM 中这句话大概做了下面3件事情：
```java
memory=allocate(); //1:分配对象的内存空间
initInstance(memory); //2:初始化对象
instance=memory; //3:设置instance指向刚分配的内存地址（执行完这步 instance才是非 null了）
```
上面3行代码中的2和3之间，可能会被重排序（在一些 JVM的即时编译器(JIT)中存在指令重排序的优化），也就是说上面的2和3的顺序是不能保证的。2和3之间重排序之后的执行时序如下：
```java
memory=allocate(); //1:分配对象的内存空间
instance=memory; //3:设置instance指向刚分配的内存地址（执行完这步 instance才是非 null了）
initInstance(memory); //2:初始化对象
```
如果发生重排序，另一个并发执行的线程B就有可能在第4行判断instance不为null。线程B接下来将访问instance所引用的对象，但此时这个对象可能还没有被A线程初始化，在使用时就会出现问题。在知晓问题发生的根源之后，我们可以想出两个办法解决:
1. 不允许2和3重排序
2. 允许2和3重排序，但不允许其他线程“看到”这个重排序

Singleton的解决方案
===
## 基于volatile的解决方案
对于前面的基于双重检查锁定的方案，只需要做一点小的修改，就可以实现线程安全的延迟初始化：
```java
public class SingletonTest {
	
	private volatile static SingletonTest instance = null;   //volatile修饰
	
	private SingletonTest() {
		System.out.println("The current thread is : " + Thread.currentThread().getName());
	}
	//double-check
	public static SingletonTest getInstance() {
		if(instance == null) {
			synchronized(SingletonTest.class) {
				if(instance == null) {
					instance = new SingletonTest();
				}
			}
		}
		return instance;
	}
}
```
使用 volatile有两个作用：
1. 这个变量不会在多个线程中存在复本，直接从内存读取
2. 这个关键字会禁止指令重排序优化。也就是说，在 volatile变量的赋值操作后面会有一个内存屏障（生成的汇编代码上），读操作不会被重排序到内存屏障之前

但是，这个事情仅在Java1.5版后有用，1.5版之前用这个变量也有问题，因为老版本的Java的内存模型是有缺陷的。

## 基于类初始化的解决方案
JVM在类的初始化阶段（即在Class被加载后，且被线程使用之前），会执行类的初始化。在执行类的初始化期间，JVM会去获取多个线程对同一个类的初始化。基于这个特性，实现的示例代码如下：
```java
public class SingletonTest {
	
	private SingletonTest() {
		System.out.println("The current thread is : " + Thread.currentThread().getName());
	}

	private static class InstanceHolder {
    		public static SingletonTest instance = new SingletonTest();
    	}
    
   	public static SingletonTest getInstance() {
    	return InstanceHolder.instance;
    }
}
```
这个方案的本质是允许前面伪代码谈到的2和3重排序，但不允许其他线程“看到”这个重排序。在SingletonTest示例代码中，首次执行getInstance()方法的线程将导致InstanceHolder类被初始化。由于Java语言是多线程的，多个线程可能在同一时间尝试去初始化同一个类或接口（比如这里多个线程可能会在同一时刻调用getInstance()方法来初始化InstanceHolder类）。Java语言规定，对于每一个类和接口C，都有一个唯一的初始化锁LC与之对应。从C到LC的映射，由JVM的具体实现去自由实现。JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类已经被初始化过了。这个方案没有性能缺陷，也不依赖 JDK 版本。

## 基于枚举的完美方案
上面的版本是旧版《Effective Java》的推荐的方式，而在新版中推荐使用枚举enum来实现：
```java
package edu.sevenvoid.jvm.thread;

/**
 * @author sevenvoid
 *
 * 2016年12月4日
 */
public enum SingletonTest {
	
	INSTANCE;
	
	private SingletonTest() {
		System.out.println("The current thread is : " + Thread.currentThread().getName());
	}
	public void doOtherSomething() {  //单例的其他功能
		System.out.println("Hello thread : " + Thread.currentThread().getName());
	}
	public static void main(String[] args) {
		
		for(int i=0; i<=5; i++) {
			Thread thread = new Thread(new Runnable() {
				@Override
				public void run() {
					SingletonTest.INSTANCE.doOtherSomething();
				}
			});
			thread.start();
		}

	}
```
运行上面的程序，从结果可以清楚的看到，实例只被初始化了一次，结果如下：
```java
The current thread is : main   //主线程完成实例化单例
Hello thread : Thread-0     //其他线程使用
Hello thread : Thread-1
Hello thread : Thread-3
Hello thread : Thread-2
Hello thread : Thread-4
Hello thread : Thread-5
```
默认枚举实例的创建是线程安全的，所以不需要担心线程安全的问题。但是在枚举中的其他任何方法的线程安全由程序员自己负责。还有防止上面的通过反射机制调用私用构造器。这种单元素的枚举类型已经成为实现Singleton的最佳方法。<br />

## Singleton其他问题
怎么？还有问题？！当然还有，请记住下面这条规则——“无论你的代码写得有多好，其只能在特定的范围内工作，超出这个范围就要出Bug了（左耳朵耗子）”，想一想还有什么情况会让这个我们上面的代码出问题吗？（当然，有些反例可能属于钻牛角尖，不过也不排除其实际可能性。

### Class Loader
JVM规范定义了两种类型的类装载器：启动内装载器(bootstrap)和用户自定义装载器(user-defined class loader)。 在一个JVM中可能存在多个ClassLoader，每个ClassLoader拥有自己的NameSpace。一个ClassLoader只能拥有一个class对象类型的实例，但是不同的ClassLoader可能拥有相同的class对象实例，这时可能产生致命的问题。如ClassLoaderA，装载了类A的类型实例A1，而ClassLoaderB，也装载了类A的对象实例A2。逻辑上讲A1=A2，但是由于A1和A2来自于不同的ClassLoader，它们实际上是完全不同的，如果A中定义了一个静态变量c，则c在不同的ClassLoader中的值是不同的。<br/>
于是，如果Singleton的双重检查版本如果面对着多个Class Loader就会有多个实例同样会被多个Class Loader创建出来，当然，这个有点牵强，不过他确实存在。我们不能在Singleton类中操作Class Loader，在这种情况下，能做的只有是——“保证多个Class Loader不会装载同一个Singleton”。

### 序例化
如果我们的这个Singleton类具有序列化的功能，那么当反序列化的时候，我们将无法控制别人不多次反序列化，这样就会造成多个对象被实例化出来。这种情况下，我们可以利用一下Serializable接口的readResolve()方法，比如：
```java
public class Singleton implements Serializable
{
    ......
    ......
    protected Object readResolve()
    {
        return getInstance(); //或者枚举情况下： return INSTANCE;
    }
}
```
### 多个Java虚拟机
如果我们的程序运行在多个Java的虚拟机中。多个虚拟机这种情况是有点极端，不过还是可能出现，比如EJB或RMI之流的东西。要在这种环境下避免多实例，看来只能通过良好的设计或非技术来解决了。

### volatile变量
关于volatile这个关键字所声明的变量可以被看作是一种 “程度较轻的同步synchronized”；与 synchronized块相比volatile变量所需的编码较少，并且运行时开销也较少，但是它所能实现的功能也仅是synchronized的一部分。当然，如前面所述，我们需要的Singleton只是在创建的时候线程同步，而后面的读取则不需要同步。所以，volatile变量并不能帮助我们即能解决问题，又有好的性能。而且，这种变量只能在JDK 1.5+版后才能使用。

### 关于继承
是的，继承于Singleton后的子类也有可能造成多实例的问题。不过，因为我们早把Singleton的构造函数声明成了私有的，所以也就杜绝了继承这种事情。

### 关于代码重用
如果我们的系统中有很多个类需要用到这个模式，如果我们在每一个类都中有这样的代码，那么就显得有点傻了。那么，我们是否可以使用一种方法，把这具模式抽象出去?Java下可能比较复杂一些，比如抽象类的模板方法等，但仍旧需要良好的设计。

上面就是一个单例类的实现方式，一步一步的分析了如何来优雅完美的实现，当然最终还是会有些极端的问题，但只要我们结合业务恰当的实现，是能够完美的解决的。在分析单例这个过程中，提到了重排序、同步等问题，接下来将会进一步讲述。

## 参考
[从一个简单的Java单例示例谈谈并发](http://hugnew.com/?p=851)
[深入浅出单实例Singleton设计模式](http://coolshell.cn/articles/265.html)

