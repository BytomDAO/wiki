# 			BTM的RPC远程调试

​	搭建完节点，顺利启动了节点并同步了区块数据。使用 ./bytomcli 命令可以成功获取数据，使用  curl -X POST get-block-count调用rpc，结果提示说get-block-count未找到命令，于是顺手加上端口号， curl -X POST localhost:9888/get-block-count，成功的获取到了json格式的数据。

​	习惯了使用Postman调试接口，于是尝试POST一个rpc请求，开始踩坑之旅哈哈。

远程调用RPC需要带上token方可请求，否则就出现BTM860的错误提示，如何获取token并设置呢？

在节点服务器本地使用cli或者curl，生成access-token。

bytomcli 方式：

```
./bytomcli create-access-token test
```

或者

```
curl -X POST create-access-token -d '{"id":"test"}'
```

返回json数据

```
{
  "created_at": "2018-05-18T16:00:25.284677605+08:00",
  "id": "test",
  "token": "test:fe50927ddaa5bcca77021e9f50fa5ef236a6140c012d1fe2eb9241f61a9228e4             "
}
```

其中 **test是远程访问的username，冒号后面的字符串是password**

Postman如何带上Auth呢？

**设置Authorization为Basic Auth，Username填刚才生成的sccess-test的id，Password填token冒号后面的字符串即可**

​	使用普通的表单提交，比如获取新地址的方法create-account-receiver ，在表单的key填account_alias ，value填账户的别名，提交请求发现返回错误信息，code为BTM003。需要将Body切到raw格式，输入json格式的参数即可，如：{"account_alias":"test"}

Java中直接通过POST请求就可以远程调用RPC，但需要构造Authorization并加入到Header中，关键代码如下

```
String auth = Username + ":" + Password;
byte[] encodedAuth = Base64.encodeBase64(auth.getBytes(Charset.forName("US-ASCII")));
String authHeader = "Basic " + new String(encodedAuth);
Map<String, String> header = new LinkedHashMap<String, String>();
header.put("Authorization", authHeader);
```

**【这里的Username即是 test，Password是fe50927xxxxxxxxxxxxx228e4，根据自己生成的token改】**

body使用JSONObject即可。



关于转账的流程，比较复杂。需要先打包交易，对交易进行签名，再进行广播。对应的rpc是：

build-transaction -> sign-transaction  -> submit-transaction 

build-transaction时带的参数根据api即可，主要是actions可能有点不清楚怎么用，比如通过地址转账时，可以设置actions如下：

```
“actions”:[

{

"account_id":"xxxx",   // 账户ID 

"amount":300000000,  //转账额度，需要包含手续费
"asset_id":"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",  //资产ID，BTM是全F，可以转合约上的其它资产
"type":"spend_account"  //表示花费的账号即转账方

}，

{
	"account_id":"xxxxx",  //接收方账户ID
	"amount":200000000, //转账额度
	"asset_id":"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
	"type":"control_address",  //类型表示是地址转账
	"address":"sm1q9s703yayrvn8g3zq80um65ul9ffg6pvx8cgpme"  //接收地址
}

]
```

转账主要是在手续费方面会踩坑，1BTM=10,000,000NEU，个人感觉NEU就是类似比特币中的一聪吧哈哈，就是小数点后八位。所以上面的amount是以NEU算的，其实就是转了2个BTM，拿1个BTM当作手续费，故转账账户需要出3个BTM。

打包完交易，就需要进行签名， 用创建账户时的密码进行签名。即是调用 sign-transaction 的rpc，参数是password和transaction，其中transaction是build成功后返回的data中的一个对象Object。签名后返回结果，可以根据 sign_complete 判断是否签名成功，如果为true即成功。

广播交易，submit-transaction 只有一个参数raw_transaction,打包完或者签名完都会有返回该参数，是一串很长的序列化后的字符串。如果广播成功，即返回tx_id，失败会返回失败信息。

这两天对btm的rpc调用研究收获了不少，还搭建了个solonet节点自己挖矿测试转账，感谢比原技术群的大牛提供不少的帮助。我会继续努力的踩坑~嘻嘻

​																		good-boy~zhangxuewen.

​																				2018-05-18