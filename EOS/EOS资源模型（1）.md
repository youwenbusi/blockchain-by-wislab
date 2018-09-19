笔者刚开始使用EOS系统时，在操作时常常会有些不习惯，甚至困惑，不清楚EOS的资源是怎么获得和分配的，相信不少朋友也有同样的疑问。针对这一问题，本文将从四个方面入手，结合代码分析EOS的资源模型(不懂编程的朋友，阅读文字阐述部分大概也能理解)。

**一、EOS资源介绍**


EOS网络中主要有3种资源，即CPU、NET和RAM。
CPU和NET通过抵押EOS获得，属于可恢复资源，用于交易的计算和带宽。
RAM需要向系统购买，属于固定资源，用于存放账户相关的数据，包括账户名、授权信息、合约代码、合约abi和智能合约的数据。

因为RAM是固定资源，买多少就拥有多少，拥有多少就能用多少，没有分配问题。账户名、授权信息、合约代码和abi占用的RAM很容易计量，智能合约数据占用的RAM通过multi_index底层的db计量。

EOS采用全内存方案，将账户相关数据放在RAM里面，加快了合约的处理速度。下面重点介绍CPU和NET。

**二、资源如何分配**


首先查看一下账户的情况：

    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get account xxxxxxxxxxxx
    permissions: 
     owner 
    1
    :
    1
     EOS72pAJiLJVUcGypwoA57UUT6gsfwJtDBR83TL9PGaDSrgJDHLza
    active 
    1
    :
    1
     EOS5ANFfT6y5ubcCzPWGUYGUR6U13KKS5T3XwE32Uo7j8Ruarfi1x1 ************
    @active
    , 
    memory: 
     quota: 
    7.953
     
    KiB
    used: 
    4.936
     
    KiB
      
    net bandwidth: 
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used:   
    341
     bytes
     available:
    1.905
     
    MiB
      
     limit:
    1.906
     
    MiB
      
    cpu bandwidth:
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used: 
    6.613
     ms   
     available:  
    354
     ms   
     limit:
    360.6
     ms   
    producers:
     chainclubeoseoscanadacomeoshenzhenio
     eoslaomaocomoraclegogogo
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get account xxxxxxxxxxxx
    permissions: 
     owner 
    1
    :
    1
     EOS72pAJiLJVUcGypwoA57UUT6gsfwJtDBR83TL9PGaDSrgJDHLza
    active 
    1
    :
    1
     EOS5ANFfT6y5ubcCzPWGUYGUR6U13KKS5T3XwE32Uo7j8Ruarfi1x1 ************
    @active
    , 
    memory: 
     quota: 
    7.953
     
    KiB
    used: 
    4.936
     
    KiB
      
    net bandwidth: 
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used:   
    341
     bytes
     available:
    1.905
     
    MiB
      
     limit:
    1.906
     
    MiB
      
    cpu bandwidth:
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used: 
    6.613
     ms   
     available:  
    351
     ms   
     limit:
    357.6
     ms   
    producers:
     chainclubeoseoscanadacomeoshenzhenio
     eoslaomaocomoraclegogogo  


拿上述账号为例，CPU抵押3.2157EOS，获得351.8ms的CPU时间。从以上代码中可以看到，账户可使用的最大资源是动态变化的，已使用的资源是固定的，可使用的资源等于可使用的最大资源减去已使用的资源。

