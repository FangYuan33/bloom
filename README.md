## 布隆过滤器

- Godoc documentation:  https://pkg.go.dev/github.com/bits-and-blooms/bloom/v3

> This library is used by popular systems such as [Milvus](https://github.com/milvus-io/milvus) and [beego](https://github.com/beego/Beego).

布隆过滤器是一种集合的简洁/压缩表示形式，其主要需求是进行成员查询，即判断一个项是否是集合的成员。**当布隆过滤报告某元素在集合中不存在时，那么它一定不存在；报告某元素存在时，允许出现“假阳性”，有时会错误地包裹某个元素在集合中，而实际上它不存在**，因为它节省了比原始集合少得多的存储空间。

> 理解“假阳性”：在的肯定在，不在的可能在

在构建布隆过滤器时，你需要知道有多少元素（所需的 **容量**）以及你愿意容忍的 **假阳性率** 是多少。常见的假阳性率是 **1%**。**假阳性率越低，所需的内存就越多**。同样，**容量越大，使用的内存也越多**。你可以通过以下方式构建一个能够接收 **100万** 个元素且假阳性率为 **1%** 的布隆过滤器：

```Go
    filter := bloom.NewWithEstimates(1000000, 0.01) 
```

应该谨慎地调用 `NewWithEstimates` 方法：如果你指定的元素数量过小，可能会超过假阳性率的限制。布隆过滤器 **不是一种动态数据结构**：你必须事先知道所需的容量。

该项目中的实现接受用于设置和测试的键为 `[]byte` 类型。因此，要添加一个字符串项，例如 `"Love"`，可以这样做：

```Go
    filter.Add([]byte("Love"))
```

同样地，如果检查 `"Love"` 是否在布隆过滤器中：

```Go
    if filter.Test([]byte("Love"))
```

对于数值数据，我们建议你查看 `encoding/binary` 库。但是，例如，要向过滤器中添加一个 `uint32`：

```Go
    i := uint32(100)
    n1 := make([]byte, 4)
    binary.BigEndian.PutUint32(n1, i)
    filter.Add(n1)
```

### 验证假阳性率

有时，实际的假阳性率可能与理论假阳性率略有不同。我们有一个函数可以估算具有 `_m_` 位和 `_k_` 个哈希函数的布隆过滤器在大小为 `_n_` 的集合中的假阳性率：

```Go
    if bloom.EstimateFalsePositiveRate(20*n, 5, n) > 0.001 ...
```

你可以使用它来验证计算得到的 `m` 和 `k` 参数：

```Go
    m, k := bloom.EstimateParameters(n, fp)
    ActualfpRate := bloom.EstimateFalsePositiveRate(m, k, n)
```

或者：

```Go
    f := bloom.NewWithEstimates(n, fp)
    ActualfpRate := bloom.EstimateFalsePositiveRate(f.m, f.k, n)
```

You would expect `ActualfpRate` to be close to the desired false-positive rate `fp` in these cases.

The `EstimateFalsePositiveRate` function creates a temporary Bloom filter. It is
also relatively expensive and only meant for validation.

在这些情况下，你可以期望 `ActualfpRate` 接近期望的假阳性率 `fp`。`EstimateFalsePositiveRate` 函数会创建一个临时的布隆过滤器。它的开销相对较大，只用于验证。

### 序列化

可以以如下方式向布隆过滤器中读写：

```Go
	f := New(1000, 4)
	var buf bytes.Buffer
	bytesWritten, err := f.WriteTo(&buf)
	if err != nil {
		t.Fatal(err.Error())
	}
	var g BloomFilter
	bytesRead, err := g.ReadFrom(&buf)
	if err != nil {
		t.Fatal(err.Error())
	}
	if bytesRead != bytesWritten {
		t.Errorf("read unexpected number of bytes %d != %d", bytesRead, bytesWritten)
	}
```

> 性能提示：在读取和写入文件或网络连接时，通过使用 `bufio` 实例来包装你的流，可能会获得更好的性能。

Eg: 

```Go
	f, err := os.Create("myfile")
	w := bufio.NewWriter(f)
```

```Go
	f, err := os.Open("myfile")
	r := bufio.NewReader(f)
```

### Design

布隆过滤器有两个参数：`_m_`，用于存储的位数，以及 `_k_`，集合元素的哈希函数数量。（实际的哈希函数也很重要，但这不是该实现的参数）。布隆过滤器由一个 [BitSet](https://github.com/bits-and-blooms/bitset) 支持；通过在每个哈希函数值（取模 `_m_`）的位置设置比特来表示一个键。集合成员资格通过“测试”每个哈希函数值（取模 `_m_`）的位置的比特是否被设置来完成。如果都被设置，则说明该项在集合中。如果该项确实在集合中，布隆过滤器永远不会失败（真正的阳性率为 1.0）；但它容易产生假阳性，技巧在于正确选择 `_k_` 和 `_m_`。

在这个实现中，使用的哈希函数是 [murmurhash](github.com/twmb/murmur3)，一种非加密的哈希函数。

鉴于特定的哈希方案，最好是通过经验来确定这一点。请注意，估计假阳性率会清空布隆过滤器。

### Goroutine safety

一般来说，使用不同的协程访问同一个过滤器是不安全的。它们为性能考虑而未进行同步。如果你想从多个协程访问一个过滤器，你应该提供同步机制。通常，这可以通过使用通道（符合 Go 风格；因此始终只有一个拥有者），或者通过使用 `sync.Mutex` 来序列化操作来完成。例外情况是，如果你从不修改过滤器的内容，那么可以从不同的协程访问同一个过滤器。