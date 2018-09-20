# 以太坊挖矿源码：ethash算法 #
> 本文具体分析以太坊的共识算法之一：实现了POW的以太坊共识引擎ethash。
>  
> 
>  关键字：ethash，共识算法，pow，Dagger Hashimoto，ASIC，struct{}，nonce，FNV hash，位运算，epoch
>  

**Ethash**

前面我们分析了以太坊挖矿的源码，挖了一个共识引擎的坑，研究了DAG有向无环图的算法，这些都是本文要研究的Ethash的基础。Ethash是目前以太坊基于POW工作量证明的一个共识引擎（也叫挖矿算法）。它的前身是Dagger Hashimoto算法。

**Dagger Hashimoto**

作为以太坊挖矿算法Ethash的前身，Dagger Hashimoto的目的是：



- 抵制矿机（ASIC，专门用于挖矿的芯片）


- 轻客户端验证


- 全链数据存储

Dagger和Hashimoto其实是两个东西，

**Hashimoto算法**

是这个人Thaddeus Dryja创造的。旨在通过IO限制来抵制矿机。在挖矿过程中，使内存读取限制条件，由于内存设备本身会比计算设备更加便宜以及普遍，在内存升级优化方面，全世界的大公司也都投入巨大，以使内存能够适应各种用户场景，所以有了随机访问内存的概念RAM，因此,现有的内存可能会比较接近最优的评估算法。Hashimoto算法使用区块链作为源数据，满足了上面的1和3的要求。

**Dagger算法**

是这个人Vitalik Buterin发明的。它利用了有向无环图DAG同时实现了Memory-Hard Function内存计算困难但易于验证Memory-easy verification的特性（我们知道这是哈希算法的重要特性之一）。它的理论依据是基于每个特定场合nonce只需要大型数据总量树的一小部分，并且针对每个特定场合nonce的子树的再计算是被禁止挖矿的。因此，需要存储树但也支持一个独立场合nonce的验证价值。Dagger算法注定要替代现存的仅内存计算困难的算法，例如Scrypt（莱特币采用的），它是计算困难同时验证亦困难的算法，当他们的内存计算困难度增加至真正安全的水平，验证的困难度也随之难上加难。然而，Dagger算法被证明是容易受到Sergio Lerner发明的共享内存硬件加速技术，随后在其他路径的研究方面，该算法被遗弃了。

**Memory-Hard Function**

直接翻译过来是内存困难函数，这是为了地址矿机而诞生的一种思想。我们知道挖矿是靠我们的电脑，但是有些硬件厂商会制造专门用于挖矿的硬件设备，它们并不是一台完整的PC机，例如ASIC、GPU以及FPGAs（我们经常能听到GPU挖矿等）。所以这些作为矿机的设备是超越普通PC挖矿的存在，这是不符合我们区块链的去中心化精神的，所以我们要让挖矿设备平等。

 那么该如何让挖矿设备是平等的呢？
 

上面谈到Dagger算法的时候其实提到了，这里换一种方式再来介绍一下，现在CPU都是多核的，如果从计算能力来讲，CPU有几核就可以模拟几台设备同时平行挖矿，自然效率就高些，但是这里采用的衡量对象是内存，一台电脑只有一个总内存。我们做过java多线程开发的朋友知道，无论机器性能有多高，但我们写的程序就是单线程的，那么这个程序运行在高配多核电脑和低配单核电脑的区别不大，只要他们的单核运算能力和内存大小一样即可。所以也是这个原理，通过Dagger算法，我们将挖矿流程锁定在以内存为衡量标准的硬件性能上，只要通过“塞一堆数据到内存中”的方式，让多核平行处理发挥不出来，降低硬件的运算优势，只与内存大小有关，这样无论是PC机还是ASIC、GPU以及FPGAs，都可达到平等挖矿的诉求，这也是ASIC-resistant原理，目前抵制矿机的主要手段。

**两个问题的研究**

在Dagger以及Dagger Hashimoto算法中，有两个问题的研究是被搁置的，