那么就有一个疑问出现，CPU抵押3.2157EOS，为什么能获得351.8ms的CPU时间？换句话说，CPU时间是怎么分配的？ CPU和NET通过抵押EOS获得，每个账户所能获得的资源为：系统总资源 * 抵押代币 / 总的抵押代币。

    // 获取账户当前可用的虚拟CPU
    int64_t
     resource_limits_manager::get_account_cpu_limit( 
    const
     account_name& name ) 
    const
     {
       
    auto
     arl = get_account_cpu_limit_ex(name);
       
    return
     arl.available;
    }
    // 获取账户的虚拟CPU限制
    account_resource_limit resource_limits_manager::get_account_cpu_limit_ex( 
    const
     account_name& name ) 
    const
     {
       
    const
     
    auto
    & state = _db.
    get
    <resource_limits_state_object>();
       
    const
     
    auto
    & usage = _db.
    get
    <resource_usage_object, by_owner>(name);
       
    const
     
    auto
    & config = _db.
    get
    <resource_limits_config_object>();
       
    int64_t
     cpu_weight, x, y;
       get_account_limits( name, x, y, cpu_weight );
       
    if
    ( cpu_weight < 
    0
     || state.total_cpu_weight == 
    0
     ) {
      
    return
     { -
    1
    , -
    1
    , -
    1
     };
       }
       account_resource_limit arl;
       
    uint128_t
     window_size = config.account_cpu_usage_average_window;
       
    // 计算窗口期(在这里为24h)内的虚拟计算能力
       
    uint128_t
     virtual_cpu_capacity_in_window = (
    uint128_t
    )state.virtual_cpu_limit * window_size;
       
    uint128_t
     user_weight = (
    uint128_t
    )cpu_weight;
       
    uint128_t
     all_user_weight = (
    uint128_t
    )state.total_cpu_weight;
       
    // 每个账户所能获得的资源为：系统总资源 * 抵押代币 / 总的抵押代币。
       
    auto
     max_user_use_in_window = (virtual_cpu_capacity_in_window * user_weight) / all_user_weight;
       
    auto
     cpu_used_in_window  = impl::integer_divide_ceil((
    uint128_t
    )usage.cpu_usage.value_ex * window_size, (
    uint128_t
    )config::rate_limiting_precision);
       
    if
    ( max_user_use_in_window <= cpu_used_in_window )
      arl.available = 
    0
    ;
       
    else
      arl.available = impl::downgrade_cast<
    int64_t
    >(max_user_use_in_window - cpu_used_in_window);
       arl.used = impl::downgrade_cast<
    int64_t
    >(cpu_used_in_window);
       arl.max = impl::downgrade_cast<
    int64_t
    >(max_user_use_in_window);
       
    return
     arl;
    }


每个账户所能获得的资源为：系统总资源 * 抵押代币 / 总的抵押代币，那么影响用户可使用的资源有2个因素：

1、系统总资源

2、权重，即抵押代币占总的抵押代币的比例

这里重点介绍系统总资源，首先看一下系统总资源：

    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get info
    {
      
    "server_version"
    : 
    "79651199"
    ,
      
    "chain_id"
    : 
    "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906"
    ,
      
    "head_block_num"
    : 
    6517556
    ,
      
    "last_irreversible_block_num"
    : 
    6517232
    ,
      
    "last_irreversible_block_id"
    : 
    "006371f00de9b98bbea9d69401df1af407a37574c0eb4357341b949b5f5598c8"
    ,
      
    "head_block_id"
    : 
    "00637334f6abfab0cd5253dd92b883cfc4560ebf9f1f8cef2e3135c69188c3f6"
    ,
      
    "head_block_time"
    : 
    "2018-07-18T12:42:04.000"
    ,
      
    "head_block_producer"
    : 
    "eosswedenorg"
    ,
      
    "virtual_block_cpu_limit"
    : 
    200000000
    ,
      
    "virtual_block_net_limit"
    : 
    1048576000
    ,
      
    "block_cpu_limit"
    : 
    199900
    ,
      
    "block_net_limit"
    : 
    1048576
    }
    右滑浏览完整代码
    
    block_cpu_limit为区块实际的最大CPU时间，现为200'000ms；
    
    block_net_limit为区块实际的最大带宽大小，现为 1024 * 1024 = 1048576K，也就是1M；
    
    virtual_block_cpu_limit为区块（虚拟）总CPU时间，该值大概为block_cpu_limit的1000倍；
    
    virtual_block_net_limit为区块（虚拟）总带宽大小，该值大概为block_net_limit的1000倍；
    
    其实virtual_block_cpu_limit和virtual_block_net_limit是不断变化的，上述的情况是系统空闲时候的状态。
    
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get info
    {
      
    "server_version"
    : 
    "79651199"
    ,
      
    "chain_id"
    : 
    "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906"
    ,
      
    "head_block_num"
    : 
    6606018
    ,
      
    "last_irreversible_block_num"
    : 
    6605699
    ,
      
    "last_irreversible_block_id"
    : 
    "0064cb83ed0342fde91c7a4299233dd4384f508b4599bbdb264ef3fc851a1115"
    ,
      
    "head_block_id"
    : 
    "0064ccc20c87cde7eb4f7bc960cdd3c1180d85a6c29ead8caec13ad5bba17ec1"
    ,
      
    "head_block_time"
    : 
    "2018-07-19T01:31:34.000"
    ,
      
    "head_block_producer"
    : 
    "eos42freedom"
    ,
      
    "virtual_block_cpu_limit"
    : 
    46622441
    ,
      
    "virtual_block_net_limit"
    : 
    1048576000
    ,
      
    "block_cpu_limit"
    : 
    199900
    ,
      
    "block_net_limit"
    : 
    1048576
    }
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ 
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get info
    {
      
    "server_version"
    : 
    "79651199"
    ,
      
    "chain_id"
    : 
    "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906"
    ,
      
    "head_block_num"
    : 
    6606065
    ,
      
    "last_irreversible_block_num"
    : 
    6605746
    ,
      
    "last_irreversible_block_id"
    : 
    "0064cbb238e95912a4be0b53b0c794c69822e763e498b33108032d1736e1c341"
    ,
      
    "head_block_id"
    : 
    "0064ccf179a79585d6621d78a0d65a43bf8f7b4804111ee7de8147f0c36bd19f"
    ,
      
    "head_block_time"
    : 
    "2018-07-19T01:31:57.500"
    ,
      
    "head_block_producer"
    : 
    "eoscafeblock"
    ,
      
    "virtual_block_cpu_limit"
    : 
    48867136
    ,
      
    "virtual_block_net_limit"
    : 
    1048576000
    ,
      
    "block_cpu_limit"
    : 
    166395
    ,
      
    "block_net_limit"
    : 
    1044416
    }


