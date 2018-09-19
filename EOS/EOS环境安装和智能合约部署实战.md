

## 硬件配置 ##

**操作系统要求**

1.Amazon 2017.09 and higher

2.Centos 7

3.Fedora 25 and higher (Fedora 27 recommended)

4.Mint 18
 
5.Ubuntu 16.04 LTS (Ubuntu 16.10 recommended)

6.Ubuntu 18.04 LTS7. MacOS Darwin 10.12 and higher (MacOS 10.13.x recommended)


**系统需求**

1.8G内存
2.20G空闲硬盘

这是官方给出的支持的操作系统，个人建议，尽量不要在windows安装linux虚拟机，会出现很多意想不到的C++安装包的问题。我用的mac版本是10.13.4。




## 安装EOS环境 ##

克隆代码

git clone https://github.com/EOSIO/eos --recursive

如果克隆代码时，没加–recursive属性，克隆完之后，需要在命令行中cd到下载的eos目录，再次执行如下命令:

git submodule update --init --recursive

执行编译脚本

在命令行中进入到eos目录，执行编译脚本:

./eosio_build.sh

大概需要10分钟，看网速情况了。编译完成后，出现如下所示log: 



请仔细看这里的error日志，提示build下面的hello.wast找不到，如果只有这个error，请忽略它(这是官方的bug，以前编译是没有的，而且，这个error也并不会影响后面的正常操作)

安装可执行脚本

cd build
sudo make install

这个安装速度会有点慢，我的是1小时左右。安装成功后，应该能看到如下所示log:

如果编译脚本或者安装可执行脚本时，出现部分安装包安装不了的情况，请重新执行命令，或者删除build目录，再次执行编译命令。如果这两种也都没法解决的，请留言给我，或者带着error日志移步到官方issue，目前来说，官方issue是解决问题最多的地方，百度和google并没有卵用。


## 启动本地私有节点 ##

在控制台中任意目录执行如下命令

    nodeos -e -p eosio --plugin eosio::chain_api_plugin --plugin eosio::history_api_plugin --replay-blockchain --max-irreversible-block-age 1801

启动成功后，会在本地产生区块，log如下: 



通过log分析，可以看到eosio root，也就是区块生成后的本地存放地点，以及ip，端口，还有一些RPC形式的API。cd到eosio root目录下:

    luoxiaohui:~ luoxiaohui$ cd /Users/luoxiaohui/Library/Application\ Support/eosio/
    luoxiaohui:eosio luoxiaohui$ ls
    nodeos
    luoxiaohui:eosio luoxiaohui$ open nodeos/

可以看到如下目录：



其中的config.ini，就是本地节点的配置文件了，里面有各个属性参数，这里不一一讲了，有想法的自己去改了玩玩。

开启keosd服务

另外打开一个命令行，执行如下命令：

    keosd --http-server-address=127.0.0.1:8900

## 使用cleos命令创建钱包/密钥对/账户/部署合约等 ##

**创建钱包**

    luoxiaohui:~ luoxiaohui$ cleos wallet create
    Creating wallet: default
    Save password to use in the future to unlock this wallet.
    Without password imported keys will not be retrievable."PW5J37EZS54NyQSugYtW6LQsERgUNB9xCpCiEpnauR9qzsJGcN8vQ"

**创建密钥对**

    luoxiaohui:~ luoxiaohui$ cleos create key
    Private key: 5JohgswbXLNLFukkLHe8Na5uPBDi7TZBTRTrkLgbfDqwo7ZCA4X
    Public key: EOS6YnSB38qj7Z6ERet1rF6r4Z9azarPADMAGzrTCrCqjanw9fzXC

**创建账户**

先强调下，我目前所在的EOS版本是1.0.5，EOS几乎每升级一个小的版本，都会有蛮大的改变。目前1.0.5的坑我是躺平了，后面如果还有升级，升级后执行cleos命令有bug，请第一时间拷贝error日志到官方issue去找答案。