- 基于区块链的工作量证明：一个POW函数包括了运行区块链上的合约。该方法被抛弃是因为这是一个长期的攻击缺陷，因为攻击者能够创建分叉，然后通过一个包含秘密的快速“trapdoor”井盖门的运行机制的合约在该分叉上殖民。


- 随机环路：一个POW函数由这个人Vlad Zamfir开发，包含了每1000个场合nonces就生成一个新的程序的功能。本质上来讲，每次选择一个新的哈希函数，会比可重配置的FPGAs(可重编程的芯片，不必重新焊接电路板就可通过软件技术重新自定义硬件功能)更快。该方法被暂时搁置，是因为它很难看到有什么机制可以用来生成随机程序是足够全面，因此它的专业化收益是较低的。然而，我们并没有看到为什么这个概念无法让它生效的根本原因，所以暂时搁置。

**Dagger Hashimoto算法**



>  （区别于Hashimoto）Dagger Hashimoto不是直接将区块链作为数据源，而是使用一个1GB的自定义生成的数据集cache。
 

这个数据集是基于区块数据每N个块就会更新。该数据集是使用Dagger算法生成，允许一个自己的高效计算，特定于每个轻客户端校验算法的场合nonce。



>  （区别于Dagger）Dagger Hashimoto克服了Dagger的缺陷，它用于查询区块数据的数据集是半永久的，只有在偶然的间隔才会被更新（例如每周一次）。这意味着生成数据集将非常容易，所以Sergio Lerner的争议共享内存加速变得微不足道了。
 

**挖矿补充**

前面我已经写了一盘关于挖矿的文章了，这一节是挖矿的补充内容。
> 
 

> 以太坊将过渡到POS（proof-of-stake)，代替传统的POW，挖矿将会被淘汰掉，所以现在不推荐再去做一名矿工（前期购买设备等成本较大，POS实现前未必能回本）。
 

挖掘以太币=网络安全=验证估算

目前以太坊的POW算法是Ethash，

 

> Ethash算法包含找到一个nonce值输入到一个算法中，得到的结果是低于一个基于特定困难度的阀值。
 

POW算法的关键点是除了暴力枚举，没有任何办法可以找到这个nonce值，但对于验证输出的结果是非常简单容易的。如果输出结果有一个均匀分布，我们就可以保证找到一个nonce值的平均所需时间取决于那个难度阀值，因此我们可以通过调整难度阀值来控制找到一个新块的时间，这就是控制出块速度的原理。

**DAG**

Ethash的POW是memory-hard，支持矿机抵御。这意味着POW计算需要选择一个固定的依赖于nonce值和块头的资源的子集。

 这个资源(大约1G大小)就是DAG!
 

**一世epoch**

每3万个块会花几个小时的时间生成一个有向无环图DAG。这个DAG被称为epoch，一世（为了好记，refer个秦二世）。DAG只取决于区块高度，它可以被预生成，如果没有预生成的话，客户端需要等待预生成流程结束以后才能继续出块操作。除非客户端真实的提前预缓存了DAG，否则在每个epoch的过渡期间，网络可能会经历一个巨大的区块延迟。

 

> 特例：当你从头启动一个结点时，挖矿工作只会在创建了现世DAG以后启动。
 

**挖矿奖励**

有三部分：



- 静态区块创建奖励，精确发放3以太币作为奖励。


- 当前区块包含的所有交易的gas钱，随着时间推移，gas会越来越便宜，获得的gas总和奖励会低于静态区块创建奖励。


- 叔块奖励，整块奖励的1/32。

**Ethash**

Ethash算法路线图：



- 存在一个种子seed，通过扫描块头为每个块计算出来那个点。


- 根据这个种子seed，可以计算一个16MB的伪随机缓存cache，轻客户端存储这个缓存。


- 从这个缓存cache中，我们能够生成一个1GB的数据集，该数据集中的每一项都取决于缓存中的一小部分。完整客户端和矿工存储了这个数据集，数
据集随着时间线性增长。


- 挖矿工作包含了抓取数据集的随机片以及运用哈希函数计算他们。校验工作能够在低内存的环境下完成，通过使用缓存再次生成所需的特性数据集的片段，所以你只需要存储缓存cache即可。
 

