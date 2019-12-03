## Bitcoinj

主页：https://bitcoinj.github.io/#introduction 



1.1.1 网络参数NetworkParameters

此类用于选择比特币所属网络。

```java
NetworkParameters params;  // 网络参数声明
params = TestNet3Params.get(); // 公共测试网络
params = RegTestParams.get(); // 私有测试网络
params = MainNetParams.get(); // 生产网络
```

1.1.2 Andiod数据存储

简单数据存储： [SharePreference](http://blog.csdn.net/crazyman2010/article/details/8454471)

复杂数据存储：sqlite



## 2 新用户注册

### 2.1功能

产生区块链确认个人账户的私钥、公钥、地址。

### 2.2技术点

2.2.1 身份证与地址对应

地址可以与身份信息多对一对应， 在中央数据库落地。

 

2.2.2 私钥生成算法

1. 生成256 bit随机数, 私钥负载。
2. 对私钥添加版本号，压缩标识，校验码，最后进行base58编码生成私钥。

bitcoinj实现方案：

```
ECKey ceKey = new ECKey();
NetworkParameters params = TestNet3Params.get();
ceKey.getPrivKey() // 私钥， BigInteger
ceKey.getPrivateKeyAsHex() // 私钥， Hex
ceKey.getPrivateKeyAsWiF(params) // 私钥， WIF(Wallet Import Format)
ceKey.getPrivKeyBytes() // 私钥 byte[]
```

2.2.3 公钥生成算法

使用私钥， 通过椭圆曲线算法生成公钥。

bitcoinj实现方案：

```
ECKey ceKey = new ECKey();
NetworkParameters params = TestNet3Params.get();
ceKey.getPublicKeyAsHex(); // 公钥Hex
ceKey.getPubKey() // 公钥原始字节数组byte[]
```

2.2.4 地址生成算法

*Address*  =  *RIPEMD*160(*SHA*256(*公钥*))

bitcoinj实现方案:

```
ECKey ceKey = new ECKey();
NetworkParameters params = TestNet3Params.get();
ceKey.toAddress(params).toBase58() // base58编码后的地址
```

### 2.3 实现方案

使用BitCoinJ的ECKey, Address两个模块。

生成的账户信息（地址，私钥，公钥）使用SharedPreferences存储。

 

## 3. 余额展示

### 3.1 功能

（1）展示用户的当前余额

（2）列出和用户相关的交易明细

### 3.2 技术点

3.2.1 UTXO的存储

从网络Node请求存储载有UTXO的Tx, 并存储于本地的SQlite中。

 

数据表结构[表名tx_utxo]：

Data  二进制数据, output所在的transaction，

index  output所在transaction的下表位置。

credit 可信度。 所在区块与尾部的距离。 如果不存在于区块， 值为0。

available 可用金额

hash  交易的hash码， 标识交易

 

建表语句：

```
CREATE TABLE IF NOT EXISTS tx_utxo(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    data BLOB,
    credit INTEGER,
    available INTEGER,
    hash VARCHAR
)
```

3.2.2 余额计算

分两步实现

（1）向Node请求与UTXOs相关的Tx列表， 并存储于数据库。

（2）获取Tx列表数据， 反序列化， 取得相关UTXO, 将金额相加。

## 4. 转账

### 4.1 功能

金额从当前账户转到指定账户

### 4.2 技术点

4.2.1 生成交易报文。

- 根据要花费的钱， 计算出使用哪些UTXO （文档Mastering Bitcoin的116页， 贪心算法实现）。
- 构造交易报文，大致流程如图。

bitcoinj实现方案：

- 构造input为空的交易。

```
String ownerPrivInBigIntegerStr = "114975678953706493006078253903895427663346921000308885839647313125562885805365";
BigInteger ownerPrivInBigInteger = new BigInteger(ownerPrivInBigIntegerStr);
NetworkParameters params = TestNet3Params.get();
// 构造初始拥有者key
ECKey ownerKey = ECKey.fromPrivate(ownerPrivInBigInteger);
Address ownerAddress = ownerKey.toAddress(params);
Transaction tx = new Transaction(params);
Coin initCoin = Coin.valueOf(1000L);
tx.addOutput(initCoin, ownerAddress);
```

- 交易序列化， 转为bytes[]

```
tx.bitcoinSerialize()
```

- 交易持久化， 二进制格式写入文件

```
FileOutputStream fo = new FileOutputStream("d:/tx.data");
tx.bitcoinSerialize(fo);
fo.close();
```

- 从二进制文件中恢复，byte[]转交易对象

```
File rf = new File("d:/tx.data");
FileInputStream fis= new FileInputStream(rf);
byte[] buf = new byte[1000000];
fis.read(buf);
Transaction txx = new Transaction(params, buf);
```

- 构造花费utxo的交易。

```
Coin valueNeeded = Coin.parseCoin("0.09"); // 转账金额
// 转账地址、转账金额
SendRequest request = SendRequest.to(recieverAddress, valueNeeded); request.changeAddress = ownerAddress;
List<ECKey> ownerKeys = new ArrayList<ECKey>();
// 设置当前用户的key
ownerKeys.add(ownerKey);
Wallet wallet = Wallet.fromKeys(params, ownerKeys);
WalletTransaction wtx = new WalletTransaction(WalletTransaction.Pool.UNSPENT, unspend);
// 将当前用户拥有的utxo所在的交易加入列表
wallet.addWalletTransaction(wtx);
// 生成交易
Transaction tx = wallet.sendCoinsOffline(request);
```

4.2.2 交易合法性验证

```
// 将交易tx2的某个Trasanction的input与交易tx1相连
tx2.getInput(0).connect(tx1,TransactionInput.ConnectMode.ABORT_ON_CONFLICT);
tx2.verify();// 交易的基本数据校验
tx2.getInput(0).verify();// 交易的验签合法
```

4.2.3 确认交易成功。

交易成功分为以下几个级别。

- 转账已受理。Node受理成功。与Node通信获得。
- 转账成功。查询到转账记录所在的区块后已经有六个后继区块。与Node通信获得。

 

4.2.4 生成/扫描二维码

转账时需要以二维码的形式呈现地址。使用开源ZXing库实现。

完全参考博文：http://blog.csdn.net/bear_huangzhen/article/details/46053327

 

4.2.5 数据存储

使用Sqlite存储相关数据

[**表名 tx_local**]：用户本地发起的交易

 

_id  自增ID

data  二进制数据，tx序列化数据

credit 可信度。 所在区块与尾部的距离。 如果不存在于区块， 值为0。

amount 交易金额

fee   手续费

hash  交易的hash码， 标识交易

 

建表语句：

```
CREATE TABLE IF NOT EXISTS tx_local
(
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    data BLOB,
    credit INTEGER,
    amount INTEGER,
    fee INTEGER,
    hash VARCHAR
)
```

[**表名 tx_net**]：从网络获取的和用户相关的交易

 

data  二进制数据

credit 可信度。 所在区块与尾部的距离。 如果不存在于区块， 值为0。