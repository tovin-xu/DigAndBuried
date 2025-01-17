# Spark 闭包中ClosureCleaner操作

在Scala，函数是第一等公民，可以作为参数的值传给相应的rdd转换和动作，进而进行迭代处理。
阅读spark源码，我们发现，spark对我们所传入的所有闭包函数都做了一次sc.clean操作，如下

    def map[U: ClassTag](f: T => U): RDD[U] = new MappedRDD(this, sc.clean(f))
    private[spark] def clean[F <: AnyRef](f: F, checkSerializable: Boolean = true): F = {
        ClosureCleaner.clean(f, checkSerializable)
        f
    }
函数clean对闭包做了一次清理的操作，那么什么是闭包清理呢？

## 闭包
我们首先看ClosureCleaner里面一个函数：

    // Check whether a class represents a Scala closure
    private def isClosure(cls: Class[_]): Boolean = {
        cls.getName.contains("$anonfun$")
    }
该函数用来检测一个Class是不是闭包类，我们看到，如果一个对象的Class-name包含"$anonfun$",那么它就是一个闭包。再看一个实例：

    //BloomFilter.scala这个文件里面有一个contains函数，函数内部使用了一个匿名函数：
    def contains(data: Array[Byte], len: Int): Boolean = {
        !hash(data,numHashes, len).exists {
          h => !bitSet.get(h % bitSetSize)       //这里是一个匿名函数
        } 
     }
对BloomFilter.scala进行编译，我们会发现，它会针对这个匿名函数生成一个"BloomFilter$$anonfun$contains$1"Class，对于该类，spark将其识别闭包。

那么闭包到底是什么？

> 在计算机科学中，闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。
> 这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。
> 所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。
> 闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。

从上面的描述来看，闭包本身就是类，它的特点是它所创建的对象实例可以引用outer函数/类里面的变量。
朴素的说法就是：闭包就是能够读取外部函数的内部变量的函数。

另外，在本质上匿名函数和闭包是不同的概念，但是匿名函数一般都会被outer函数所包含，它有读取outer函数变量的能力，因此可以简单的把匿名函数理解为闭包。

简单的总结一下：闭包就是拥有对outer函数/类的变量的引用，从而可以在外面函数栈执行结束以后，依然握有外面函数栈/堆变量的引用，并可以改变他们的值。
说到这里，相信大家也看到闭包有对外部变量的引用的能力，这个能力是有潜在风险的。首先它会影响变量的GC，另外他会影响函数对象的序列化.  
再回头看一下clean函数第三个参数checkSerializable: Boolean = true，即是否检查序列化的问题，默认是true。
在scala中函数对象都是可以被序列化，从而可以传输到各个slave中进行计算，
但是如果一个函数对象引用了outer函数/对象的变量是不可以被序列化，那么就导致整个函数对象序列化失败。

## java中"闭包"仿真

java8版本引入Lambda表达式和闭包的支持,但是java8之前版本都没有支持,需要通过java(匿名)内部类来模拟实现,参考spark的rdd map函数的java-api

    <R> JavaRDD<R> map(Function<T,R> f)
    public interface Function<T1,R>
    extends java.io.Serializable
    
    //实现的时候可以
    rdd.map(new Function<String, String>{
        public String class(String strIn) {
            return strIn;
        }
    });
    
闭包和匿名内部类肯定还不是一个层次上的概念,要不然java8也不会在已有内部类的情况引入Lambda和闭包,那么它们之间有什么区别呢?
这里我首先总结一下java内部类的概念,

+   java内部类可以分为成员内部类,静态内部类,局部内部类,匿名内部类这个类别.
+   成员内部类可以访问外部对象所有的成员变量,无论他是否是static,final,public和private
+   成员内部类对成员变量访问可以直接访问,或者通过(外部类名称.this.非stattic变量)和(外部类名称.static变量名称)来访问,
如果内部类和外部类有相同的成员变量名称,那么访问内部的成员变量可以通过(变量名称)和(this.变量名称)来访问,
但是访问外部类的变量时候必须通过(外部类名称.this.变量名称)
+   成员内部类里面不能定义static类变量和static函数;但是静态内部类里面可以.
+   静态内部类不能访问外部类里面的非static成员变量,内部类没有(外部类名称.this)外部类的指针.
+   成员内部类的对象创建,必须通过(外部类名称.内部类名称 对象变量 = 外部类对象.new 外部类名称.内部类名称),
(注意:尽管new的方式不一样,但是new出来的两个内部对象的类型是相等,后面会谈到scala内部类,这点和scala是很不同,下面的实例提前做一个比较)

        //JAVA
        OuterClass outerClass1 = new OuterClass();
        OuterClass outerClass2 = new OuterClass();
        OuterClass.InnerClass innerClass1 = outerClass1.new InnerClass();
        OuterClass.InnerClass innerClass2 = outerClass2.new InnerClass();
        //two will be success
        outerClass1.runWithInnerClass(innerClass1);
        outerClass1.runWithInnerClass(innerClass2);
        
        //SCALA
        val scalaOuterClass1 = new ScalaOuterClass;
        val scalaOuterClass2 = new ScalaOuterClass;
        val scalaInnerClass1 = new scalaOuterClass1.ScalaInnerClass;
        val scalaInnerClass2 = new scalaOuterClass2.ScalaInnerClass;
        
        scalaOuterClass1.runWithInnerClass(scalaInnerClass1);
        //
        //error: type mismatch;
        //[INFO]  found   : scalaOuterClass2.ScalaInnerClass
        //[INFO]  required: scalaOuterClass1.ScalaInnerClass
        scalaOuterClass1.runWithInnerClass(scalaInnerClass2);
        
