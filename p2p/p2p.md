p2p 实现了四个主要功能：
* 1. address book 用于邻居地址存储和索引。
* 2. pex secure connection 用于数据加密存储，采用[ECDH 算法](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)
* 3. Reactor 模式的消息转发。
* 4. Upnp的扩展。
