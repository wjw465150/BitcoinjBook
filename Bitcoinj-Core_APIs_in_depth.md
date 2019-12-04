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
3. Find which blocks the transaction appears in using `getAppearsInHashes()`. Because the transaction only stores the hashes of the block headers, you may need to use a populated `BlockStore` to get the block data itself.
4. Learn about the state of the transaction in the chain using `getConfidence`. This returns a `TransactionConfidence` object representing various bits of data about when and where the transaction was included into the block chain.

## Confidence levels {#Confidence_levels}
A transaction has an associated *confidence*. This is data you can use to calculate the probability of the transaction being reversed (double spent). This is always a risk in Bitcoin although at high network speeds, the probability becomes extremely low, certainly relative to traditional payment systems.

Confidence is modelled with the `TransactionConfidence` object. You can get one by calling `Transaction.getConfidence()`. Confidence data does not exist in Bitcoin serialization, but will survive Java and protobuf serialization.

A confidence object has one of several states:

- If **BUILDING**, then the transaction is included in the best chain and your confidence in it is increasing.
- If **PENDING**, then the transaction is unconfirmed and should be included shortly as long as it is being broadcast from time to time and is considered valid by the network. A pending transaction will be announced if the containing wallet has been attached to a live `PeerGroup` using `PeerGroup.addWallet()`. You can estimate how likely the transaction is to be included by measuring how many nodes announce it after sending it to one peer, using `TransactionConfidence.numBroadcastPeers()`. Or if you saw it from a trusted peer, you can assume it’s valid and will get mined sooner or later as well.
- If **DEAD**, then it means the transaction won’t confirm unless there is another re-org, because some other transaction is spending one of its inputs. Such transactions should be alerted to the user so they can take action, eg, suspending shipment of goods if they are a merchant.
- **UNKNOWN** is used if we have no information about the confidence of this transaction, because for example it has been deserialized from a Bitcoin structure but not broadcast or seen in the chain yet. UNKNOWN is the default state.

The confidence type, available via `TransactionConfidence.getConfidenceType()`, is a general statement of the transactions state. You can get a more precise view using getters on the object. For example, in the `BUILDING` state, `getDepthInBlocks()` should tell you how deeply buried the transaction is, in terms of blocks. The deeper it is buried in the chain, the less chance you have of the transaction being reversed.

Depth in blocks is easy to understand and roughly corresponds to how long the transaction has been confirmed for (1 block == 10 minutes on average). However, this is not a stable measure of how much effort it takes to reverse a transaction because the amount of *work done* on a block varies over time, depending on how much mining is happening, which itself depends on the exchange rate (vs the dollar/euro/etc).

### Understanding difficulty and confidence {#Understanding_difficulty_and_confidence}
The most common reason you are interested in confidence is you wish to measure the risk of losing money that was sent to you, for example, to delay dispatching of goods or provision of services. The Bitcoin community uses a rule of thumb of zero confirmations for things that aren’t of much value like MP3s or ebooks, one or two blocks for things (10-20 minutes) for things that stand a risk of a double spend attack, or 6 blocks (an hour) for where rock solid certainty is required, like with currency exchanges.

In practice, reports of merchants suffering double-spend fraud are rare so this discussion is somewhat theoretical.

You’ll notice that the rules of thumb quoted above are expressed as blocks, not work done. So if the exchange rate and thus mining falls, 2 blocks provides less assurance than before, meaning you may wish to wait longer. Conversely if the exchange rate rises, mining activity will increase, meaning you can wait less time before a transaction is valid, resulting in happier customers. This is why we also provide a way to measure confidence as work done.

## How transactions can be used {#How_transactions_can_be_used}
The most common and obvious way to use transactions is to:

1. Download them into your wallet from the block chain or network broadcasts
2. Create spends and then broadcast them

However there are many other possibilities.

### Direct transfer {#Direct_transfer}
It’s possible to send someone money by directly giving them a transaction, which they can then broadcast at their leisure, or further modify. These use cases aren’t well supported today, but in future may become a common way to use Bitcoin.

