---
title: bytom源码分析-protobuf生成核心代码
date: 2018-08-17 18:30:17
categories:
tags: [golang, bytom]
---


# bytom源码分析-protobuf生成比原核心代码

## 简介

https://github.com/Bytom/bytom

本章介绍bytom代码Api-Server接口服务

> 作者使用MacOS操作系统，其他平台也大同小异

> Golang Version: 1.8



## protobuf生成比原核心代码

### protobuf介绍
Protocol buffers是一个灵活的、高效的、自动化的用于对结构化数据进行序列化的协议。Protocol buffers序列化后的码流更小、速度更快、操作更简单。只需要将序列化的数据结构(.proto文件)，便可以生成的源代码。

### protobuf 3.0语法介绍
[protobuf 语法](https://developers.google.com/protocol-buffers/docs/proto3)


### protobuf 安装

#### 安装protobuf 3.4.0
[protobuf download](https://github.com/google/protobuf/releases)

``` bash
./configure
make
make install
protoc —version
```
#### 安装grpc-go
``` bash
export PATH=$PATH:$GOPATH/bin
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

### 查看比原bc.proto核心文件
** protocol/bc/bc.proto **

``` bash
syntax = "proto3";

package bc;

message Hash {
  fixed64 v0 = 1;
  fixed64 v1 = 2;
  fixed64 v2 = 3;
  fixed64 v3 = 4;
}

message Program {
  uint64 vm_version = 1;
  bytes  code       = 2;
}

// This message type duplicates Hash, above. One alternative is to
// embed a Hash inside an AssetID. But it's useful for AssetID to be
// plain old data (without pointers). Another alternative is use Hash
// in any protobuf types where an AssetID is called for, but it's
// preferable to have type safety.
message AssetID {
  fixed64 v0 = 1;
  fixed64 v1 = 2;
  fixed64 v2 = 3;
  fixed64 v3 = 4;
}

message AssetAmount {
  AssetID asset_id = 1;
  uint64  amount   = 2;
}

message AssetDefinition {
  Program issuance_program = 1;
  Hash    data             = 2;
}

message ValueSource {
  Hash        ref      = 1;
  AssetAmount value    = 2;
  uint64      position = 3;
}

message ValueDestination {
  Hash        ref      = 1;
  AssetAmount value    = 2;
  uint64      position = 3;
}

message BlockHeader {
  uint64            version                 = 1;
  uint64            height                  = 2;
  Hash              previous_block_id       = 3;
  uint64            timestamp               = 4;
  Hash              transactions_root       = 5;
  Hash              transaction_status_hash = 6;
  uint64            nonce                   = 7;
  uint64            bits                    = 8;
  TransactionStatus transaction_status      = 9;
}

message TxHeader {
  uint64        version         = 1;
  uint64        serialized_size = 2;
  uint64        time_range      = 3;
  repeated Hash result_ids      = 4;
}

message TxVerifyResult {
  bool status_fail = 1;
}

message TransactionStatus {
  uint64                  version        = 1;
  repeated TxVerifyResult verify_status  = 2;
}

message Mux {
  repeated ValueSource      sources              = 1; // issuances, spends, and muxes
  Program                   program              = 2;
  repeated ValueDestination witness_destinations = 3; // outputs, retirements, and muxes
  repeated bytes            witness_arguments    = 4;
}

message Coinbase {
  ValueDestination witness_destination = 1;
  bytes            arbitrary           = 2;
}

message Output {
  ValueSource source          = 1;
  Program     control_program = 2;
  uint64      ordinal         = 3;
}

message Retirement {
  ValueSource source   = 1;
  uint64      ordinal  = 2;
}

message Issuance {
  Hash             nonce_hash               = 1;
  AssetAmount      value                    = 2;
  ValueDestination witness_destination      = 3;
  AssetDefinition  witness_asset_definition = 4;
  repeated bytes   witness_arguments        = 5;
  uint64           ordinal                  = 6;
}

message Spend {
  Hash             spent_output_id     = 1;
  ValueDestination witness_destination = 2;
  repeated bytes   witness_arguments   = 3;
  uint64           ordinal             = 4;
}
```

### 根据bc.proto生成bc.pb.go代码

``` base
protoc -I/usr/local/include -I. \
      -I${GOPATH}/src \
      --go_out=plugins=grpc:. \
      ./*.proto
```
执行完上面命令，我们会看到当前目录下生成的bc.pb.go文件，该文件在比原链中承载这block、transaction、coinbase等重要数据结构