> 以上提到的大数据集是每3万个块更新一次，所以绝大多数的矿工的工作是读取该数据集而不是改变它。
 

**pkg ethash源码分析**

以上我们将所有的概念抽象梳理了一下，包括POW，挖矿，Ethash原理流程等，下面我们带着这些理论知识走进源代码中去分析具体的实现。正如我们的题目，本文主要分析的是ethash算法，因此整个源码范围仅限于go-ethereum/consensus/ethash包，该包实现了ethash pow的共识引擎。

**入口**

分析源码要有个入口，这个入口就是在《以太坊源码机制：挖矿》中挖下的坑“Seal方法”，原文留下了这个印子，在本文进行展开讨论。

在go-ethereum/consensus/consensus.go 接口中定义了如下的方法，正是对应上面的“Seal方法”，该接口方法的定义如下：


    Seal(chain ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error)//该方法通过输入一个包含本地矿工挖出的最高区块在主干上生成一个新块。
参数有ChainReader，Block，stop结构体信号，返回一个主链上的新出的块实体。



- ChainReader

    // 定义了一些方法，用于在区块头验证以及叔块验证期间，访问本地区块链。

    type ChainReader interface {
    // 获取区块链的链配置
    Config() *params.ChainConfig
    
    // 从本地链获取当前块头
    CurrentHeader() *types.Header
    
    // 通过hash和number从主链中获取一个区块头
    GetHeader(hash common.Hash, number uint64) *types.Header
    
    // 通过number从主链中获取一个区块头
    GetHeaderByNumber(number uint64) *types.Header
    
    // 通过hash从主链中获取一个区块头
    GetHeaderByHash(hash common.Hash) *types.Header
    
    // 通过hash和number从主链中获取一个区块
    GetBlock(hash common.Hash, number uint64) *types.Block
    }

总结，ChainReader定义了几个方法：从本地区块链获取配置、区块头，从主链中获取区块头、区块，参数条件包括hash和number，随意组合。



- Block

    // Block代表以太坊区块链中的一个完整的区块

    type Block struct {
    header   *Header // 区块包括头
    uncles   []*Header // 叔块
    transactions Transactions // 交易集合
    
    // caches缓存
    hash atomic.Value
    size atomic.Value
    
    // Td用于core包存储所有的链上的难度
    td *big.Int
    
    // 这些字段用于eth包来跟踪inter-peer内部端点区块的接替
    ReceivedAt   time.Time
    ReceivedFrom interface{}
    }

总结，Block除了我们熟知的区块中必有的区块头、叔块以及打包存储的交易信息，还有cache缓存的内容，以及每个块之于链的难度值，还有用于跟踪内部端点的字段。



- stop

stop是一个空结构体作为信号源。



>  关于空结构体的讨论，为什么go里面经常出现struct{}?
 

go中除了struct{}类型以外，其他类型都是width，占有存储，而struct{}没有字段，没有方法，width为0，灵活性高，不占内存空间，这可能是让Gopher青睐的原因。

**sealer**

seal方法有两个实现，我们选择ethash，该方法存在于consensus/ethash/sealer.go文件中，第一个函数就是seal的实现，先来看该方法的声明部分：

    // 尝试找到一个nonce值能够满足区块难度需求。
    func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
