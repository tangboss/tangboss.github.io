---
layout:     post
title:      "Scala含集合List类型对象的判等方法"
subtitle:   " \"SortCompare\""
date:       2019-03-23 20:12:00
author:     "tangboss"
tags:
    - scala
    - hashcode
    - equals
---


## 需求

一般情况下Scala内，通过定义case class，原生override了equals和hashcode方法，可以实现对象间的比较。但若类里包含List类型，如List(a,b,c)其实不等于List(b,a,c)，

需要进行equals和hashcode的重载，方便对象间的比较。

---

## equals&hashcode

在Java里，equals和hashcode方法都是用来对比两个对象是否相等一致。



- equals()相等的两个对象他们的hashCode()肯定相等，也就是用equals()对比是绝对可靠的。


- hashCode()相等的两个对象他们的equals()不一定相等，也就是hashCode()不是绝对可靠的。
比较流程：

对于需要大量并且快速的对比的话如果都用equals()去做显然效率太低，所以解决方式是，每当需要对比的时候，首先用hashCode()去对比，

如果hashCode()不一样，则表示这两个对象肯定不相等（也就是不必再用equals()去再对比了）,如果hashCode()相同，此时再对比他们的equals()，如果equals()也相同，则表示这两个对象是真的相同了，这样既能大大提高了效率也保证了对比的绝对正确性。这种大量的并且快速的对象对比一般使用的hash容器中，比如HashSet,HashMap,HashTable等等，比如HashSet里要求对象不能重复。


一般情况下对于普通类，hashCode()和equals()一样都是基本类Object里的方法，而和equals()一样，Object里hashCode()里面只是返回当前对象的地址，对于相同的一个类，new两个对象，由于他们在内存里的地址不同，则他们的hashCode（）不同，所以这显然不是我们想要的，所以我们必须重写我们类的hashCode()方法

**开发原则：**

一般情况下，只要重写 equals，就必须重写 hashCode，

如果重写了equals，比如说是基于对象的内容实现的，而保留hashCode的实现不变，那么很可能某两个对象明明是“相等”，而hashCode却不一样。

这样，当你用其中的一个作为键保存到hashMap、hashTable或hashSet中，再以“相等的”找另一个作为键值去查找他们的时候，则根本找不到。

---

## case class

Scala里的case class进行了多个方法 的封装，

经过scala编译器优化后，被更好的用于模式匹配规则的类
（1）Case class的每个参数默认以val(不变形式)存在，除非显式的声明为var
（2）自动产生伴生对象，、且半生对象中自动产生appay方法来构建对象
（3）半生对象自动产生unapply方法，提取主构造器的参数进行模式匹配
（4）自动产生copy方法，来构建一个与现有值相同的新对象
（5）class中自动产生hashcode，toString，equals方法

可以定义一个case class，进行反编译：

1）hello.scala：case class Hello(world:String,hi:Int)

2）进行编译产生两个文件：scalac hello.scala

     Hello.class  Hello$.class

3）进行反编译：javap -p -classpath . Hello.class

	root@ZTE:~/code/testCase# cat hello.scala
	case class Hello(world:String,hi:Int)
	root@ZTE:~/code/testCase# scalac hello.scala
	root@ZTE:~/code/testCase# ls
	Hello.class  Hello$.class  hello.scala
	root@ZTE:~/code/testCase# javap -p -classpath . Hello.class
	Compiled from "hello.scala"
	public class Hello implements scala.Product,scala.Serializable {
	  private final java.lang.String world;
	  private final int hi;
	  public static scala.Function1<scala.Tuple2<java.lang.String, java.lang.Object>, Hello> tupled();
	  public static scala.Function1<java.lang.String, scala.Function1<java.lang.Object, Hello>> curried();
	  public java.lang.String world();
	  public int hi();
	  public Hello copy(java.lang.String, int);
	  public java.lang.String copy$default$1();
	  public int copy$default$2();
	  public java.lang.String productPrefix();
	  public int productArity();
	  public java.lang.Object productElement(int);
	  public scala.collection.Iterator<java.lang.Object> productIterator();
	  public boolean canEqual(java.lang.Object);
	  public int hashCode();
	  public java.lang.String toString();
	  public boolean equals(java.lang.Object);
	  public Hello(java.lang.String, int);
	}

可以看出case class增加继承于product，并基于product实现了equals和hashcode方法：

	javap -c -classpath . Hello.class
	public int hashCode();
	    Code:
	       0: ldc           #85                 // int -889275714
	       2: istore_1
	       3: iload_1
	       4: aload_0
	       5: invokevirtual #46                 // Method world:()Ljava/lang/String;
	       8: invokestatic  #91                 // Method scala/runtime/Statics.anyHash:(Ljava/lang/Object;)I
	      11: invokestatic  #95                 // Method scala/runtime/Statics.mix:(II)I
	      14: istore_1
	      15: iload_1
	      16: aload_0
	      17: invokevirtual #49                 // Method hi:()I
	      20: invokestatic  #95                 // Method scala/runtime/Statics.mix:(II)I
	      23: istore_1
	      24: iload_1
	      25: iconst_2
	      26: invokestatic  #98                 // Method scala/runtime/Statics.finalizeHash:(II)I
	      29: ireturn

	ScalaRunTime.scala
	
	def _toString(x: Product): String =
	  x.productIterator.mkString(x.productPrefix + "(", ",", ")")
	 
	def _hashCode(x: Product): Int = scala.util.hashing.MurmurHash3.productHash(x)


