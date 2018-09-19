# EOS交易打包&区块同步流程 #
tips：

本文所有代码均在libraries\chainbase\include\chainbase.cpp或者plugins/producer_plugin/producer_plugin.cpp中，由于篇幅的原因，部分代码用“...”表示。



**一、主要的数据结构**


本文主要简述EOS交易打包&区块同步流程，而controller_imp类是交易打包&区块同步的执行核心，所以从controller_imp类出发，可以顺藤摸瓜，了解整个流程。

首先，我们来看一下controller_impl类的主要成员变量：
    struct
     controller_impl {
       controller&self;
       chainbase::databasedb;   
    // 用于存储已执行的区块，这些区块可以回滚，一旦提交就成了不可逆的区块
       chainbase::databasereversible_blocks; 
    ///< a special database to persist blocks that have successfully been applied but are still reversible
       block_log  blog;
       optional<pending_state>pending;  
    // 尚未出的块
       block_state_ptrhead; 
    // 当前区块的状态
       fork_database  fork_db;  
    // 用于存储可分叉区块的数据库，可分叉的区块都放到这里
       wasm_interface wasmif;   
    // wasm虚拟机的runtime
       resource_limits_managerresource_limits;  
    // 资源管理
       authorization_manager  authorization;
    // 权限管理
       controller::config conf;
       chain_id_type  chain_id;
       bool   replaying = 
    false
    ;
       bool   in_trx_requiring_checks = 
    false
    ; 
    ///< if true, checks that are normally skipped on replay (e.g. auth checks) cannot be skipped
    }

下面重点讲一下其中几个成员。

db用于存储已执行的区块，这些区块可以回滚，一旦提交就不可逆；

reversible_blocks用于存储已执行，但是这些区块是可逆的；

pending用于存放当前正在生产或者验证（收到）的区块；

fork_db用于存放所有区块（包括自己生产和收到的），这些区块链是可分叉的。

由于db和reversible_blocks都是chainbase的数据库，这里简单介绍一下：

EOS 中的数据库索引是依赖chainbase实现的，这个数据库的实现比较简单，主要是用内存映射文件（memory mappedfile)。它认为LevelDB这种数据库，性能不行，也不方便做多级索引，对区块链来说，状态数据库只是账本日志的快照，对持久化要求没那么高，所以可以更激进的利用内存。

chainbase是一个基于boost::multi_index_container实现的适合频繁顺序读写的区块链内存数据库。它有以下几个特性：

1. 多表多索引

2. 状态state可以持久化并多进程共享

3. 嵌套的写事务并支持undo处理

chainbase::database定义在libraries\chainbase\include\chainbase.cpp和chainbase.hpp文件中，由于代码太多，这里就不贴出来了，感兴趣的同学可以自行阅读。

