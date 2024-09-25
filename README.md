
业余时间用 .net 写了一个免费的在线客服系统：升讯威在线客服与营销系统。


时常有朋友问我性能方面的问题，正好有一个真实客户，在线的访客数量达到了 2000 人。在争得客户同意后，我录了一个视频。


升讯威在线客服系统可以在极低配置的服务器环境下，轻松应对这种情况，依然可以做到：


### 消息毫秒级送达，操作毫秒级响应




 您的浏览器不支持 video 标签。

## 性能



> 以官方在线使用环境为例，每日处理 HTTPS 请求数大于 16 万次， PV 请求大于 25 万 次的情况下，服务端主程序内存占用小于 300MB，服务器 CPU 占用小于 5%。



> 每日处理 HTTPS 请求数大于 16 万次：


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/a144abd2-6bf2-4fb1-a6e5-66c42ee4fd09.png)



> 每日处理 PV 请求大于 25 万 次：


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/5c1c9f12-18ef-4b85-a3fb-7fab1ae2c64f.png)



> 服务端主程序内存占用小于 300MB：


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/0b5fd54a-2a54-4d75-9087-8b7f25e2c773.png)



> 服务器 CPU （Intel Xeon Platinum 8163 / 4 核 2\.5 GHz） 占用稳定约 5%：


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/a61caa23-7318-4ce4-a1fa-19cc550414e2.png)


## 安全性


* 访客端使用 https 与 wss 安全连接，数据全程加密传输。
* 客服端数据报文使用 AES 加密传输。（Advanced Encryption Standard，美国联邦政府区块加密标准）。
* 可以 100% 私有化部署在您的自有服务器。



> 拦截的报文中消息以密文传输：


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/7c51c714-40e2-41e4-87be-186214bdc2a4.png)




---


## 实现效果


客服端


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/1d43bed9-b5a8-4941-a2c6-56ef4e3152cf.png)


访客端