那么EOS中为什么会有虚拟CPU资源和NET资源？我们用生活中的例子来说明。如同现实生活中的水、电和流量，忙时和闲时是分开定价的，使用价格手段鼓励用户错峰用电。EOS也允许用户在系统闲时使用更多的资源，在系统忙时保证能使用对应权重的资源，解决了以太坊拥堵时低gas交易无法打包的问题。

同样多的钱，在闲时可以买到更多的水电；同理，抵押同样数量的代币，在系统闲时可使用更多的资源。 这里就引出EOS使用资源的基本原则：使用一分，记录一分。

为了实现动态调节的机理，EOS引入了虚拟资源这一概念，最大可使用的虚拟资源为实际可使用资源的1000倍，也就是说，用户在系统闲时可用的最大资源为实际可用的1000倍。

现实生活中的水、电的忙闲是通过时段区分的，比如白天是忙时，半夜是闲时。EOS则需要通过监测60s内的区块资源的使用情况来区分，现阶段是60s内的区块资源使用低于最大可用的10%，就（小比例）增加系统可用的虚拟资源，否则就（小比例）减少。

    static
     
    const
     
    uint32_t
     block_cpu_usage_average_window_ms= 
    60
    *
    1000l
    ;
    static
     
    const
     
    uint32_t
     block_size_average_window_ms = 
    60
    *
    1000l
    ;
    class
     resource_limits_state_object : 
    public
     chainbase::object<resource_limits_state_object_type, resource_limits_state_object> {
      OBJECT_CTOR(resource_limits_state_object);
      id_type id;
      
    /**
       * Track the average netusage for blocks
       */
      usage_accumulator average_block_net_usage;
      
    /**
       * Track the average cpu usage for blocks
       */
      usage_accumulator average_block_cpu_usage;
      
    void
     update_virtual_net_limit( 
    const
     resource_limits_config_object& cfg );
      
    void
     update_virtual_cpu_limit( 
    const
     resource_limits_config_object& cfg );
      
    uint64_t
     pending_net_usage = 
    0ULL
    ;
      
    uint64_t
     pending_cpu_usage = 
    0ULL
    ;
      
    uint64_t
     total_net_weight = 
    0ULL
    ;
      
    uint64_t
     total_cpu_weight = 
    0ULL
    ;
      
    uint64_t
     total_ram_bytes = 
    0ULL
    ;
      
    /**
       * The virtual number of bytes that would be consumed over blocksize_average_window_ms
       * if all blocks were at their maximum virtual size. This is virtual because the
       * real maximum block is less, this virtual number is only used for rate limiting users.
       *
       * It's lowest possible value is max_block_size * blocksize_average_window_ms / block_interval
       * It's highest possible value is 1000 times its lowest possible value
       *
       * This means that the most an account can consume during idle periods is 1000x the bandwidth
       * it is gauranteed under congestion.
       *
       * Increases when average_block_size < target_block_size, decreases when
       * average_block_size > target_block_size, with a cap at 1000x max_block_size
       * and a floor at max_block_size;
       **/
      
    uint64_t
     virtual_net_limit = 
    0ULL
    ;
      
    /**
       *  Increases when average_bloc
       */
      
    uint64_t
     virtual_cpu_limit = 
    0ULL
    ;
       };
    static
     
    uint64_t
     update_elastic_limit(
    uint64_t
     current_limit, 
    uint64_t
     average_usage, 
    const
     elastic_limit_parameters& params) {
       
    uint64_t
     result = current_limit;
       
    if
     (average_usage > params.target ) {
      result = result * params.contract_rate;
       } 
    else
     {
      result = result * params.expand_rate;
       }
       
    return
     std::min(std::max(result, params.max), params.max * params.max_multiplier);
    }
    void
     resource_limits_state_object::update_virtual_cpu_limit( 
    const
     resource_limits_config_object& cfg ) {
       
    //idump((average_block_cpu_usage.average()));
       virtual_cpu_limit = update_elastic_limit(virtual_cpu_limit, average_block_cpu_usage.average(), cfg.cpu_limit_parameters);
       
    //idump((virtual_cpu_limit));
    }
    // 计算1分钟内块的CPU和NET的使用情况，用于计算下一个块最大可用的虚拟CPU和NET，通常在controller生成块的时候调用
    void
     resource_limits_manager::process_block_usage(
    uint32_t
     block_num) {
       
    const
     
    auto
    & s = _db.
    get
    <resource_limits_state_object>();
       
    const
     
    auto
    & config = _db.
    get
    <resource_limits_config_object>();
       _db.modify(s, [&](resource_limits_state_object& state){
      
    // apply pending usage, update virtual limits and reset the pending
      state.average_block_cpu_usage.add(state.pending_cpu_usage, block_num, config.cpu_limit_parameters.periods);
      state.update_virtual_cpu_limit(config);
      state.pending_cpu_usage = 
    0
    ;
      state.average_block_net_usage.add(state.pending_net_usage, block_num, config.net_limit_parameters.periods);
      state.update_virtual_net_limit(config);
      state.pending_net_usage = 
    0
    ;
       });
    }


