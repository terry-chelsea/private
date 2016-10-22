#**Scala学习笔记1——初始Scala**

## **引入**

最近因为Spark的兴起，Scala也炙手可热，个人并不认为它是一个新兴的编程语言，虽然它提供了全新的语法，本文主要介绍Scala环境和几种运行方式，以及通过实例观察Scala和JAVA的关系，学习Scala主要参考[Scala语法手册](http://www.runoob.com/scala/scala-tutorial.html)和[Scala实例](https://www.cs.helsinki.fi/u/wikla/OTS/Sisalto/examples/index.html)。

## **Scala环境**
Linux上可以使用类似于JDK环境安装的方式，下载Scala，设置PATH，不过我在Scala官网上找了半天Linux包才在[这里](http://www.scala-lang.org/download/2.11.8.html)找到。这种方式同样适用于Windows，不过Windows可以在Scala官网上下载[安装包](http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.msi)，并且可以下载基于eclipse等IDE的[Scala IDE](http://www.scala-lang.org/download/2.11.8.html)，Scala有两种使用方式。

* 类似于Python的方式直接作为交互式方式运行，直接运行scala命令就可以进入交互式环境，或者编写一个scala脚本文件，在不进行编译的情况下直接使用scala命令运行（类似于Python），不过scala不能像java源码中那样定义package。
* 类似于Java的方式编译运行，类似于java，scala还提供了scalac/scalap等工具，通过scalac编译之后的scala文件生成.class文件，和javac的结果相同，然后可以使用scala命令执行程序，下面使用Hello World的实例分别展示这几种方式的使用方式。

## **实例**

### **交互式命令行**
	$ scala
	Welcome to Scala 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_101).
	Type in expressions for evaluation. Or try :help.
	
	scala> println("Hello World !")
	Hello World !
	
	scala> var s = "Hello World !"
	s: String = Hello World !
	
	scala> println(s)
	Hello World !

### **scala作为脚本执行**
编辑helloWorld.scala文件：
	
	var s = "Hello World !"
	println(s)
	
	hzfengyu@HIH-L-1888 MINGW64 /e/Scala
	$ scala ./helloWorld.scala
	Hello World !


### **使用scala编译执行**
编辑HelloWorld.java文件

	object HelloWorld {
	    def main(args : Array[String]) : Unit = {
	        var s = "Hello World !"
	        println(s)
	    }
	}

#### **1. 直接使用scala执行**

	$ scala ./HelloWorld.scala
	Hello World !

#### **2. 编译执行**

	$ scalac -cp . ./HelloWorld.scala
	
	hzfengyu@HIH-L-1888 MINGW64 /e
	$ ls HelloWorld*
	'HelloWorld$.class'  HelloWorld.class  HelloWorld.scala
	
	hzfengyu@HIH-L-1888 MINGW64 /e
	$ scala -cp . HelloWorld
	Hello World !

#### **3. 使用java命令执行**

	$ java -cp . HelloWorld
	Exception in thread "main" java.lang.NoClassDefFoundError: scala/Predef$
	        at HelloWorld$.main(HelloWorld.scala:4)
	        at HelloWorld.main(HelloWorld.scala)
	Caused by: java.lang.ClassNotFoundException: scala.Predef$
	        at java.net.URLClassLoader.findClass(Unknown Source)
	        at java.lang.ClassLoader.loadClass(Unknown Source)
	        at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
	        at java.lang.ClassLoader.loadClass(Unknown Source)
	        ... 2 more

可以看出使用java来运行这个class缺少一些jar，因此我们尝试将scala安装包lib下面的jar加入到classpath中。It Works！

	$ java -cp .:$SCALA_HOME/lib/* HelloWorld 
	Hello World !

### **反编译成Java**

通过这几个实例可以看出，scala是基于java的，个人觉得它其实就是提供了一种新的编程语法，提供了scala自己的一些函数式编程的函数，然后通过编译将其转换成JVM能够识别的class文件，所以使用java命令也是可以运行的，设置我们可以通过[jad命令](http://varaneckas.com/jad/)（不是jdk里面的工具）将scala代码反编译成java代码，在上面可以看到，HelloWorld.scala文件通过scalac编译之后生成了两个class文件，分别将它们反编译。如下：
	$ jad ./HelloWorld.class 
	Parsing ./HelloWorld.class...The class file version is 50.0 (only 45.3, 46.0 and 47.0 are supported)
	Generating HelloWorld.jad
	
	$ cat HelloWorld.jad 
	// Decompiled by Jad v1.5.8e. Copyright 2001 Pavel Kouznetsov.
	// Jad home page: http://www.geocities.com/kpdus/jad.html
	// Decompiler options: packimports(3) 
	// Source File Name:   HelloWorld.scala
	
	
	public final class HelloWorld
	{
	
	    public static void main(String args[])
	    {
	        HelloWorld$.MODULE$.main(args);
	    }
	}

	$ jad ./HelloWorld$.class 
	Parsing ./HelloWorld$.class...The class file version is 50.0 (only 45.3, 46.0 and 47.0 are supported)
	Generating HelloWorld$.jad
	$ cat HelloWorld$.jad 
	// Decompiled by Jad v1.5.8e. Copyright 2001 Pavel Kouznetsov.
	// Jad home page: http://www.geocities.com/kpdus/jad.html
	// Decompiler options: packimports(3) 
	// Source File Name:   HelloWorld.scala
	
	import scala.Predef$;
	
	public final class HelloWorld$
	{
	
	    public void main(String args[])
	    {
	        String s = "Hello World !";
	        Predef$.MODULE$.println(s);
	    }
	
	    private HelloWorld$()
	    {
	    }
	
	    public static final HelloWorld$ MODULE$ = this;
	
	    static 
	    {
	        new HelloWorld$();
	    }
	}

这个反编译回来的文件就是熟悉的java代码了，可以看到它导入了scala.Predef$这个类，这个是scala语言包里面的类，所以当我们使用java执行该程序，在classpath中加上scala包之后就可以把它当做一个普通java程序运行了。

## **总结**

所以我觉得scalac类似于执行javac -cp $SCALA_HOME/lib的效果，scala类似于执行scala -cp $SCALA_HOME/lib的效果，而scala本身包含了编译scala代码功能，所以最简单的接受scala的方式是可以把它当做一个java库，这不过这个库需要使用全新的语法写代码，然后使用特定的工具编译程序。所以再熟悉JAVA的基础上学习Scala还是有一点优势的，不过乍一看Scala的代码感觉天马行空的，后面我们会通过实例来学习Scala语法和Scala的库。