fork_db的类型是fork_database，用于存放所有区块。
    /**
    * @class fork_database
    * @brief manages light-weight state for all potential unconfirmed forks
    *
    * As new blocks are received, they are pushed into the fork database. The fork
    * database tracks the longest chain and the last irreversible block number. All
    * blocks older than the last irreversible block are freed after emitting the
    * irreversible signal.
    */
       
    class
     fork_database {
      
    public
    :
     fork_database( 
    const
     fc::path& data_dir );
     ~fork_database();
     
    void
     close();
     block_state_ptr  get_block(
    const
     block_id_type& id)
    const
    ;
     block_state_ptr  get_block_in_current_chain_by_num( 
    uint32_t
     n )
    const
    ;
    // vector<block_state_ptr>get_blocks_by_number(uint32_t n)const;
     
    /**
      *  Provides a "valid" blockstate upon which other forks may build.
      */
     
    void
    
    set
    ( block_state_ptr s );
     
    /** this method will attempt to append the block to an exsting
      * block_state and will return a pointer to the new block state or
      * throw on error.
      */
     block_state_ptr add( signed_block_ptr b, bool trust = 
    false
     );
     block_state_ptr add( block_state_ptr next_block );
     
    void
    remove( 
    const
     block_id_type& id );
     
    void
    add( 
    const
     header_confirmation& c );
     
    const
     block_state_ptr& head()
    const
    ;
     
    /**
      *  Given two head blocks, return two branches of the fork graph that
      *  end with a common ancestor (same prior block)
      */
     pair< branch_type, branch_type >  fetch_branch_from( 
    const
     block_id_type& first,
      
    const
     block_id_type& second )
    const
    ;
     
    /**
      * If the block is invalid, it will be removed. If it is valid, then blocks older
      * than the LIB are pruned after emitting irreversible signal.
      */
     
    void
     set_validity( 
    const
     block_state_ptr& h, bool valid );
     
    void
     mark_in_current_chain( 
    const
     block_state_ptr& h, bool in_current_chain );
     
    void
     prune( 
    const
     block_state_ptr& h );
     
    /**
      * This signal is emited when a block state becomes irreversible, once irreversible
      * it is removed unless it is the head block.
      */
     signal<
    void
    (block_state_ptr)> irreversible;
      
    private
    :
     
    void
     set_bft_irreversible( block_id_type id );
     unique_ptr<fork_database_impl> my;
       }
    pending的类型是pending_state的，用于存放当前正在生产或者验证（收到）的区块。
    
    struct
     pending_state {
       pending_state( database::session&& s )
       :_db_session( move(s) ){}
       database::session  _db_session;  
    // 会话，用于和db数据库交互
       block_state_ptr_pending_block_state; 
    // pending区块的状态
       vector<action_receipt> _actions; 
    // push_transaction后的action收据
       controller::block_status   _block_status = controller::block_status::incomplete;   
    // pending区块的验证状态，从incomplete到irreversible
       
    void
     push() {
      _db_session.push();
       }
    };


**二、EOS处理交易和出块的流程**


如果对具体代码实现没有兴趣，看上面流程图就也足够。交易和区块处理主要由生产者插件完成，主要代码在plugins/producer_plugin/producer_plugin.cpp。