到这里，有些小伙伴可能还会有这样的疑问：系统总资源是怎么计算出来的？
诚如上述，60s内的区块资源使用低于最大可用的10%，就（小比例）增加系统总资源，否则就（小比例）减少。

而virtual_block_cpu_limit和virtual_block_net_limit的总资源的初始值分别为block_cpu_limit和block_net_limit。也就是说，虚拟资源一开始等于实际资源，然后随着系统忙闲不断调整，最低值等于实际资源，最高值等于实际资源的1000倍。

    void
     resource_limits_manager::initialize_database() {
       
    const
     
    auto
    & config = _db.create<resource_limits_config_object>([](resource_limits_config_object& config){
      
    // see default settings in the declaration
       });
       _db.create<resource_limits_state_object>([&config](resource_limits_state_object& state){
      
    // see default settings in the declaration
      
    // start the chain off in a way that it is "congested" aka slow-start
      state.virtual_cpu_limit = config.cpu_limit_parameters.max;
      state.virtual_net_limit = config.net_limit_parameters.max;
       });
    }


账户CPU和NET属于可恢复资源，在EOS系统中，完全恢复周期为24h，也就是说，24h后账户的CPU和NET资源都会恢复。

    static
     
    const
     
    uint32_t
     account_cpu_usage_average_window_ms  = 
    24
    *
    60
    *
    60
    *
    1000l
    ;
    static
     
    const
     
    uint32_t
     account_net_usage_average_window_ms  = 
    24
    *
    60
    *
    60
    *
    1000l
    ;


虽然说CPU和NET的完全恢复周期为24h，但不代表你需要等24小时才能使用，其实EOS的资源是每时每刻都在恢复的。

比如现在距离你上笔交易正好是12h，上一笔交易后，NET的使用量是120KB，你当下可用的NET为( 1 - 12 / 24) * 120 = 60KB，NET的使用量也为60KB，此时你发起一笔交易，该交易需要消耗1KB的交易，那么交易成功后，NET的使用量为61KB。

