# SilkPay - SLP  上币技术指南


SLP是SilkPay社区价值共识，是基于Silubium公链开发的[通证](https://github.com/SilkPayVIP/SLP/blob/master/slp.sol)，SilkPay是全球首例技术突破去中心化加密货币支付与法币支付壁垒的开源支付平台。

[Silubium](https://github.com/SilubiumProject/slucore)是基于比特币UTXO和以太坊EVM开发的集多种功能于一体的公链，区块最快确认速度为4秒，完全满足高性能应用需求。

## 交易所上架SLP准备工作

* 在自己的服务器上安装[Silubium节点](http://update.silubium.org/silubium-bitcore-3.14.16.26.tar.gz)；或直接用[官方超级节点](https://github.com/SilubiumProject/silubium-java-lib#%E4%BA%A4%E6%98%93%E6%89%80%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)，推荐使用。

* 交易所按[BIP44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)规则为用户[生成SLP地址](#离线生成SLP地址)

* 用户充币，[监听扫描区块信息](#扫描实例)，判断是否为自己交易所的地址，若有收款信息，则为用户进行充币操作（确认6次后可视为到帐）

* 用户提币，建议将用户地址上的SLP，定期归集到交易所的提币地址上，用户提币时，可统一从交易所提币地址[转帐](#SLP转帐)。

* 转帐SLU和SLP均需消耗SLU。SLU转帐手续费标准为0.0001SLU/Kb，最大手续费不超过0.5SLU。SLP转帐会预扣燃料0.0801SLU，区块确认一次后，未用完燃料会返还给发送地址，实际消耗燃料在0.0037SLU左右。

* 需要使用的技术资料参见[Silubium官方](https://github.com/SilubiumProject/silubium-java-lib#%E4%BA%A4%E6%98%93%E6%89%80%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)。

### 离线生成SLP地址
```html
//Silubium原生资产为SLU，SLP与SLU使用同一地址。SLP转帐需要消耗SLU燃料。
@Test
public void testbip44EthereumEcKey() throws MnemonicException.MnemonicLengthException, ValidationException {                    
    List<String> mnemonicWordsInAList = new ArrayList<>();
    mnemonicWordsInAList.add("cupboard");
    mnemonicWordsInAList.add("shed");
    mnemonicWordsInAList.add("accident");
    mnemonicWordsInAList.add("simple");
    mnemonicWordsInAList.add("marble");
    mnemonicWordsInAList.add("drive");
    mnemonicWordsInAList.add("put");
    mnemonicWordsInAList.add("crew");
    mnemonicWordsInAList.add("marine");
    mnemonicWordsInAList.add("mistake");
    mnemonicWordsInAList.add("shop");
    mnemonicWordsInAList.add("chimney");
    mnemonicWordsInAList.add("plate");
    mnemonicWordsInAList.add("throw");
    mnemonicWordsInAList.add("cable");
    // 123为默认密码，可自行替换
    byte[] seed = MnemonicCode.toSeed(mnemonicWordsInAList, "123");

    ExtendedKey extendedKey = ExtendedKey.create(seed);
    CoinPairDerive coinKeyPair = new CoinPairDerive(extendedKey);
    // 使用bip44路径生成，建议BIP44.m().purpose44().coinType(CoinTypes.SLIUBIUM).account(1).external().address(此处为地址索引，为遵循bip44规则，其值范围为0到19最佳);
    // 当address索引累计到19后，可以增加account索引以生成更多地址
    //使用bip44路径生成，建议BIP44.m().purpose44().coinType(CoinTypes.SLIUBIUM).account(增加此处索引).external().address(19);
    
    AddressIndex address = BIP44.m().purpose44().coinType(CoinTypes.SLIUBIUM).account(1).external().address(1);
    // 第一个参数为地址索引路径，然后是链参数，是否为测试链
    ECKeyPair master = coinKeyPair.derive(address, new SLUNetworkParameters(), !CurrentNetParams.getUseMainNet());
    System.out.println(address.toString());
    try {
        // 合法的slu地址正式链前三个字符为SLU，测试链前三个字符为SLS，截掉"SL"后地址规则满足base58编码
        String sluAddress = "SL"+master.getAddress();
        System.out.println("privateKey" + "..." + master.getPrivateKey());
        System.out.println("publicKey" + "..." + master.getPublicKey());
    } catch (Exception e) {
        e.printStackTrace();
    }
  }
```

### 扫描实例
```html
// 起始页码为0
int i = 0;
// 获取指定高度第一页的交易，pageSize为分页大小参数，建议设置为500，且不要随意变动该值，防止分页出现粘连数据
transaction = Generator.executeSync(SilubiumServiceSingalUtil.getSilubiumService().listTransaction(nextBlockNumber.toString(), i, pageSize));
// 对获取的交易进行解析
replayBlock(transaction.getTxs());
// 对获取交易设置休眠时间，因Silubium最快打包速度为4s，所以正常是1s获取一次即可，如果出现网络异常或者已经到最新高度，也建议休眠4s后再尝试获取交易
long waitTime = 1000L;
while (true) {
    try {
        if (transaction == null) {
            logger.info("已经是最新交易信息");
            transaction = Generator.executeSync(SilubiumServiceSingalUtil.getSilubiumService().listTransaction(nextBlockNumber.toString(), i, pageSize));
        } else {
            // 当前高度的交易是否已经扫描完，如果扫描完，则对高度进行累加
            if (transaction.getPagesTotal() == i + 1) {
                //查询下一个块
                setNextBlockNumber(nextBlockNumber + 1);
                logger.info("扫描到：{} 第1页", nextBlockNumber);
                i = 0;
                transaction = Generator.executeSync(SilubiumServiceSingalUtil.getSilubiumService().listTransaction(nextBlockNumber.toString(), i, pageSize));
            } else {
                // 否则就对当前高度的页码进行累加
                logger.info("扫描到：{} 第{}页", nextBlockNumber, i + 2);
                i++;
                transaction = Generator.executeSync(SilubiumServiceSingalUtil.getSilubiumService().listTransaction(nextBlockNumber.toString(), i, pageSize));
            }
        }
    } catch (Exception e) {
        // 如果网络异常，或者已经高于最新高度，则将高度重置，且休眠4s
        logger.info("异常 {} 当前扫描高度 {}", e.getMessage(), nextBlockNumber);
        if (StringUtils.equalsIgnoreCase(e.getMessage(), errorMessage)) {
            //重置块高度
            setNextBlockNumber(nextBlockNumber - 1);
        }
        transaction = null;
        waitTime = 4000L;
    } finally {
        Thread.sleep(waitTime);
    }
}
```
```html
// 前一步已经取得块的交易信息
// 现在对块的交易信息进行逐个解析
// 通过对com.spark.bc.wallet.api.entity.bcc.TxsBean属性issrc20Transfer判断是TOKEN交易还是SLU交易
// txBean.isIssrc20Transfer() 如果是则交易包含TOKEN交易和SLU交易，如果不是则只包含SLU交易
// 分析TOEKN交易，类似ETH ERC20 ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef 该字符串表示TOKEN转账
// 获取交易中涉及的TOKEN转账信息
List<ReceiptBean> receiptBeans = txBean.getReceipt();
// 遍历所有交易，注意判断receiptBeans 为 null的情况，否则该循环会报错
for (ReceiptBean tokenTransaction : receiptBeans) {
  // 判断该笔合约交易是否执行成功
  if ("None".equalsIgnoreCase(tokenTransaction.getExcepted())) {
    // 执行成功，则获取对应合约交易记录
    List<LogBean> logBeans = tokenTransaction.getLog();
    // 遍历所有合约交易记录，注意判断logBeans 为 null的情况，否则该循环会报错
    for (LogBean logBean : logBeans) {
        // 是否为指定合约地址的转账交易，不是则继续遍历下一个
        if (!"ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef".equalsIgnoreCase(logBean.getTopics().get(0)) 
        || !StringUtils.equalsIgnoreCase("TOKEN 合约地址", logBean.getAddress())) {
          continue;
        }
        // 是的话，解析交易信息
        // 转账金额 coinToken.getCoinDecimals()为对应币种精度一般为8
        com.spark.bc.wallet.api.util.AmountFormate.amount(logBean.getData(), String.valueOf(coinToken.getCoinDecimals())).doubleValue()
        // 发送地址
        String sendAddress = logBean.getTopics().get(1);
        // 接收地址
        String toAddress = logBean.getTopics().get(2);
        //将交易信息进行持久化
    }        
  }                        
}
// 分析SLU交易java.util.List<BccTransferResult> sluTransferResults = TransactionUtil.analysis(txsBean, null);
// 使用该方式可获取交易中所有地址的交易金额，如果金额小于0 则为转出，否则为转入
```

### SLP转帐
```html
/**
  * 测试创建SLU交易或者TOKEN交易 （可以同时发送SLU和TOKEN）
  * 每笔交易均会消耗slu，token不会被消耗
  */
@Test
public void testCreateNewTx() throws UnsupportedEncodingException {
  try {
      SendRawTransactionRequest sendRawTransactionRequest = new SendRawTransactionRequest();
      sendRawTransactionRequest.setAllowAbsurdFees(true);

      //接收地址
      List<SluTransferResult> addresses = new ArrayList<>();

      // SLU接收地址
      List<SendGasResult> sendGasResults = new ArrayList<>();
      sendGasResults.add(new SendGasResult(Address.fromBase58(CurrentNetParams.getNetParams(), "SLSSFpE5Gbbg84v2FqFkZAasrmqfNNNZqvwr"),new BigDecimal("10000")));
      

      // TOKEN接收地址
      BigDecimal bigDecimal = new BigDecimal("10000");
      addresses.add(new SluTransferResult("SLSSFpE5Gbbg84v2FqFkZAasrmqfNNNZqvwr",bigDecimal));
      
      Map<String, String> map = new HashMap(1);
      // 合约地址 发送TOKEN交易时，合约地址必须填写
      //  所有合约http://silkchain.silubium.org/token/tokenlist.html
      //  SLP合约地址为：56465a84d048dfcaa06ef5bd818ecd257b139cd0
      String contractAddress = "56465a84d048dfcaa06ef5bd818ecd257b139cd0";
      // 该map只能放一个  发送地址和私钥
      map.put("sendAddress","privateKey");

      try {
          // 构建交易
          TransactionCheck transactionCheck = TransactionUtil.createNewTx(map //发送地址，有且仅有一个
          // 合约地址，当合约地址为空时，则为SLU交易，当合约地址不为空时，则为合约交易或者（合约和SLU）交易
          , contractAddress
          , addresses
          // TOKEN 接收地址，可多个，建议每次低于10，防止交易体积过大，具体可以实测，理论上可以放1000个
          , new BigDecimal("0.0001").toPlainString()
          // 默认基础手SLU续费，其余SLU手续费会根据交易体积自动计算，应确保发送地址SLU至少大于0.0801 SLU
          , sendGasResults
          // SLU 发送，可多个，建议每次低于10个，防止交易体积过大，具体可以实测，理论上可以放1200个
          , BigDecimal.ZERO
          // 该金额用于过滤 utxo 中过小的金额，如果为0则不会过滤任何 utxo 金额
          ,null          
          // 该处可用于高并发处理，将前面一笔交易使用的utxo暂时保存，构建新的交易时，将已经消耗过的utxo放于该变量，
          // 然后程序会排除这些utxo（用于钱包广播节点和网络造成的问题，解决短时间同一笔utxo被多笔交易使用的问题）
          );
          sendRawTransactionRequest.setRawtx(transactionCheck.getTransactionBytes());
      } catch (Exception e) {
          e.printStackTrace();
      }
      SilubiumService rpcService = Generator.createService(SilubiumService.class, CurrentNetParams.getBaseUrl());
      // 广播交易 返回交易hash
      SendResult sendResult = Generator.executeSync(rpcService.sendRawTransaction(sendRawTransactionRequest));
      System.out.println(sendResult.getTxid());
  } catch (ApiException e) {
      System.out.println(e.getError());
  }
}
```