目前，1.0.5版本去部署bios合约时，有了大的改版，在前面提到的config.ini文件中屏蔽掉private-key，并设置如下属性:


    signature-provider=EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV=KEY:5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3

然后，创建eos的账户需要用到密钥对中的公钥，而在使用cleos命令创建账户之前，需要将此公钥所对应的私钥导入到钱包中，才能创建账户，否则会报错。最最重要的，就是只有部署了合约的账号，才能去创建账号，具体命令如下：

    luoxiaohui:eos luoxiaohui$ cleos wallet createCreating wallet: default
    Save password to use in the future to unlock this wallet.
    Without password imported keys will not be retrievable."PW5Kjfs7MYmbFtN4dFKoNUh1nKMvYV3nvyMcnDpLW3ZbfH7P8VSzn"luoxiaohui:eos
    luoxiaohui$ cleos wallet import EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
    Error 3010003: Invalid private keyPrivate key should be encoded in base58 WIF
    Error Details:
    Invalid private key: EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
    luoxiaohui:eos luoxiaohui$ cleos wallet import 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
    imported private key for: EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
    luoxiaohui:eos luoxiaohui$ cleos set contract eosio build/contracts/eosio.bios -p eosio
    Reading WAST/WASM from build/contracts/eosio.bios/eosio.bios.wasm...Using already assembled WASM...
    Publishing contract...
    executed transaction: ce79d77c59edcafffea30cc94661785c07a89a576e8dc6448745f4812c09548b  3720 bytes  7414 us
    # eosio <= eosio::setcode   {"account":"eosio","vmtype":0,"vmversion":0,"code":"0061736d0100000001621260037f7e7f0060057f7e7e7e7e...
    # eosio <= eosio::setabi{"account":"eosio","abi":"0e656f73696f3a3a6162692f312e30050c6163636f756e745f6e616d65046e616d650f7065...
    warning: transaction executed locally, but may not be confirmed by the network yet
    luoxiaohui:eos luoxiaohui$ cleos create keyPrivate key: 5JLZJyDGjhfZz2k6Yzm3LbRnpBuMrZN3rq2nJYHdRXYSyU4JF9xPublic key: EOS8aG1zYTGTXCLobmwqN5dYQz68st3qhwGiL7ekrYZFT2Jxk5QuV
    luoxiaohui:eos luoxiaohui$ cleos wallet import 5JLZJyDGjhfZz2k6Yzm3LbRnpBuMrZN3rq2nJYHdRXYSyU4JF9x
    imported private key for: EOS8aG1zYTGTXCLobmwqN5dYQz68st3qhwGiL7ekrYZFT2Jxk5QuV
    luoxiaohui:eos luoxiaohui$ cleos create account eosio test1 EOS8aG1zYTGTXCLobmwqN5dYQz68st3qhwGiL7ekrYZFT2Jxk5QuV
    executed transaction: 0a4175febf8d6001634ee0d525eaee6ee4a2e70730f7af336ff47ad4bb801c82  200 bytes  2740 us
    # eosio <= eosio::newaccount{"creator":"eosio","name":"test1","owner":{"threshold":1,"keys":[{"key":"EOS8aG1zYTGTXCLobmwqN5dYQz6...
    warning: transaction executed locally, but may not be confirmed by the network yet
    luoxiaohui:eos luoxiaohui$ cleos create account eosio test2 EOS8aG1zYTGTXCLobmwqN5dYQz68st3qhwGiL7ekrYZFT2Jxk5QuV
    executed transaction: 11fa7405e486403ce250b7ecb942aef27b3a2943241e6892abebbfb5a7faef2c  200 bytes  234 us
    # eosio <= eosio::newaccount{"creator":"eosio","name":"test2","owner":{"threshold":1,"keys":[{"key":"EOS8aG1zYTGTXCLobmwqN5dYQz6...
    warning: transaction executed locally, but may not be confirmed by the network yet

