---
title: skipList&redis
tag: 
- implement
categories: redis
---



redis里的sorted set，内部实现使用的调表



## 为什么使用skipList

### redis里的zset需要实现哪些功能？

- 支持随机的插入和删除
- 支持有序
- 支持去重



1. 在有序性的情况下，数组不是很合适，插入较小或者删除较小的节点，需要移动
2. 使用单向链表的话同样也会有问题，如果想插入查找某个节点时，需要首先定位到该节点O(N)，(数组一般是可以通过二分查找O(nlogn))
3. 引入了组长制，每一部分的数据都会有个代表进行层级跃升，有些节点包含了多个层级，那么在查找的时候从最高层级进行遍历，层级递减来减少查找的时间复杂度

![](https://i.loli.net/2020/08/02/myELxf7qGOMsI46.png)

如图，由1->6的单向链表变成下面有层级的链表，比如查找5这个节点的过程不再是从头结点1开始往后遍历，而从最高层级level1开始遍历，查找路径就变成了2->4->5



### 基础skipList的实现

#### 插入节点的层级(level)如何确定

- 每个节点肯定都有第一层指针(level0)

- 跳表维护了一个概率p，p代表跨越当前层级i，进入i+1的概率。
- 有最大层数限制 maxLevel

获取level的代码如下:

```java
public int getLevelRandomly() {
    int level = 1;
    //Math.random() < probability 跨越一层的概率为p
    while (Math.random() < probability && level < maxLevel) {
        level++;
    }

    return level;
}
```

影响层级的是两个因素：

1. 概率p
2. 最大层数maxLevel



#### 分析参数与性能

产生层级越高的节点，概率越低

- 节点层级为1是100%，层级恰好为1的概率是(1-p)
- 节点层级恰好是2的概率 p(跨越第一层) * p(但没有跨越到第三层) = p * (1 - p )  
- ...

计算出平均节点： 

![](https://i.loli.net/2020/08/02/ON3hRcBpPjvr8wY.png)

### skipList与平衡树的比较

|          | 实现 | 内存占用 | 查找 |
| -------- | ---- | -------- | ---- |
| skipList | 插入和删除只用修改目标节点的前后节点，简单 | 主要是每个节点的指针，每个节点的占用跟p有关，1/(1-p) | 支持范围查找，只用找到范围的小值节点后x，在x.level[0]后进行遍历即可；单key查找，时间复杂度O(logn) |
| 平衡树   | 插入删除可能会引发子树的调整，逻辑较复杂 | 每个节点包含两个指针(左右) | 范围查找，找到最小值后，需要中序遍历，寻找不超过它的大值；单key查找O(logn) |



## redis里对skipList扩展

- 支持根据排名范围查找
- 支持通过key找到score
- 支持查找排名
- 支持同样的score，key按字典序排序

![](https://i.loli.net/2020/08/02/LOAiVolzBwjrTYU.jpg)



1. 查找排名：每一个节点有一个span字段用来记录节点的当前层级的next指针跨越多少个节点，方便后续查找排名，只用累加span即可
2. 支持通过key找到score：redis里有字典表(dict)以k-v形式存储key和score，同时每个节点也指向了这一份数据
3. 支持根据排名范围查找：根据span的累加找到小值的节点(x)，然后在x->level[0]向后遍历直到大值





## redis里skipList的实现

这里用java实现了部分方法

```java
/**
 * feat.
 * 1. 满足基础的skipList
 * 2. skipList进行扩展，支持以下操作;
 * 2.1 zscore:           查找key对应的数据
 * 2.2 zrevrank:         查看key的排名
 * 2.3 zrange:        查看多少名内的数据
 * 2.4 zrevrangebyscore: 查看分数范围内的数据，从小到大排序
 *
 * @author lqs
 * @version 1.0
 * @date 2020/7/26 12:20 下午
 */
public class SkipList {

    /**
     * 默认的最大层级
     */
    private static final int DEFAULT_MAX_LEVEL = 32;

    /**
     * 默认跨越层级为p
     */
    private static final double DEFAULT_PROBABILITY = 0.25;

    private SkipListNode head;

    private SkipListNode tail;

    /**
     * 长度
     */
    private int length;

    /**
     * 最大层级
     */
    private int maxLevel;

    /**
     * 层级
     */
    private int level;

    /**
     * 当前层级为第n层，当前节点分配到 (n+1)层的概率
     */
    private double probability;

    public SkipList(double probability, int maxLevel) {
        this.probability = probability;
        this.maxLevel = maxLevel;
        this.length = 0;
        this.level = 1;
        this.head = new SkipListNode(maxLevel);
    }
  
     public SkipList() {
        this(DEFAULT_PROBABILITY, DEFAULT_MAX_LEVEL);
    }
  
    public SkipListNode insert(int score, String dictName) {
        int level = getLevelRandomly();

        //update数组：用来记录查找插入节点的路径
        SkipListNode[] update = new SkipListNode[Math.max(level, this.level)];
        SkipListNode newNode = new SkipListNode(score, maxLevel, dictName);

        //rank数组：用来记录与update数组对应的span
        int[] rank = new int[update.length];

        if (level > this.level) {
            for (int i = this.level; i < level; i++) {
                update[i] = head;
                rank[i] = 0;
                head.getPointers()[i].setSpan(length);
            }

            //更新最大跳表的level
            this.level = level;
        }

        //找到每层最近的左节点，方便后续插入
        for (int i = 0; i < level; i++) {

            SkipListNode cur = head;
            SkipListNode next = cur.getPointers()[i].getNext();
            while (next != null && (next.getScore() < score || next.getScore() == score && next.getDictName().compareTo(dictName) > 0)) {
                //这里累加左节点
                rank[i] += cur.getPointers()[i].getSpan();
                cur = next;
                next = cur.getPointers()[i].getNext();
            }

            update[i] = cur;
        }

        for (int i = 0; i < level; i++) {
            SkipListNode.Pointer pointer = update[i].getPointers()[i];
            pointer.setNext(newNode);
            int span = pointer.getSpan();

            newNode.getPointers()[i].setSpan(span - (rank[0] - rank[i]));
            //+1是包含新的节点
            pointer.setSpan(rank[0] - rank[i] + 1);
        }

        if (update[0] == null) {
            newNode.setPrev(head);
        } else {
            newNode.setPrev(update[0]);
        }

        SkipListNode next = newNode.getPointers()[0].getNext();
        if (next != null) {
            next.setPrev(newNode);
        } else {
            tail = newNode;
        }

        length++;
        return newNode;
    }

    public int getLevelRandomly() {
        int level = 1;
        //Math.random() < probability 跨越一层的概率为p
        while (Math.random() < probability && level < maxLevel) {
            level++;
        }

        return level;
    }
  
  
  /**
     * 根据字典名查找对应的数据排名，从头向尾
     *
     * @param dictName
     * @return
     */
    public int zrank(String dictName) {
        //1. 根据zset的dictName从dict中获取score
        //2. 从header的level - 1 开始往后遍历，累加游走节点的span
        Dict dictObj = (Dict) getObjectFromDict(dictName);
        double score = dictObj.score;
        int span = 0;

        SkipListNode curNode = head;
        SkipListNode next;

        for (int i = level - 1; i >= 0; i--) {
            SkipListNode.Pointer curNodePointer = curNode.getPointers()[i];
            next = curNodePointer.getNext();

            while (next != null && (next.getScore() < score || next.getScore() == score && dictName.compareTo(next.getDictName()) <= 0)) {

                span += curNodePointer.getSpan();
                curNode = next;
                next = curNode.getPointers()[i].getNext();
            }

            //已经找到了本身该节点
            if (dictObj.equals(new Dict(curNode.getDictName(), curNode.getScore()))) {
                return span;
            }
        }

        return 0;
    }
  
    /**
     * 查找在排名范围内的字典数据
     *
     * @param low
     * @param high
     * @return
     */
    public List<String> zrange(int low, int high) {
        //1. 从最高层往第0层遍历累加span(rank),找到rank等于low的节点target，
        //2. 遍历第0层的target往后的 high - low

        SkipListNode curNode = this.head;
        SkipListNode next;
        int rank = 0;

        for (int i = level; i >= 0; i--) {

            next = curNode.getPointers()[i].getNext();
            while (next != null && (rank + curNode.getPointers()[i].getSpan()) <= low) {
                rank += curNode.getPointers()[i].getSpan();
                curNode = next;
                next = curNode.getPointers()[i].getNext();
            }

            if (rank == low) {
                break;
            }
        }

        List<String> res = new ArrayList<>();
        res.add(curNode.getDictName());
        for (int i = low; i < high; i++) {
            SkipListNode next1 = curNode.getPointers()[0].getNext();
            if (next1 == null) {
                break;
            }

            curNode = next1;
            res.add(curNode.getDictName());
        }

        return res;
    }
}
```





## 参考

[Redis 为什么用跳表而不用平衡树？](https://juejin.im/post/57fa935b0e3dd90057c50fbc)

[Redis源码剖析--跳跃表zskiplist](https://juejin.im/post/57fa935b0e3dd90057c50fbc)

[redis的设计与实现-有序集合对象](http://redisbook.com/preview/object/sorted_set.html)

