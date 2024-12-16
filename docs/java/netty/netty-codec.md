# Netty 编解码器

## RocketMQ 自定义解码器 NettyDecoder 基于 LengthFieldBaseFrameDecoder 实现原理

**RocketMQ 基于 LengthFieldBaseFrameDecoder 自定义的编码协议**，共包括5个部分：

+ 4字节：记录报文的总长度，值：1 + 3 + header.length + body.length，不包含记录报文总长度的本身（长度字段）的4字节，这里之所以不算这4字节是因为 LengthFieldBaseFrameDecoder 解码时会自动加上，看后文源码。

*  1字节：记录序列化编码协议类型：0：JSON, 1: ROCKETMQ。
*  3字节：记录头部数据（header）的长度。
*  m字节：存储header数据 (RemotingCommand 对象序列化后的内容)。
*  n字节：存储消息体内容 (RemotingCommand 对象中 body 的内容)。

另外 LengthFieldBaseFrameDecoder 注释中还列举了一些编码协议示例。

**原理分析**：

> 一个完整的数据报文被成为数据帧。

```java
//NettyDecoder

public NettyDecoder() {
    //调用 LengthFieldBaseFrameDecoder构造器，这里的参数表示
    //数据帧最大长度（RocketMQ中默认设置是16777216）；
    //长度字段在数据包中的偏移量0,长度字段的字节长度为4,即数据包开始4个字段的int值表示整个数据包的长度; 
    //lengthAdjustment = 0, initialBytesToStrip = 4
    super(FRAME_MAX_LENGTH, 0, 4, 0, 4);
}

@Override
public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    ByteBuf frame = null;
    //Stopwatch timer = Stopwatch.createStarted();
    try {
        // 经过 LengthFieldBaseFrameDecoder 解码之后，这里 frame 就是除去 lengthField 的数据帧内容，即前面 1 + 3 + header.length + body.length 部分
        frame = (ByteBuf) super.decode(ctx, in);
        if (null == frame) {
            return null;
        }
        // 后面就是业务逻辑部分的解码了
        RemotingCommand cmd = RemotingCommand.decode(frame);
        log.info("NettyDecoder#decode(), cmd={}", cmd);
        //cmd.setProcessTimer(timer);
        return cmd;
    } catch (Exception e) {
        log.error("decode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
        RemotingHelper.closeChannel(ctx.channel());
    } finally {
        if (null != frame) {
            frame.release();
        }
    }
    return null;
}
```



```java
public class LengthFieldBasedFrameDecoder extends ByteToMessageDecoder {
    private final ByteOrder byteOrder;
    //数据帧最大长度
    //数据从Channel中读取出来需要先放到缓冲中，根据编码协议解析出完整的数据数据包后才能取出放到新的ByteBuf或对象中然后交给Pipeline处理，如果一个完整的数据包的数据超过这个限制且配置，Netty会丢弃这个包
    private final int maxFrameLength;
    //长度字段在整个数据包中的偏移量
    private final int lengthFieldOffset;
    //长度字段的长度（占几个字节）
    private final int lengthFieldLength;
    //长度字段结束偏移量, 等于 lengthFieldOffset + lengthFieldLength;
    private final int lengthFieldEndOffset;
    //
    private final int lengthAdjustment;
    private final int initialBytesToStrip;
    private final boolean failFast;
    //是否丢弃太长的帧
    private boolean discardingTooLongFrame;
    private long tooLongFrameLength;
    private long bytesToDiscard;

    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        if (discardingTooLongFrame) {
            discardingTooLongFrame(in);
        }

        if (in.readableBytes() < lengthFieldEndOffset) {
            return null;
        }
		// 数据帧长度字段在ByteBuf中实际的偏移量
        int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;
        // 数据帧的长度，内部是将长度字段转成int返回
        long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);
        if (frameLength < 0) {
            failOnNegativeLengthField(in, frameLength, lengthFieldEndOffset);
        }
		// 数据帧的长度再加上 lengthAdjustment + lengthFieldEndOffset
        // 实际是加上长度字段的长度后的数据帧的长度，所以长度字段的值不包含长度字段本身的长度
        frameLength += lengthAdjustment + lengthFieldEndOffset;

        if (frameLength < lengthFieldEndOffset) {
            failOnFrameLengthLessThanLengthFieldEndOffset(in, frameLength, lengthFieldEndOffset);
        }
        if (frameLength > maxFrameLength) {	//数据帧太长的话丢弃这个数据帧
            exceededFrameLength(in, frameLength);
            return null;
        }

        // 经过上面过滤到这里的数据帧都不会超过 maxFrameLength
        int frameLengthInt = (int) frameLength;
        if (in.readableBytes() < frameLengthInt) {
            return null;
        }
        if (initialBytesToStrip > frameLengthInt) {
            failOnFrameLengthLessThanInitialBytesToStrip(in, frameLength, initialBytesToStrip);
        }
        // 跳过 initialBytesToStrip 字节，这里其实是跳过 lengthField
        in.skipBytes(initialBytesToStrip);

        // extract frame
        int readerIndex = in.readerIndex();
        int actualFrameLength = frameLengthInt - initialBytesToStrip;
        // 这里提取出来的数据就是除去 lengthField 的数据帧内容，即前面 1 + 3 + header.length + body.length 部分
        ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
        in.readerIndex(readerIndex + actualFrameLength);
        return frame;
    }
    
    ...
}
```