可以看出这个方法是属于Ethash的指针对象的，

    type Ethash struct {
    // cache配置
    cachedir string // 缓存位置
    cachesinmem  int// 在内存中缓存的数量
    cachesondisk int// 在硬盘中缓存的数量
    
    // DAG挖矿数据集配置
    dagdir   string // DAG位置，存储全部挖矿数据集
    dagsinmemint// 在内存中DAG的数量
    dagsondisk   int// 在硬盘中DAG的数量
    
    // 内存cache
    caches   map[uint64]*cache   // 内存缓存，可反复使用避免再生太频繁
    fcache   *cache  // 为了下一世估算的预生产缓存
    
    // 内存数据集
    datasets map[uint64]*dataset // 内存数据集，可反复使用避免再生太频繁
    fdataset *dataset// 为了下一世估算的预生产数据集
    
    // 挖矿相关字段
    rand *rand.Rand// 随机工具，用来为nonce做适当的种子
    threads  int   // 如果在挖矿，代表挖矿的线程编号
    update   chan struct{} // 更新挖矿中参数的通道
    hashrate metrics.Meter // 测量跟踪平均哈希率
    
    // 以下字段是用于测试
    testerbool  // 是否使用一个小型测试数据集的标志位
    shared*Ethash   // 共享pow模式，无法再生缓存
    fakeMode  bool  // Fake模式，是否取消POW检查的标志位
    fakeFull  bool  // 是否取消所有共识规则的标志位
    fakeFail  uint64// 未通过POW检查的区块号（包含fake模式）
    fakeDelay time.Duration // 验证工作返回消息前的休眠延迟时间
    
    lock sync.Mutex // 为了内存中的缓存和挖矿字段，保证线程安全
    }
为了更好的读懂之后的代码，我们要对区块头的数据结构进行一个分析：

    type Header struct {
    ParentHash  common.Hash`json:"parentHash"   gencodec:"required"`
    UncleHash   common.Hash`json:"sha3Uncles"   gencodec:"required"`
    Coinbasecommon.Address `json:"miner"gencodec:"required"`
    Rootcommon.Hash`json:"stateRoot"gencodec:"required"`
    TxHash  common.Hash`json:"transactionsRoot" gencodec:"required"`
    ReceiptHash common.Hash`json:"receiptsRoot" gencodec:"required"`
    Bloom   Bloom  `json:"logsBloom"gencodec:"required"`
    Difficulty  *big.Int   `json:"difficulty"   gencodec:"required"`
    Number  *big.Int   `json:"number"   gencodec:"required"`
    GasLimit*big.Int   `json:"gasLimit" gencodec:"required"`
    GasUsed *big.Int   `json:"gasUsed"  gencodec:"required"`
    Time*big.Int   `json:"timestamp"gencodec:"required"`
    Extra   []byte `json:"extraData"gencodec:"required"`
    MixDigest   common.Hash`json:"mixHash"  gencodec:"required"`
    Nonce   BlockNonce `json:"nonce"gencodec:"required"`
    }
可以看到一个区块头包含了父块hash值，叔块hash值，Coinbase结点账户地址，状态根，交易hash，接受者hash，日志，难度值，块编号，最低支付gas，花费的gas，时间戳，额外数据，混合hash，nonce值（8个byte)。我们要对这些区块头的成员属性了然于胸，后面的源码内容才能更好的理解。下面我们继续Seal方法，下面展示完整代码：

    func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
    // fake模式立即返回0 nonce
    if ethash.fakeMode {
    header := block.Header()
    header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
    return block.WithSeal(header), nil
    }
    // 共享pow的话，则转到它的共享对象执行Seal操作
    if ethash.shared != nil {
    return ethash.shared.Seal(chain, block, stop)
    }
    // 创建一个runner以及它指挥的多重搜索线程
    abort := make(chan struct{})
    found := make(chan *types.Block)
    
    ethash.lock.Lock() // 线程上锁，保证内存的缓存（包含挖矿字段）安全
    threads := ethash.threads // 挖矿的线程s
    if ethash.rand == nil {// rand为空，则为ethash的字段rand赋值
    // 获得种子
    seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64))
    if err != nil {// 执行失败，有报错
    ethash.lock.Unlock() // 先解锁
    return nil, err // 程序中止，直接返回空块和报错信息
    }
    ethash.rand = rand.New(rand.NewSource(seed.Int64())) // 执行成功，拿到合法种子seed，通过其获得rand对象，赋值。
    }
    ethash.lock.Unlock() // 解锁
    if threads == 0 {// 挖矿线程编号为0，则通过方法返回当前物理上可用CPU编号
    threads = runtime.NumCPU()
    }
    if threads < 0 { // 非法结果
    threads = 0 // 置为0，允许在本地或远程没有额外逻辑的情况下，取消本地挖矿操作
    }
    var pend sync.WaitGroup // 创建一个倒计时锁对象，go语法参照 http://www.cnblogs.com/Evsward/p/goPipeline.html#sync.waitgroup
    for i := 0; i < threads; i++ {
    pend.Add(1)
    go func(id int, nonce uint64) {// 核心代码通过闭包多线程技术来执行。
    defer pend.Done()
    ethash.mine(block, id, nonce, abort, found) // Seal核心工作
    }(i, uint64(ethash.rand.Int63()))//闭包第二个参数表达式uint64(ethash.rand.Int63())通过上面准备好的rand函数随机数结果作为nonce实参传入方法体
    }
    // 直到seal操作被中止或者找到了一个nonce值，否则一直等
    var result *types.Block // 定义一个区块对象result，用于接收操作结果并作为返回值返回上一层
    select { // go语法参照 http://www.cnblogs.com/Evsward/p/go.html#select
    case <-stop:
    // 外部意外中止，停止所有挖矿线程
    close(abort)
    case result = <-found:
    // 其中一个线程挖到正确块，中止其他所有线程
    close(abort)
    case <-ethash.update:
    // ethash对象发生改变，停止当前所有操作，重启当前方法
    close(abort)
    pend.Wait()
    return ethash.Seal(chain, block, stop)
    }
    // 等待所有矿工停止或者返回一个区块
    pend.Wait()
    return result, nil
    }
