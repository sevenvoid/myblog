---
title: Java 8 特性：Stream（一）
date: 2018/01/06
categories:
- 函数式编程
tags:
- Java 8
- 函数式编程
- stream
---
### 1、流是什么
流是Java 8 新出现的成员，它允许你以声明性方式处理数据集合(通过查询语句来表达，而不是临时编写一个实现，或者说是函数式风格处理流中的元素)，并且可以并行的处理。例如，假设有个菜单列表 (menus)，它包含诺干个菜品 (dish)，每个菜品都包含名字 (name)，卡路里量 (calories)，类型(type)，现在需要查询出只包含300以下卡路里且按照卡路里排序的所有菜品名字时，用Stream的实现方式如下所示：
```
	List<String> dishNames = menus.stream()
			.filter(()-> dish.getCalories() < 300)
			.sorted(comparing(Dish::getCalories))
			.map(Dish::getName)
			.collect(Collectors.toList())
```
换句话说，流是“从支持数据处理操作的源生成的元素序列”：
1. 元素序列——就像集合一样，流也提供了一个接口，可以访问特定元素类型的一组有序 值。因为集合是数据结构，所以它的主要目的是以特定的时间/空间复杂度存储和访问元素(如ArrayList 与 LinkedList)。但流的目的在于表达计算。
2. 源——流会使用一个提供数据的源，如集合、数组或输入/输出资源。 请注意，从有序集合生成流时会保留原有的顺序。
3. 数据处理操作——流的数据处理功能支持类似于数据库的操作，以及函数式编程语言中的常用操作，如filter、map、reduce、find、match、sort等。流操作可以顺序执行，也可并行执行。

<!--more-->

此外，流操作有两个重要的特点。
3. 流水线——很多流操作本身会返回一个流，这样多个操作就可以链接起来，形成一个大的流水线。流水线的操作可以看作对数据源进行数据库式查询。
4. 内部迭代——与使用迭代器显式迭代的集合不同，流的迭代操作是在背后进行的。Streams库的内部迭代可以自动选择一种适合你硬件的数据表示和并行实现。

### 2、流与集合的区别
流与集合有如下几个区别：
1. 流不能存储：流不是数据结构不能用于存储元素，相反的，流能将元素传递到一个流水线形式的计算操作集合。
2. 流的操作是函数式风格：也就是流的处理是无副作用的，对一个流的操作会产生新的结果而不是修改原始的流，而且同一流也只能被消费一次。
3. 流的操作是懒处理：流的操作可以分为中间操作和终端操作，只有终端操作被执行时，中间操作才会开始执行。
4. 流可能是无界：集合有界，而流可能是无界的，流的短路操作如 limit、findFirst操作，可以在一个有限的时间内在一个无限的流上操作。

### 3、流的实现
在Java的类库中，[java.util.stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) 包包含了4中类型的流对象，Stream、IntStream、LongStream、DoubleStream分别代表object对象流，原始数据类型的 int, long, double流。常见的可以通过如下方式获得流对象：
1. 通过Collection接口的stream() 和 parallelStream() 方法
2. 通过Arrays的stream(Object [ ])方法
3. 通过流对象的静态方法 Stream.of(Object [ ])、IntStream.range(int, int)、Stream.iterate(Object, UnaryOperator)、Stream.generate(Supplier<T> s)
4. 或者提供一个Spliterator 的实现，自己构造出一个流

Stream接口实现了AutoCloseable接口，但是几乎所有的流在使用后都不需要手动的close()，只有I/O Channel相关的流需要显示的Closing，也可以放在 *try-with-resources* 语句中，流被关闭时，注册在onClose(Runnable closeHandler) closeHandler将会被顺序依次调用执行。 

#### 流的操作
[java.util.strem.Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html) 类中提供了大量关于流的操作，主要如下所示：
```
**Intermediate operations** :
Stream<T> filter(Predicate<? super T> predicate)
<R> Stream<R> map(Function<? super T, ? extends R> mapper)
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)
Stream<T> peek(Consumer<? super T> action)
S sequential()
S parallel()
```
```
**Stateful intermediate operations** :
Stream<T> distinct()
Stream<T> skip(long n)
Stream<T> sorted()
Stream<T> sorted(Comparator<? super T> comparator)
```
```
**Short-circuiting Stateful intermediate operations** :
Stream<T> limit(long maxSize)
```
```
**Terminal operations** : 
void forEach(Consumer<? super T> action)
void forEachOrdered(Consumer<? super T> action)
Object[] toArray()
T reduce(T identity, BinaryOperator<T> accumulator)
Optional<T> reduce(BinaryOperator<T> accumulator)
<U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner)
<R> R collect(Supplier<R> supplier,BiConsumer<R, ? super T> accumulator,BiConsumer<R, R> combiner)
<R, A> R collect(Collector<? super T, A, R> collector)
Optional<T> min(Comparator<? super T> comparator)
Optional<T> max(Comparator<? super T> comparator)
long count()
Spliterator<T> spliterator()
Iterator<T> iterator()
```
```
**Short-circuiting terminal operations**:
boolean allMatch(Predicate<? super T> predicate)
boolean anyMath(Predicate<? super T> predicate)
boolean noneMatch(Predicate<? super T> predicate)
Optional<T> findFirst()
Optional<T> findAny()
```
流的操作主要分为两种类型的操作：中间操作和终端操作。中间操作会返回一个新的流；终端操作返回一个非Stream类型的值或者是消费流中的元素（副作用），只有终端操作被执行时，中间操作才开始执行。两者又包括部分短路操作。

中间操作又分为无状态的和有状态的操作，无状态的操作流中的每个元素将会单独处理，有状态的操作操作一个新元素时，需要关注之前元素的状态。

#### 参数化行为的特性
大多数的操作都接收行为参数化的参数，也即函数式接口，或Lambda表达式，或方法引用。为了保持操作有正确地行为，或者能并行执行的特性，参数化行为的行为参数需要有如下特性：
1.  无干涉 (Non-interference) : 也就是说行为参数在执行时不能修改原始流中的元素，除非流本身来自于支持并发修改的数据结构，比如ConcurrentHashMap等。如果是非并发数据结构产生的流的修改操作（非线程安全的）可能会造成异常、错误、或者不可预知的结果。但是在终端操作之前，对流的原始数据结果的修改是支持的，不会出现并发修改异常，并会反映到流。

2.  大多数无状态 (Stateless behaviors)： 行为参数在处理某个元素时，不应该依赖于之前的元素。有状态的行为（sort、distinct 等），如果不同步状态可能会造成数据并发修改的错误，如果同步状态，又可能出现竞态，或者降低并行执行的效率。

3. 副作用 (Side-effects)：尽量不要产生副作用，但大部分是需要副作用的，如map函数。
4.  结合性 (Associativity):  为了流水线可以并行执行，需要满足操作的可结合性，即：a op b op c op d == (a op b) op ( c op d)。

此外，流中元素的结果顺序，取决于数据源，以及中间操作，大多数情况，如果是有序的数据结构流则元素处理是有序的，产生有序的结果，无序的，就会产生无序的结果，并行执行时，有时无序的结果效率更高，可以是由BaseStream.unordered()方法使得元素无序。


### 参考
[Java 8 in action]()
[Java doc](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)
