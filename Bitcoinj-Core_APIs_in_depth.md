# Working with transactions(处理交易)

## 介绍 {#Introduction1}
交易是比特币协议的基本要素。 他们封装了对某个价值的主张，并随后要求对该价值本身要求的条件。

本文将讨论：

- 什么是交易
- 交易类及其给您的好处
- 交易可信度
- 可以使用不同的交易方式

## 什么是交易？ {#What_is_a_transaction_}
交易本质上是*输入*和*输出*的集合。 它们还具有锁定时间（当前的比特币网络中未使用该锁定时间），以及仅以比特币形式表示的少量元数据（如上次交易时的时间）和置信度度量。 所有这些都使用`Transaction`类表示。

输出是指定 *value* 和 *script* 的数据结构，通常称为 *scriptPubKey* 。 输出将由交易输入收集的某些值分配给特定程序。 任何可以满足该程序（使其返回true）的人都可以要求在自己的交易中输出值。 输出脚本通常会根据公共密钥检查签名，但是它们可以做很多其他事情。

输入是一个数据结构，其中包含一个 *script*（实际上只是一个字节数组的列表）和一个 *outpoint* ，这是对另一个交易的输出的引用。 由于比特币通过其哈希来标识交易，因此出口是（哈希，索引）对，其中索引仅标识给定交易中的预期输出。我们说输入是 *connected* 到输出的。 输入脚本通常称为 *scriptSigs* ，理论上可以包含任何脚本操作码，但是由于程序是在没有输入的情况下运行的，因此这样做毫无意义，因此，实际的输入脚本只能将诸如签名和键之类的常量推入堆栈。