以上Seal方法体，针对ethash的各种状态进行了校验和流程处理，以及对线程资源的控制，下面看Seal核心工作的内容（sealer.go文件只有两个函数，一个是Seal方法，另一个就是mine方法，可以看出Seal方法是对外的，而mine方法是内部方法，只能被当前ethash包域调用）：mine方法

    // mine函数是真正的pow矿工，用来搜索一个nonce值，nonce值开始于seed值，seed值是能最终产生正确的可匹配可验证的区块难度
    func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
    // 从区块头中提取出一些数据，放在一个全局变量域中
    var (
    header = block.Header()
    hash   = header.HashNoNonce().Bytes()
    target = new(big.Int).Div(maxUint256, header.Difficulty) // 后面有大用，这是用来验证的target
    
    number  = header.Number.Uint64()
    dataset = ethash.dataset(number)
    )
    // 开始生成随机nonce值知道我们中止或者成功找到了一个合适的值
    var (
    attempts = int64(0) // 初始化一个尝试次数的变量，下面会利用该变量耍一些花枪
    nonce= seed // 初始化为seed值，后面每次尝试以后会累加
    )
    logger := log.New("miner", id)
    logger.Trace("Started ethash search for new nonces", "seed", seed)
    for {
    select {
    case <-abort: // 中止命令
    // 挖矿中止，更新状态，中止当前操作，返回空
    logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
    ethash.hashrate.Mark(attempts)
    return
    
    default: // 默认执行
    // 我们没必要在每一次尝试nonce值的时候更新hash率，可以在尝试了2的X次方nonce值以后再更新即可
    attempts++ // 通过次数attemp来控制
    if (attempts % (1 << 15)) == 0 {// 这里是定的2的15次方，位操作符请参考 http://www.cnblogs.com/Evsward/p/go.html#%E5%B8%B8%E9%87%8F
    ethash.hashrate.Mark(attempts) // 满足条件了以后，要更新ethash的hash率字段的状态值
    attempts = 0 // 重置尝试次数
    }
    // 为这个nonce值计算pow值
    digest, result := hashimotoFull(dataset, hash, nonce) // 调用的hashimotoFull函数在本包的算法库中，后面会介绍。
    if new(big.Int).SetBytes(result).Cmp(target) <= 0 { // 验证标准，后面介绍
    // 找到正确nonce值，创建一个基于它的新的区块头
    header = types.CopyHeader(header)
    header.Nonce = types.EncodeNonce(nonce) // 将输入的整型值转换为一个区块nonce值
    header.MixDigest = common.BytesToHash(digest) // 将字节数组转换为Hash对象【Hash是32位的根据任意输入数据的Keccak256哈希算法的返回值】
    
    // 封装返回一个区块
    select {
    case found <- block.WithSeal(header):
    logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
    case <-abort:
    logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
    }
    return
    }
    nonce++ // 累加nonce
    }
    }
    }
