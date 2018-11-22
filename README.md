### EOS合约安全指南

#### 概述
随着EOS DAPP生态的爆发，引发了各种合约安全事件，主要以菠菜游戏为主，EOS俨然成为黑客的提款机，现在我们就来盘点一下。以下数据来自IMEOS的数据分析报告：

| 字段名        | DApp名称          | 漏洞细节|      损失EOS  |
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
|11.04|EOSeven|内部成员进行大额转账，抛售SVN|

这些漏洞大部分是由于程序员的疏忽或者对eos合约的原理不熟悉导致的，归纳起来可以分为几类：数据溢出，RAM被合约吞噬，假币，假转账通知，随机数破击，失败回滚。RAM被合约吞噬，是项目方故意为之，不属于漏洞。这里之所以列出来主要是为了让更多人了解EOS的内存消耗机智。

####  数据溢出

这个话题没什么好讲的，任何一个有素养的程序员都知道怎么回事。这里想说的是，eos库中的很多类都实现了运算符重载，重载函数中进行了溢出判断，比如asset类。所以尽量用重载函数运算。 下面是asset乘以一个int64_t的重载函数。

```CPP
  asset& operator*=( int64_t a ) {
         eosio_assert( a == 0 || (amount * a) / a == amount, "multiplication overflow or underflow" );
         eosio_assert( -max_amount <= amount, "multiplication underflow" );
         eosio_assert( amount <= max_amount,  "multiplication overflow" );
         amount *= a;
         return *this;
      }
```

#### RAM被合约吞噬
什么鬼，用户的RAM怎么能被合约吞噬！这个得从eos数据库说起。eos数据库跟我们传统的数据库不太一样，传统的数据库大部分数据是放在磁盘上面的，所以不消耗太多RAM（当然也有一些内存数据库），而EOS的数据库的数据是全部在RAM中的。内存数据库的好处当然是为了更快的读写速度。eos提供了一个类multi_index来使用数据库，对于合约来说，我们不是直接操作数据库，而是操作这个类。 合约要保存数据，必然需要消耗一定的RAM。所以在multi_index的插入数据emplace和修改数据modify函数中的第一个参数指定了RAM的支付者payer。

```
  template<typename Lambda>
  const_iterator emplace( uint64_t payer, Lambda&& constructor ) {
    ...
}
```
```
 template<typename Lambda>
  void modify( const T& obj, uint64_t payer, Lambda&& updater ){
   ...
}
```
只要有了用户授权，合约就可以消耗用户的RAM来存数据。这里顺便提一下，为什么往一个没有某种token的账号转账时消耗的是转出者的RAM。下面是token的合约，可以看到当转入的账号在数据库中不存在时，调用emplace函数，ram_payer是转出者账号。n（ram_payer不可能是转入者账号，因为转账操作没有转入者的授权）
```
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
```
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
所以EOS不适合大量空投token，因为要消耗项目方的RAM。当然项目方也想到了解决办法，就是让用户自己领空投。具体就就是在代币合约上增加一个action，用户调用该action领取token，因为这时已经取得用户的授权，所以就可以使用用户的RAM来存数据。项目方若是想作恶，可以在转账的action或者领空投的action中增加一些代码，存大量数据，大量消耗用户的RAM。

### 假币漏洞
在说这个