输入还包含 *序列号* 。 序列号当前未在比特币网络上使用，并且没有被bitcoinj公开。 它们的存在是为了支持[contracts(合同)](https://en.bitcoin.it/wiki/Contracts)。

交易输入收集的值与其输出花费的值之间的不匹配（如有）称为 *fee(费用)* 。 显然，在输出中具有比其输入所收集的价值更高的交易被视为无效，但反之则不成立。 谁成功开采了包含该交易的区块，谁都可以合法地主张未重新分配的价值。

请注意，输入不包含值字段。 要查找输入的值，必须检查其连接的输出的值。 这意味着，对于一个独立的交易，除非您还拥有其输入连接的所有交易，否则您将不知道其费用是多少。

交易可以被序列化，或者通过P2P网络广播，在这种情况下，矿工可以开始尝试将其包含在一个块中，或者使用其他协议在网络外部传递。

## 交易类 {#The_Transaction_class}
`Transaction`类允许反序列化、序列化、构建和检查交易。它还可以帮助您跟踪关于交易的有趣的元数据，比如已经包含了哪些块(如果有的话)，以及您对交易不会被逆转/重复使用的信心有多大。

输入由`TransactionInput`类表示，输出当然由`TransactionOutput`类处理。 输入可以是无符号的，在这种情况下，网络不会认为交易有效，但是中间状态有时会很有用。

**常见的任务**:

1. 使用 `getInputs()` , `getOutputs()` , `addInput()` 和 `addOutput()`访问输入和输出。注意，`addInput()` 有两种形式，其中一种采用 `TransactionInput` —这非常合乎逻辑。另一个是 `TransactionOutput` ，它将为您创建一个连接到输出的无符号输入。
2. 用`signInputs()`签署交易输入。 当前，您必须始终将`SigHash.ALL`作为第一个参数传递。 将来会支持其他类型的`SigHash`标志。 它们用于合同。 此方法还需要一个钱包，其中包含输入连接到输出所使用的密钥。 如果您没有私钥，那么显然您无法声明该值。
3. 使用`getAppearsInHashes()`查找交易中出现的块。因为交易只存储块头的散列，所以您可能需要使用一个已填充的`BlockStore(块存储)`来获取块数据本身。
4. 使用`getConfidence`了解链中的交易状态。它将返回一个`TransactionConfidence`对象，该对象表示关于交易何时何地被包含到块链中的各种数据位。

## 置信级别 {#Confidence_levels}
交易具有关联的 *confidence(信心)* 。 这些数据可用于计算交易被撤消（double spent双花）的可能性。 尽管在高网络速度下，这相对于传统支付系统而言，但这始终是比特币中的一种风险，尽管概率非常低。

置信度是使用`TransactionConfidence`对象建模的。 您可以通过调用`Transaction.getConfidence()`获得一个。 置信度数据在比特币序列化中不存在，但可以在Java和protobuf序列化中保留下来。

置信对象具有以下几种状态之一：

- 如果是 **BUILDING**，则表示该交易已包含在最佳链中，您对此的信心正在增加。
- 如果 **PENDING** ，则该交易未经确认，只要不时广播，并应被网络认为是有效的，则应将其包括在内。 如果包含的钱包已使用`PeerGroup.addWallet()`附加到实时的`PeerGroup`中，则将宣布待处理交易。 您可以使用`TransactionConfidence.numBroadcastPeers()`，通过测量在将交易发送给一个对等点后宣布该交易的节点数，来估计交易被包含的可能性。 或者，如果您从受信任的对等方看到它，则可以假定它是有效的，并且迟早也会被挖出。
- 如果为 **DEAD**，则表示除非再次进行重组，否则该交易将无法确认，因为其他某项交易正在消耗其投入之一。 应该向用户提醒此类交易，以便他们可以采取行动，例如，如果他们是商人，则中止货物的运输。
- 如果我们没有有关此交易的置信度的信息，则使用 **UNKNOWN** ，因为例如，它已从比特币结构中反序列化，但尚未在链中广播或看到。 UNKNOWN是默认状态。

可通过`TransactionConfidence.getConfidenceType()`获得的置信度类型是交易状态的一般说明。 您可以在对象上使用`getters`获得更精确的视图。 例如，在`BUILDING`状态下，`getDepthInBlocks()`应该以块为单位告诉您交易的深度。 它被埋在链中的深度越深，交易被撤销的机会就越少。

区块深度很容易理解，大致对应于确认交易的时间（平均1区块== 10分钟）。 但是，这并不是衡量撤消交易需要花费多少精力的稳定方法，因为在一个区块上完成的工作量会随时间而变化，这取决于正在进行的挖矿数量，而挖矿本身取决于汇率（vs 美元/欧元/等）。

### 理解困难与信心 {#Understanding_difficulty_and_confidence}
对`confidence信心`感兴趣的最常见原因是，您希望度量损失发送给您的钱的风险，例如延迟发送货物或提供服务。比特币社区使用零确认的经验法则来处理诸如MP3或电子书之类价值不高的事物，对于可能遭受双重消费攻击的事物使用一两个块（10-20分钟），或者 6个块（一小时），用于需要坚如磐石的确定性环境，例如货币兑换。

在实践中，关于商家遭受双重消费欺诈的报道很少，因此这种讨论多少有些理论性。

您会注意到上面引用的经验法则是用块表示的，而不是完成的工作。因此，如果汇率下跌，从而导致采矿减少，2个块提供的保障比以前少，这意味着你可能希望等待更长的时间。相反，如果汇率上升，采矿活动将增加，这意味着你可以等待更少的时间，直到交易有效，从而使客户更快乐。这就是为什么我们还提供了一种方法来衡量信心。

## 如何使用交易 {#How_transactions_can_be_used}
最常见和最明显的交易方式是:

1. 将它们从区块链或网络广播下载到您的钱包中
2. 创建支出，然后广播

但是，还有许多其他可能性。

### 直接转账 {#Direct_transfer}
可以通过直接给某人一笔交易来给某人汇款，然后他们可以在空闲时广播这笔交易，或者进一步修改这笔交易。这些用例目前还没有得到很好的支持，但将来可能会成为使用比特币的一种常见方式。

### 参与合约 {#Participation_in_contracts}
[Contracts(合约)](https://en.bitcoin.it/wiki/Contracts)允许在比特币网络的介导下进行各种低信任度交易。 通过精心构建具有特定脚本，签名和结构的交易，您可以进行低信任度的争议调解，锁定在任意条件下的硬币（例如，期货合约），保证合约，[智能财产](https://en.bitcoin.it/wiki/Smart_Property)和许多其他可能性。

您可以在文章[WorkingWithContracts](https://bitcoinj.github.io/working-with-contracts)中了解有关此主题的更多信息。



# 使用钱包 {#Working_with_the_wallet}

*了解如何使用wallet类并使用它处理定制交易。*

## 介绍 {#Introduction2}
Wallet类是bitcoinj中最重要的类之一。 它存储密钥和为这些密钥分配值或从中分配值的交易。 它允许您创建新的交易，使用之前存储的交易输出，并在钱包内容更改时通知您。

你需要学习如何使用Wallet来创建多种应用程序。

本文假定您已阅读Satoshi的白皮书和[WorkingWithTransactions](https://bitcoinj.github.io/working-with-transactions)文章。

## 设置 {#Setup}
为了实现最佳操作，钱包需要连接到`BlockChain区块链`和`Peer对等`或`PeerGroup`。 可以在其构造函数中将钱包传递给区块链。 它将在收到钱包块时发送它们，以便钱包可以查找并提取 *相关交易*，即将硬币发送或接收到其中存储的密钥的交易。 `Peer/Group`将发送钱包交易，这些交易在整个网络中广播，然后再出现在一个区块中。

无论区块链包含什么内容，钱包在开始使用时没有任何交易，因此余额为零。要使用它，你需要下载区块链，这将加载钱包的交易，可以分析和消费。

```java
Wallet wallet = new Wallet(params);
BlockChain chain = new BlockChain(params, wallet, ...);
PeerGroup peerGroup = new PeerGroup(params, chain);
peerGroup.addWallet(wallet);
peerGroup.startAndWait();
```

## 获取地址 {#Getting_addresses}
当然，这段代码是毫无用处的，因为没有办法把钱弄进去。您可以通过以下API调用从钱包中获取密钥和地址:

```java
Address a = wallet.currentReceiveAddress();
ECKey b = wallet.currentReceiveKey();
Address c = wallet.freshReceiveAddress();

assert b.toAddress(wallet.getParams()).equals(a);
assert !c.equals(a);
```

然后可以将其分发以接收付款。 电子钱包具有“当前”地址的概念。 这适用于希望始终显示地址的GUI钱包。 一旦看到当前地址正在使用，它将更改为新地址。 另一方面，`freshReceiveKey/Address`方法始终返回新派生的地址。

## 种子和助记码 {#Seeds_and_mnemonic_codes}
这些方法返回的密钥和地址使用`BIP 32` 和 `BIP 39` 中列出的算法从种子中确定性地得出。密钥的寿命如下所示：

1. 一个新的Wallet对象使用`SecureRandom`选择128位随机熵。
2. 这种随机性被转化为“mnemonic code 助记码”;使用`BIP 39`标准预先定义的字典的一组12个单词。
3. 12个单词的字符串形式用作密钥派生算法（PBKDF2）的输入，并进行多次迭代以获得“seed 种子”。 请注意，种子并非只是您所期望的使用单词表示的原始随机熵，而是种子本身的UTF-8字节序列的派生词。
4. 然后将新计算出的种子分为主私钥和“chain code 链码”。 在一起，它们允许使用`BIP 32`中指定的算法来迭代密钥树。此算法利用椭圆曲线数学的属性，允许迭代序列中的公钥而无需访问等效的私钥，这对于在提供密码时不需要对钱包解密的自动售货地址非常有用。 bitcoinj使用`BIP 32`中推荐的默认推荐树结构。
5. 钱包会预先计算一组 *lookahead keys超前密钥* 。 这些密钥尚未由电子钱包通过`current/freshReceiveKey` API发布，但将会在将来发布。 预计算可以实现几个目标。 第一，它使这些API变得快速，这对于不想等待可能慢的EC数学完成的GUI应用程序很有用。 第二，它允许钱包注意到与尚未发行的密钥进行的交易/来自尚未发行的密钥的交易-如果已将钱包种子克隆到多个设备，并且正在重播区块链，则可能发生这种情况。

（预先）计算出的种子和密钥将保存到磁盘中，以避免在加载钱包时出现缓慢的重新分配循环。

与原始私钥相比，“wallet words 钱包里的单词”旨在更易于使用和写下来：用户可以使用笔和纸轻松地与他们一起使用，而很少有意外将事情记错的可能性。 因此，建议您将这些字词作为备份机制提供给用户（请确保他们也记下日期，以加快还原速度）。

您可以像这样处理种子：

```java
DeterministicSeed seed = wallet.getKeyChainSeed();
println("Seed words are: " + Joiner.on(" ").join(seed.getMnemonicCode()));
println("Seed birthday is: " + seed.getCreationTimeSeconds());

String seedCode = "yard impulse luxury drive today throw farm pepper survey wreck glass federal";
long creationtime = 1409478661L;
DeterministicSeed seed = new DeterministicSeed(seedCode, null, "", creationtime);
Wallet restoredWallet = Wallet.fromSeed(params, seed);
// now sync the restored wallet as described below.
```

在使钱包保持同步时，超前区域发挥着重要作用。 默认区域为100个键。 这意味着，如果将钱包A克隆到钱包B，并且钱包A发行了50个密钥，而实际上只有最后一个密钥用于接收付款，则钱包B仍会注意到该付款并移动其超前区域，这样B总共跟踪了150个密钥。如果A钱包分发了120个钥匙，而只有110个收到了付款，B钱包就不会注意到发生了什么。因此，在同步钱包的时候，你需要知道在任何时候有多少未完成的地址在等待付款。默认值100被选择为适用于消费者钱包，但是在商家场景中，您可能需要更大的区域。

## Replaying the chain {#Replaying_the_chain}
If you import non-fresh keys to a wallet that already has transactions in it, to get the transactions for the added keys you must remove transactions by resetting the wallet (using the `reset` method) and re-download the chain. Currently, there is no way to replay the chain into a wallet that already has transactions in it and attempting to do so may corrupt the wallet. This is likely to change in future. Alternatively, you could download the raw transaction data from some other source, like a block explorer, and then insert the transactions directly into the wallet. However this is currently unsupported and untested. For most users, importing existing keys is a bad idea and reflects some deeper missing feature. Talk to us if you feel a burning need to import keys into wallets regularly.

The Wallet works with other classes in the system to speed up synchronisation with the block chain, but only some optimisations are on by default. To understand what is done and how to configure a Wallet/PeerGroup for optimal performance, please read [SpeedingUpChainSync](https://bitcoinj.github.io/speeding-up-chain-sync).

## Creating spends {#Creating_spends}
After catching up with the chain, you may have some coins available for spending:

```
System.out.println("You have " + Coin.FRIENDLY_FORMAT.format(wallet.getBalance()));
```

Spending money consists of four steps:

1. Create a send request
2. Complete it
3. Commit the transaction and then save the wallet
4. Broadcast the generated transaction

For convenience there are helper methods that do these steps for you. In the simplest case:

```
// Get the address 1RbxbA1yP2Lebauuef3cBiBho853f7jxs in object form.
Address targetAddress = new Address(params, "1RbxbA1yP2Lebauuef3cBiBho853f7jxs");
// Do the send of 1 BTC in the background. This could throw InsufficientMoneyException.
Wallet.SendResult result = wallet.sendCoins(peerGroup, targetAddress, Coin.COIN);
// Save the wallet to disk, optional if using auto saving (see below).
wallet.saveToFile(....);
// Wait for the transaction to propagate across the P2P network, indicating acceptance.
result.broadcastComplete.get();
```

The `sendCoins` method returns both the transaction that was created, and a `ListenableFuture` that let’s you block until the transaction has been accepted by the network (sent to one peer, heard back from the others). You can also register a callback on the returned future to learn when propagation was complete, or register your own `TransactionConfidence.Listener` on the transaction to watch the progress of propagation and mining yourself.

At a lower level, we can do these steps ourselves:

```
// Make sure this code is run in a single thread at once.
SendRequest request = SendRequest.to(address, value);
// The SendRequest object can be customized at this point to modify how the transaction will be created.
wallet.completeTx(request);
// Ensure these funds won't be spent again.
wallet.commitTx(request.tx);
wallet.saveToFile(...);
// A proposed transaction is now sitting in request.tx - send it in the background.
ListenableFuture<Transaction> future = peerGroup.broadcastTransaction(request.tx);

// The future will complete when we've seen the transaction ripple across the network to a sufficient degree.
// Here, we just wait for it to finish, but we can also attach a listener that'll get run on a background
// thread when finished. Or we could just assume the network accepts the transaction and carry on.
future.get();
```

To create a transaction we first build a `SendRequest` object using one of the static helper methods. A `SendRequest` consists of a partially complete (invalid) `Transaction` object. It has other fields let you customize things that aren’t done yet like the fee, change address and maybe in future privacy features like how coins are selected. You can modify the partial transaction if you like, or simply construct your own from scratch. The static helper methods on `SendRequest` are simply different ways to construct the partial transaction.

Then we complete the request - that means the transaction in the send request has inputs/outputs added and signed to make the transaction valid. It’s now acceptable to the Bitcoin network.

Note that between `completeTx` and `commitTx` no lock is being held. So it’s possible for this code to race and fail if the wallet changes out from underneath you - for example, if the keys in the wallet have been exported and used elsewhere, and a transaction that spends the selected outputs comes in between the two calls. When you use the simpler construction the wallet is locked whilst both operations are run, ensuring that you don’t end up trying to commit a double spend.

## When to commit a transaction {#When_to_commit_a_transaction}
To *commit* a transaction means to update the spent flags of the wallets transactions so they won’t be re-used. It’s important to commit a transaction at the right time and there are different strategies for doing so.

The default sendCoins() behaviour is to commit and then broadcast, which is a good choice most of the time. It means there’s no chance of accidentally creating a double spend if there’s a network problem or if there are multiple threads trying to create and broadcast transactions simultaneously. On the other hand, it means that if the network doesn’t accept your transaction for some reason (eg, insufficient fees/non-standard form) the wallet thinks the money is spent when it wasn’t and you’ll have to fix things yourself.

You can also just not call `wallet.commitTx` and use `peerGroup.broadcastTransaction` instead. Once a transaction has been seen by a bunch of peers it will be given to the wallet which will then commit it for you. The main reason you may want to commit after successful broadcast is if you’re experimenting with new code and are creating transactions that won’t necessarily be accepted. It’s annoying to have to constantly roll back your wallet in this case. Once you know the network will always accept your transactions you can create the send request, complete it and commit the resulting transaction all under a single lock so multiple threads won’t create double spends by mistake.

## Understanding balances and coin selection {#Understanding_balances_and_coin_selection}
The `Wallet.getBalance()` call has by default behaviour that may surprise you. If you broadcast a transaction that sends money and then immediately afterwards check the balance, it may be lower than what you expect (or even be zero).

The reason is that bitcoinj has a somewhat complex notion of *balance*. You need to understand this in order to write robust applications. The `getBalance()` method has two alternative forms. In one, you pass in a `Wallet.BalanceType` enum. It has four possible values, `AVAILABLE`, `ESTIMATED`, `AVAILABLE_SPENDABLE` and `ESTIMATED_SPENDABLE`. Often these will be the same, but sometimes they will vary. The other overload of `getBalance()` takes a `CoinSelector` object.

The `ESTIMATED` balance is perhaps what you may imagine as a “balance” - it’s simply all outputs in the wallet which are not yet spent. However, that doesn’t mean it’s really safe to spend them. Because Bitcoin is a global consensus system, deciding when you can really trust you received money and the transaction won’t be rolled back can be a subtle art.

This concept of safety is therefore exposed via the `AVAILABLE` balance type, and the `CoinSelector` abstraction. A coin selector is just an object that implements the `CoinSelector` interface. That interface has a single method, which given a list of all unspent outputs and a target value, returns a smaller list of outputs that adds up to at least the target (and possibly more). Coin selection can be a complex process that trades off safety, privacy, fee-efficiency and other factors, which is why it’s pluggable.

The default coin selector bitcoinj provides (`DefaultCoinSelector`) implements a relatively safe policy: it requires at least one confirmation for any transaction to be considered for selection, except for transactions created by the wallet itself which are considered spendable as long as it was seen propagating across the network. This is the balance you get back when you use `getBalance(BalanceType.AVAILABLE)` or the simplest form, `getBalance()` - it’s the amount of money that the spend creation methods will consider for usage. The reason for this policy is as follows:

1. You trust your own transactions implicitly, as you “know” you are trustworthy. However, it’s still possible for bugs and other things to cause you to create unconfirmable transactions - for instance if you don’t include sufficient fees. Therefore we don’t want to spend change outputs unless we saw that network nodes relayed the transaction, giving a high degree of confidence that it will be mined upon.
2. For other transactions, we wait until we saw at least one block because in SPV mode, you cannot check for yourself that a transaction is valid. If your internet connection was hacked, you might be talking to a fake Bitcoin network that feeds you a nonsense transaction which spends non-existent Bitcoins. Please read the [SecurityModel](https://bitcoinj.github.io/security-model) article to learn more about this. Waiting for the transaction to appear in a block gives you confidence the transaction is real. The original Bitcoin client waits for 6 confirmations, but this value was picked at a time when mining hash rate was much lower and it was thus much easier to forge a small fake chain. We’ve not yet heard reports of merchants being defrauded using invalid blocks, so waiting for one block is sufficient.

The default coin selector also takes into account the age of the outputs, in order to maximise the probability of your transaction getting confirmed in the next block.

The default selector is somewhat customisable via subclassing. But normally the only customisation you’re interested in is being able to spend unconfirmed transactions. If your application knows the money you’re receiving is valid via some other mechanism (e.g. you are connected to a trusted node), or you just don’t care about fraud because the value at stake is very low, then you can call `Wallet.allowSpendingUnconfirmedTransactions()` to make the wallet use a different selector that drops the 1-block policy.

You can also choose the coin selector on a per-payment basis, using the `SendRequest.coinSelector` field.

As a consequence of all of the above, if you query the `AVAILABLE` balance immediately after doing a `PeerGroup.broadcastTransaction`, you are likely to see a balance lower than what you anticipate, as you’re racing the process of broadcasting the transacation and seeing it propagate across the network. If you instead call `getBalance()` after the returned `ListenableFuture` completes successfully, you will see the balance you expect.

The difference between `AVAILABLE_SPENDABLE` and `AVAILABLE` variants is whether the wallet considers outputs for which it does not have the keys. In a “watching wallet”, the wallet may be tracking the balance of another wallet elsewhere by watching for the public keys. By default, watched addresses/scripts/keys are considered to be a part of the balance even though attempting to spend those outputs would fail. The only time these two notions of balance differ is if you have a mix of private keys and public-only keys in your wallet: this can occur when writing advanced contracts based applications but otherwise should never crop up.

## Using fees {#Using_fees}
Transactions can have fees attached to them when they are completed by the wallet. To control this, the `SendRequest` object has several fields that can be used. The simplest is `SendRequest.fee` which is an absolute override. If set, that is the fee that will be attached. A more useful field is `SendRequest.feePerKb` which allows you to scale the final fee with the size of the completed transaction. When block space is limited, miners decide ranking by fee-per-1000-bytes so typically you do want to pay more for larger transactions, otherwise you may fall below miners fee thresholds.

There are also some fee rules in place intended to avoid cheap flooding attacks. Most notably, any transaction that has an output of value less than 0.01 coins ($1) requires the min fee which is currently set to be 10,000 satoshis (0.0001 coins or at June 2013 exchange rates about $0.01). By default nodes will refuse to relay transactions that don’t meet that rule, although mining them is of course allowed. You can disable it and thus create transactions that may be un-relayable by changing `SendRequest.ensureMinRequiredFee` to false.

bitcoinj will by default ensure you always attach a small fee per kilobyte to each transaction regardless of whether one is technically necessary or not. The amount of fee used depends on the size of the transaction. You can find out what fee was attached by reading the fee field of `SendRequest` after completion. There are several reasons for why bitcoinj always sets a fee by default:

- Most apps were already setting a fixed fee anyway.
- There is only 27kb in each block for free “high priority” transactions. This is a quite small amount of space which will often be used up already.
- SPV clients have no way to know whether that 27kb of free-high-priority space is already full in the next block.
- The default fee is quite low.
- It more or less guarantees that the transaction will confirm quickly.

Over time, it’d be nice to get back to a state where most transactions are free. However it will require some changes on the C++ and mining side along with careful co-ordinated rollouts.

## Learning about changes {#Learning_about_changes}
The wallet provides the `WalletEventListener` interface for learning about changes to its contents. You can derive from `AbstractWalletEventListener` to get a default implementation of these methods. You get callbacks on:

- Receiving money.
- Money being sent from the wallet (regardless of whether the tx was created by it or not).
- Changes in transaction confidence. See [WorkingWithTransactions](https://bitcoinj.github.io/working-with-transactions) for information about this.

## Saving the wallet at the right times {#Saving_the_wallet_at_the_right_times}
By default the Wallet is just an in-memory object, it won’t save by itself. You can use `saveToFile()` or `saveToOutputStream()` when you want to persist the wallet. It’s best to use `saveToFile()` if possible because this will write to a temporary file and then atomically rename it, giving you assurance that the wallet won’t be half-written or corrupted if something goes wrong half way through the saving process.

It can be difficult to know exactly when to save the wallet, and if you do it too aggressively you can negatively affect the performance of your app. To help solve this, the wallet can auto-save itself to a named file. Use the `autoSaveToFile()` method to set this up. You can optionally provide a delay period, eg, of a few hundred milliseconds. This will create a background thread that saves the wallet every N milliseconds if it needs saving. Note that some important operations, like adding a key, always trigger an immediate auto-save. Delaying writes of the wallet can help improve performance in some cases, eg, if you’re catching up a wallet that is very busy (has lots of transactions moving in and out). You can register an auto-save listener to learn when the wallet saved itself.

## Wallet maintenance and key rotation {#Wallet_maintenance_and_key_rotation}
The wallet has a notion of *maintenance*, which currently exists purely to support *key rotation*. Key rotation is useful when you believe some keys might be weak or compromised and want to stop using them. The wallet knows how to create a fresh HD key hierarchy and create spends that automatically move coins from rotating keys to the new keys. To start this process, you tell the wallet the time at which you believe the existing keys became weak, and then use the `doMaintenance(KeyParameter, boolean)` method to obtain transactions that move coins to the fresh keys. Note that you can’t mark individual keys as weak, only groups based on their creation time.

Examples of when this can be useful:

- You learn that the random number generator used to create some keys was predictable
- You have a backup somewhere that you can’t reliably delete, and worry that its keys may be vulnerable to theft. For example wallets written to flash/solid state disks can be hard to reliably erase.

The `doMaintenance` method takes the users password key if there is one, and a boolean controlling whether the needed maintenance transactions will actually be broadcast on the network. It returns a future that completes either immediately (if the bool argument was false), or when all maintenance transactions have broadcast, and the future vends a list of the transactions that were created. The method might potentially throw an exception if it needs the users password key, even if the bool argument is false, so you should be ready to catch that and ask the user to provide their password - probably with an explanation of why. If no exception is thrown and the argument is false, you can use `get()` on the returned future to obtain the list and then check if it’s empty. If it’s empty, no maintenance is required. If it’s not, you need to tell the user what’s going on and let it broadcast the transactions.

The concept of maintenance is general. In future, the wallet might generate maintenance transactions for things like defragmenting the wallet or increasing user privacy. But currently, if you don’t use key rotation, this method will do nothing.

## Creating multi-sends and other contracts {#Creating_multi_sends_and_other_contracts}
The default `SendRequest` static methods help you construct transactions of common forms, but what if you want something more advanced? You can customize or build your own transaction and put it into a `SendRequest` yourself.

Here’s a simple example - sending to multiple addresses at once. A fresh `Transaction` object is first created with the outputs specifying where to send various quantities of coins. Then the incomplete, invalid transaction is *completed*, meaning the wallet adds inputs (and a change output) as necessary to make the transaction valid.

```
Address target1 = new Address(params, "17kzeh4N8g49GFvdDzSf8PjaPfyoD1MndL");
Address target2 = new Address(params, "1RbxbA1yP2Lebauuef3cBiBho853f7jxs");
Transaction tx = new Transaction(params);
tx.addOutput(Utils.toNanoCoins(1, 10), target1);
tx.addOutput(Utils.toNanoCoins(2, 20), target2);
SendRequest request = SendRequest.forTx(tx);
if (!wallet.completeTx(request)) {
  // Cannot afford this!
} else {
  wallet.commitTx(request.tx);
  peerGroup.broadcastTransaction(request.tx).get();
}
```

You can add arbitrary `TransactionOutput` objects and in this way, build transactions with exotic reclaim conditions. For examples of what is achievable, see [contracts](https://en.bitcoin.it/wiki/Contracts).

At this time `SendRequest` does not allow you to request unusual forms of signing like `SIGHASH_ANYONECANPAY`. If you want to do that, you must use the lower level APIs. Fortunately it isn’t hard - you can see examples of how to do that in the article [WorkingWithContracts](https://bitcoinj.github.io/working-with-contracts).

## Encrypting the private keys {#Encrypting_the_private_keys}
It’s a good idea to encrypt your private keys if you only spend money from your wallet rarely. The `Wallet.encrypt("password")` method will derive an AES key from an Scrypt hash of the given password string and use it to encrypt the private keys in the wallet, you can then provide the password when signing transactions or to fully decrypt the wallet. You can also provide your own AES keys if you don’t want to derive them from a password, and you can also customize the Scrypt hash parameters.

[Scrypt](http://www.tarsnap.com/scrypt.html) is a hash function designed to be harder to brute force at high speed than alternative algorithms. By selecting difficult parameters, a password can be made to take multiple seconds to turn into a key. In the WalletTemplate application you can find code that shows how to measure the speed of the host computer and then select scrypt parameters to give encryption/decryption time of a couple of seconds.

Once encrypted, you will need to provide the AES key whenever you try and sign transactions. You do this via the `SendRequest` object:

```
Address a = new Address(params, "1RbxbA1yP2Lebauuef3cBiBho853f7jxs");
SendRequest req = SendRequest.to(a, Coin.parseCoin("0.12"));
req.aesKey = wallet.getKeyCrypter().deriveKey("password");
wallet.sendCoins(req);
```

The wallet can be decrypted by using the `wallet.decrypt` method which takes either a textual password or a `KeyParameter`.

Note that because bitcoinj saves wallets by creating temp files and then renaming them, private key material may still exist on disk even after encryption. Especially with modern SSD based systems deleting data on disk is rarely completely possible. Encryption should be seen as a reasonable way to raise the bar against adversaries that are not extremely sophisticated. But if someone has already gained access to their wallet file, they could theoretically also just wait for the user to type in their password and obtain the private keys this way. Encryption is useful but should not be regarded as a silver bullet.

## Watching wallets {#Watching_wallets}
A wallet can cloned such that the clone is able to follow transactions from the P2P network but doesn’t have the private keys needed for spending. This arrangement is very useful and makes a lot of sense for things like online web shops. The web server can observe all payments into the shops wallet, so knows when customers have paid, but cannot authorise withdrawals from that wallet itself thus significantly reducing the risk of hacking.

To create a watching wallet from another HD BIP32/bitcoinj wallet:

```
Wallet toWatch = ....;

DeterministicKey watchingKey = toWatch.getWatchingKey();

// Get the standardised base58 encoded serialization
System.out.println("Watching key data: " + watchingKey.serializePubB58());
System.out.println("Watching key birthday: " + watchingKey.getCreationTimeSeconds());

//////////////////////////////////////////////////////////////////////////////////////////

DeterministicKey key = DeterministicKey.deserializeB58(null, "key data goes here");
long keyBirthday = 12345678L;
Wallet watchingWallet = Wallet.fromWatchingKey(params, key, keyBirthday);
// now attach watchingWallet and replay the chain as normal.
```

The “watching key” in this case is the public form of the BIP32 account zero key i.e. it’s the first key derived from the master. Because of how BIP32 works, the watching key cannot be the master public key itself. The internal (change) and external (for handing out) chains are both below this and thus all keys generated by each wallet will be in sync. In the web server case, the server can just call `wallet.freshReceiveAddress()` as normal. The original wallet will be able to see all the payments received to the watching wallet and also spend them, when synced. But it can be stored offline (a so called “cold wallet”) for extra security.

You can also create a wallet that watches arbitrary addresses or scripts even if you don’t have the HD watching key:

```
wallet.addWatchedAddress(...);
wallet.addWatchedScripts(List.of(script, script, script));
```

Obviously in this case you don’t get auto synchronisation of new keys/addresses/scripts.

## Married/multi-signature wallets and pluggable signers {#Married_multi_signature_wallets_and_pluggable_signers}
Starting from bitcoinj 0.12 there is some support for wallets which require multiple signers to cooperate to sign transactions. This allows the usage of a remote “risk analysis service” which may only countersign transactions if some additional authorisation has been obtained or if the transaction seems to be low risk. A wallet that requires the cooperation of a third party in this way is called a *married wallet*, because it needs the permission of its spouse to spend money :-)

To marry a wallet to another one, the spouse must also be an HD wallet using the default BIP 32 tree. You must obtain the watching key of that other wallet via some external protocol (it can be retrieved as outlined above) and then do:

```
DeterministicKey spouseKey = ....;
wallet.addFollowingAccountKeys(Lists.newArrayList(spouseKey), 2);   // threshold of 2 keys, i.e. both are needed to spend.

Address a = wallet.freshReceiveAddress();
assert a.isP2SHAddress();
```

Once a wallet has a “following key” it becomes married and the API changes. You can no longer call `freshReceiveKey` or `currentReceiveKey` because those are specified to return single keys and a married wallet can only have money sent to it using pay-to-script-hash (P2SH) addresses, which start with the number 3 when written down. Instead you should use `currentReceiveAddress` and `freshReceiveAddress` exclusively.

The following key HD hierarchy must match the key hierarchy used by the wallet itself. That is, it’s not possible for the remote server to have a single HD key hierarchy that is shared between all users: each user must have their own key tree.

The way you spend money also changes. Whilst the high level API remains the same, the Wallet will throw an exception if you try to send coins without installing a pluggable transaction signer. The reason is, whilst it knows how to sign with its own private keys, it doesn’t know how to talk to the remote service and get the relevant signatures to finish things off.

A pluggable transaction signer is an object that implements the `TransactionSigner` interface. It must have a no-args constructor and can provide data that will be serialized to the wallet when saved, such as authentication credentials. During setup it should be created, configured with whatever info is required to let it talk to the remote service, authenticate and obtain signatures. Once configured the `isReady` method should start returning true, and it can now be added to the wallet using the `Wallet.addTransactionSigner` method. This only has to be done once as deserialization will recreate the object using reflection and then use the `TransactionSigner.deserialize(byte[])` method to reload it.

The Wallet class runs transaction signers in sequence. First comes the `LocalTransactionSigner` which knows how to use the private keys in the wallet’s `KeyBag`. Then comes yours. The `TransactionSigner.signInputs(ProposedTransaction, KeyBag)` method is called and the transaction inside the `ProposedTransaction` object should have signatures fetched from the remote service and inserted as appropriate. The HD key paths to use can be retrieved from the `ProposedTransaction` object.

Married wallets support is relatively new and experimental. If you do something with it, or encounter problems, let us know!

## Connecting the wallet to a UTXO store {#Connecting_the_wallet_to_a_UTXO_store}
By default the wallet expects to find unspent outputs by scanning the block chain using Bloom filters, in the normal SPV manner. In some use cases you may already have this data from some other source, for example from a database built by one of the [full pruned block stores](https://bitcoinj.github.io/full-verification). You can configure the Wallet class to use such a source instead of its own data and as such, avoid the need to directly use the block chain. This can be convenient when attempting to build something like a web wallet, where wallets need to be loaded and discarded very quickly.

To configure the wallet in this manner, you need an implementation of the `UTXOProvider` interface. This interface is fairly simple and has two core methods, one to return the block height known by the provider and the other to return a list of `UTXO` objects given a list of `Address` objects:

```
public interface UTXOProvider {
    List<UTXO> getOpenTransactionOutputs(List<Address> addresses) throws UTXOProviderException;
    ....
 }
```

Note that this API will probably change in future to take a list of scripts instead of/as well as a list of addresses, as not every output type can be expressed with an address.

Conveniently the database backed `FullPrunedBlockStore` implementations that bitcoinj ships with (Postgres, MySQL, H2) implement this interface, so if you use them you can connect the wallet to the databases directly.

Once you have a provider, just use `Wallet.setUTXOProvider` to plug it in. The provider will then be queried for spendable outputs whenever the balance is calculated or a spend is completed.

Note that the UTXO provider association is not serialized and must be reset every time the wallet is loaded. For typical web wallet scenarios however this isn’t a big deal as most likely, the wallet will be re-constructed from the random seed each time it is needed rather than being explicitly serialized.



# Working with monetary amounts

## The Coin class {#The_Coin_class}
Bitcoin amounts in the bitcoinj API are represented using the `Coin` class. `Coin` is superficially similar to `BigInteger` except that it wraps a long internally and thus cannot represent arbitrarily large quantities of bitcoin. As there is a global limit on how many bitcoins can exist this is unlikely to pose an issue for the forseeable future. The raw number of satoshis represented by a `Coin` instance is accessible via the public final field `value`, and the rest of the class is about making it easier and more type safe to work with these amounts.

Methods are provided to do the following operations in a type safe manner:

1. add, subtract, multiply, divide, divideAndRemainder
2. isPositive, isNegative, isZero, isGreaterThan, isLessThan
3. shift left, shift right, negate
4. parse from string, for example `Coin.parseCoin("1.23")` will parse the given amount as a bitcoin quantity.
5. build from a raw value in satoshis

There are also static instances representing ZERO, COIN, CENT, MILLICOIN, MICROCOIN, SATOSHI and FIFTY_COINS.

`Coin` implements `Comparable`, `Serializable` and also the bitcoinj provided `Monetary` interface. The `Monetary` interface represents any monetary amount in any currency, represented as a long with a “smallest unit exponent”. For Bitcoin this is 8 but for most national/state currencies it will be either two or zero.

## Formatting {#Formatting}
bitcoinj provides two APIs for formatting `Coin` amounts to and from strings:

1. `MonetaryFormat` is an immutable formatting class that can be configured to use various different denominations (BTC, mBTC, uBTC etc), varying numbers of digits after the decimal point, rounding modes, and allowing the currency code to be placed before or after the amount. Some pre-configured instances are provided as static instances on both `MonetaryFormat` and `Coin`. `MonetaryFormat` is intended to be usable with fiat currencies as well as Bitcoin.
2. `BtcFormat` is a larger and more complex API that builds on the Java `java.text.Format` infrastructure. It has extensive JavaDocs that explain its many features, including but not limited to: automatic selection of the most appropriate denomination, ability to parse strings that include unicode Bitcoin symbols and denomination codes, padding of strings so that formatted amounts can align around the decimal point, locale sensitivity for formatting, and more.

Which API to use is up to you: they were contributed by two different contributors and we’re waiting to gain experience in which ones are most popular, with an eye to creating a unified API in future.



# How to handle networking/peer APIs

*This article applies to the code in git master only*

## Introduction {#Introduction3}
The bitcoinj networking APIs have a few options targeted at different use-cases - you can spin up individual Peers and manage them yourself or bring up a `PeerGroup` to let it manage them, you can use one-off sockets or socket managers, and you can use blocking sockets or NIO/non-blocking sockets. This page attempts to explain the tradeoffs and use-cases for each choice, as well as provide some basic examples to doing more advanced networking.

The bitcoinj networking API is built up in a series of layers. On the bottom are simple wrapper classes that provide an API to open new connections using blocking sockets or java NIO (asynchronous select()-based sockets). On top of those sit various parsers that parse the network traffic into messages (ie into the Bitcoin messages). On top of those are `Peer` objects, which handle message handling (exchanging initial version handshake, downloading blocks, etc) for each individual remote peer and provide a simple event listener interface. Finally a `PeerGroup` can be layered on top to keep track of peers, ensuring there are always enough connections to the network and keeping track of network sync progress.

## The simple case {#The_simple_case}
In the case that you simply want a connection to the P2P network (ie in the vast majority of cases), all you need to do is instantiate a PeerGroup and connect a few objects:

```
PeerGroup peerGroup = new PeerGroup(networkParameters, blockChain);
peerGroup.setUserAgent("Sample App", 1.0);
peerGroup.addWallet(wallet);
peerGroup.addPeerDiscovery(new DnsDiscovery(networkParameters));
peerGroup.startAndWait();
peerGroup.downloadBlockChain();
```

After this code completes you’ve connected to some peers and fully downloaded the blockchain up to the latest block, filling in missing wallet transactions as it goes. peerGroup.startAndWait() and peerGroup.downloadBlockChain() can be replaced with asynchronous versions peerGroup.start() followed by peerGroup.startBlockchainDownload(listener) when the future returned by start() completes.

## Proxying {#Proxying}
If you wish to connect to the network using a SOCKS proxy, you must use blocking sockets instead of nio ones. This makes network slightly less efficient, but it should not be noticeable for anything short of very heavy workloads. Then you simply set the Java system properties for a SOCKS proxy (as below) and connections will automatically flow over the proxy.

```
System.setProperty("socksProxyHost", "127.0.0.1");
System.setProperty("socksProxyPort", "9050");
peerGroup = new PeerGroup(params, chain, new BlockingClientManager());
```

## Using the lower level API’s {#Using_the_lower_level_API_s}
You can build a Peer at a lower level, controlling the socket to be used, using code like this:

```
Peer peer = new Peer(params, bitcoin.chain(), new PeerAddress(InetAddress.getLocalHost(), 8333), "test app", "0.1") {
    @Override
    public void connectionOpened() {
        super.connectionOpened();
        System.out.println("TCP connect done");
    }
};
peer.addEventListener(new AbstractPeerEventListener() {
    @Override
    public void onPeerConnected(Peer peer, int peerCount) {
        System.out.println("Version handshake done");
    }
});
new BlockingClient(address, peer, 10000, SocketFactory.getDefault(), null);
```

Of course you would provide your own `SocketFactory` instead of using the default.

If you want access to a raw stream of messages with no higher level logic, subclass `PeerSocketHandler` and override the `processMessage` abstract method. This class implements `StreamParser` which breaks raw byte streams into the right subclass of `Message` for you, and then lets you handle those messages as you see fit. Create instances of your new object and pass them to an implementation of `ClientConnectionManager`, typically either `BlockingClientManager` or `NioClientManager` to use epoll/select based async IO.



