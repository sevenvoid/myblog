---
title: Jar包冲突表象、本质与解决方案
date: 2018/12/15
comments: true
categories:
- Java
tags:
- Java
- Jar包冲突
---
使用Java语言进行项目开发时，尤其是在面对大型复杂型项目时，Jar包或Class冲突可以说是非常常见的。最直接的原因就是项目依赖了大量的二方库，这些二方库又依赖了其他的库，因此会间接引入大量Jar，导致大量Jar包冲突，需要进行大量的排包工作。很容易想到的原因就是构建工具如maven关联Jar引起的。因为众所周知，maven的传递性依赖非常方便，但也会让我们陷入依赖的地狱。

<!-- more -->
## 表象
那有哪些现象我们可以初步定位是Jar包或者Class冲突导致的问题呢？通常分为两大类：一类是比较隐蔽的，在编译的时候不会报错，但是程序运行时的结果却与预期的不一致；另一类比较直观，编译或者运行时常常会报一个异常，常见异常如下：
```java
java.lang.ClassNotFundException
java.lang.NoSuchMethodError
java.lang.NoClassDefFoundError
java.lang.LinkageError
```
## 本质
上面那些异常总的来说可以归结为两种类型的Jar包冲突问题：
**第一类：同一个Jar包出现了多个不同的类版本：应用程序依赖的同一个Jar包出现了多个不同版本，并选择了错误的版本而导致JVM加载不到需要的类或加载了错误版本的类**

随着项目的迭代进行，势必有越来越多的二方包会自我升级，有些包的升级可能修改了某个类的签名或者方法签名。针对这类问题，大多数时候都是由Jar包依赖管理工具引发的冲突。不管是Maven，还是Gradle，其传递性依赖为我们带来了极大地便利性，但同时也引入了问题 - 版本冲突。以Maven为例，当同一个Jar包出现了多个不同的版本，针对该问题Maven也有一套仲裁机制来决定最终选用哪个版本，但Maven的选择往往不一定是我们所期望的，这也是产生Jar包冲突最常见的原因之一。先来看下Maven的仲裁机制：
+ 优先按照依赖管理<dependencyManagement>元素中指定的版本声明进行仲裁，此时下面的两个原则都无效了
+ 若无版本声明，则按照“短路径优先”的原则（Maven2.0）进行仲裁，即选择依赖树中路径最短的版本
+ 若路径长度一致，则按照“第一声明优先”的原则进行仲裁，即选择POM中最先声明的版本从maven的仲裁机制中可以发现，除了第一条仲裁规则（这也是解决Jar包冲突的常用手段之一）外，后面的两条原则，对于同一个Jar包不同版本的选择，Maven的选择有点“随机性”了。但每个应用都有其特殊性，该依赖哪个版本，Maven也是没办法帮你完全搞定，如果你没有规规矩矩地使用<dependencyManagement>来进行依赖管理，就注定了逃脱不了第一类Jar包冲突问题。

**第二类不同的Jar包出现了类路径一致的类：同样的类（类的全限定名完全一样）出现在多个不同的依赖Jar包中，即该类有多个版本，并由于Jar包加载的先后顺序导致JVM加载了错误版本的类**

这类冲突比较隐蔽，这种情况下JVM加载的类由Jar所在的加载路径和文件系统的文件加载顺序确定的。对于这类Jar包冲突问题，即多个不同的Jar包有类冲突，这相对于第一类问题就显得更为棘手。为什么这么说呢？
在这种情况下，两个不同的Jar包，假设为A、 B，它们的名称互不相同，甚至可能完全不沾边，如果不是出现冲突问题，你可能都不会发现它们有共有的类！对于A、B这两个Jar包，Maven就显得无能为力了，因为Maven只会为你针对同一个Jar包的不同版本进行仲裁，而这俩是属于不同的Jar包，超出了Maven的依赖管理范畴。此时，当A、B都出现在应用程序的类路径下时，就会存在潜在的冲突风险，即A、B的加载先后顺序就决定着JVM最终选择的类版本，如果选错了，就会出现诡异的第二类冲突问题。那么Jar包的加载顺序都由哪些因素决定的呢？具体如下：
+ Jar包所处的加载路径，或者换个说法就是加载该Jar包的类加载器在JVM类加载器树结构中所处层级。由于JVM类加载的双亲委派机制，层级越高的类加载器越先加载其加载路径下的类，顾名思义，引导类加载器（bootstrap ClassLoader，也叫启动类加载器）是最先加载其路径下Jar包的，其次是扩展类加载器（extension ClassLoader），再次是系统类加载器（system ClassLoader，也就是应用加载器appClassLoader），Jar包所处加载路径的不同，就决定了它的加载顺序的不同。
+ 文件系统的文件加载顺序。这个因素很容易被忽略。因tomcat等容器的ClassLoader获取加载路径下的文件列表时是不排序的，这就依赖于底层文件系统返回的顺序，那么当不同环境之间的文件系统不一致时，就会出现有的环境没问题，有的环境出现冲突。比如：测试环境怎么测都没问题，但一上线就出现冲突问题，规避这种问题的最佳办法就是尽量保证测试环境与线上一致。很显然大多数时候都是应用Jar包，因此大多可能原因是文件加载顺序导致的。

此外Maven还提供了几种依赖范围，用于控制在不同的环境下使用不同的依赖，也可能会导致上述问题，这些范围是：
+ compile: 默认依赖范围，不申明的话就是compile，对于编译、测试、运行三种classpath都有效
+ test: 只对测试有效，在编译主代码或者运行项目时无法使用此类依赖
+ provided: 对编译和测试有效，运行时无效，运行时已有其他方式添加了此依赖，比如servlet-api
+ runtime: 对测试和运行时有效，在编译主代码时无效，比如JDBC驱动实现
+ system: 和provided依赖范围一致

