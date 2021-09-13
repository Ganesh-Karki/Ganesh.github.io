---
title: POST IMPLEMENTATION TEST
tags: test
categories: algorithm
---

* TOC
{:toc}

Table of Content
1. TEST 1
2. TEST 2
3. TEST 3
4. TEST 4


# TEST 1

~~~java
This is test implementation 1.
~~~

# TEST 2

~~~java
List<Future<Chunk>> splitFutureList = new ArrayList<>();
while (true){
    line = br.readLine();
    if(line != null){
        chunkRows.add(line);
    }
    if(line == null || chunkRows.size() >= initialChunkSize){
        if(chunkRows.size() > 0){
            final int rn = rowNum;
            final List<String> cr = chunkRows;
            rowNum += chunkRows.size();
            chunkRows = new ArrayList<>();
            Future<Chunk> chunk = threadPoolExecutor.submit(() -> {
                cr.sort(comparator);
                return initialChunk(rn, cr, file);
            });
            splitFutureList.add(chunk);
        }
    }
    if(line == null){
        break;
    }
}
chunkList = splitFutureList.stream().map(this::get).collect(Collectors.toList());
~~~


# TEST 3

~~~java
int currentLevel = INITIAL_CHUNK_LEVEL;
List<Future<Chunk>> mergeFutureList = new ArrayList<>();
while (true) {
    //从队列中获取一组chunk
    List<Chunk> pollChunks = pollChunks(chunkQueue, currentLevel);
    //未取到同级chunk, 表示此级别应合并完成
    if (CollectionUtils.isEmpty(pollChunks)) {
        mergeFutureList.stream().map(this::get).forEach(chunkQueue::add);
        mergeFutureList.clear();
        //chunkQueue 中只有一个元素，表示此次合并是最终合并
        if (chunkQueue.size() == 1) {
            break;
        } else {
            currentLevel++;
            continue;
        }
    }
    Future<Chunk> chunk = threadPoolExecutor.submit(() -> merge(pollChunks, original));
    mergeFutureList.add(chunk);
}
~~~

可以看到合并任务与拆分任务有些不同，拆分任务是在循环退出后才执行`Future.get`，因为拆分不用考虑先后；
而合并任务在每次获取当前阶段的chunk结束时执行`Future.get`，这样才能避免不同的阶段之间产生混乱。

 [完整代码][完整代码]  
    
[上一篇]:https://bit-ranger.github.io/blog/algorithm/large-file-diff/
[完整代码]:https://github.com/bit-ranger/architecture/blob/d9083d2fb71763557e6d4eb6875f9c001fd41596/core/src/main/java/com/rainyalley/architecture/core/arithmetic/sort/FileSorter.java

# TEST 4

在上一篇中，使用单线程，1千万条数据排序耗时13秒；
在同一台电脑上，使用多线程后，耗时6秒，时间减少了一半。

在拆分过程中，每个线程都要在内存中进行排序，
在拆分和合并过程中，每个线程都要持有自己的读写缓冲区，这无疑会增大内存的使用量。

究竟消耗了多少内存，我们可以使用`Java Mission Control`来观察，jdk8的bin目录下`jmc.exe`即为此工具。