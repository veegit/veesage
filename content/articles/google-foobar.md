+++
title = 'Secret Google FooBar'
date = 2025-05-01T19:05:31-08:00
+++

What started as a simple Google search for "CyclicBarrier in Java" turned into an unexpected journey through Google's Foo Bar coding challenge. Eight years ago, while debugging thread synchronization issues, my browser suddenly displayed a black screen with the message: "You've been invited to try Google Foo Bar. Do you accept? (yes/no)". It seemed like every level has number of problems = level, except for level 4 and 5. 

## The Invitation

My coding competition days participation was long over. So accepting the challenge made me nervious but I decided to dive in. Accepting the challenge dropped me into a command-line interface with no GUI. 

## Level 1: Starting Simple

The first challenge was straightforward—counting pattern occurrences in a string (like counting ">" symbols that interact with following "<" symbols). A simple loop and counter did the trick:

```java
for (char c : string.toCharArray()) {
    if (c == '>') count++;
    else if (c == '<') result += count;
}
```

{{< githubscrollablefile user="veegit" repo="Algorithms" file="src/main/java/com/vee/algorithms/problems/foobar/GooFooCrossings.java" lang="java" height="500px" >}}

## Level 2: Knight's Moves

The difficulty ramped up with a chess knight problem—finding minimum moves between squares on an 8x8 board. This called for breadth-first search:

```java
Queue<Point> q = new ArrayDeque<>();
q.offer(startPoint);
while (!q.isEmpty()) {
    Point u = q.poll();
    for (Point n : getNeighbours(u, moves, size)) {
        if (!visited[v]) {
            visited[v] = true;
            distance[v] = distance[uIndex] + 1;
            if (v == destIndex) return distance[v];
            q.offer(n);
        }
    }
}
```

It was followed by another problem around a post order traversal of a binary tree

{{< githubscrollablefile user="veegit" repo="Algorithms" file="src/main/java/com/vee/algorithms/problems/foobar/GooFooPostOrderRoot.java" lang="java" height="500px" >}}

## Level 3: Mathematical Challenges

The problems evolved into elaborate sci-fi scenarios involving Commander Lambda and bunny prisoners. One "Lucky Triples" challenge required dynamic programming to count triples where each element divides the next:

```java
if (S[j] % S[i] == 0) {
    num[j]++;           // i -> j is valid step
    triples += num[i];  // extend existing triples
}
```

Another involved XOR checksum calculations for massive datasets. The trick was recognizing that XOR operations from 0 to N follow a pattern every 4 numbers, allowing constant-time computation instead of iterating through millions of values.

{{< githubscrollablefile user="veegit" repo="Algorithms" file="src/main/java/com/vee/algorithms/problems/foobar/GooFooChecksum.java" lang="java" height="500px" >}}

The most complex challenge involved "self-replicating bombs" with astronomically large numbers (up to 10^50). This turned out to be a disguised Euclidean algorithm problem, requiring BigInteger arithmetic and reverse GCD logic.

{{< githubscrollablefile user="veegit" repo="Algorithms" file="src/main/java/com/vee/algorithms/problems/foobar/GooFooMF.java" lang="java" height="500px" >}}

## Stranded on Level 4

One problem, cryptically named "Bounces," remained unsolved and I got busy with life. I never got to complete the challenge since they only give you couple of days to solve it. 

Although I was reached out by a Google Recruiter, it didn't work out at the end as evident from a "not a match" message. But I did end up learning a lot from the challenge and the solutions available in the [repository](https://github.com/veegit/Algorithms/tree/master/src/main/java/com/vee/algorithms/problems/foobar).

![](/images/google-no-recruit.png)