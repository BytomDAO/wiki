
## 反应堆设计模式
是一种并行处理服务请求分发的事件模型。服务处理器分离收到的请求并调度相对用的请求处理器同步的处理各自的请求。

### 组成结构
***Resources(可以被监听的事件)***
任何可以向系统输入或者消耗输出的资源(linux 上对应的可以用epoll监听的事件)。
***Synchronous Event Demultiplexer(同步的事件解复用器)***
用一个EventLoop 阻塞所有的资源。解复用器触发调度器，同步的操作变为非阻塞的资源。(实例：
- 同步系统调用read()函数 将会在没有数据可读时是阻塞的。
- 解复用器用select()函数来监听该资源，直到该资源变为可读时。
- read()函数不会被阻塞， 解复用器可以将该资源发送给调度器。)
**Dispatcher(调度器)**
处理请求处理程序的注册和注销。将资源从多路分解器调度到相关联的请求处理程序。
**Request Handler(请求处理程序)**
应用程序定义的与它自己相关联资源的请求处理程序。
### 属性
所有反应堆系统都是单线程的，但可以存在于多线程环境中。
### 优点
反应堆模式完全将应用程序特定的代码与反应堆实现分离，这意味着应用程序组件可以分为模块化的，可重复使用的部分。此外，由于请求处理程序的同步调用，反应器模式允许简单的粗粒度并发，而不会将多个线程的复杂性增加到系统。
### 缺点
由于反向控制流程，反应堆模式可能比由程序模式更难调试。而且，通过只同时调用请求处理程序，反应堆模式限制了最大的并发性，特别是在对称多处理硬件上。反应堆模式的可扩展性不仅受到同步调用请求处理程序的限制，还受到解复用器的限制。

### reactor 模式实例
***解复用器， EventLoop 循环， 将资源传送给调度器调用***
``` go
// recvRoutine reads msgPackets and reconstructs the message using the channels' "recving" buffer.
// After a whole message has been assembled, it's pushed to onReceive().
// Blocks depending on how the connection is throttled.
func (c *MConnection) recvRoutine() {
    defer c._recover()

FOR_LOOP:
    for {
        // Block until .recvMonitor says we can read.
        c.recvMonitor.Limit(maxMsgPacketTotalSize, atomic.LoadInt64(&c.config.RecvRate), true)

        /*
            // Peek into bufReader for debugging
            if numBytes := c.bufReader.Buffered(); numBytes > 0 {
                log.Info("Peek connection buffer", "numBytes", numBytes, "bytes", log15.Lazy{func() []byte {
                    bytes, err := c.bufReader.Peek(MinInt(numBytes, 100))
                    if err == nil {
                        return bytes
                    } else {
                        log.Warn("Error peeking connection buffer", "error", err)
                        return nil
                    }
                }})
            }
        */

        // Read packet type
        var n int
        var err error
        pktType := wire.ReadByte(c.bufReader, &n, &err)
        c.recvMonitor.Update(int(n))
        if err != nil {
            if c.IsRunning() {
                c.Logger.Error("Connection failed @ recvRoutine (reading byte)", "conn", c, "error", err)
                c.stopForError(err)
            }
            break FOR_LOOP
        }

        // Read more depending on packet type.
        switch pktType {
        case packetTypePing:
            // TODO: prevent abuse, as they cause flush()'s.
            c.Logger.Debug("Receive Ping")
            c.pong <- struct{}{}
        case packetTypePong:
            // do nothing
            c.Logger.Debug("Receive Pong")
        case packetTypeMsg:
            pkt, n, err := msgPacket{}, int(0), error(nil)
            wire.ReadBinaryPtr(&pkt, c.bufReader, maxMsgPacketTotalSize, &n, &err)
            c.recvMonitor.Update(int(n))
            if err != nil {
                if c.IsRunning() {
                    c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "error", err)
                    c.stopForError(err)
                }
                break FOR_LOOP
            }
            channel, ok := c.channelsIdx[pkt.ChannelID]
            if !ok || channel == nil {
                cmn.PanicQ(cmn.Fmt("Unknown channel %X", pkt.ChannelID))
            }
            msgBytes, err := channel.recvMsgPacket(pkt)
            if err != nil {
                if c.IsRunning() {
                    c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "error", err)
                    c.stopForError(err)
                }
                break FOR_LOOP
            }
            if msgBytes != nil {
                c.Logger.Debug("Received bytes", "chID", pkt.ChannelID, "msgBytes", msgBytes)
                c.onReceive(pkt.ChannelID, msgBytes)
            }
        default:
            cmn.PanicSanity(cmn.Fmt("Unknown message type %X", pktType))
        }

        // TODO: shouldn't this go in the sendRoutine?
        // Better to send a ping packet when *we* haven't sent anything for a while.
        c.pingTimer.Reset()
    }

    // Cleanup
    close(c.pong)
    for _ = range c.pong {
        // Drain
    }
}
```
***调度器 负责注册资源，以及资源和请求程序的绑定***
``` go
func (sw *Switch) AddReactor(name string, reactor Reactor) Reactor {
    // Validate the reactor.
    // No two reactors can share the same channel.
    reactorChannels := reactor.GetChannels()
    for _, chDesc := range reactorChannels {
        chID := chDesc.ID
        if sw.reactorsByCh[chID] != nil {
            cmn.PanicSanity(fmt.Sprintf("Channel %X has multiple reactors %v & %v", chID, sw.reactorsByCh[chID], reactor))
        }
        sw.chDescs = append(sw.chDescs, chDesc)
        sw.reactorsByCh[chID] = reactor
    }
    sw.reactors[name] = reactor
    reactor.SetSwitch(sw)
    return reactor
}
```
***抽象出来的接口， 编程只要继承这些接口即可***
``` go
type BaseReactor struct {
    cmn.BaseService // Provides Start, Stop, .Quit
    Switch          *Switch
}

func NewBaseReactor(name string, impl Reactor) *BaseReactor {
    return &BaseReactor{
        BaseService: *cmn.NewBaseService(nil, name, impl),
        Switch:      nil,
    }
}

func (br *BaseReactor) SetSwitch(sw *Switch) {
    br.Switch = sw
}
func (_ *BaseReactor) GetChannels() []*ChannelDescriptor              { return nil }
func (_ *BaseReactor) AddPeer(peer *Peer)                             {}
func (_ *BaseReactor) RemovePeer(peer *Peer, reason interface{})      {}
func (_ *BaseReactor) Receive(chID byte, peer *Peer, msgBytes []byte) {}
```
***编写代码范本***
``` go
type MyReactor struct{}

// 对应于资源
func (reactor MyReactor) GetChannels() []*ChannelDescriptor {
    return []*ChannelDescriptor{ChannelDescriptor{ID:MyChannelID, Priority: 1}}
}
// 资源相对应的请求处理程序
func (reactor MyReactor) Receive(chID byte, peer *Peer, msgBytes []byte) {
    r, n, err := bytes.NewBuffer(msgBytes), new(int64), new(error)
    msgString := ReadString(r, n, err)
    fmt.Println(msgString)
}
```
