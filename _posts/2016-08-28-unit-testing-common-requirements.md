---
layout: post
title: Unit Testing Common Requirements
date: 2016-08-28 20:25
author: matejoseph
comments: true
categories: [Java, java, Programming, Unit Testing]
---
Suppose you wanted to unit test java.util.Map. How would you verify each implementation? Take a quick look at the <a href="https://docs.oracle.com/javase/7/docs/api/java/util/Map.html">Map javadoc</a>, and you'll find 20 implementing classes. Each Map implementation has common unit tests for requirements that you want to avoid copying. In this post I illustrate two techniques you can use for testing common requirements over implementations by testing my simplified version of java.util.Map.

I also encountered a similar problem when taking the Coursera Scala Parallel Programming course. The course's assignments required sequential and parallel implementations. I noticed I ended up copying and pasting my tests of the sequential implementation to the parallel one. I came up with these solutions to de-duplicate the code.
<h1>Introduction to Example</h1>
In this example we implement a simplified Map interface with three methods, get, put, and getKeys:

```java
public interface Map<K,V>  {
	void put(K key, V value);
	V get(K key);
	List getKeys();
}
```

We provide two implementations of the interface. InefficientMap uses two ArrayLists to store the keys and values and SortedMap which guarantees that getKeys() returns the keys in sorted order. The implementations are purposely naive because the focus of this article is the testing aspect, not performance.
<h1>Method 1: Abstract Class</h1>
Abstract classes can be used to test the common requirements of differing implementations. This method provides the least duplicated code. You have all the common test cases in the base class which makes use of a factory method for generating the implementation.

To apply the inheritance you follow these steps:
<ol>
	<li>Create an abstract class for the common unit tests</li>
	<li>Add an abstract method that expects the extender provide a new instance the real implementation</li>
	<li>Add @Tests the verify the requirements of the interface</li>
	<li>Create a unit test class for each implementation and extend from the abstract unit test class</li>
	<li>Provide the real implementation expected by the abstract method</li>
	<li>Add any additional @Tests that test requirements specific to the implementation</li>
</ol>
<h2>Step 1 and 2: Creating the Abstract Class</h2>
You create an abstract class with an abstract factory method for returning the real implementation.

```java
public abstract class InheritingMapTest {
	protect abstract Map<K,V> makeMap();
}
```
<h2>Step 3: Add @Tests</h2>
In the abstract unit test class, we include requirements that are common to all maps. The tests make use of the factory method for getting an instance of the implementation.

Here we look at only one of the many test cases for the Map interface. The factory provides us with a mechanism for obtaining a new instance of the implementation.

Getting the value using a key that was replaced is should return the most recent value.

```java
@Test
public void testValueReplaced() {
	Map<Integer,Integer> map = makeMap();
	map.put(0, 10);
	map.put(0, 12);
	assertEquals(Integer.valueOf(12), map.get(0));
}
```
<h2>Step 4 and 5: Create a Unit Test Class for each Implementation</h2>
Now for each implementation we extend from the abstract class and implement the factory method.

```java
public class InheritingSortedMapTest extends InheritingMapTest {
	protected Map<Integer,Integer> makeMap() {
		return new SortedMap<>(IntComparator.intComparator);
	}
}
```
<h2>Step 6: Add Additional Tests of Uncommon Requirements to Subclass</h2>
The sorted map comes with the additional requirement that the getKeys() method returns the keys in sorted order. We add tests of this new requirement into the subclass.

```java
@Test
public void testKeysSorted() {
	SortedMap<Integer,Integer> map = new SortedMap<>(IntComparator.intComparator);
	map.put(2,12);
	map.put(1,11);
	map.put(3,13);
	List keys = map.getKeys();
	for(int i = 0 ; i < 3; i++) {
		assertEquals(Integer.valueOf(i+1),keys.get(i));
	}
}
```
<h1>Method 2: Composition</h1>
An alternative to this pattern is to include all the tests in a helper class and have each unit test call the testAll() method. The problem with this solution is that you end up having a method which calls all your test methods which duplicates some code. More importantly, it makes it more difficult to figure out which test failed because you have to inspect the stack trace instead of click on the failed test in an IDE like eclipse.
<ol>
	<li>Create a factory interface</li>
	<li>Create class containing that accepts the factory in the constructor</li>
	<li>Add tests the verify the requirements of the interface</li>
	<li>Call each test method from a public 'testAll()' method</li>
	<li>Create a unit test class for each implementation that creates and instance of the test class</li>
	<li>Provide an implementation of the factory to the test class</li>
	<li>Add any additional @Tests that test requirements specific to the implementation</li>
</ol>
<h2>Step 1 and 2: Create the Factory and Test Class</h2>
The 'Testing Class' has a Factory for creating new instances of the implementation when it needs it.

```java
public class CompositionMapTest {
	public interface MapFactory {
		/**
		 * Return a new instance of your implementation of Map<>;
		 */
		Map<Integer,Integer> make() ;
	}

	private MapFactory mapFactory;

	public CompositionMapTest(MapFactory mapFactory) {
		this.mapFactory = mapFactory;
	}
}
```
<h2>Step 3 and 4: Add Tests</h2>
We make use of the factory to create an instance of the implementation and verify the object matches the specification.

```java
private void testValueReplaced() {
	Map<Integer,Integer> map = mapFactory.make();
	map.put(0, 10);
	map.put(0, 12);
	assertEquals(Integer.valueOf(12), map.get(0));
}
```

The 'Testing Class' invokes all the test methods in a "testAll()" method which the real test classes will call.

```java
public void testAll() {
	...
	testValueReplaced();
	...
}
```

<h2>Step 5 and 6: Create a class for each implementation</h2>
To reuse the existing tests of the 'Test Class' we create an instance of it, providing it with a Factory for creating implementations. We invoke the test class's "testAll()" method to check our implementation.

```java
public class CompositionSortedMapTest {
	@Test
	public void testBasicRequirements() {
		CompositionMapTest basicTests = new CompositionMapTest(
			new CompositionMapTest.MapFactory() {
				public Map<Integer,Integer> make() {
					return new SortedMap<>(IntComparator.intComparator);
				}
			});
		basicTests.testAll();
	}
}
```
<h2>Step 7: Add any additional tests for uncommon requirements</h2>
We write tests as we would normally for requirements that are not common to all Maps.

```java
@Test
public void testKeysSorted() {
	SortedMap<Integer,Integer> map = new SortedMap<>(IntComparator.intComparator);
	map.put(2,12);
	map.put(1,11);
	map.put(3,13);
	List keys = map.getKeys();
	for(int i = 0 ; i < 3; i++) {
		assertEquals(Integer.valueOf(i+1),keys.get(i));
	}
}
```
<h1>Conclusion</h1>
If you notice you're copying and pasting unit test code for common requirements, consider one of these two techniques. However, these techniques are only guides because you will encounter cases that do not fit cleanly. As an exercise for the reader: how would you extend these techniques to apply to java.util.WeakHashMap? If you want to have a go I've made the source available on <a href="https://github.com/josephmate/inheriting_unit_tests">my github</a>.

