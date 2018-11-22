# Stately

Stately is a state utility library to facilitate the Native side of Kotlin Multiplatform.

The library consists of some useful expect/actual definitions, as well as a set of multithreaded collection classes that 
will allow multithreaded mutation in Kotlin/Native.

Kotlin/Native has a fairly different model of concurrency than what JVM developers are used to. This includes very 
different rules around sharing state. These rules are intended to reduce the likelihood of concurrency related issues.

One of the rules is that shared state is immutable. While generally a good idea, some shared state mutability is also 
pretty useful. Kotlin/Native provides a set of Atomics to allow for some state to be changed in a safe way. Stately provides
a set of collection classes that use Atomics under the hood to function, as well as some multiplatform definitions to 
make using Atomics and Kotlin/Native related state management simpler.

The documentation here is about the structure, or the *what* of the library. A follow on blog post will talk a bit more 
about the *why*.

## Status

[![Build status](https://build.appcenter.ms/v0.1/apps/fcda190b-7ec8-43b7-8216-6fc1be836332/branches/master/badge)](https://appcenter.ms)  [ ![Download](https://api.bintray.com/packages/touchlabpublic/kotlin/Stately/images/download.svg) ](https://bintray.com/touchlabpublic/kotlin/Stately/_latestVersion) 

This a pretty early version. Expect changes in the near future.

## Maven

```groovy
repositories {
    maven { url 'https://dl.bintray.com/touchlabpublic/kotlin' }
}

dependencies {
    implementation 'co.touchlab.stately:Stately:$statelyVersion'
}
```

## Annotations

Kotlin/Native provides some annotations to support state characteristics. These do not have common kotlin analogs, which 
makes multiplatform code somewhat more difficult to define. Stately defines expect/actual parallels which have no impact 
on JVM or JS.

### ThreadLocal

Place on top level var or object to make it available as thread local. See [`ThreadLocal`](https://github.com/JetBrains/kotlin-native/blob/master/runtime/src/main/kotlin/kotlin/native/Annotations.kt#L51).

### SharedImmutable

Top level var will default to main thread only. To share it as an immutable, add this annotation. See [`SharedImmutable`](https://github.com/JetBrains/kotlin-native/blob/master/runtime/src/main/kotlin/kotlin/native/Annotations.kt#L59).

## Functions/Extensions

### freeze()

Multiplatform definition for Kotlin/Native 'freeze'.

### isFrozen()

Call on any object to find out if it's frozen.

### isNativeFrozen()

Will return true on non-native platforms. On native, will return true if frozen. Usually you're trying to figure out if
an instance can be shared among threads. Because the rules are different, the answer is always "true" on JVM and JS. This
method makes that logic simpler.

### isNative (val)

Simple way to find out if we're in a native context.

### Iterator.toList()

Converts an Iterator to a List. This is an extension function that works on any Iterator. Mostly useful for testing.

## Collections

### Copy On Write List

A MutableList that updates a stable copy on each edit. When you get an iterator, that iterator remains stable regardless
of operations happening on the list. The JVM version is actually [https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CopyOnWriteArrayList.html](CopyOnWriteArrayList.java). 
The Native version updates a frozen ArrayList internally by copying and replacing on edits.

```kotlin
val list:MutableList<SampleData> = frozenCopyOnWriteList<SampleData>()
```

### Shared Linked List

Rather than copying the full list on each edit, this structure uses atomic references in the nodes to allow modification
that is reasonably performant relative to the Copy On Write list in situations that may have larger lists and/or frequent updates.

It's important to note, performance is OK *relative* to copying the list on each edit. The list locks aggressively to 
achieve consistency. There are lockless algorithms for linked lists that may be explored in the future.

There are two versions.

### SharedLinkedList

Iterators reflect changes to the underlying list. If values are added/removed while iterating, your iterator will
see them.

```kotlin
val list:MutableList<SampleData> = frozenLinkedList<SampleData>()

```

### CopyOnIterateLinkedList

Iterators are stable. Calling iterator creates an internal copy of the list which the iterator uses. Underlying changes
to the source list are NOT reflected in the iterator. This will trigger a lock and copy on calling iterator, which 
may be a performance consideration with large lists.

Similar to CopyOnWriteList, and likely to perform better in general cases. It is a linked list, however, so indexed 
access requires a scan.

```kotlin
val list:MutableList<SampleData> = frozenLinkedList<SampleData>(stableIterator = true)

```

### Hash Map

A standard hash map architected as a simple Java-like implementation. Optionally pass in initial capacity and load factor.
Will grow as needed. This also locks on most operations, so from a performance perspective I would expect it 
to fall pretty far behind the standard single threaded implementation, but from an order of operations perspective, it should be similar.

One note. In Java 8 (not sure about kotlin native), in buckets, after length 8, a tree is used instead of a linked list.
Worst case performance then flattens out to lgN (ish). Java 7, and our implementation, do not, so expect worst case to
be N. Try to use good hashes.

```kotlin
val sharedMap:MutableMap<String, SampleData> = frozenHashMap<String, SampleData>()
```

### LRU Cache

Hash map and linked list combined into an LRU cache implementation.

You can supply a callback to be notified when an entry is removed.

```kotlin
val lruRemovedCount = AtomicInt(0)
val lruCache = frozenLruCache<String, SampleData>(5) {
    lruRemovedCount.increment()
}
```

Note that the lambda for onRemove is optional. However, since it is multithreaded, be aware that anything in there will
be frozen.

## Usage

See the [Sample App](Sample/) for usage.


License
=======

    Copyright 2018 Touchlab, Inc.

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
