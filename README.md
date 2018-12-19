# EOS智能合约安全

## 目录
- [概述](#introduction)
- [数据溢出](#overflow)
- [RAM被合约吞噬](#ram)
- [假币漏洞](#counterfeit)
- [假转账通知](#counterfeit_transfer)
- [失败回滚](#roll_back)
- [重放攻击](#redo)
- [拒绝收款](#refuse)
- [随机数攻破](#random)
- [EOS合约都是开源的](#open)
- [onchain调用](#onchain)
- [EOS不可逆块与双花攻击](#block)


###  <a name="introduction">概述</a>
随着EOS DAPP生态的爆发，引发了各种合约安全事件，主要以菠菜游戏为主，EOS俨然成为黑客的提款机，现在我们就来盘点一下。以下数据来自IMEOS的数据分析报告：

| 时间        | DApp名称          | 漏洞细节|      损失EOS  |
| ---------- | ----------- |----| ------------------------------ |
| 7.25| 狼人游戏 | 底层asset结构体存在缺陷，导致出现溢出| 60686|
| 8.26| EOSBET| RAM被恶意合约吞噬|未知|
| 8.27| Luckyos| 随机数产⽣的规律被黑客破解|未知|
| 9.02 | EOS.WIN|受到黑客随机数攻击|2000|
|9.10| DEOSBET|随机数产⽣的规律被黑客破解|4000|
|9.12|EOS Happy Slot|⿊客利用重放攻击，⼀次性获取多次中奖收益|5000|
|9.12 |FairDice|由于其随机算法与时间相关，同笔在不同时间会收获不同的结果。⿊客利用这一特性拒绝失败的开奖结果|未知|
|9.12|EOSBet|黑客利用代码漏洞，假币套取真币，在未投注的情况下中奖|42000|
|9.14|EOSBet|游戏合约在 apply 里没有校验 transfer action 的调用方必须是 eosio.token 或者是自己的游戏代币合约|
|9.14|Newdex|黑客利用假币在交易所转账换取真币|11803|
|9.15|EOS.WIN|受到假币攻击|4000|
|10.15|EOSBet|“假通知”漏洞：在处理 transfer 通知时未校验 transfer 中的 to 是否为 self|145321|（被追回）|
|10.16|WORLD CONQUEST|黑客利用游戏缴税规则，拒绝后续的买家，导致游戏非正常结束|4555|
|10.26|EosRoyale|随机数的漏洞，黑客能够设法通过使用先前的区块信息来计算未来的随机数|10800|
|10.28|EOS POKER|游戏方在拓展服务器时忘记将种子放入数据库中，因此有玩家利用获胜的种子赢取奖池|1374|
|10.31|EOSCast|游戏合约在 apply 里没有校验 transfer action的调用方必须是 eosio.token 或者是自己的游戏代币合约|70000|
|11.04|EOSDICE|随机数被攻破|2545|
|12.19|Betdice/EOSMax/TrueBet/BigGame/Tobet| 双花攻击| 几十万 |


这些漏洞大部分是由于程序员的疏忽或者对eos合约的原理不熟悉导致的，归纳起来可以分为几类：数据溢出，RAM被合约吞噬，假币，假转账通知，失败回滚，拒绝转账，随机数破击，双花攻击等。RAM被合约吞噬，是项目方故意为之，不属于漏洞。这里之所以列出来主要是为了让更多人了解EOS的内存消耗机制。

###  <a name="overflow">数据溢出</a>

这个话题没什么好讲的，任何一个有素养的程序员都知道怎么回事。这里想说的是，eos库中的很多类都实现了运算符重载，并在重载函数中做了溢出判断，比如asset类。所以尽量用重载函数运算。 下面是asset乘以一个int64_t的重载函数。

```CPP
  asset& operator*=( int64_t a ) {
         eosio_assert( a == 0 || (amount * a) / a == amount, "multiplication overflow or underflow" );
         eosio_assert( -max_amount <= amount, "multiplication underflow" );
         eosio_assert( amount <= max_amount,  "multiplication overflow" );
         amount *= a;
         return *this;
      }
```

###  <a name="ram">RAM被合约吞噬</a>
什么鬼，用户的RAM怎么能被合约吞噬！这个得从eos数据库说起。eos数据库跟我们传统的数据库不太一样，传统的数据库大部分数据是放在磁盘上面的，所以不用消耗太多的RAM（当然也有一些内存数据库），而EOS的数据库的数据是全部在内存中的。内存数据库的好处当然是为了更快的读写速度。eos提供了一个类multi_index来使用数据库，对于合约来说，我们不是直接操作数据库，而是操作这个类。合约要保存数据，必然需要消耗一定的RAM。所以在multi_index的插入数据emplace和修改数据modify函数中的第一个参数指定了RAM的支付者payer。

```CPP
  template<typename Lambda>
  const_iterator emplace( uint64_t payer, Lambda&& constructor ) {
    ...
}
```
```CPP
 template<typename Lambda>
  void modify( const T& obj, uint64_t payer, Lambda&& updater ){
   ...
}
```
只要有了用户授权，合约就可以消耗用户的RAM来存数据。这里顺便提一下，为什么往一个没有某种token的账号转账时消耗的是转出者的RAM。下面是token的合约，可以看到当转入的账号在数据库中不存在时，调用emplace函数，ram_payer是转出者账号。（ram_payer不可能是转入者账号，因为转账操作没有转入者的授权。完整的token代码[https://github.com/liyue201/eos-security/tree/master/eosio.token](https://github.com/liyue201/eos-security/tree/master/eosio.token)）

```CPP
void token::add_balance( account_name owner, asset value, const currency_stats& st, account_name ram_payer )
{
   accounts to_acnts( _self, owner );
   auto to = to_acnts.find( value.symbol.name() );
   if( to == to_acnts.end() ) {
      to_acnts.emplace( ram_payer, [&]( auto& a ){
        a.balance = value;
      });
   } else {
      to_acnts.modify( to, 0, [&]( auto& a ) {
        a.balance += value;
      });
   }
}
```
代币转走完了之后，从数据库中删除改账号，RAM会归还用户。
```CPP
void token::sub_balance( account_name owner, asset value, const currency_stats& st ) {
   accounts from_acnts( _self, owner );

   const auto& from = from_acnts.get( value.symbol.name() );
   eosio_assert( from.balance.amount >= value.amount, "overdrawn balance" );

   if( from.balance.amount == value.amount ) {
      from_acnts.erase( from );
   } else {
      from_acnts.modify( from, owner, [&]( auto& a ) {
          a.balance -= value;
      });
   }
}
```
所以EOS不适合大量空投token，因为要消耗项目方的RAM。当然项目方也想到了解决办法，就是让用户自己去领空投。具体就是在代币合约上增加一个action，用户调用该action领取token。因为这时已经取得用户的授权，所以就可以使用用户的RAM来存数据。项目方若是想作恶，可以在转账或者领空投的action中增加一些代码，存储大量数据，大量消耗用户的RAM。

###  <a name="counterfeit">假币漏洞</a>
EOS的token由两个要素构成，即发行合约（contract）和符号（symbol）。比如真正的EOS合约是eosio.token,符号是EOS。一个假币漏洞的例子是黑客发行了一个token，它的符号也叫EOS，然后他拿这个假的EOS去交易所购买别的token，交易所没有校验发行的contract是否是eosio.token，把它当成了真的EOS。对于合约怎么防止这种漏洞，请看下一个内容假转账通知。

###  <a name="counterfeit_transfer">假转账通知</a>
EOS中一个合约触发另外一个合约有两种方式，一个是直接调合约的action,另一个是使用require_recipient通知。一般在token合约的转账函数tranfer里面，会调用require_recipient，通知转出者和接收者。若转出者或接收者是合约账号，就可以收到通知，做进一步处理。

```CPP
void token::transfer( account_name from,
                      account_name to,
                      asset        quantity,
                      string       /*memo*/ )
{
    eosio_assert( from != to, "cannot transfer to self" );
    require_auth( from );
    eosio_assert( is_account( to ), "to account does not exist");
    auto sym = quantity.symbol.name();
    stats statstable( _self, sym );
    const auto& st = statstable.get( sym );

    require_recipient( from );
    require_recipient( to );

    eosio_assert( quantity.is_valid(), "invalid quantity" );
    eosio_assert( quantity.amount > 0, "must transfer positive quantity" );
    eosio_assert( quantity.symbol == st.supply.symbol, "symbol precision mismatch" );


    sub_balance( from, quantity, st );
    add_balance( to, quantity, st, from );
}
```

一般去中心化交易所或者菠菜游戏的合约代码大概是这么写的，在on _transfer中处理转账通知。代码中的两处注释就是防止假转账通知和假币。
若没有注释1中的判断，黑客就可以写一个攻击合约，用他的另外一个账号给他的攻击合约转账，在他的合约里面调require_recipient通知我们的合约。我们的合约以为给我们转账，实际上他的to并不是我们的合约。

```CPP
class game : public contract {
public:
    game(account_name self)
    {
    }
    void test()
    {
    }
    void on_transfer(const currency::transfer& t, account_name code)
    {
        require_auth(t.from);

        //1. 判断假转账通知
        if (t.from == _self || t.to != _self) {
            return;
        }

        //2. 判断是否是假币
        if (code != N(eosio.token) && t.quantity.symbol == S(4, EOS))
        {
           return;
        }

        eosio_assert(t.quantity.is_valid(), "invalid quantity");
    }

    void apply(account_name code, account_name action)
    {
        if (action == N(transfer)) {
            on_transfer(unpack_action_data<currency::transfer>(), code);
            return;
        }

        if (code != _self)
            return;

        auto& thiscontract = *this;
        switch (action) {
            EOSIO_API(game, (test));
        };
    }
};

extern "C" {
[[noreturn]] void apply(uint64_t receiver, uint64_t code, uint64_t action) {
    game app(receiver);
    app.apply(code, action);
    eosio_exit(0);
}
}
```

###  <a name="roll_back">失败回滚</a>
早期有一些骰子游戏逻辑是这样的，合约收到玩家的下注后，立即在合约内部根据区块随机数据还有时间戳等一些信息作为种子，生成随机数，判断输赢，若赢则立即给玩家转账。这种设计主要是因为初学者对eos合约的运行原理不熟悉导致的。eos一个transaction可以包含多个action，只要其中一个action运行失败，整个transaction都会失败，类似数据库的事务。黑客可以利用这个原理，实现这样一个攻击合约。代码如下：

```CPP
class mycontract : public eosio::contract
{
public:
  void attack()
  {
    //1: 读自己账号的余额
    uint64_t old_balance = getBalance();

    //2,这里调用游戏合约下注
    bet();

    //3,触发attackafter
    SEND_INLINE_ACTION(*this, attackafter, {_self, N(active)}, {old_balance});
  }

  void attackafter(uint64_t old_balance)
  {
      if (old_balance > getBalance()) {
           //输了，终止action，回滚之前的操作
           eosio_assert(0,  "lose");
      }
  }
};
```
黑客写了两个action，第一个先读自己合约账号的余额，再去调用游戏合约下注，最后触发另一个action，也就是上面的attackafter，再读一次余额。如果余额减少那肯定是输了，这时只要终止这个action，前面的action也都会失败，相当于没有下注，黑客的eos并有损失。只有赢的时候，整个transaction才会执行。黑客用这个合约下注就可以做到只赢不输。有些人可能会问，这里为什么要单独写attackafter这个action，而不是在attack函数中直接读处理。原因是eos合约的action都是异步执行的，这里一共3个action，在步骤2中调用合约的action不是立即执行，它要等到attack执行完才执行，所以在attack中读到的余额还是原来那个。attackafter是最后执行的action，当然能读到改变后的余额。后来的骰子游戏几乎都改成延后开奖的模式了，即玩家投注之后，再由合约或者中心服务器触发另外一个transation开奖。 但这并不意味着绝对安全，于是又有了重放攻击。

### <a name="redo">重放攻击</a>
黑客生成随机种子使用攻击合约小额下注，若赢了，开奖的action中会给攻击合约转账，这时攻击合约拒绝这个action执行，只需加一行代码eosio_assert(0,  "lose")，于是开奖失败。黑客再用这个随机种子大额下注。这类漏洞是因为随机种子使用次数的限制没处理好。


### <a name="refuse">拒绝收款</a>
拒绝收款和重放攻击类似。一个例子是WORLD CONQUES，黑客利用游戏缴税规则，拒绝后续的买家，导致游戏非正常结束。拒绝收款的合约大概是这样的

```CPP
class mycontract : public contract {
public:
    void on_transfer(const currency::transfer& t, account_name code)
    {
        if (t.to == _self)
        {
            //拒绝任何人给我转账
            eosio_assert(0, "");
        }
    }

    void apply(account_name code, account_name action)
    {
        if (action == N(transfer)) {
            on_transfer(unpack_action_data<currency::transfer>(), code);
            return;
        }
    }
};

extern "C" {
[[noreturn]] void apply(uint64_t receiver, uint64_t code, uint64_t action) {
    mycontract app(receiver);
    app.apply(code, action);
    eosio_exit(0);
}
}
```

###  <a name="random">随机数攻破</a>
BM提出了一个随机数方案，关于它的原理网上已经有很多了，这里就不重复了。那些随机数被攻破的项目自己反省一下。首先随机种子最好不要用链上的数据了，你能拿到的数据，别人都能拿到。

### <a name="open">EOS合约都是开源的</a>
你以为你的合约不开源就是安全的，但还是被攻击了。因为在黑客的眼里你的合约就是开源的。由于区块链的公开透明的特性，区块链上的数据所有人都能获取。EOS合约使用C++编写，编译成WASM格式部署，可以从区块链浏览器上拿到，再用一个工具将WASM转换成WAST格式。WAST是一种可读性良好的编程语言，是要稍加学习，就能读懂，再翻译成C++也不是难事。所以千万不要把私钥之类的重要信息放在合约代码里。这里推荐两个在线工具

* C++转WAST工具https://wasdk.github.io/WasmFiddle/
* WASM转WAST工具https://webassembly.github.io/wabt/demo/wasm2wat/

### <a name="onchain"> onchain调用 </a>

EOS ABI文件不是必须的，ABI只是合约的接口描述文件，就算不部署，也用被其他合约调用。

### <a name="block">EOS不可逆块与双花攻击</a>
EOS不可逆区块数大约335个，按0.5s出一个块，相当于167s。具体的原理可以参考这里：

* http://blog.eosdata.io/index.php/2018/09/21/guan-yu-eos-bu-ke-ni-kuai/

对于中心化或者伪中心化交易所，比如newdex这类链下撮合的交易所，需要考虑区块可逆的情况，收到转账后需要等待一定的区块数确认才算真正收到。若是完全去中心化交易所（链上撮合，链上结算的交易所）则不用考虑。12月19日凌晨多个菠菜合约遭到攻击就是基于这个原理。黑客抓住的DAPP节点没有读写分离的漏洞，直接用DAPP的读节点去发送交易，那么该节点会最早执行合约的逻辑计算DICE结果，如果黑客赢那就不做任何操作，等该节点广播同步到出块节点就赢了。如果黑客输了，黑客同时发送一笔转账操作到目前正在出块的节点，让余额不足以完成之前的交易，那么之前那笔交易就失效了，黑客也就不会输。






