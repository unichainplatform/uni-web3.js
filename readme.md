安装及引用：

```
npm i unichain-web3 -S

import * as unichain from 'unichain-web3'; 
```

根据rpc功能划分了不同的域：
- unichain.account.*: 包括查看账户、资产等功能
- unichain.dpos.*: 包括查看竞选、出块节点信息以及dpos共识相关信息等功能
- unichain.uni.*: 包括查看区块、交易信息以及签名、发送交易等功能
- unichain.fee.*: 包括查看不同账户手续费的功能
- unichain.miner.*: 包括启动挖矿、停止挖矿、设置coinbase等功能
- unichain.p2p.*: 包括增加、删除节点等功能
- unichain.txpool.*: 包括获取txpool信息以及设置gas price等功能
- unichain.utils.*: 包括rlp编码、合约payload编码等功能
> rpc文档：https://github.com/unichainplatform/unichain/wiki/JSON-RPC

demo1：设置节点信息、查看账户信息
> 节点rpc默认是htp://127.0.0.1:8545，如不是，必须先设置好节点信息，才可进行后续操作
```
import * as unichain from 'unichain-web3';

const nodeInfo = 'htp://127.0.0.1:8545';  
unichain.utils.setProvider(nodeInfo);  //设置节点rpc信息

try {
  const accountInfo = await unichain.account.getAccountByName('unichain.admin');
  ...
} catch (error) {
  console(error);  
}

```
demo2: 发送多签名交易

```
import * as unichain from 'unichain-web3';

// 交易完整结构体：{chainId, gasAssetId, gasPrice, actions:[{actionType, accountName, nonce, gasLimit, toAccountName, assetId, amount, payload, remark}]}
// 其中chainId, gasAssetId, nonce, payload 以及 remark这几个参数如果不传，sdk会根据实际情况自动填充
txInfo = {...}
signInfo1 = await unichain.uni.signTx(txInfo, privateKey1);  //获取第一个签名
signInfo2 = await unichain.uni.signTx(txInfo, privateKey2);  //获取第二个签名
multiSignInfos = [{signInfo1, [0]}, {signInfo2, [1]}];
parentIndex = 0; // 签名级别，如果为0，表示是自己账号签名，如果为1，表示父账号签名，如果为2，表示父父账号签名
unichain.uni.sendSeniorSigTransaction(txInfo, multiSignInfos, 0).then(txHash => {...}).catch(error => {...});  // 发送多签名交易
```
demo3: 发送单签名交易

```
import * as unichain from 'unichain-web3';

txInfo = {...}
signInfo1 = await unichain.uni.signTx(txInfo, privateKey1, 0);
unichain.uni.sendSingleSigTransaction(txInfo, signInfo1).then(txHash => {...}).catch(error => {...});
```
demo4: 合约方法调用
- 合约代码：
```
contract hello {
    string greeting;
    
    function hello(string _greeting) public {
        greeting = _greeting;
    }

    function say() constant public returns (string) {
        return greeting;
    }
}
```
- sdk调用方式

```
import * as unichain from 'unichain-web3';

// 调用合约的hello方法
const payload = unichain.utils.getContractPayload('hello', ['string'], ['unichain blockchain']); //payload会填入txInfo中
const txInfo = {...}  // 构造合约交易对象
signInfo1 = await unichain.uni.signTx(txInfo, privateKey1);
unichain.uni.sendSingleSigTransaction(txInfo, signInfo1, 0).then(txHash => {...}).catch(error => {...});

// 调用合约的say方法，由于say是一个constant类型的方法，只需要从链上读取数据，因此不需要发送交易，只要调用rpc中的call方法即可获得结果
const payload = unichain.utils.getContractPayload('say', [], []);
const callInfo = {actionType:0, from:'youraccount', to: 'contractAccountName', assetId:0, gas:100000, gasPrice:10, value:100, data:payload, remark:''};
unichain.uni.call(callInfo, 'latest').then(result=>{...});
```
