---
layout: post
title: Java PriorityQueue Pitfalls
---

One thing I've always loved about Java is the `java.util` and `java.util.concurrent` data structures.
They're predictable, stable, and multiple implemetations of each data structure are often available.
Still, every so often I fall victim to a "gotcha."
Java's `PriorityQueue` in particular makes it very easy to shoot oneself in the foot, an unusual trait for a Java standard library class.

Recently, I found myself working with a particulary difficult API that returns real-time, timestamped data out of order occassionally.
Though annoying, the fix we easy enough: build up a buffer with API results, ordered by timestamp, and only begin consuming results once the buffer is large enough to eradicate out-of-order results. Enter the `PriorityQueue`.

A PriorityQueue is the perfect tool for this job because it lets you how you want to queue to be arranged via `Comparable` elements or by providing a custom `Comparator`.
The result objects I work with have no natural ordering, so I had to seek out the latter option:

```java
public final class ResultComparator
        implements Comparator<Result> {
    
    @Override
    public int compare(final Result lhs, final Result rhs) {
        return Long.compareTo(
            lhs.getUnixMillis(),
            rhs.getUnixMillis());
    }
}
```

```java
public final class BufferedResultListener
        implements NextResultListener {

    /**
     * When the buffer has at least this number of results,
     * we can guarantee they will come out of the queue in
     * order.
     */
    private static final int MAX_BUFFER_SIZE = 1000;

    private final Queue<Result> buffer = new PriorityQueue(
            MAX_BUFFER_SIZE, new ResultComparator());

    @Override
    public final onNextResult(final Result result) {
        buffer.offer(result);

        if (buffer.size() > MAX_BUFFER_SIZE) {
            // Order is guaranteed for the next result, so
            // it may be consumed.
            final Result result = buffer.remove();

            // Use result here.
        }
    }

}
```

Given the use of Comparator and PriorityQueue together, it seems reasonable to assume the items in the queue are ordered.
The whole point of a PriorityQueue, after all, is to return the contained object with the highest priority when remove(), poll(), element(), or peek() are called (the standard Queue accessor methods).
Similarly reasonable is the assumption that iterating over a PriorityQueue thusly returns objects with highest priority first.
There are a number of scenarios where one might want to iterate over a PriorityQueue, and it seems intuitive that this iteration should happen in a manner consistent with how the contained items might come out of the queue, their prioirty order.

```java
/**
 * Get the first item in the queue with a priority (timestamp)
 * greater than targetUnixMillis.
 */
Result findFirstResultAfterInstant(long targetUnixMillis) {
    if (buffer.size() == 0) {
        throw new IllegalStateException("Queue is empty.");
    }


    for (Result item : buffer) {
        if (item.getUnixMillis() > targetUnixMillis) {
            // Will always return here, since size is
            // limited by validation block at top of
            // method.
            return item;
        }
    }

    return null;
}
```

The above function takes advantage of these assumptions and will usually return earlier than it would if it had to iterate over an unordered set instead of an ordered priority queue.
But it doesn't work.
Why?
The answer is in the documentation.
[PriorityQueue](http://docs.oracle.com/javase/8/docs/api/java/util/PriorityQueue.html) is implemented as a heap in Java, and unfortunately for us, the only way to get an ordered list of items out of a heap is to destroy it.
This turns the simple function above into this:

```java
Result findFirstResultAfterInstant(long targetUnixMillis) {
    if (buffer.size() == 0) {
        throw new IllegalStateException("Queue is empty.");
    }

    long firstDiff = Long.MAX_LONG;
    Result first = null;

    for (Result item : sortedResults) {
        long diff = item.getUnixMillis() - targetUnixMillis;
        if (item.getUnixMillis() > targetUnixMillis && diff < firstDiff) {
            first = item;
            firstDiff = diff;
        }
    }

    return first;
}
```

The above method is more complex and may no longer return early, but now works as expected.

Why does Java not iterate in order? It's because they decided that out-of-order iteration is the most efficient even though it invalidates the assumptions one could logically make about priority queue iteration order.
Still, it would be nice if there was some built-in way to get an iterator that will work in order.
This could be provided by a `Queue.orderedIterator()` method, similar to how Lists provide ListIterators, and it might look like this:

```java
public class PriorityQueue<T> {

    ...

    public Iterator<T> orderedIterator() {
        List<T> copy = new ArrayList(this);

        if (this.comparator != null) {
            Collections.sort(copy);
        } else {
            Collections.sort(copy, this.comparator);
        }

        return copy.iterator();
    }

}
```

Sure, it's inefficient compared to the standard PriorityQueue.iterator(), but it's the only reasonable way to in-order-traverse a PriorityQueue without destroying it.
As long as the (in)efficiency is noted in the Javadoc, I don't see a problem with it existing in the standard library.
Just having the orderedIterator method in the list of methods provided by PriorityQueue would go a long way in reducing programmers' confusion when using this otherwise very straightforward class.
To those who argue this would introduce unnecessary bloat into the Java standard library, I'd hand them a page from Python's PEP 20, which applies to way more than just Python:

> Beautiful is better than ugly.  
> Explicit is better than implicit.  
> ...  
> Readability counts.  
> Special cases aren't special enough to break the rules.  
> Although practicality beats purity.  
> ...  
> There should be one-- and preferably only one --obvious way to do it.  
> ...  
> Now is better than never.  
> ...  
> If the implementation is hard to explain, it's a bad idea.  
> If the implementation is easy to explain, it may be a good idea.  