nodeos程序根据全局时间(epoch)重复执行schedule_production_loop函数，以达到周期性出块的目的。schedule_production_loop函数通过start_block函数判断该节点当前是否具有epoch的出块权限，如果有，则出块。
 
    void
     producer_plugin_impl::schedule_production_loop() {
       chain::controller& chain = app().get_plugin<chain_plugin>().chain();
       _timer.cancel();
       std::weak_ptr<producer_plugin_impl> weak_this = shared_from_this();
    
    // 开始区块
       
    auto
     result = start_block();
       
    if
     (result == start_block_result::failed) {
      elog(
    "Failed to start a pending block, will try again later"
    );
      _timer.expires_from_now( boost::posix_time::microseconds( config::block_interval_us  / 
    10
     ));
      
    // we failed to start a block, so try again later?
      _timer.async_wait([weak_this,cid=++_timer_corelation_id](
    const
     boost::system::error_code& ec) {
     
    auto
     self = weak_this.lock();
     
    if
     (self && ec != boost::asio::error::operation_aborted && cid == self->_timer_corelation_id) {
    self->schedule_production_loop();
     }
      });
       } 
    else
     
    if
     (_pending_block_mode == pending_block_mode::producing) {
      
    // we succeeded but block may be exhausted
      
    if
     (result == start_block_result::succeeded) {
     
    // ship this block off no later than its deadline
     
    static
     
    const
     boost::posix_time::ptime epoch(boost::gregorian::date(
    1970
    , 
    1
    , 
    1
    ));
     _timer.expires_at(epoch + boost::posix_time::microseconds(chain.pending_block_time().time_since_epoch().count()));
     fc_dlog(_log, 
    "Scheduling Block Production on Normal Block #${num} for ${time}"
    , (
    "num"
    , chain.pending_block_state()->block_num)(
    "time"
    ,chain.pending_block_time()));
      } 
    else
     {
     
    // ship this block off immediately
     _timer.expires_from_now( boost::posix_time::microseconds( 
    0
     ));
     fc_dlog(_log, 
    "Scheduling Block Production on Exhausted Block #${num} immediately"
    , (
    "num"
    , chain.pending_block_state()->block_num));
      }
      
    // 出块时间到了，打包区块
      _timer.async_wait([&chain,weak_this,cid=++_timer_corelation_id](
    const
     boost::system::error_code& ec) {
     
    auto
     self = weak_this.lock();
     
    if
     (self && ec != boost::asio::error::operation_aborted && cid == self->_timer_corelation_id) {
    
    auto
     res = self->maybe_produce_block();
    fc_dlog(_log, 
    "Producing Block #${num} returned: ${res}"
    , (
    "num"
    , chain.pending_block_state()->block_num)(
    "res"
    , res) );
     }
      });
       }
    }
    
    start_block函数代码太多，这里只贴出相关代码。
    producer_plugin_impl::start_block_result producer_plugin_impl::start_block() {
    chain::controller& chain = app().get_plugin<chain_plugin>().chain();
    
    // 调用controller::start_block函数
    chain.start_block(block_time, blocks_to_confirm);
    }
    
    现在回到schedule_production_loop函数，轮到当前节点出块时，会设置一个定时器，定时出块，定时器超时会调用maybe_produce_block函数，停止打包，并提交区块。
    bool producer_plugin_impl::maybe_produce_block() {
       
    auto
     reschedule = fc::make_scoped_exit([
    this
    ]{
      schedule_production_loop();
       });
       
    try
     {
      
    // 出块
      produce_block();
      
    return
     
    true
    ;
       } FC_LOG_AND_DROP();
       fc_dlog(_log, 
    "Aborting block due to produce_block error"
    );
       chain::controller& chain = app().get_plugin<chain_plugin>().chain();
       chain.abort_block();
       
    return
     
    false
    ;
    }
    void
     producer_plugin_impl::produce_block() {
       FC_ASSERT(_pending_block_mode == pending_block_mode::producing, 
    "called produce_block while not actually producing"
    );
       chain::controller& chain = app().get_plugin<chain_plugin>().chain();
       
    const
     
    auto
    & pbs = chain.pending_block_state();
       
    const
     
    auto
    & hbs = chain.head_block_state();
       FC_ASSERT(pbs, 
    "pending_block_state does not exist but it should, another plugin may have corrupted it"
    );
       
    auto
     signature_provider_itr = _signature_providers.find( pbs->block_signing_key );
       FC_ASSERT(signature_provider_itr != _signature_providers.end(), 
    "Attempting to produce a block for which we don't have the private key"
    );
       
    //idump( (fc::time_point::now() - chain.pending_block_time()) );
       
    // 交易停止打包到区块
       chain.finalize_block();
       
    // 对区块进行签名
       chain.sign_block( [&]( 
    const
     digest_type& d ) {
      
    auto
     debug_logger = maybe_make_debug_time_logger();
      
    return
     signature_provider_itr->second(d);
       } );
       
    // 提交区块到db数据库
       chain.commit_block();
       
    auto
     hbt = chain.head_block_time();
       
    //idump((fc::time_point::now() - hbt));
       block_state_ptr new_bs = chain.head_block_state();
       ...
    }
    
    现在我们看看commit_block函数，该函数将区块放到到fork_db和db数据库中。
    void
     commit_block( bool add_to_fork_db ) {
      
    if
    ( add_to_fork_db ) {
     pending->_pending_block_state->validated = 
    true
    ;
     
    // 将区块放入fork_db
     
    auto
     new_bsp = fork_db.add( pending->_pending_block_state );
     emit( self.accepted_block_header, pending->_pending_block_state );
     head = fork_db.head();
     FC_ASSERT( new_bsp == head, 
    "committed block did not become the new head in fork database"
     );
      }
      
    //ilog((fc::json::to_pretty_string(*pending->_pending_block_state->block)));
      emit( self.accepted_block, pending->_pending_block_state );
      
    if
    ( !replaying ) {
     reversible_blocks.create<reversible_block_object>( [&]( 
    auto
    & ubo ) {
    ubo.blocknum = pending->_pending_block_state->block_num;
    ubo.set_block( pending->_pending_block_state->block );
     });
      }
      
    // push到db中
      pending->push();
      pending.reset();
       }
    
    fork_db每次新增区块，都会判断是否有区块不可逆，如果有，向db数据库发信号，db数据库就会将不可逆的区块提交到数据库中。
    // fork_database.cpp
       block_state_ptr fork_database::add( block_state_ptr n ) {
      
    auto
     inserted = my->index.insert(n);
      FC_ASSERT( inserted.second, 
    "duplicate block added?"
     );
      my->head = *my->index.
    get
    <by_lib_block_num>().begin();
      
    auto
     lib= my->head->dpos_irreversible_blocknum;
      
    auto
     oldest = *my->index.
    get
    <by_block_num>().begin();
      
    // 移除比最后不可逆区块还旧的区块
      
    if
    ( oldest->block_num < lib ) {
     prune( oldest );
      }
      
    return
     n;
       }
       
    void
     fork_database::prune( 
    const
     block_state_ptr& h ) {
      
    auto
     num = h->block_num;
      
    auto
    & by_bn = my->index.
    get
    <by_block_num>();
      
    auto
     bni = by_bn.begin();
      
    while
    ( bni != by_bn.end() && (*bni)->block_num < num ) {
     prune( *bni );
     bni = by_bn.begin();
      }
      
    auto
     itr = my->index.find( h->id );
      
    if
    ( itr != my->index.end() ) {
     
    // 发送不可逆信号，使得不可逆区块写入db
     irreversible(*itr);
     
    // 移除不可逆区块
     my->index.erase(itr);
      }
    ...
       }
    // controller.cpp   
      
    void
     on_irreversible( 
    const
     block_state_ptr& s ) {
      
    if
    ( !blog.head() )
     blog.read_head();
      
    const
     
    auto
    & log_head = blog.head();
      FC_ASSERT( log_head );
      
    auto
     lh_block_num = log_head->block_num();
      
    // 将不可逆区块广播到net_plugin
      emit( self.irreversible_block, s );
      
    // 将不可逆区块提交到db
      db.commit( s->block_num );
      
    if
    ( s->block_num <= lh_block_num ) {
    // edump((s->block_num)("double call to on_irr"));
    // edump((s->block_num)(s->block->previous)(log_head->id()));
     
    return
    ;
      }
    ...
       }