### Participation in contracts {#Participation_in_contracts}
[Contracts](https://en.bitcoin.it/wiki/Contracts) allow for a variety of low trust trades to take place, mediated by the Bitcoin network. By carefully constructing transactions with particular scripts, signatures and structures you can have low-trust dispute mediation, coins locked to arbitrary conditions (eg, futures contracts), assurance contracts, [smart property](https://en.bitcoin.it/wiki/Smart_Property) and many other possibilities.

You can learn more about this topic in the article [WorkingWithContracts](https://bitcoinj.github.io/working-with-contracts).



# Working with the wallet

*Learn how to use the wallet class and craft custom transactions with it.*

## Introduction {#Introduction2}
The Wallet class is one of the most important classes in bitcoinj. It stores keys and the transactions that assign value to/from those keys. It lets you create new transactions which spend the previously stored transactions outputs, and it notifies you when the contents of the wallet have changed.

You’ll need to learn how to use the Wallet to build many kinds of apps.

This article assumes you’ve read Satoshi’s white paper and the [WorkingWithTransactions](https://bitcoinj.github.io/working-with-transactions) article.

## Setup {#Setup}
For optimal operation the wallet needs to be connected to a `BlockChain` and a `Peer` or `PeerGroup`. The block chain can be passed a Wallet in its constructor. It will send the wallet blocks as they are received so the wallet can find and extract *relevant transactions*, that is, transactions which send or receive coins to keys stored within it. The Peer/Group will send the wallet transactions which are broadcast across the network before they appear in a block.

A Wallet starts out its life with no transactions in it, and thus a balance of zero, regardless of what the block chain contains. To use it you need to download the block chain, which will load the wallet up with transactions that can be analyzed and spent.

```
Wallet wallet = new Wallet(params);
BlockChain chain = new BlockChain(params, wallet, ...);
PeerGroup peerGroup = new PeerGroup(params, chain);
peerGroup.addWallet(wallet);
peerGroup.startAndWait();
```

## Getting addresses {#Getting_addresses}
Of course, the snippet of code is fairly useless because there’s no way to get money into it. You can obtain keys and addresses from the wallet with the following API calls:

```
Address a = wallet.currentReceiveAddress();
ECKey b = wallet.currentReceiveKey();
Address c = wallet.freshReceiveAddress();

assert b.toAddress(wallet.getParams()).equals(a);
assert !c.equals(a);
```

These can then be handed out to receive payments on. The Wallet has a notion of a “current” address. This is intended for GUI wallets that wish to display an address at all times. Once the current address is seen being used, it changes to a new one. The freshReceiveKey/Address methods on the other hand always return a newly derived address.

## Seeds and mnemonic codes {#Seeds_and_mnemonic_codes}
The keys and addresses returned by these methods are derived deterministically from a seed, using the algorithms laid out in BIP 32 and BIP 39. The life of a key looks like this:

1. A new Wallet object selects 128 bits of random entropy using `SecureRandom`.
2. This randomness is transformed into a “mnemonic code”; a set of 12 words using a dictionary pre-defined by the BIP 39 standard.
3. The string form of the 12 words is used as input to a key derivation algorithm (PBKDF2) and iterated numerous times to obtain the “seed”. Note that the seed is *not* simply the original random entropy represented using words as you might expect, rather, the seed is a derivative of the UTF-8 byte sequence of the words themselves.
4. The newly calculated seed is then split into a master private key and a “chain code”. Together, these allow for iteration of a key tree using the algorithms specified in BIP 32. This algorithm exploits properties of elliptic curve mathematics to allow the public keys in a sequence to be iterated without having access to the equivalent private keys, which is very useful for vending addresses without needing the wallet to be decrypted if a password has been supplied. bitcoinj uses the default recommended tree structure from BIP 32.
5. The wallet pre-calculates a set of *lookahead keys*. These are keys that were not issued by the Wallet yet via the current/freshReceiveKey APIs, but will be in future. Precalculation achieves several goals. One, it makes these APIs fast, which is useful for GUI apps that don’t want to wait for potentially slow EC math to be done. Two, it allows the wallet to notice transactions being made to/from keys that were not issued yet - this can happen if the wallet seed has been cloned to multiple devices, and if the block chain is being replayed.

The seed and keys that were (pre) calculated are saved to disk in order to avoid slow rederivation loops when the wallet is loaded.

The “wallet words” are intended to be easier to work with and write down than a raw private key: users can easily work with them using a pen and paper with less chance of accidentally writing things down wrong. So you’re encouraged to expose the words to users as a backup mechanism (make sure they write down the date too, to speed up restore).

You can work with seeds like this:

```
DeterministicSeed seed = wallet.getKeyChainSeed();
println("Seed words are: " + Joiner.on(" ").join(seed.getMnemonicCode()));
println("Seed birthday is: " + seed.getCreationTimeSeconds());

String seedCode = "yard impulse luxury drive today throw farm pepper survey wreck glass federal";
long creationtime = 1409478661L;
DeterministicSeed seed = new DeterministicSeed(seedCode, null, "", creationtime);
Wallet restoredWallet = Wallet.fromSeed(params, seed);
// now sync the restored wallet as described below.
```

The lookahead zone plays an important role when keeping wallets synchronised together. The default zone is 100 keys in size. This means that if wallet A is cloned to wallet B, and wallet A issues 50 keys of which only the last one is actually used to receive payment, wallet B will still notice that payment and move its lookahead zone such that B is tracking 150 keys in total. If wallet A handed out 120 keys and only the 110th received payment, wallet B would not notice anything had happened. For this reason when trying to keep wallets in sync it’s important that you have some idea of how many outstanding addresses there may be awaiting payment at any given time. The default of 100 is selected to be appropriate for consumer wallets, but in a merchant scenario you may need a larger zone.

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