## 定位与解决
当遇到上述这些冲突时，可以有如下方式来快速定位解决问题：
**方式一：使用mvn dependency:tree 命令定位**

它有如下参数：-Dverbose: verbose参数将详细显示项目所依赖的所有Jar包，包括冲突和未冲突的；-Dincludes: 指定要查找显示的包，格式为\[groupId]:\[artifactid]:\[type]:\[version]，支持 * 匹配，用逗号分隔多个；-Dexcludes: 指定不显示的包，格式和includes相同。一般来说遇到包冲突时，将冲突的包按上述方式找出来，并排除，能解决大部分的问题。排除包的时候，也只需要我们在本项目的POM文件中对于依赖的Jar包，在<dependency>下使用<exclusion>标签即可，并在需要时主动依赖被排除的包。

**方式二：IDE协助查找类冲突**

使用IDE查找某个类的方式，当键入类名称时，如果出来了类全限定名完全相同的但位于不同Jar包中的多个类时，就很可能有冲突了

**方式三：使用IDE的Maven helper插件定位**

**方式四：使用第三方Java诊断工具来排查**

比如[Arthas](https://alibaba.github.io/arthas/)

**方式五：也可以使用如下代码来诊断**
```java
import java.io.IOException;
import java.net.URL;
import java.util.Enumeration;
import java.util.HashSet;
import java.util.Set;

public class ClassConflictCheck {

    private ClassConflictCheck() {}

    public static void checkConflictClass(Class cls) {
        checkConflictClass(cls.getName().replace(".","/") + ".class");
    }

    private static void checkConflictClass(String path) {
        try {
            Enumeration<URL> urls =  Thread.currentThread().getContextClassLoader().getResources(path);
            Set<String> files = new HashSet<String>();
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                if (null != url) {
                    String file = url.getFile();
                    if (null != file && file.length() > 0) {
                        files.add(file);
                    }
                }
            }
            System.out.println("Conflict class of " + path + " in " + files.size() + " jar: " + files);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ClassConflictCheck.checkConflictClass(java.net.URL.class);
    }
}
```

**方式六：使用Maven插件**
maven-enforcer-plugin，这个强大的maven插件，配合extra-enforcer-rules工具，能自动扫描Jar包将冲突检测并打印出来，汗颜的是。其原理其实也比较简单，通过扫描Jar包中的class，记录每个class对应的Jar包列表，如果有多个即是冲突了，故不必深究，我们只需要关注如何用它即可。

在最终需要打包运行的应用模块pom中，引入maven-enforcer-plugin的依赖，在build阶段即可发现问题，并解决它。比如对于具有parent pom的多模块项目，需要将插件依赖声明在应用模块的pom中。这里有童鞋可能会疑问，为什么不把插件依赖声明在parent pom中呢？那样依赖它的应用子模块岂不是都能复用了？

这里之所以强调“打包运行的应用模块pom”，是因为冲突检测针对的是最终集成的应用，关注的是应用运行时是否会出现冲突问题，而每个不同的应用模块，各自依赖的Jar包集合是不同的，由此而产生的<ignoreClasses>列表也是有差异的，因此只能针对应用模块pom分别引入该插件。
```xml
...
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>1.4.1</version>
  <executions>
    <execution>
      <id>enforce</id>
      <configuration>
        <rules>
          <dependencyConvergence/>
        </rules>
      </configuration>
      <goals>
        <goal>enforce</goal>
      </goals>
    </execution>
    <execution>
      <id>enforce-ban-duplicate-classes</id>
      <goals>
        <goal>enforce</goal>
      </goals>
      <configuration>
        <rules>
          <banDuplicateClasses>
            <ignoreClasses>
              <ignoreClass>javax.*</ignoreClass>
              <ignoreClass>org.junit.*</ignoreClass>
              <ignoreClass>net.sf.cglib.*</ignoreClass>
              <ignoreClass>org.apache.commons.logging.*</ignoreClass>
              <ignoreClass>org.springframework.remoting.rmi.RmiInvocationHandler</ignoreClass>
            </ignoreClasses>
            <findAllDuplicates>true</findAllDuplicates>
          </banDuplicateClasses>
        </rules>
        <fail>true</fail>
      </configuration>
    </execution>
  </executions>
  <dependencies>
    <dependency>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>extra-enforcer-rules</artifactId>
      <version>1.0-beta-6</version>
    </dependency>
  </dependencies>
</plugin>
```
maven-enforcer-plugin是通过很多预定义的标准规则（[standard rules](http://maven.apache.org/enforcer/enforcer-rules/index.html)）和用户自定义规则，来约束maven的环境因素，如maven版本、JDK版本等等，它有很多好用的特性，具体可参见官网。而Extra Enforcer Rules则是MojoHaus项目下的针对maven-enforcer-plugin而开发的提供额外规则的插件，这其中就包含前面所提的重复类检测功能，具体用法可参见[官网](http://www.mojohaus.org/extra-enforcer-rules/)，这里就不详细叙述了。

**方式七：最小化依赖**

最后也是最重要的规则就是最小化项目的依赖，每一次有新的依赖Jar包，都可以仔细去分析一下该Jar的依赖情况，排除不需要的Jar包，可以使用`mvn dependency:tree -Dverbose > dp.txt` 命令分析。

## 总结
以上就是对于Jar包冲突问题的现象分析，以及提出的解决方案。希望下次遇到同样的问题时，能快速定位并解决，提高自己的工作效率！







