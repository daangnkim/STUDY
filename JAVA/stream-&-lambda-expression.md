## Stream

- The **Stream API** (introduced in Java 8) is a way to process collections using **a functional programming** approach. Instead of writing traditional loops, you create a "pipeline" of operations
- Streams can be created from different element sources e.g. collections or arrays
- stream operation process
	- create
	```java
	numbers.stream() // Converts List<Integer> to Stream<Integer>
	```
	- intermediate operation - build pipeline
		- These operations return another Stream, which means you can chain them together. They don't execute immediately - they just build up a series of instructions.
	```java
	.map(num -> num * 2)        // Transform each element
	.filter(num -> num % 2 == 0) // Keep elements that match condition
	```
	- terminal operation  - execute pipeline
		- The key insight is that **nothing actually happens until you call a terminal operation**
	```java
	.collect(Collectors.toList())  // Gather results into a new List
	```

## Lambda Expression

- `->`syntax is called a **lambda expression**. It's a short way to write anonymous functions.

## Collecting


[[모던 자바] 컬렉터(Collector)란 무엇인가? — Bonglog - 기록과 정리의 공간](https://devbksheen.tistory.com/entry/%EB%AA%A8%EB%8D%98-%EC%9E%90%EB%B0%94-%EC%BB%AC%EB%A0%89%ED%84%B0Collector%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80)
[Introduction to Java Streams | Baeldung](https://www.baeldung.com/java-8-streams-introduction)