**三、EOS收到区块&处理出块流程**


producer通过on_incoming_block函数接收区块，on_incoming_block会调用controller::push_block()函数。
 
    void
     on_incoming_block(
    const
     signed_block_ptr& block) {
     fc_dlog(_log, 
    "received incoming block ${id}"
    , (
    "id"
    , block->id()));
     FC_ASSERT( block->timestamp < (fc::time_point::now() + fc::seconds(
    7
    )), 
    "received a block from the future, ignoring it"
     );
     chain::controller& chain = app().get_plugin<chain_plugin>().chain();
     
    /* de-dupe here... no point in aborting block if we already know the block */
     
    auto
     id = block->id();
     
    // 区块已存在本地，丢弃该区块
     
    auto
     existing = chain.fetch_block_by_id( id );
     
    if
    ( existing ) { 
    return
    ; }
     
    // abort the pending block
     chain.abort_block();
     
    // exceptions throw out, make sure we restart our loop
     
    auto
     ensure = fc::make_scoped_exit([
    this
    ](){
    schedule_production_loop();
     });
     
    // push the new block
     bool except = 
    false
    ;
     
    try
     {
    
    // 调用controller::push_block()函数
    chain.push_block(block);
     } 
    catch
    ( 
    const
     fc::exception& e ) {
    elog((e.to_detail_string()));
    except = 
    true
    ;
     }
     ...
      }

