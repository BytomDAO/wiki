---
title: bytom源码分析-孤块管理
date: 2018-08-16 11:53:55
categories:
tags: [golang, bytom]
---
# bytom源码分析-孤块管理

## 简介

https://github.com/Bytom/bytom

本章介绍bytom代码孤块管理

> 作者使用MacOS操作系统，其他平台也大同小异

> Golang Version: 1.8


## 孤块介绍
### 什么是孤块
当节点收到了一个有效的区块，而在现有的主链中却未找到它的父区块，那么这个区块被认为是“孤块”。父区块是指当前区块的PreviousBlockHash字段指向上一区块的hash值。

接收到的孤块会被存储在孤块池中，直到它们的父区块被节点收到。一旦收到了父区块，节点就会将孤块从孤块池中取出，并且连接到它的父区块，让它作为区块链的一部分。


### 孤块出现的原因
当两个或多个区块在很短的时间间隔内被挖出来，节点有可能会以不同的顺序接收到它们，这个时候孤块现象就会出现。

我们假设有三个高度分别为100、101、102的块，分别以102、101、100的颠倒顺序被节点接收。此时节点将102、101放入到孤块管理缓存池中，等待彼此的父块。当高度为100的区块被同步进来时，会被验证区块和交易，然后存储到区块链上。这时会对孤块缓存池进行递归查询，根据高度为100的区块找到101的区块并存储到区块链上，再根据高度为101的区块找到102的区块并存储到区块链上。




## 孤块源码分析
### 孤块管理缓存池结构体
** protocol/orphan_manage.go **

```
type OrphanManage struct {
	orphan      map[bc.Hash]*types.Block
	prevOrphans map[bc.Hash][]*bc.Hash
	mtx         sync.RWMutex
}

func NewOrphanManage() *OrphanManage {
	return &OrphanManage{
		orphan:      make(map[bc.Hash]*types.Block),
		prevOrphans: make(map[bc.Hash][]*bc.Hash),
	}
}
```

* orphan 存储孤块，key为block hash，value为block结构体
* prevOrphans 存储孤块的父块
* mtx 互斥锁，保护map结构在多并发读写状态下保持数据一致



### 添加孤块到缓存池
```
func (o *OrphanManage) Add(block *types.Block) {
	blockHash := block.Hash()
	o.mtx.Lock()
	defer o.mtx.Unlock()

	if _, ok := o.orphan[blockHash]; ok {
		return
	}

	o.orphan[blockHash] = block
	o.prevOrphans[block.PreviousBlockHash] = append(o.prevOrphans[block.PreviousBlockHash], &blockHash)

	log.WithFields(log.Fields{"hash": blockHash.String(), "height": block.Height}).Info("add block to orphan")
}
```

当一个孤块被添加到缓存池中，还需要记录该孤块的父块hash。用于父块hash的查询

### 查询孤块和父孤块
```
func (o *OrphanManage) Get(hash *bc.Hash) (*types.Block, bool) {
	o.mtx.RLock()
	block, ok := o.orphan[*hash]
	o.mtx.RUnlock()
	return block, ok
}

func (o *OrphanManage) GetPrevOrphans(hash *bc.Hash) ([]*bc.Hash, bool) {
	o.mtx.RLock()
	prevOrphans, ok := o.prevOrphans[*hash]
	o.mtx.RUnlock()
	return prevOrphans, ok
}
```

### 删除孤块
```
func (o *OrphanManage) Delete(hash *bc.Hash) {
	o.mtx.Lock()
	defer o.mtx.Unlock()
	block, ok := o.orphan[*hash]
	if !ok {
		return
	}
	delete(o.orphan, *hash)

	prevOrphans, ok := o.prevOrphans[block.PreviousBlockHash]
	if !ok || len(prevOrphans) == 1 {
		delete(o.prevOrphans, block.PreviousBlockHash)
		return
	}

	for i, preOrphan := range prevOrphans {
		if preOrphan == hash {
			o.prevOrphans[block.PreviousBlockHash] = append(prevOrphans[:i], prevOrphans[i+1:]...)
			return
		}
	}
}
```
删除孤块的过程中，同时删除父块

### 孤块处理逻辑
** protocol/block.go **

```
func (c *Chain) processBlock(block *types.Block) (bool, error) {
blockHash := block.Hash()
	if c.BlockExist(&blockHash) {
		log.WithFields(log.Fields{"hash": blockHash.String(), "height": block.Height}).Info("block has been processed")
		return c.orphanManage.BlockExist(&blockHash), nil
	}

	if parent := c.index.GetNode(&block.PreviousBlockHash); parent == nil {
		c.orphanManage.Add(block)
		return true, nil
	}

	if err := c.saveBlock(block); err != nil {
		return false, err
	}

	bestBlock := c.saveSubBlock(block)
	// ...
}	
```
processBlock函数处理block块加入区块链上之前的过程。

c.BlockExist判断当前block块是否存在于区块链上或是否存在孤块缓存池中，如果存在则返回。

c.index.GetNode判断block块的父节点是否存在。如果在现有的主链中却未找到它的父区块则将block块添加到孤块缓存池。

c.saveBlock走到了这一步说明，block父节点是存在于区块链，则将block块存储到区块链。该函数会验证区块和交易有效性。

saveSubBlock 代码如下：

```

func (c *Chain) saveSubBlock(block *types.Block) *types.Block {
	blockHash := block.Hash()
	prevOrphans, ok := c.orphanManage.GetPrevOrphans(&blockHash)
	if !ok {
		return block
	}

	bestBlock := block
	for _, prevOrphan := range prevOrphans {
		orphanBlock, ok := c.orphanManage.Get(prevOrphan)
		if !ok {
			log.WithFields(log.Fields{"hash": prevOrphan.String()}).Warning("saveSubBlock fail to get block from orphanManage")
			continue
		}
		if err := c.saveBlock(orphanBlock); err != nil {
			log.WithFields(log.Fields{"hash": prevOrphan.String(), "height": orphanBlock.Height}).Warning("saveSubBlock fail to save block")
			continue
		}

		if subBestBlock := c.saveSubBlock(orphanBlock); subBestBlock.Height > bestBlock.Height {
			bestBlock = subBestBlock
		}
	}
	return bestBlock
}
```


saveSubBlock 在孤块缓存池中查询是否存在当前区块的下一个区块。比如当前区块高度为100，则在孤块缓存池中查询是否有区块高度为101的区块。如果存在则将101区块存储到区块链并从孤块缓存池中删除该区块。

saveSubBlock是一个递归函数的实现。目的是为了寻找最深叶子节点的递归方式。比如当前区块高度为100的，递归查询出高度为99、98、97等高度的区块。