mine方法主要就是对nonce的操作，以及对区块头的重建操作，注释中我们也留了一个坑就是对于nonce尝试的工作，这部分内容会转到算法库中来介绍。

**algorithm**

ethash包中包含几个algorithm开头的文件，这些文件的内容是pow核心算法，用来支持挖矿操作。首先我们继续上面留的坑继续研究。

**hashimotoFull函数**

该函数位于ethash/algorithm.go文件中，

    // 在传入的数据集中通过hash和nonce值计算加密值
    func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
    // 本函数核心代码段：定义一个lookup函数，用于在数据集中查找数据
    lookup := func(index uint32) []uint32 {
    offset := index * hashWords // hashWords是上面定义的常量值= 16
    return dataset[offset : offset+hashWords]
    }
    // hashimotoFull函数做的工作就是将原始数据集进行了读取分割，然后传给hashimoto函数。
    return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
    }
**hashimoto函数**

继续分析，上面的hashimotoFull函数返回的是hashimoto函数的返回值，hashimoto算法我们在上面概念部分已经介绍过了，读源码的朋友不理解的可以翻回上面仔细了解一番再回到这里继续研究。

    // 该函数与hashimotoFull有着相同的愿景：在传入的数据集中通过hash和nonce值计算加密值
    func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
    // 计算数据集的理论的行数
    rows := uint32(size / mixBytes)
    
    // 合并header+nonce到一个40字节的seed
    seed := make([]byte, 40) // 创建一个长度为40的字节数组，名字为seed
    copy(seed, hash)// 将区块头的hash（上面提到了Hash对象是32字节大小）拷贝到seed中。
    binary.LittleEndian.PutUint64(seed[32:], nonce) // 将nonce值填入seed的后（40-32=8）字节中去，（nonce本身就是uint64类型，是64位，对应8字节大小），正好把hash和nonce完整的填满了40字节的seed
    
    seed = crypto.Keccak512(seed) // seed经历一遍Keccak512加密
    seedHead := binary.LittleEndian.Uint32(seed) // 从seed中获取区块头，代码后面详解
    
    // 开始与重复seed的混合
    mix := make([]uint32, mixBytes/4)// mixBytes常量= 128，mix的长度为32，元素为uint32，是32位，对应为4字节大小。所以mix总共大小为4*32=128字节大小
    for i := 0; i < len(mix); i++ {
    mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])// 共循环32次，前16和后16位的元素值相同
    }
    // 做一个temp，与mix结构相同，长度相同
    temp := make([]uint32, len(mix))
    
    for i := 0; i < loopAccesses; i++ { // loopAccesses常量 = 64，循环64次
    parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows // mix[i%len(mix)]是循环依次调用mix的元素值，fnv函数在本代码后面详解
    for j := uint32(0); j < mixBytes/hashBytes; j++ {
    copy(temp[j*hashWords:], lookup(2*parent+j))// 通过用种子seed生成的mix数据进行FNV哈希操作以后的数值作为参数去查找源数据（太绕了）拷贝到temp中去。
    }
    fnvHash(mix, temp) // 将mix中所有元素都与temp中对应位置的元素进行FNV hash运算
    }
    // mix大混淆
    for i := 0; i < len(mix); i += 4 {
    mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
    }
    // 最后有效数据只在前8个位置，后面的数据经过上面的循环混淆以后没有价值了，所以将mix的长度减到8，保留前8位有效数据。
    mix = mix[:len(mix)/4]
    
    digest := make([]byte, common.HashLength) // common.HashLength=32，创建一个长度为32的字节数组digest
    for i, val := range mix {
    binary.LittleEndian.PutUint32(digest[i*4:], val)// 再把长度为8的mix分散到32位的digest中去。
    }
    return digest, crypto.Keccak256(append(seed, digest...))
    }
