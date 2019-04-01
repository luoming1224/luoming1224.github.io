---
title: 'TreeSet,LinkedHashSet以及HashSet之间的区别'
date: 2018-04-09 19:55:40
categories: 'java学习笔记'
tags:
---
原文链接：[Difference between TreeSet, LinkedHashSet and HashSet in Java with Example](http://javarevisited.blogspot.hk/2012/11/difference-between-treeset-hashset-vs-linkedhashset-java.html)

<!--more-->

TreeSet,LinkedHashSet以及HashSet均实现了Set接口，因此他们都遵循Set接口的协议，比如他们都不允许重复的元素。尽管他们都来自同一类型的层次结构，但是他们之间还是有很多区别；理解他们之间的区别非常重要，这样您就可以根据您的需求选择最合适的集合实现。另外，TreeSet和HashSet（或者LinkedHashSet）之间的区别是一个非常流行的Java集合面试题，虽然不如“比较Hashtable和HashMap”或者“比较ArrayList和Vector”这类问题普遍，但是依然出现在各种Java面试中。在这篇文章中，我们将看看HashSet,TreeSet以及LinkedHashSet在各个方面之间的区别，比如元素顺序、性能、是否允许null值等，然后我们会看到在Java中什么时候使用TreeSet、LinkedHashSet、HashSet。

# TreeSet,LinkedHashSet以及HashSet之间的区别
TreeSet,LinkedHashSet以及HashSet是Java集合框架中Set接口的三种实现类型，并且像许多其他集合类型一样，他们也用于存储对象。TreeSet的主要特性是其中元素时排序的，LinkedHashSet的主要特性是其中元素是基于插入顺序的，HashSet则仅仅是用于存储对象这一通用集合功能。在Java中HashSet是基于HashMap实现的，而TreeSet是基于TreeMap实现的。TreeSet同时是一个SortedSet接口的实现，因此允许TreeSet中元素基于Comparable或者Comparator接口定义保持元素顺序排序。Comparable通常用于按照对象自然顺序排序，Comparator通常用于对象自定义排序，可以在创建TreeSet实例时提供这两种接口之一用于实例中元素排序。在了解TreeSet, LinkedHashSet 以及HashSet之间的区别之前，我们先看看他们之间的相同之处。

1）是否允许重复元素：三者都实现了Set接口，意味着他们都不允许存储重复元素。

2）线程安全性：HashSet, TreeSet 以及 LinkedHashSet都不是线程安全的。 如果你将他们用于多线程环境下，并且至少有一个线程修改Set集合中的内容的话，你需要使用外部的同步机制来同步他们。

3）快速失败迭代器（Fail-Fast Iterator）：TreeSet, LinkedHashSet 以及 HashSet返回的迭代器均为快速失败迭代器。关于更多 [fail-fast vs fail-safe Iterator](http://javarevisited.blogspot.sg/2012/02/fail-safe-vs-fail-fast-iterator-in-java.html) 请阅读这里。

现在让我们来看看**HashSet, LinkedHashSet 以及 TreeSet之间的区别**

**性能和速度：**他们之间的第一个区别来自速度，HashSet 是最快的，LinkedHashSet 在性能方面排第二，或者近乎接近于HashSet，TreeSet稍微慢一些，因为每一次插入元素都需要执行排序操作。TreeSet 对于常见操作如add,remove,contains等能够保证时间复杂度为 O(log(n)) ，而 HashSet 和 LinkedHashSet 对于add，contains，remove等操作提供时间复杂度为常数时间的性能，因为HashSet为给定散列函数在桶中均匀分配元素。

**排序性：**HashSet 不保证元素的任何顺序，而LinkedHashSet 保持元素的插入顺序，更多的像List接口一样，TreeSet 按照元素实现的比较接口保持元素的顺序。

**内部实现：**HashSet底层基于一个HashMap实例，LinkedHashSet 基于HashSet和LinkedList实现，TreeSet底层基于NavigableMap接口实现，java中默认用TreeMap实现。

**null值：**HashSet 和 LinkedHashSet允许null值，但是TreeSet 不支持null值，并且当你向TreeSet中插入null值时会抛出 java.lang.NullPointerException 异常。因为TreeSet应用 compareTo() 方法于各个元素来比较他们，当比较null值时会抛出 NullPointerException异常。

**比较（Comparison ）：**HashSet 和 LinkedHashSet 用 equals() 方法用于比较，但是TreeSet 用 compareTo() 方法用于维持元素顺序。这就是为什么在java中compareTo()和 equals() 必须保持一致，否则会打破Set接口的一些通用规则，比如可能会允许重复值。

# TreeSet vs HashSet vs LinkedHashSet - 使用示例
让我们通过java编程来比较所有这些Set接口实现在一些方面的异同。在这个例子中我们会证明 TreeSet, HashSet 和 LinkedHashSet在元素顺序和插入1M数据记录所花费的时间之间的差异。这将会帮助我们巩固前面讨论的内容，已经帮助我们决定在java中什么时候应该用HashSet, LinkedHashSet 或者 TreeSet。

	import java.util.Arrays;
	import java.util.HashSet;
	import java.util.LinkedHashSet;
	import java.util.Set;
	import java.util.TreeSet;
	
	/**
	 * Java program to demonstrate difference between TreeSet, HashSet and LinkedHashSet
	 * in Java Collection.
	 * @author
	 */
	public class SetComparision {
  
	    public static void main(String args[]){             
	        HashSet<String> fruitsStore = new HashSet<String>();
	        LinkedHashSet<String> fruitMarket = new LinkedHashSet<String>();
	        TreeSet<String> fruitBuzz = new TreeSet<String>();
	      
	        for(String fruit: Arrays.asList("mango", "apple", "banana")){
	            fruitsStore.add(fruit);
	            fruitMarket.add(fruit);
	            fruitBuzz.add(fruit);
	        }
	       
	        //no ordering in HashSet – elements stored in random order
	        System.out.println("Ordering in HashSet :" + fruitsStore);
	
	        //insertion order or elements – LinkedHashSet storeds elements as insertion
	        System.err.println("Order of element in LinkedHashSet :" + fruitMarket);
	
	        //should be sorted order – TreeSet stores element in sorted order 
	        System.out.println("Order of objects in TreeSet :" + fruitBuzz); 
	     
	
	        //Performance test to insert 10M elements in HashSet, LinkedHashSet and TreeSet
	        Set<Integer> numbers = new HashSet<Integer>();
	        long startTime = System.nanoTime();
	        for(int i =0; i<10000000; i++){
	            numbers.add(i);
	        }
	
	        long endTime = System.nanoTime();
	        System.out.println("Total time to insert 10M elements in HashSet in sec : "
	                            + (endTime - startTime));
	      
	      
	        // LinkedHashSet performance Test – inserting 10M objects
	        numbers = new LinkedHashSet<Integer>();
	        startTime = System.nanoTime();
	        for(int i =0; i<10000000; i++){
	            numbers.add(i);
	        }
	        endTime = System.nanoTime();
	        System.out.println("Total time to insert 10M elements in LinkedHashSet in sec : "
	                            + (endTime - startTime));
	       
	        // TreeSet performance Test – inserting 10M objects
	        numbers = new TreeSet<Integer>();
	        startTime = System.nanoTime();
	        for(int i =0; i<10000000; i++){
	            numbers.add(i);
	        }
	        endTime = System.nanoTime();
	        System.out.println("Total time to insert 10M elements in TreeSet in sec : "
	                            + (endTime - startTime));
	    }
	}

	Output
	Ordering in HashSet :[banana, apple, mango]
	Order of element in LinkedHashSet :[mango, apple, banana]
	Order of objects in TreeSet :[apple, banana, mango]
	Total time to insert 10M elements in HashSet in sec : 3564570637
	Total time to insert 10M elements in LinkedHashSet in sec : 3511277551
	Total time to insert 10M elements in TreeSet in sec : 10968043705


# 何时用 HashSet, TreeSet 以及 LinkedHashSet
所有三种Set接口实现都可以用于通用的Set操作，比如不允许重复元素，但是三者又都有其独特的特性，所以在特定的环境下可以有更合适的选择。 由于TreeSet保持元素顺序，所以当你需要一个集合保持元素顺序并且不允许重复元素时，可以使用TreeSet；HashSet是通用的Set集合实现，如果你需要一个效率比较快、且不允许重复元素的集合，则可以使用HashSet；LinkedHashSet 是HashSet的扩展，它更适合当你需要保持元素的插入顺序时使用，更像一个不牺牲昂贵性能的List。LinkedHashSet 的另一个用法是用于创建一个已经存在的Set集合的副本，因为LinkedHashSet 保持插入顺序，它返回包含同一顺序的相同元素的集合，更像精确的复制。总之，尽管他们三个都是Set接口的实现，但是他们均有各自独特的特性，HashSet是一个通用功能的Set，而LinkedHashSet 提供元素插入顺序保证，TreeSet是一个SortedSet实现，由Comparator 或者 Comparable指定的元素顺序存储元素。

# 如何从一个集合中复制对象到另一个集合
下面是一个LinkedHashSet 的代码示例，来展示LinkedHashSet 如何从一个集合复制对象到另一个集合而且能够保持元素顺序。你将得到一个源Set的精确复制，无论是内容还是顺序都一致。这里静态方法copy(Set source)使用泛型来实现，这样的参数化方法可以提供类型安全，以及帮助避免运行时异常。

	import java.util.Arrays;
	import java.util.HashSet;
	import java.util.LinkedHashSet;
	import java.util.Set;
	
	/**
	 * Java program to copy object from one HashSet to another using LinkedHashSet.
	 * LinkedHashSet preserves order of element while copying elements.
	 * 
	 * @author Javin
	 */
	public class SetUtils{
	    
	    public static void main(String args[]) {
	        
	        HashSet<String> source = new HashSet<String>(Arrays.asList("Set, List, Map"));
	        System.out.println("source : " + source);
	        Set<String> copy = SetUtils.copy(source);
	        System.out.println("copy of HashSet using LinkedHashSet: " + copy);
	    }
	    
	    /*
	     * Static utility method to copy Set in Java
	     */
	    public static <T> Set<T> copy(Set<T> source){
	           return new LinkedHashSet<T>(source);
	    }
	}
	
	Output:
	source : [Set, List, Map]
	copy of HashSet using LinkedHashSet: [Set, List, Map]

通常在代码中用接口而不是具体的接口实现类型，可以在需求变化时很方便的替换为HashSet、LinkedHashSet 或者TreeSet。这就是java中HashSet, LinkedHashSet 以及 TreeSet之间所有的区别，如果你了解这三者之间任何其他更有意义的区别，请在评论区中评论。