---

## Scala实现

故为了变更对象的比较行为，需重写equals和hashcode方法。基于本需求对于含有集合List的类对象，需要对List进行排序后再进行比较。

	trait SortWithCompare {
	  override def equals(that: Any): Boolean = that match {
	    case that: SortWithCompare =>
	      this.recursive_sort;that.recursive_sort
	      getClass.getDeclaredFields.forall(f => {
	        f.setAccessible(true)
	        if (f.getType == classOf[scala.collection.immutable.List[_]]) {
	          f.get(this).asInstanceOf[List[_]].sortBy(_.toString) ==
	            f.get(that).asInstanceOf[List[_]].sortBy(_.toString)}
	        else f.get(this) == f.get(that)})
	    case _ => false}
	 
	  private def recursive_sort: SortWithCompare = {
	    getClass.getDeclaredFields.foreach(f => {
	      if (f.getType == classOf[scala.collection.immutable.List[_]]) {
	        f.setAccessible(true)
	        val result = f.get(this).asInstanceOf[List[_]].map({
	          case e: SortWithCompare => e.recursive_sort
	          case b: Any => b})
	        f.set(this, result.sortBy(_.toString))}})
	    this}
	 
	  override def hashCode(): Int = {
	    val state = getClass.getDeclaredFields.map(para=>{
	      para.setAccessible(true)
	      para.get(this)
	    }).filterNot(_.eq(null)).toSeq
	    state.map(_.hashCode()).foldLeft(0)((a, b) => 31 * a + b)
	  }
	 
	  override def toString:String = {
	    getClass.getDeclaredFields.map(para=>{
	      para.setAccessible(true)
	      para.get(this)
	    }).mkString("(", ",", ")")
	  }
	}

主要通过反射获取对象的每个字段，对于包含List字段的进行递归排序，目前排序的基准是通过toString方法实现。为了保证toString的唯一性，进行了toString的重写。

**注：**若基于hashcode来排序，但不同对象可能有相同的hashcode，如字符串"gdejicbegh"以及"hgebcijedg"具有相同的hash值，会导致List("gdejicbegh","hgebcijedg")与List("hgebcijedg","gdejicbegh")比较不等。

---
## 用例
	case class CompareTest3(para3:List[CompareTest2],para30:String = "test30") extends SortWithCompare
	case class CompareTest2(para2:List[CompareTest1],para20:String = "test20") extends SortWithCompare
	case class CompareTest1(para1:String)
	 
	class CompareTestClass3(val para3:List[CompareTestClass2],para30:String = "test30") extends SortWithCompare
	class CompareTestClass2(val para2:List[CompareTestClass1],para20:String = "test20") extends SortWithCompare
	class CompareTestClass1(val para1:String) extends SortWithCompare
	 
	 
	@RunWith(classOf[JUnitRunner])
	class ContentCompareSpec extends FunSpec with ScalatraSuite with BeforeAndAfterEach with Matchers {
	 
	  describe("Content Compare test |") {
	    it("test case class with List equals and hashCode"){
	      val s1= new CompareTest1("gdejicbegh")
	      val s2= new CompareTest1("hgebcijedg")
	      val s3 =new CompareTest1("gdejicbegh")
	      s1.hashCode() should be (s2.hashCode())
	      s1 should be(s3)
	      s1 should not be s2
	      val c1 = new CompareTest2(List(s1,s2))
	      val c2 = new CompareTest2(List(s2,s1))
	      c1 should be (c2)
	      c1.hashCode() should be(c2.hashCode())
	      val t1 = new CompareTest3(List(c1,c2))
	      val t2 = new CompareTest3(List(c2,c1))
	      t1.hashCode() should be(t2.hashCode())
	      t1 should be (t2)
	    }
	 
	    it("test class with List equals and hashCode"){
	      val s1= new CompareTestClass1("gdejicbegh")
	      val s2= new CompareTestClass1("hgebcijedg")
	      val s3 =new CompareTestClass1("gdejicbegh")
	      s1 should be(s3)
	      s1 should not be s2
	      val c1 = new CompareTestClass2(List(s1,s2))
	      val c2 = new CompareTestClass2(List(s2,s1))
	      c1 should be (c2)
	      c1.hashCode() should be(c2.hashCode())
	      val t1 = new CompareTestClass3(List(c1,c2))
	      val t2 = new CompareTestClass3(List(c2,c1))
	      t1.hashCode() should be(t2.hashCode())
	      t1 should be (t2)
	    }
	}

