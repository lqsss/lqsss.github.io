---
IINAtitle: 毕业前夕提升系列(二)：Netty总结(12)——装饰者模式
tag: 
- summary
categories: Netty
---





## 为什么要使用装饰者模式

动态地将责任附加到对象上。若要扩展功能，装饰者提供了比继承更加有弹性的替代方案

## 形式

- 修饰者和被修饰者对象有相同的超类型
- 可以用一个或者多个修饰者包装一个对象
- 修饰者和被修饰者有相同超类，那么可以在任何需要原始对象（被包装的）场合，用被包装过的对象代替它。
- 装饰者可以在委托给装饰者包装的行为之前或者之后加上自己的行为，来达到特定的目的

## netty中的修饰者模式场景



1. 被修饰者

``ByteBuf``

```java
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf> {
  //
}
```



2. 修饰者基类

 2.1 修饰者基类``WrappedByteBuf``

```java
class WrappedByteBuf extends ByteBuf {
    protected final ByteBuf buf;

    protected WrappedByteBuf(ByteBuf buf) {
        if (buf == null) {
            throw new NullPointerException("buf");
        } else {
            this.buf = buf;
        }
    }
}
```

该类继承了``ByteBuf``，是装饰者的基类，包装成员变量``buf``

2.2 具体修饰类

``SimpleLeakAwareByteBuf``

```java
class SimpleLeakAwareByteBuf extends WrappedByteBuf {

    /**
     * This object's is associated with the {@link ResourceLeakTracker}. When {@link ResourceLeakTracker#close(Object)}
     * is called this object will be used as the argument. It is also assumed that this object is used when
     * {@link ResourceLeakDetector#track(Object)} is called to create {@link #leak}.
     */
    private final ByteBuf trackedByteBuf;
    final ResourceLeakTracker<ByteBuf> leak;

    SimpleLeakAwareByteBuf(ByteBuf wrapped, ByteBuf trackedByteBuf, ResourceLeakTracker<ByteBuf> leak) {
        super(wrapped);
        this.trackedByteBuf = ObjectUtil.checkNotNull(trackedByteBuf, "trackedByteBuf");
        this.leak = ObjectUtil.checkNotNull(leak, "leak");
    }

    SimpleLeakAwareByteBuf(ByteBuf wrapped, ResourceLeakTracker<ByteBuf> leak) {
        this(wrapped, wrapped, leak);
    }
  
  //...
}
```

``UnreleasableByteBuf``

```java
final class UnreleasableByteBuf extends WrappedByteBuf {

    private SwappedByteBuf swappedBuf;

    UnreleasableByteBuf(ByteBuf buf) {
        super(buf instanceof UnreleasableByteBuf ? buf.unwrap() : buf);
    }
  //...
  
  //重写了release，如此进行动态扩展
    @Override
    public boolean release() {
        return false;
    }
}
```

