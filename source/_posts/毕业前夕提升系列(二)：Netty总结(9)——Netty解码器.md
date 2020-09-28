---
title: 毕业前夕提升系列(二)：Netty总结(9)——Netty解码器
tag: 
- summary
categories: Netty
---

根据不同的协议来进行数据流到对象的转换，decoder是网络编程中不可或缺的一个步骤。Netty有一个``ByteToMessageDecoder``抽象解码器，上层应用实现该类的``decode()``方法。

### 解码步骤

1. 通过Cumulator积累字节流数据
2. 执行实现ByteToMessageDecoder子类的方法来解析对象
3. 将解析的ByteBuf对象向下传播

## 源码

``ByteToMessageDecoder``抽象类实现了``ChannelInboundHandlerAdapter``，加入到pipeline中，channelRead作为入口进行解码操作。

``channelRead(ChannelHandlerContext ctx, Object msg)``

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
          	//第一次就是当前收到的data
            if (first) {
                cumulation = data;
            } else {
                //1. 累加器进行字节流的累加
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            //2. 执行实现ByteToMessageDecoder子类的方法来解析对象
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            } else if (++ numReads >= discardAfterReads) {
                // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                // See https://github.com/netty/netty/issues/4275
                numReads = 0;
                discardSomeReadBytes();
            }

            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            //3. 将解析的ByteBuf对象向下传播
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```



1. 累加器进行字节流的累加

默认累加器实现``ByteToMessageDecoder#MERGE_CUMULATOR``

```java
public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        try {
            final ByteBuf buffer;
            //1.1 超出累加器的容量
            if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
                // Expand cumulation (by replace it) when either there is not more room in the buffer
                // or if the refCnt is greater then 1 which may happen when the user use slice().retain() or
                // duplicate().retain() or if its read-only.
                //
                // See:
                // - https://github.com/netty/netty/issues/2327
                // - https://github.com/netty/netty/issues/1764
                //扩容
                buffer = expandCumulation(alloc, cumulation, in.readableBytes());
            } else {
                buffer = cumulation;
            }
            //
            buffer.writeBytes(in);
            return buffer;
        } finally {
            // We must release in in all cases as otherwise it may produce a leak if writeBytes(...) throw
            // for whatever release (for example because of OutOfMemoryError)
            in.release();
        }
    }
};
```



2. 执行实现ByteToMessageDecoder子类的方法来解析对象

``callDecode(ctx, cumulation, out)``



- ctx：上下文
- in：这次接收到的数据
- out：得到的字节数据转成功的对象

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        //外层是循环
        while (in.isReadable()) {
            //先mark一下的对象的数量
            int outSize = out.size();

            //如果存在已生成的对象
            if (outSize > 0) {
                //传递到下文
                fireChannelRead(ctx, out, outSize);
                //清理该list
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }

            //mark一下当前数据的可读字节
            int oldInputLength = in.readableBytes();
            //调用子类实现的decode，将in的字节数据解码成对象到out里
            decodeRemovalReentryProtection(ctx, in, out);

            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) {
                break;
            }

            //经过解码后，没有新生成的对象
            if (outSize == out.size()) {
                //而且in的字节流也没有被消耗，说明这一次无法
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    //可能被消耗了部分，需要后面几次继续解码才能形成一个对象
                    continue;
                }
            }

            //没有消耗任何数据就解码出了对象，有问题
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                    StringUtil.simpleClassName(getClass()) +
                    ".decode() did not read anything but decoded a message.");
            }

            //如果是进行单次解码方法调用，直接break
            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```



3. 将解析的ByteBuf对象向下传播

``fireChannelRead(ctx, out, size);``

```java
static void fireChannelRead(ChannelHandlerContext ctx, CodecOutputList msgs, int numElements) {
        for (int i = 0; i < numElements; i ++) {
            ctx.fireChannelRead(msgs.getUnsafe(i));
        }
}
```

