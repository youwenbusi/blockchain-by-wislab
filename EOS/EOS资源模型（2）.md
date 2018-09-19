
**三、资源的计量**


对NET的计量主要是交易的大小，对CPU的计量主要是交易消耗的时间； 对NET的计量概念比较直观易懂，这里重点介绍对CPU的计量。

对交易的计量主要是在transaction_context.cpp中完成的，即构造transaction_context类对象时开始计时，初始化( init()函数 )各钟参数，计算付费账户，交易处理结束后( finalize()函数 )停止计时，更新付费账户的资源使用情况。

由于代码过多，阅读不便，部分函数只列出了函数头，有兴趣的小伙伴自行阅读源码。

    transaction_context
    ::
    transaction_context
    (
     controller
    &
     c
    ,
     
    const
     signed_transaction
    &
     t
    ,
     
    const
     transaction_id_type
    &
     trx_id
    ,
     fc
    ::
    time_point s 
    )
    :
    control
    (
    c
    )
    ,
    trx
    (
    t
    )
    ,
    id
    (
    trx_id
    )
    ,
    undo_session
    (
    c
    .
    db
    ().
    start_undo_session
    (
    true
    ))
    ,
    trace
    (
    std
    ::
    make_shared
    <
    transaction_trace
    >())
    ,
    start
    (
    s
    )
    ,
    net_usage
    (
    trace
    ->
    net_usage
    )
    ,
    pseudo_start
    (
    s
    )
    {
      trace
    ->
    id 
    =
     id
    ;
      executed
    .
    reserve
    (
     trx
    .
    total_actions
    ()
     
    );
      FC_ASSERT
    (
     trx
    .
    transaction_extensions
    .
    size
    ()
     
    ==
     
    0
    ,
     
    "we don't support any extensions yet"
     
    );
    }
    void
     transaction_context
    ::
    exec
    ()
     
    {
    
    ...
    }
    void
     transaction_context
    ::
    init
    (
    uint64_t
     initial_net_usage 
    )
     
    {
      
    ...
    }
    void
     transaction_context
    ::
    finalize
    ()
     
    {
      
    ...
      net_usage 
    =
     
    ((
    net_usage 
    +
     
    7
    )/
    8
    )*
    8
    ;
     
    // Round up to nearest multiple of word size (8 bytes)
      eager_net_limit 
    =
     net_limit
    ;
      check_net_usage
    ();
      
    auto
     now 
    =
     fc
    ::
    time_point
    ::
    now
    ();
      trace
    ->
    elapsed 
    =
     now 
    -
     start
    ;
      
    if
    (
     billed_cpu_time_us 
    ==
     
    0
     
    )
     
    {
     
    const
     
    auto
    &
     cfg 
    =
     control
    .
    get_global_properties
    ().
    configuration
    ;
     billed_cpu_time_us 
    =
     std
    ::
    max
    (
     
    (
    now 
    -
     pseudo_start
    ).
    count
    (),
     static_cast
    <
    int64_t
    >(
    cfg
    .
    min_transaction_cpu_usage
    )
     
    );
      
    }
      validate_cpu_usage_to_bill
    (
     billed_cpu_time_us 
    );
      rl
    .
    add_transaction_usage
    (
     bill_to_accounts
    ,
     static_cast
    <
    uint64_t
    >(
    billed_cpu_time_us
    ),
     net_usage
    ,
    block_timestamp_type
    (
    control
    .
    pending_block_time
    ()).
    slot 
    );
     
    // Should never fail
    }
    

读到这里，或许会产生一个疑问：EOS有21个出块节点，每个出块节点的性能是不一样的，同样的交易在不同的出块节点执行，消耗的CPU时间是不一样的，那要怎么保证计量的统一性呢？

