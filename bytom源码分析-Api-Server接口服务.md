---
title: bytom源码分析-Api-Server接口服务
date: 2018-08-16 19:31:27
categories:
tags:
---

# bytom源码分析-Api-Server接口服务

## 简介

https://github.com/Bytom/bytom

本章介绍bytom代码Api-Server接口服务

> 作者使用MacOS操作系统，其他平台也大同小异

> Golang Version: 1.8



## Api-Server接口服务
Api Server是比原链中非常重要的一个功能，在比原链的架构中专门服务于bytomcli和dashboard，他的功能是接收并处理用户和矿池相关的请求。默认启动9888端口。总之主要功能如下：

* 接收并处理用户或矿池发送的请求
* 管理交易：打包、签名、提交等操作
* 管理本地比原钱包
* 管理本地p2p节点信息
* 管理本地矿工挖矿操作等



在Api Server服务过程中，在监听地址listener上接收bytomcli或dashboard的请求访问。对每一个请求，Api Server均会创建一个新的goroutine来处理请求。首先Api Server读取请求内容，解析请求，接着匹配相应的路由项，随后调用路由项的Handler回调函数来处理。最后Handler处理完请求之后给bytomcli响应该请求。

## Api-Server源码分析


在bytomd启动过程中，bytomd使用golang标准库http.NewServeMux()创建一个router路由器，提供请求的路由分发功能。创建Api Server主要有三部分组成：

* 初始化http.NewServeMux()得到mux
* 为mux.Handle添加多个有效的router路由项。每一个路由项由HTTP请求方法（GET、POST、PUT、DELET）、URL和Handler回调函数组成
* 将监听地址作为参数，最终执行Serve(listener)开始服务于外部请求

### 创建Api对象
** node/node.go **

```
func (n *Node) initAndstartApiServer() {
	n.api = api.NewAPI(n.syncManager, n.wallet, n.txfeed, n.cpuMiner, n.miningPool, n.chain, n.config, n.accessTokens)

	listenAddr := env.String("LISTEN", n.config.ApiAddress)
	env.Parse()
	n.api.StartServer(*listenAddr)
}
```

** api/api.go **

```
func NewAPI(sync *netsync.SyncManager, wallet *wallet.Wallet, txfeeds *txfeed.Tracker, cpuMiner *cpuminer.CPUMiner, miningPool *miningpool.MiningPool, chain *protocol.Chain, config *cfg.Config, token *accesstoken.CredentialStore) *API {
	api := &API{
		sync:          sync,
		wallet:        wallet,
		chain:         chain,
		accessTokens:  token,
		txFeedTracker: txfeeds,
		cpuMiner:      cpuMiner,
		miningPool:    miningPool,
	}
	api.buildHandler()
	api.initServer(config)

	return api
}
```
首先，实例化api对象。Api-server管理的事情很多，所以参数也相对较多。
listenAddr本地端口，如果系统没有设置LISTEN变量则使用config.ApiAddress配置地址，默认为9888

NewAPI函数我们看到有三个操作：

1. 实例化api对象
2. api.buildHandler添加router路由项
3. api.initServer实例化http.Server，配置auth验证等


### router路由项
```
func (a *API) buildHandler() {
	walletEnable := false
	m := http.NewServeMux()
	if a.wallet != nil {
		walletEnable = true

		m.Handle("/create-account", jsonHandler(a.createAccount))
		m.Handle("/list-accounts", jsonHandler(a.listAccounts))
		m.Handle("/delete-account", jsonHandler(a.deleteAccount))
	// ...
	}
}
```
router路由项过多。这里只介绍关于账号相关的handler。其他的handler大同小异。
```
m.Handle("/create-account", jsonHandler(a.createAccount))
```
我们可以看到一条router项由url和对应的handle回调函数组成。当我们请求的url匹配到/create-account时，Api-Server会执行a.createAccount函数，并将用户的传参也带过去。


### 启动Api-Server服务
** api/api.go **

```
func (a *API) StartServer(address string) {
	log.WithField("api address:", address).Info("Rpc listen")
	listener, err := net.Listen("tcp", address)
	if err != nil {
		cmn.Exit(cmn.Fmt("Failed to register tcp port: %v", err))
	}

	go func() {
		if err := a.server.Serve(listener); err != nil {
			log.WithField("error", errors.Wrap(err, "Serve")).Error("Rpc server")
		}
	}()
}
```

通过golang标准库net.listen方法，监听本地的地址端口。由于http服务是一个持久运行的服务，我们启动一个go程专门运行http服务。当运行a.server.Serve没有任何报错时，我们可以看到服务器上启动的9888端口。此时Api-Server已经处于等待接收用户的请求。