该函数除了被hashimotoFull函数调用以外，还会被hashimotoLight函数调用。顾名思义，hashimotoLight是相对于hashimotoFull的存在。hashimotoLight在后面有机会就介绍（看看能不能绕进我们的route吧）。

下划线与位运算|
以上代码中的seedHead := binary.LittleEndian.Uint32(seed)，我们挑出来单练，跳转到内部方法为：

    func (littleEndian) Uint32(b []byte) uint32 {
    _ = b[3] // bounds check hint to compiler; see golang.org/issue/14808
    return uint32(b[0]) | uint32(b[1])<<8 | uint32(b[2])<<16 | uint32(b[3])<<24
    }


- go语法补充：下划线变量代表Go语言“垃圾桶”的意思，这个垃圾桶并不是说销毁一个对象，而是针对go语言报错机制来处理的，所以b[3]这一行可以是b[3]未使用防止go报“xxx未使用”的错误，同时观察后面的官方注释，也是为了在真正使用b[3]数据前进行边界检查，如果b[3]为空，则会提前报错，不会引发程序问题。


- 位运算，我们在《掌握一门语言GO》中对左移和右移进行了介绍，这里针对或|和与&进行介绍。位运算都是将原数据转换为二进制进行运算，或|就是0和1或得1，例如1和2或得3，因为1的二进制表达为01,2的二进制表达为10，01和10或运算以后就是11，等于3。同理，与&运算就是，0和1与得0，所以1和2的与运算结果为0，因为与&运算是只有都为1才能得1。
**FNV hash 算法**

FNV是由三位创建者的名字得来的，我们知道hash算法最重要的目标就是要平均分布（高度分散），避免碰撞，最好相近的源数据加密后完全不同，哪怕他们只有一个字母不一样，FNV hash算法就是这样的一种算法。

    func fnv(a, b uint32) uint32 {
    return a*0x01000193 ^ b
    }
    
    func fnvHash(mix []uint32, data []uint32) {
    for i := 0; i < len(mix); i++ {
    mix[i] = mix[i]*0x01000193 ^ data[i]
    }
    }
0x01000193是FNV hash算法的一个hash质数（Prime number，又叫素数，只能被1和其本身整除），哈希算法会基于一个常数来做散列操作。0x01000193是FNV针对32 bit数据的散列质数。

**验证方式**

我们一直提，pow是难于计算，上面这么长篇章深刻体现了这一点，但是pow是易于验证的，所以本节讨论的是ethash的pow的验证方式，这个验证方式也很容易找到，就是上面mine方法中我在注释里留下的坑：

new(big.Int).SetBytes(result).Cmp(target) <= 0
我们的核心计算nonce对应的加密值digest方法hashimoto算法返回了一个digest和一个result两个值，而由这行代码可知，与验证方式相关的就是result的值。result在hashimoto算法中最终还经过了crypto.Keccak256(append(seed, digest...)的Keccak256加密，参数列表中也看到了digest值。得到result值以后，就要执行上面这行代码的表达式了。这行表达式很简单，主要含义就是将result值和target值进行比较，如果小于等于0，即为通过。

 **那么target是什么？**
 

target被定义在mine方法体中靠前的变量声明部分，

    target = new(big.Int).Div(maxUint256, header.Difficulty)
可以看出，target的定义是根据区块头中的难度值运算而得出的。所以，这就验证了我们最早在概念部分中提到的，我们可以通过调整Difficulty值，来控制pow运算难度，生成正确nonce的难度，达到pow工作量可控的目标。

**总结**

代码读到这里，已经完成了一个闭环，结合前面的《挖矿》，我们已经走通了以太坊pow的全部流程，整个流程我没有丝毫懈怠，从入口深入到内核，我们把源码扒了底掉（实际上，目前为止的流程中，以太坊的pow并未真正使用到如我所想的DAG）。到目前为止，我们对pow，以及以太坊ethash的实现有了深刻的理解与认识，相信如果让我们去实现一套pow，也是完全有能力的。大家在阅读本文时有任何疑问均可留言给我，我一定会及时回复。