这里给出解答。对交易消耗的资源的计量由将交易打包到区块的出块节点决定，该出块节点将交易打包到区块时，会开一张收据，这张收据记录交易消耗的CPU时间和NET带宽，其它出块节点(和全节点)统一认同这个收据。

    /**
    * When a transaction is referenced by a block it could imply one of several outcomes which
    * describe the state-transition undertaken by the block producer.
    */
       
    struct
     transaction_receipt_header 
    {
      
    enum
     status_enum 
    {
     executed  
    =
     
    0
    ,
     
    ///< succeed, no error handler executed
     soft_fail 
    =
     
    1
    ,
     
    ///< objectively failed (not executed), error handler executed
     hard_fail 
    =
     
    2
    ,
     
    ///< objectively failed and error handler objectively failed thus no state change
     delayed   
    =
     
    3
    ,
     
    ///< transaction delayed/deferred/scheduled for future execution
     expired   
    =
     
    4
      
    ///< transaction expired and storage space refuned to user
      
    };
      transaction_receipt_header
    ():
    status
    (
    hard_fail
    ){}
      transaction_receipt_header
    (
     status_enum s 
    ):
    status
    (
    s
    ){}
      friend 
    inline
     bool 
    operator
     
    ==(
     
    const
     transaction_receipt_header
    &
     lhs
    ,
     
    const
     transaction_receipt_header
    &
     rhs 
    )
     
    {
     
    return
     std
    ::
    tie
    (
    lhs
    .
    status
    ,
     lhs
    .
    cpu_usage_us
    ,
     lhs
    .
    net_usage_words
    )
     
    ==
     std
    ::
    tie
    (
    rhs
    .
    status
    ,
     rhs
    .
    cpu_usage_us
    ,
     rhs
    .
    net_usage_words
    );
      
    }
      fc
    ::
    enum_type
    <
    uint8_t
    ,
    status_enum
    >
       status
    ;
      
    uint32_t
     cpu_usage_us
    ;
     
    ///< total billed CPU usage (microseconds)
      fc
    ::
    unsigned_int net_usage_words
    ;
     
    ///<  total billed NET usage, so we can reconstruct resource state when skipping context free data... hard failures...
       
    };
       
    struct
     transaction_receipt 
    :
     
    public
     transaction_receipt_header 
    {
      transaction_receipt
    ():
    transaction_receipt_header
    (){}
      transaction_receipt
    (
     transaction_id_type tid 
    ):
    transaction_receipt_header
    (
    executed
    ),
    trx
    (
    tid
    ){}
      transaction_receipt
    (
     packed_transaction ptrx 
    ):
    transaction_receipt_header
    (
    executed
    ),
    trx
    (
    ptrx
    ){}
      fc
    ::
    static_variant
    <
    transaction_id_type
    ,
     packed_transaction
    >
     trx
    ;
      digest_type digest
    ()
    const
     
    {
     digest_type
    ::
    encoder enc
    ;
     fc
    ::
    raw
    ::
    pack
    (
     enc
    ,
     status 
    );
     fc
    ::
    raw
    ::
    pack
    (
     enc
    ,
     cpu_usage_us 
    );
     fc
    ::
    raw
    ::
    pack
    (
     enc
    ,
     net_usage_words 
    );
     
    if
    (
     trx
    .
    contains
    <
    transaction_id_type
    >()
     
    )
    fc
    ::
    raw
    ::
    pack
    (
     enc
    ,
     trx
    .
    get
    <
    transaction_id_type
    >()
     
    );
     
    else
    fc
    ::
    raw
    ::
    pack
    (
     enc
    ,
     trx
    .
    get
    <
    packed_transaction
    >().
    packed_digest
    ()
     
    );
     
    return
     enc
    .
    result
    ();
      
    }
       
    };
    

**四、资源的管理**


EOS对CPU、NET和RAM资源进行统一管理，对系统可用资源和账户可用资源集中记账。
    // 将交易消耗的CPU和NET资源计入账户，并且加到块消耗的CPU和NET资源内，通常在交易验证后调用
    void
     resource_limits_manager
    ::
    add_transaction_usage
    (
    const
     flat_set
    <
    account_name
    >&
     accounts
    ,
     
    uint64_t
     cpu_usage
    ,
     
    uint64_t
     net_usage
    ,
     
    uint32_t
     time_slot 
    )
     
    {
      
    ...
    }
    // 更新RAM的使用量
    void
     resource_limits_manager
    ::
    add_pending_ram_usage
    (
     
    const
     account_name account
    ,
     
    int64_t
     ram_delta 
    )
     
    {
    
    ...
    }
    // 更新各账户的资源限制，通常在controller生成块的时候调用
    void
     resource_limits_manager
    ::
    process_account_limit_updates
    ()
     
    {
       
    ...
    }

**此文转自 EcoBall生态球公众号**