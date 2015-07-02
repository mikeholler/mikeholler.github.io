---
layout: post
title: Java PrioirtyQueue Pitfalls
---

One thing I've always loved about Java is the `java.util` and `java.util.concurrent` data structures.
They're predictable, stable, and multiple implemetations of each data structure are often available.
Still, every so often I fall victim to a "gotcha".
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

    private final Queue<Result> mBuffer = new PriorityQueue(
            MAX_BUFFER_SIZE, new ResultComparator());

    @Override
    public final onNextResult(final Result result) {
        mBuffer.offer(result);

        if (mBuffer.size() > MAX_BUFFER_SIZE) {
            // Order is guaranteed for the next result, so
            // it may be consumed.
            final Result result = mBuffer.remove();

            // Use result here.
        }
    }

}
```