![](https://docs-api.shengxunwei.com/StaticFiles/Upload/15ea0fe9-0392-4acc-bc5a-12735d16d537.png)




---


## 怎么做到的？技术详解


### 使用NetworkStream的TCP服务器


在Pipelines之前用.NET编写的典型代码如下所示：



```
async Task ProcessLinesAsync(NetworkStream stream)
{
    var buffer = new byte[1024];
    await stream.ReadAsync(buffer, 0, buffer.Length);

    // 在buffer中处理一行消息
    ProcessLine(buffer);
}

```

此代码可能在本地测试时正确工作，但它有几个潜在错误：


一次ReadAsync调用可能没有收到整个消息（行尾）。
它忽略了stream.ReadAsync()返回值中实际填充到buffer中的数据量。（译者注：即不一定将buffer填充满）
一次ReadAsync调用不能处理多条消息。
这些是读取流数据时常见的一些缺陷。为了解决这个问题，我们需要做一些改变：


我们需要缓冲传入的数据，直到找到新的行。
我们需要解析缓冲区中返回的所有行



```
async Task ProcessLinesAsync(NetworkStream stream)
{
    var buffer = new byte[1024];
    var bytesBuffered = 0;
    var bytesConsumed = 0;

    while (true)
    {
        var bytesRead = await stream.ReadAsync(buffer, bytesBuffered, buffer.Length - bytesBuffered);
        if (bytesRead == 0)
        {
            // EOF 已经到末尾
            break;
        }
        // 跟踪已缓冲的字节数
        bytesBuffered += bytesRead;

        var linePosition = -1;

        do
        {
            // 在缓冲数据中查找找一个行末尾
            linePosition = Array.IndexOf(buffer, (byte)'\n', bytesConsumed, bytesBuffered - bytesConsumed);

            if (linePosition >= 0)
            {
                // 根据偏移量计算一行的长度
                var lineLength = linePosition - bytesConsumed;

                // 处理这一行
                ProcessLine(buffer, bytesConsumed, lineLength);

                // 移动bytesConsumed为了跳过我们已经处理掉的行 (包括\n)
                bytesConsumed += lineLength + 1;
            }
        }
        while (linePosition >= 0);
    }
}

```

这一次，这可能适用于本地开发，但一行可能大于1KiB（1024字节）。我们需要调整输入缓冲区的大小，直到找到新行。


因此，我们可以在堆上分配缓冲区去处理更长的一行。我们从客户端解析较长的一行时，可以通过使用ArrayPool避免重复分配缓冲区来改进这一点。



```
async Task ProcessLinesAsync(NetworkStream stream)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
    var bytesBuffered = 0;
    var bytesConsumed = 0;

    while (true)
    {
        // 在buffer中计算中剩余的字节数
        var bytesRemaining = buffer.Length - bytesBuffered;

        if (bytesRemaining == 0)
        {
            // 将buffer size翻倍 并且将之前缓冲的数据复制到新的缓冲区
            var newBuffer = ArrayPool<byte>.Shared.Rent(buffer.Length * 2);
            Buffer.BlockCopy(buffer, 0, newBuffer, 0, buffer.Length);
            // 将旧的buffer丢回池中
            ArrayPool<byte>.Shared.Return(buffer);
            buffer = newBuffer;
            bytesRemaining = buffer.Length - bytesBuffered;
        }

        var bytesRead = await stream.ReadAsync(buffer, bytesBuffered, bytesRemaining);
        if (bytesRead == 0)
        {
            // EOF 末尾
            break;
        }

        // 跟踪已缓冲的字节数
        bytesBuffered += bytesRead;

        do
        {
            // 在缓冲数据中查找找一个行末尾
            linePosition = Array.IndexOf(buffer, (byte)'\n', bytesConsumed, bytesBuffered - bytesConsumed);

            if (linePosition >= 0)
            {
                // 根据偏移量计算一行的长度
                var lineLength = linePosition - bytesConsumed;

                // 处理这一行
                ProcessLine(buffer, bytesConsumed, lineLength);

                // 移动bytesConsumed为了跳过我们已经处理掉的行 (包括\n)
                bytesConsumed += lineLength + 1;
            }
        }
        while (linePosition >= 0);
    }
}

```

这段代码有效，但现在我们正在重新调整缓冲区大小，从而产生更多缓冲区副本。它将使用更多内存，因为根据代码在处理一行行后不会缩缓冲区的大小。为避免这种情况，我们可以存储缓冲区序列，而不是每次超过1KiB大小时调整大小。


此外，我们不会增长1KiB的 缓冲区，直到它完全为空。这意味着我们最终传递给ReadAsync越来越小的缓冲区，这将导致对操作系统的更多调用。


为了缓解这种情况，我们将在现有缓冲区中剩余少于512个字节时分配一个新缓冲区：



```
public class BufferSegment
{
    public byte[] Buffer { get; set; }
    public int Count { get; set; }

    public int Remaining => Buffer.Length - Count;
}

async Task ProcessLinesAsync(NetworkStream stream)
{
    const int minimumBufferSize = 512;

    var segments = new List();
    var bytesConsumed = 0;
    var bytesConsumedBufferIndex = 0;
    var segment = new BufferSegment { Buffer = ArrayPool<byte>.Shared.Rent(1024) };

    segments.Add(segment);

    while (true)
    {
        // Calculate the amount of bytes remaining in the buffer
        if (segment.Remaining < minimumBufferSize)
        {
            // Allocate a new segment
            segment = new BufferSegment { Buffer = ArrayPool<byte>.Shared.Rent(1024) };
            segments.Add(segment);
        }

        var bytesRead = await stream.ReadAsync(segment.Buffer, segment.Count, segment.Remaining);
        if (bytesRead == 0)
        {
            break;
        }

        // Keep track of the amount of buffered bytes
        segment.Count += bytesRead;

        while (true)
        {
            // Look for a EOL in the list of segments
            var (segmentIndex, segmentOffset) = IndexOf(segments, (byte)'\n', bytesConsumedBufferIndex, bytesConsumed);

            if (segmentIndex >= 0)
            {
                // Process the line
                ProcessLine(segments, segmentIndex, segmentOffset);

                bytesConsumedBufferIndex = segmentOffset;
                bytesConsumed = segmentOffset + 1;
            }
            else
            {
                break;
            }
        }

        // Drop fully consumed segments from the list so we don't look at them again
        for (var i = bytesConsumedBufferIndex; i >= 0; --i)
        {
            var consumedSegment = segments[i];
            // Return all segments unless this is the current segment
            if (consumedSegment != segment)
            {
                ArrayPool<byte>.Shared.Return(consumedSegment.Buffer);
                segments.RemoveAt(i);
            }
        }
    }
}

(int segmentIndex, int segmentOffest) IndexOf(List segments, byte value, int startBufferIndex, int startSegmentOffset)
{
    var first = true;
    for (var i = startBufferIndex; i < segments.Count; ++i)
    {
        var segment = segments[i];
        // Start from the correct offset
        var offset = first ? startSegmentOffset : 0;
        var index = Array.IndexOf(segment.Buffer, value, offset, segment.Count - offset);

        if (index >= 0)
        {
            // Return the buffer index and the index within that segment where EOL was found
            return (i, index);
        }

        first = false;
    }
    return (-1, -1);
}

```

此代码只是得到很多更加复杂。当我们正在寻找分隔符时，我们同时跟踪已填充的缓冲区序列。为此，我们此处使用List查找新行分隔符时表示缓冲数据。其结果是，ProcessLine和IndexOf现在接受List作为参数，而不是一个byte\[]，offset和count。我们的解析逻辑现在需要处理一个或多个缓冲区序列。


我们的服务器现在处理部分消息，它使用池化内存来减少总体内存消耗，但我们还需要进行更多更改：


* 我们使用的byte\[]和ArrayPool的只是普通的托管数组。这意味着无论何时我们执行ReadAsync或WriteAsync，这些缓冲区都会在异步操作的生命周期内被固定（以便与操作系统上的本机IO API互操作）。这对GC有性能影响，因为无法移动固定内存，这可能导致堆碎片。根据异步操作挂起的时间长短，池的实现可能需要更改。
* 可以通过解耦读取逻辑和处理逻辑来优化吞吐量。这会创建一个批处理效果，使解析逻辑可以使用更大的缓冲区块，而不是仅在解析单个行后才读取更多数据。这引入了一些额外的复杂性
* 我们需要两个彼此独立运行的循环。一个读取Socket和一个解析缓冲区。
* 当数据可用时，我们需要一种方法来向解析逻辑发出信号。
* 我们需要决定如果循环读取Socket“太快”会发生什么。如果解析逻辑无法跟上，我们需要一种方法来限制读取循环（逻辑）。这通常被称为“流量控制”或“背压”。
* 我们需要确保事情是线程安全的。我们现在在读取循环和解析循环之间共享多个缓冲区，并且这些缓冲区在不同的线程上独立运行。
* 内存管理逻辑现在分布在两个不同的代码段中，从填充缓冲区池的代码是从套接字读取的，而从缓冲区池取数据的代码是解析逻辑。
* 我们需要非常小心在解析逻辑完成之后我们如何处理缓冲区序列。如果我们不小心，我们可能会返回一个仍由Socket读取逻辑写入的缓冲区序列。




---


## 在线体验或下载完整私有化部署包：



> [https://kf.shengxunwei.com](https://github.com)


希望能够打造： **开放、开源、共享。努力打造 .net 社区的一款优秀开源产品。**


### 钟意的话请给个赞支持一下吧，谢谢\~


 本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