**部署合约**

    luoxiaohui:eos luoxiaohui$ cleos set contract test1 build/contracts/eosio.token -p test1
    Reading WAST/WASM from build/contracts/eosio.token/eosio.token.wasm...Using already assembled WASM...
    Publishing contract...
    executed transaction: b5dbab3a5ce41cc59ca3240529e78936155cde8bdea7201a8adecf268c3cc0fc  8104 bytes  5913 us
    # eosio <= eosio::setcode   {"account":"test1","vmtype":0,"vmversion":0,"code":"0061736d01000000017e1560037f7e7f0060057f7e7e7f7f...
    # eosio <= eosio::setabi{"account":"test1","abi":"0e656f73696f3a3a6162692f312e30010c6163636f756e745f6e616d65046e616d65050874...
    warning: transaction executed locally, but may not be confirmed by the network yet

**创建代币**

切记，只有部署过合约的账户，才能创建代币. 

创建一个叫FB的代币

    luoxiaohui:eos luoxiaohui$ cleos push action test1 create '[ "eosio", "100000000.0000 FB" ]' -p test1
    executed transaction: 237196f8ffacff045cf8a5e7b8aa0a6b11b8d149f64a9f02465f5da476c1198c  120 bytes  2247 us
    # test1 <= test1::create{"issuer":"eosio","maximum_supply":"100000000.0000 FB"}
    warning: transaction executed locally, but may not be confirmed by the network yet
    luoxiaohui:eos luoxiaohui$ cleos create account eosio test3 EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
    executed transaction: 8697a79671dd815cb6938e83b10609adccc68bb32931e7ec3043547861751777  200 bytes  212 us
    # eosio <= eosio::newaccount{"creator":"eosio","name":"test3","owner":{"threshold":1,"keys":[{"key":"EOS6MRyAjQq8ud7hVNYcfnVPJqc...
    warning: transaction executed locally, but may not be confirmed by the network yet
    luoxiaohui:eos luoxiaohui$ cleos push action test1 issue '[ "test3","100000000.0000 FB", "memo" ]' -p eosio
    executed transaction: 00e87d6a189087934ff81e1f33a0c4f004cf1ad1fb73371ab25ae7aa0237be74  128 bytes  2581 us
    # test1 <= test1::issue {"to":"test3","quantity":"100000000.0000 FB","memo":"memo"}
    # test1 <= test1::transfer  {"from":"eosio","to":"test3","quantity":"100000000.0000 FB","memo":"memo"}
    # eosio <= test1::transfer  {"from":"eosio","to":"test3","quantity":"100000000.0000 FB","memo":"memo"}
    # test3 <= test1::transfer  {"from":"eosio","to":"test3","quantity":"100000000.0000 FB","memo":"memo"}
    warning: transaction executed locally, but may not be confirmed by the network yet

**转账**

账户test3转账给test2

    1.luoxiaohui:eos luoxiaohui$ cleos push action test1 transfer '[ "test3", "test2", "25.0000 FB", "m" ]' -p test3  
    2.executed transaction: a5bb2c5bd92d7fee1a24efb4fa0afcc66ac8da39be0036aa55a2cf070049e8ac  128 bytes  595 us
    3.# test1 <= test1::transfer  {"from":"test3","to":"test2","quantity":"25.0000 FB","memo":"m"}
    4.# test3 <= test1::transfer  {"from":"test3","to":"test2","quantity":"25.0000 FB","memo":"m"}
    5.# test2 <= test1::transfer  {"from":"test3","to":"test2","quantity":"25.0000 FB","memo":"m"}
    warning: transaction executed locally, but may not be confirmed by the network yet

**查询代币余额**

这里的test1，表示创建了FB这个代币的账户，test2表示需要查询的账户，FB表示代币名称。

    1.luoxiaohui:eos luoxiaohui$ cleos get currency balance test1 test2 FB
    2.25.0000 FB