请注意，通过get account获取到的CPU和NET的使用量都是上一笔交易后的状态，账户当前的可用资源信息必须通过交易来更新。

    /**
       *  This class accumulates and exponential moving average based on inputs
       *  This accumulator assumes there are no drops in input data
       *
       *  The value stored is Precision times the sum of the inputs.
       */
      template<
    uint64_t
     
    Precision
     = config::rate_limiting_precision>
      
    struct
     exponential_moving_average_accumulator
      {
     static_assert( 
    Precision
     > 
    0
    , 
    "Precision must be positive"
     );
     
    static
     constexpr 
    uint64_t
     max_raw_value = std::numeric_limits<
    uint64_t
    >::max() / 
    Precision
    ;
     exponential_moving_average_accumulator()
     : last_ordinal(
    0
    )
     , value_ex(
    0
    )
     , consumed(
    0
    )
     {
     }
     
    uint32_t
       last_ordinal;  
    ///< The ordinal of the last period which has contributed to the average
     
    uint64_t
       value_ex;  
    ///< The current average pre-multiplied by Precision
     
    uint64_t
       consumed;   
    ///< The last periods average + the current periods contribution so far
     
    /**
      * return the average value
      */
     
    uint64_t
     average() 
    const
     {
    
    return
     integer_divide_ceil(value_ex, 
    Precision
    );
     }
     
    void
     add( 
    uint64_t
     units, 
    uint32_t
     ordinal, 
    uint32_t
     window_size 
    /* must be positive */
     )
     {
    
    // check for some numerical limits before doing any state mutations
    EOS_ASSERT(units <= max_raw_value, rate_limiting_state_inconsistent, 
    "Usage exceeds maximum value representable after extending for precision"
    );
    EOS_ASSERT(std::numeric_limits<decltype(consumed)>::max() - consumed >= units, rate_limiting_state_inconsistent, 
    "Overflow in tracked usage when adding usage!"
    );
    
    auto
     value_ex_contrib = downgrade_cast<
    uint64_t
    >(integer_divide_ceil((
    uint128_t
    )units * 
    Precision
    , (
    uint128_t
    )window_size));
    EOS_ASSERT(std::numeric_limits<decltype(value_ex)>::max() - value_ex >= value_ex_contrib, rate_limiting_state_inconsistent, 
    "Overflow in accumulated value when adding usage!"
    );
    
    if
    ( last_ordinal != ordinal ) {
       FC_ASSERT( ordinal > last_ordinal, 
    "new ordinal cannot be less than the previous ordinal"
     );
       
    if
    ( (
    uint64_t
    )last_ordinal + window_size > (
    uint64_t
    )ordinal ) {
      
    const
     
    auto
     delta = ordinal - last_ordinal; 
    // clearly 0 < delta < window_size
      
    const
     
    auto
     decay = make_ratio(
      (
    uint64_t
    )window_size - delta,
      (
    uint64_t
    )window_size
      );
      value_ex = value_ex * decay;
       } 
    else
     {
      value_ex = 
    0
    ;
       }
       last_ordinal = ordinal;
       consumed = average();
    }
    consumed += units;
    value_ex += value_ex_contrib;
     }
      };
       }
       using usage_accumulator = impl::exponential_moving_average_accumulator<>;
    
    struct
     resource_usage_object : 
    public
     chainbase::object<resource_usage_object_type, resource_usage_object> {
      OBJECT_CTOR(resource_usage_object)
      id_type id;
      account_name owner;
      usage_accumulatornet_usage;
      usage_accumulatorcpu_usage;
      
    uint64_t
     ram_usage = 
    0
    ;
       };
    右滑浏览完整代码
    
    在EOS中，交易都会消耗CPU和NET，下面是投票交易，大家留意第2次投票消耗的CPU和NET。
    
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io system voteproducer prods xxxxxxxxxxxx chainclubeos oraclegogogo eoscanadacom eoshenzhenio eoslaomaocom -p xxxxxxxxxxxx@active
    1725682ms
     thread-
    0
       main.cpp:
    429
      create_action] result: {
    "binargs"
    :
    "a0a662fb519c856400000000000000000580a93a3aa2e94c43202932c94c83305540dd54ed4fd530552029a2465213315540196594a988cca5"
    } arg: {
    "code"
    :
    "eosio"
    ,
    "action"
    :
    "voteproducer"
    ,
    "args"
    :{
    "voter"
    :
    "xxxxxxxxxxxx"
    ,
    "proxy"
    :
    ""
    ,
    "producers"
    :[
    "chainclubeos"
    ,
    "eoscanadacom"
    ,
    "eoshenzhenio"
    ,
    "eoslaomaocom"
    ,
    "oraclegogogo"
    ]}} 
    executed transaction: 
    0008ead46d05915c8eb129f234766481ba2cf672e20e96f94587b42cb1b59f52
      
    152
     bytes  
    3641
     us
    # eosio <= eosio::voteproducer  {
    "voter"
    :
    "xxxxxxxxxxxx"
    ,
    "proxy"
    :
    ""
    ,
    "producers"
    :[
    "chainclubeos"
    ,
    "eoscanadacom"
    ,
    "eoshenzhenio"
    ,
    "
    eoslao...
    warning: transaction executed locally, but may not be confirmed by the network yet
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ 
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ 
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get account xxxxxxxxxxxx
    permissions: 
     owner 
    1
    :
    1
     EOS72pAJiLJVUcGypwoA57UUT6gsfwJtDBR83TL9PGaDSrgJDHLza
    active 
    1
    :
    1
     EOS5ANFfT6y5ubcCzPWGUYGUR6U13KKS5T3XwE32Uo7j8Ruarfi1x1 ************
    @active
    , 
    memory: 
     quota: 
    7.953
     
    KiB
    used: 
    4.936
     
    KiB
      
    net bandwidth: 
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used:   
    351
     bytes
     available:
    1.906
     
    MiB
      
     limit:
    1.906
     
    MiB
      
    cpu bandwidth:
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used: 
    7.371
     ms   
     available:  
    373
     ms   
     limit:
    380.4
     ms   
    producers:
     chainclubeoseoscanadacomeoshenzhenio
     eoslaomaocomoraclegogogo
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io system voteproducer prods xxxxxxxxxxxx chainclubeos oraclegogogo eoscanadacom eoshenzhenio eoslaomaocom -p xxxxxxxxxxxx@active
    1750647ms
     thread-
    0
       main.cpp:
    429
      create_action] result: {
    "binargs"
    :
    "a0a662fb519c856400000000000000000580a93a3aa2e94c43202932c94c83305540dd54ed4fd530552029a2465213315540196594a988cca5"
    } arg: {
    "code": "eosio",
    "action"
    :
    "voteproducer"
    ,
    "args"
    :{
    "voter"
    :
    "xxxxxxxxxxxx"
    ,
    "proxy"
    :
    ""
    ,
    "producers"
    :[
    "chainclubeos"
    ,
    "eoscanadacom"
    ,
    "eoshenzhenio"
    ,
    "eoslaomaocom"
    ,
    "oraclegogogo"
    ]}} 
    executed transaction: 
    25271df8a7a1346125fd8c5a3e5119dbb9596dadb029810101a59881d7736751
      
    152
     bytes  
    5343
     us
    # eosio <= eosio::voteproducer  {
    "voter"
    :
    "xxxxxxxxxxxx"
    ,
    "proxy"
    :
    ""
    ,
    "producers"
    :[
    "chainclubeos"
    ,
    "eoscanadacom"
    ,
    "eoshenzhenio"
    ,
    "
    eoslao...
    warning: transaction executed locally, but may not be confirmed by the network yet
    ubuntu@ubuntu:~
    /eos/
    build/programs/cleos$ ./cleos -u https:
    //api.oraclechain.io get account xxxxxxxxxxxx
    permissions: 
     owner 
    1
    :
    1
     EOS72pAJiLJVUcGypwoA57UUT6gsfwJtDBR83TL9PGaDSrgJDHLza
    active 
    1
    :
    1
     EOS5ANFfT6y5ubcCzPWGUYGUR6U13KKS5T3XwE32Uo7j8Ruarfi1x1 ************
    @active
    , 
    memory: 
     quota: 
    7.953
     
    KiB
    used: 
    4.936
     
    KiB
      
    net bandwidth: 
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used:   
    503
     bytes
     available:
    1.906
     
    MiB
      
     limit:
    1.906
     
    MiB
      
    cpu bandwidth:
     staked:  
    3.2157
     EOS   (total stake delegated from account to self)
     delegated:   
    0.0000
     EOS   (total staked delegated to account from others)
     used: 
    10.95
     ms   
     available:
    369.4
     ms   
     limit:
    380.4
     ms   
    producers:
     chainclubeoseoscanadacomeoshenzhenio
     eoslaomaocomoraclegogogo

**此文转自 EcoBall生态球公众号**