+   静态内部类和可以直接通过(外部类名称.内部类名称 对象变量 = new 外部类名称.内部类名称),即静态内部类与外部类的对象之间不存在对应关系.
+   成员内部类,静态内部类都是定义类里,与传统的成员变量/静态变量相似.还有另外一种作用域里的内部类:局部内部类,即定义在方法里的内部类,
它和成员内部类的区别是,它除了拥有外部类的变量的可见性以外,还拥有方法内的部分局部变量的可见性.
+   局部内部类中所拥有的方法中局部变量可见性指的是final变量,普通变量不具备可读性.
+   成员内部类和静态内部类都是编译为"外部类$内部类.class",而局部内部类很根据定义的次序编译为"外部类$次序编号+内部类.class".
+   匿名内部类,匿名内部类是局部内部类的一个子集,它是定义在局部方法内部,具有与局部内部类相同的外部类变量和局部变量的可读性.局部内部类的实现需要依赖接口来实现.匿名内部类会被编译成"外部类$次序编号.class".
+   总结,从上面来看,内部类拥有外部类成员变量的可见性,但是内部类(局部/匿名)不能读取定义域非final局部变量.

上面简单的对java内部类进行简单总结,发现它和闭包有几个区别

+   编译出来的class不一样.
+   局部/匿名内部类与也是局部定义的闭包对局部变量的可见性不同.

在对java的内部类与scala的闭包的区别进行分析之前,先来看一下scala对内部类的支持.

+   scala也有成员内部类,静态内部类,局部内部类以及匿名类;其中静态内部类是定义在Object的类;
+   scala中对内部类的支持与java大体一直,连编译出来的class名称也与java完全一样.不一样的三点是:
上述的内部类的类型机制不一样;局部内部类对局部变量的可见性不一样;引入路径依赖类型和类型投影的概念
+   重要:scala局部内部类对局部变量的可见性没有final/val变量的要求,比如下面的例子:

        def runWithInnerClass(inner:ScalaInnerClass): Unit = {
            var test= 2;
            class functionClass {
                def doSome1(): Unit = {
                    inner.doSome();//可以读取函数的参数
                    println(test)//可以读取函数局部变量
                    test=3;//可以修改函数的局部变量
                    println(test)//
                    print(ScalaOuterClass.this.test3);//可以读取外部类的成员变量
                }
            };
        }
    