我们现在来看controller::push_block()函数。
    // 压入区块
       
    void
     push_block( 
    const
     signed_block_ptr& b, controller::block_status s ) {
    
    //  idump((fc::json::to_pretty_string(*b)));
      FC_ASSERT(!pending, 
    "it is not valid to push a block when there is a pending block"
    );
      
    try
     {
     FC_ASSERT( b );
     FC_ASSERT( s != controller::block_status::incomplete, 
    "invalid block status for a completed block"
     );
     bool trust = !conf.force_all_checks && (s == controller::block_status::irreversible || s == controller::block_status::validated);
     
    // 将收到的区块加到fork_db
     
    auto
     new_header_state = fork_db.add( b, trust );
     emit( self.accepted_block_header, new_header_state );
     maybe_switch_forks( s );
      } FC_LOG_AND_RETHROW( )
       }

controller::push_block()函数将区块加到fork_db后，会判断接收到的区块和本地区块链是否在同一条链上，如果是，那么就执行验证交易；如果不是，那么切换到新的分支。
    void
     maybe_switch_forks( controller::block_status s = controller::block_status::complete ) {
      
    auto
     new_head = fork_db.head();
      
    // 节点当前不出块，当收到新区块时，判断收到的区块和自己本地的区块是否能连上(是否分叉)
      
    if
    ( new_head->header.previous == head->id ) {
     
    try
     {
    
    // 验证区块和交易
    apply_block( new_head->block, s );
    fork_db.mark_in_current_chain( new_head, 
    true
     );
    fork_db.set_validity( new_head, 
    true
     );
    head = new_head;
     } 
    catch
     ( 
    const
     fc::exception& e ) {
    fork_db.set_validity( new_head, 
    false
     ); 
    // Removes new_head from fork_db index, so no need to mark it as not in the current chain.
    
    throw
    ;
     }
      } 
    else
     
    if
    ( new_head->id != head->id ) {   
    // 切换分支
     ilog(
    "switching forks from ${current_head_id} (block number ${current_head_num}) to ${new_head_id} (block number ${new_head_num})"
    ,
      (
    "current_head_id"
    , head->id)(
    "current_head_num"
    , head->block_num)(
    "new_head_id"
    , new_head->id)(
    "new_head_num"
    , new_head->block_num) );
     
    auto
     branches = fork_db.fetch_branch_from( new_head->id, head->id );
     
    for
    ( 
    auto
     itr = branches.second.begin(); itr != branches.second.end(); ++itr ) {
    fork_db.mark_in_current_chain( *itr , 
    false
     );
    
    // 将原来的区块pop出来，撤销之前验证执行的交易
    pop_block();
     }
     FC_ASSERT( self.head_block_id() == branches.second.back()->header.previous,
    
    "loss of sync between fork_db and chainbase during fork switch"
     ); 
    // _should_ never fail
     
    for
    ( 
    auto
     ritr = branches.first.rbegin(); ritr != branches.first.rend(); ++ritr) {
    optional<fc::exception> except;
    
    try
     {
      
    // 验证执行当前区块里面的交易
       apply_block( (*ritr)->block, (*ritr)->validated ? controller::block_status::validated : controller::block_status::complete );
       head = *ritr;
       fork_db.mark_in_current_chain( *ritr, 
    true
     );
       (*ritr)->validated = 
    true
    ;
    }
    
    catch
     (
    const
     fc::exception& e) { except = e; }
    
    if
     (except) {
       ...
    } 
    // end if exception
     } 
    /// end for each block in branch
     ilog(
    "successfully switched fork to new head ${new_head_id}"
    , (
    "new_head_id"
    , new_head->id));
      }
       } 
    /// push_block

最后我们来看一下pop_block函数。
    // 撤销当前区块和状态，恢复到上一个区块和状态
      
    // 撤销交易，交易转为unapplied
       
    void
     pop_block() {
      
    auto
     prev = fork_db.get_block( head->header.previous );
      FC_ASSERT( prev, 
    "attempt to pop beyond last irreversible block"
     );
      
    if
    ( 
    const
     
    auto
    * b = reversible_blocks.find<reversible_block_object,by_num>(head->block_num) )
      {
     reversible_blocks.remove( *b );
      }
      
    for
    ( 
    const
     
    auto
    & t : head->trxs )
     unapplied_transactions[t->signed_id] = t;
      head = prev;
      db.undo();
       }


**此文转自 EcoBall生态球公众号**