+   针对"外部类名称.内部类名称"这样的格式的类型,引入"路径依赖类型"；比如 A.this.B就是一个路径依赖类型,
其中A.this会因为this的实例的不同而不同，比如 a1 和 a2 就是两个不同的路径，所以a1.B 与 a2.B也是不同的类型
+   路径依赖类型a1.B与a2.B是两个不同类型,但是她们都有一个超类型A.B,那么如果一个方法希望接受所有A.B,那怎么写?类型投影，用 A#B的形式表示。
那么def foo(b: A#B)就可以接受a1.B和a2.B.

从上面我看到看到仅仅Java对局部(匿名)类做了"只能读取final局部变量"的限制?为什么有这个限制?

首先对于外部类的成员变量没有访问限制的原因外部类的this引用在编译为字节码时已经作为内部类(不含静态内部类)的一个成员变量添加为内部类中,
从而不管是scala还是java,对外部类的成员变量的访问都没有限制.  
其次针对局部变量final条件的限制也是一种无可奈何的选择,java函数的运行是围绕进栈和出栈操作而进行,对于处于函数中局部变量(包括基本类型和引用类型)
在运行开始会放入栈中,函数运行结束会从栈中离开,离开就代表这个局部变量不存在了.同时对于处于函数中的局部内部类的生命周期明显比函数生命周期要长,在函数运行结束以后,
内部类依然引用局部类的栈变量,而此时栈已经出栈,此时内部类就会引用一个不存在的数据,这是内部类不可接受的.  

但是如果限制变量为final,那么就采用"值复制"的方式来解除内部类对外部函数局部函数栈变量的引用.  
对于基本类型,final类型在函数和内部类都不会被修改,因为可以复制,从而不会因为复制以后,导致两者修改都对应不同的变量.  
对于引用类型,final类型代表不能修改引用的指向,但是可以修改指向的对象内部值.这样可以保证函数内部和局部内部类所修改的都指向同一个对象,
并且在外部函数运行结束以后,内部类还依然引用相应对象,从而该对象不会被虚拟机GC.

通过上面限制java就可以无bug的实现局部内部类,虽然有限制,但是还是够用,所以一直以来,局部内部类/匿名内部类广泛应用在回调操作上.

那么为什么scala的局部内部类可以访问函数的var变量呢?如下面的实例:

    //scala
    def run(): Unit ={
        var data=1;
        class InnerClass{
            def runInnerClass(): Unit = {
                println(data);
                data = 2;
            }
        }
        (new InnerClass()).runInnerClass();
        print(data);
    }
    
    //截取javap中针对内部类构造函数和runnInnerClass
    public void runInnerClass();
        flags: ACC_PUBLIC
        Code:
          stack=2, locals=1, args_size=1
             0: getstatic     #21                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
             3: aload_0       
             4: getfield      #23                 // Field data$1:Lscala/runtime/IntRef;
             7: getfield      #29                 // Field scala/runtime/IntRef.elem:I
            10: invokestatic  #35                 // Method scala/runtime/BoxesRunTime.boxToInteger:(I)Ljava/lang/Integer;
            13: invokevirtual #39                 // Method scala/Predef$.println:(Ljava/lang/Object;)V
            16: return        
    public com.baidu.bcs.dataplatform.ScalaOuterClass$InnerClass$1(com.baidu.bcs.dataplatform.ScalaOuterClass, scala.runtime.IntRef);
    
    //截取javap中关于外部类run函数
    public void run();
        flags: ACC_PUBLIC
        Code:
          stack=4, locals=2, args_size=1
             0: new           #22                 // class scala/runtime/IntRef
             3: dup           
             4: iconst_1      
             5: invokespecial #26                 // Method scala/runtime/IntRef."<init>":(I)V
             8: astore_1      
             9: new           #28                 // class com/baidu/bcs/dataplatform/ScalaOuterClass$InnerClass$1
            12: dup           
            13: aload_0       
            14: aload_1       
            15: invokespecial #31                 // Method com/baidu/bcs/dataplatform/ScalaOuterClass$InnerClass$1."<init>":(Lcom/baidu/bcs/dataplatform/ScalaOuterClass;Lscala/runtime/IntRef;)V
            18: invokevirtual #34                 // Method com/baidu/bcs/dataplatform/ScalaOuterClass$InnerClass$1.runInnerClass:()V
            21: getstatic     #39                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
            24: aload_1       
            25: getfield      #43                 // Field scala/runtime/IntRef.elem:I
            28: invokestatic  #49                 // Method scala/runtime/BoxesRunTime.boxToInteger:(I)Ljava/lang/Integer;
            31: invokevirtual #53                 // Method scala/Predef$.print:(Ljava/lang/Object;)V
            34: return        
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                   0      35     0  this   Lcom/baidu/bcs/dataplatform/ScalaOuterClass;
                   9      25     1  data   Lscala/runtime/IntRef; 
    
通过javap我们可以看到,外部函数的定义局部变量data,在run函数中被包装成成一个scala/runtime/IntRef对象,并且在内部类InnerClass的构造函数中,将其传入到构造函数中,
因为这个对象是scala自己生成的,所以可以肯定的被包装这个对象是不会改变引用,相当于java final对象,然后函数和内部类之间就可以进行操作了.
一句话,把本身属于栈的基本类型变量,转换为引用类型,从而实现scala内部类可以读取外部定义函数的局部变量,并且不受final的限制.

那如果不是基本类型而是引用类型呢?很简单,封装为scala/runtime/ObjectRef.

哈哈哈!!!!终于理清楚java内部类和scala的内部的区别了;虽然这篇问题是要讲闭包,但是我相信很多人和我一样,对闭包与内部类之间的差别很模糊!!!!清晰了

## 闭包清理的实现
 
    
  


临时备注:
参考http://www.cnblogs.com/chenssy/p/3388487.html和http://www.cnblogs.com/yjmyzz/p/3448330.html对内部类/匿名类的实现,更好的来解释这个含义

     
