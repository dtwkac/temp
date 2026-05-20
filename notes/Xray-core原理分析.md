# Xray-core 翻墙原理深度分析 — VLESS / VMess 结合完整代码

---

## 一、整体架构：代理链路

```
[App] → [VLESS/VMess Outbound] → [Transport: TCP/TLS/REALITY] → GFW →
[Transport: TCP/TLS/REALITY] → [VLESS/VMess Inbound] → [Dispatcher] → [Freedom Outbound] → [目标网站]
```

**核心入口**：

`main/main.go` → `core/core.go` 的 `New` 创建 `Instance` → `Instance.Start` 启动所有 feature → 每个 proxy 的 `init()` 通过 `common.RegisterConfig` 注册到全局。

---

## 二、连接建立阶段——握手协议

### 2.1 VMess Client 端握手 — `proxy/vmess/encoding/client.go`

`ClientSession.EncodeRequestHeader` 构建加密的请求头：

```go
func (c *ClientSession) EncodeRequestHeader(header *protocol.RequestHeader, writer io.Writer) error {
    // 1. 构建明文部分: 版本(1) + bodyIV(16) + bodyKey(16) + responseHeader(1) + option(1)
    //    + security+padding(1) + reserved(1) + command(1) + 目标地址 + 随机填充 + FNV 校验
    buffer.WriteByte(Version)          // 协议版本: 1
    buffer.Write(c.requestBodyIV[:])   // 16 字节随机请求体初始向量
    buffer.Write(c.requestBodyKey[:])  // 16 字节随机请求体加密密钥
    buffer.WriteByte(c.responseHeader) // 1 字节响应头标识（用于验证）
    buffer.WriteByte(byte(header.Option)) // 选项标记
    security := byte(paddingLen<<4) | byte(header.Security)
    buffer.Write([]byte{security, byte(0), byte(header.Command)})
    addrParser.WriteAddressPort(buffer, header.Address, header.Port) // 目标地址+端口
    // 随机填充 0-15 字节，使包长度不可预测
    buffer.ReadFullFrom(rand.Reader, int32(paddingLen))
    // FNV-1a 哈希校验
    fnv1a.Write(buffer.Bytes())
    hashBytes := buffer.Extend(int32(fnv1a.Size()))
    fnv1a.Sum(hashBytes[:0])

    // 2. 关键：使用 CmdKey 对整个请求头进行 AEAD 加密
    // CmdKey 由用户 UUID 经过 KDF 派生而来
    copy(fixedLengthCmdKey[:], account.ID.CmdKey())
    vmessout := vmessaead.SealVMessAEADHeader(fixedLengthCmdKey, buffer.Bytes())
    io.Copy(writer, bytes.NewReader(vmessout))
    return nil
}
```

**Vmess 关键安全特性**：
- 整个请求头（含目标地址）被 AEAD 加密，GFW 无法看到用户访问的目标
- 每连接随机生成 key/IV，无固定特征
- FNV 校验保证完整性
- 随机 padding 防止包长度指纹分析

`ClientSession.NewClientSession` 中生成随机 key：

```go
func NewClientSession(ctx context.Context, behaviorSeed int64) *ClientSession {
    randomBytes := make([]byte, 33) // 16+16+1
    rand.Read(randomBytes)
    copy(session.requestBodyKey[:], randomBytes[:16])   // 请求体 AES 密钥
    copy(session.requestBodyIV[:], randomBytes[16:32])  // 请求体初始向量
    session.responseHeader = randomBytes[32]             // 响应头验证字节
    // 响应体密钥 = SHA256(请求体密钥) 的前 16 字节
    BodyKey := sha256.Sum256(session.requestBodyKey[:])
    copy(session.responseBodyKey[:], BodyKey[:16])
    ...
}
```

### 2.2 VMess Server 端解密 — `proxy/vmess/encoding/server.go`

`ServerSession.DecodeRequestHeader` 反向过程：

```go
func (s *ServerSession) DecodeRequestHeader(reader io.Reader, isDrain bool) (*protocol.RequestHeader, error) {
    // 1. 读 16 字节身份认证 ID（也是 UUID 派生的）
    buffer.ReadFullFrom(reader, protocol.IDBytesLen)

    // 2. 尝试 AEAD 解密 — 用所有已知用户的 CmdKey 尝试
    user, foundAEAD, errorAEAD := s.userValidator.GetAEAD(buffer.Bytes())

    // 3. 用 CmdKey 解密 AEAD 头部
    copy(fixedSizeCmdKey[:], vmessAccount.ID.CmdKey())
    aeadData, shouldDrain, bytesRead, errorReason :=
        vmessaead.OpenVMessAEADHeader(fixedSizeCmdKey, fixedSizeAuthID, reader)

    // 4. 防重放攻击: 检查 session 是否已被使用过
    if !s.sessionHistory.addIfNotExits(sid) {
        return nil, errors.New("duplicated session id, possibly under replay attack")
    }

    // 5. 从解密后的数据中解析目标地址、命令、安全类型
    request.Command = protocol.RequestCommand(buffer.Byte(37))
    addrParser.ReadAddressPort(buffer, decryptor) // 读取目标地址

    // 6. 验证 FNV-1a 完整性校验
    fnv1a.Write(buffer.BytesTo(-4))
    actualHash := fnv1a.Sum32()
    expectedHash := binary.BigEndian.Uint32(buffer.BytesFrom(-4))
    if actualHash != expectedHash {
        return nil, errors.New("invalid auth")
    }
    return request, nil
}
```

`SessionHistory.addIfNotExits` 是防重放攻击的核心：

```go
type sessionID struct {
    user  [16]byte  // 用户 ID
    key   [16]byte  // 请求体 Key
    nonce [16]byte  // 请求体 IV
}

func (h *SessionHistory) addIfNotExits(session sessionID) bool {
    if expire, found := h.cache[session]; found && expire.After(time.Now()) {
        return false  // 拒绝重放
    }
    h.cache[session] = time.Now().Add(time.Minute * 3)  // 3 分钟有效期
    return true
}
```

### 2.3 VLESS Client 端握手 — `proxy/vless/encoding/encoding.go`

`EncodeRequestHeader` 极其简单——**与 VMess 形成鲜明对比**：

```go
func EncodeRequestHeader(writer io.Writer, request *protocol.RequestHeader, requestAddons *Addons) error {
    buffer.WriteByte(request.Version)                        // 版本: 0
    buffer.Write(request.User.Account.(*vless.MemoryAccount).ID.Bytes()) // 16 字节 UUID（明文！）
    EncodeHeaderAddons(&buffer, requestAddons)                // 附加信息（如 flow 类型）
    buffer.WriteByte(byte(request.Command))                   // 命令: TCP/UDP/Mux
    if request.Command != protocol.RequestCommandMux {
        addrParser.WriteAddressPort(&buffer, request.Address, request.Port) // 目标地址（明文！）
    }
    writer.Write(buffer.Bytes())
    return nil
}
```

**VLESS 协议头的特点**：UUID 和目标地址都是**明文**传输。这看起来是"不安全"的，但 VLESS 的哲学是——**加密下沉到传输层**（TLS/REALITY），协议层保持极简。整个 VLESS 头部会被 TLS 加密，所以明文不是问题。

### 2.4 VLESS Server 端验证 — 同文件 `DecodeRequestHeader`

```go
func DecodeRequestHeader(isfb bool, first *buf.Buffer, reader io.Reader,
    validator vless.Validator) ([]byte, *protocol.RequestHeader, *Addons, bool, error) {

    request.Version = buffer.Byte(0)
    copy(id[:], buffer.Bytes()) // 读取 16 字节 UUID
    request.User = validator.Get(id) // 验证用户
    if request.User == nil {
        return nil, nil, nil, isfb, errors.New("invalid request user id: " + u.String())
    }
    // 读取命令类型
    request.Command = protocol.RequestCommand(buffer.Byte(0))
    switch request.Command {
    case protocol.RequestCommandTCP, protocol.RequestCommandUDP:
        addrParser.ReadAddressPort(&buffer, reader) // 读取目标地址
    }
    return id[:], request, requestAddons, false, nil
}
```

---

## 三、数据体加密传输

### 3.1 VMess 数据体加密 — `proxy/vmess/encoding/client.go`

`ClientSession.EncodeRequestBody` 根据安全类型选择不同加密算法：

```go
func (c *ClientSession) EncodeRequestBody(request *protocol.RequestHeader, writer io.Writer) (buf.Writer, error) {
    // 分块大小解析器 — 决定数据如何分块
    var sizeParser crypto.ChunkSizeEncoder = crypto.PlainChunkSizeParser{}
    if request.Option.Has(protocol.RequestOptionChunkMasking) {
        // 使用 SHAKE128 掩盖真实数据块大小
        sizeParser = NewShakeSizeParser(c.requestBodyIV[:])
    }

    switch request.Security {
    case protocol.SecurityType_AES128_GCM:
        aead := crypto.NewAesGcm(c.requestBodyKey[:])    // AES-128-GCM
        auth := &crypto.AEADAuthenticator{
            AEAD:                    aead,
            NonceGenerator:          GenerateChunkNonce(c.requestBodyIV[:], ...),
            AdditionalDataGenerator: crypto.GenerateEmptyBytes(),
        }
        // 每个数据块独立加密，递增 nonce
        return crypto.NewAuthenticationWriter(auth, sizeParser, writer, ...), nil

    case protocol.SecurityType_CHACHA20_POLY1305:
        key := GenerateChacha20Poly1305Key(c.requestBodyKey[:])
        aead, _ := chacha20poly1305.New(key)
        // 同样使用 AEAD，但算法不同
        auth := &crypto.AEADAuthenticator{AEAD: aead, ...}
        return crypto.NewAuthenticationWriter(auth, sizeParser, writer, ...), nil
    }
}
```

`ShakeSizeParser` (`proxy/vmess/encoding/auth.go`) 的作用——掩盖真实的数据块大小：

```go
func (s *ShakeSizeParser) Encode(size uint16, b []byte) []byte {
    mask := s.next()                          // SHAKE128 伪随机流
    binary.BigEndian.PutUint16(b, mask ^ size) // XOR 掩盖真实长度
    return b[:2]
}

func (s *ShakeSizeParser) NextPaddingLen() uint16 {
    return s.next() % 64  // 每个块后随机附加 0-63 字节填充
}
```

**这就是对抗流量分析的核心**——GFW 常用的手段是通过分析数据包大小模式来识别代理流量，ShakeSizeParser 使每个数据块的大小看起来是随机的。

**服务端解密** — 同文件 `ServerSession.DecodeRequestBody`，对称操作，使用相同的安全类型选择逻辑。

### 3.2 VLESS 数据体 — 无加密，但有 XTLS Vision

VLESS 的数据体用 `encoding.DecodeBodyAddons` / `encoding.EncodeBodyAddons` 处理，这些是对传输层的**透明包装**，不做加密。

真正的加密由传输层完成。如果使用 `flow: "xtls-rprx-vision"`，则在数据体处理中插入一个关键的 Vision Reader：

**VLESS Outbound** — `proxy/vless/outbound/outbound.go` 的 `Handler.Process`：

```go
// 构建数据流向
clientReader := link.Reader
clientWriter := link.Writer

// 对请求体进行编码（无加密，但包装 XTLS Flow）
serverWriter := encoding.EncodeBodyAddons(bufferWriter, request, requestAddons,
    trafficState, true, ctx, conn, ob)

// 使用 XTLS Vision 处理响应
serverReader := encoding.DecodeBodyAddons(conn, request, responseAddons)
if requestAddons.Flow == vless.XRV {
    serverReader = proxy.NewVisionReader(serverReader, trafficState, false, ctx,
        conn, input, rawInput, ob)
}
```

---

## 四、XTLS Vision — 真正的翻墙技术核心

这是 Xray-core 区别于 V2Ray 的最大创新，代码在 `proxy/vless/inbound/inbound.go` 和 `proxy/vless/outbound/outbound.go`。

### 4.1 原理：避免 TLS-in-TLS

传统的 TLS 代理有两个 TLS 层：

```
客户端 TLS → 代理服务器 TLS → 目标网站 TLS
```

GFW 可以通过检测 "两层 TLS 握手" 特征来识别。XTLS Vision 的思路：**让客户端的 TLS 连接直接穿透到目标**，中间不解密。

### 4.2 XTLS Vision 客户端 — `proxy/vless/outbound/outbound.go` 的 `Handler.Process`

```go
// 检测 flow 是否为 "xtls-rprx-vision"
switch requestAddons.Flow {
case vless.XRV:
    ob.CanSpliceCopy = 2  // 标记可以 splice
    switch request.Command {
    case protocol.RequestCommandTCP, protocol.RequestCommandMux, protocol.RequestCommandRvs:
        // 关键：使用 unsafe 直接访问 TLS 连接的内部缓冲区
        // 获取 Go crypto/tls.Conn 内部的 input 和 rawInput 字段
        var t reflect.Type
        var p uintptr
        if tlsConn, ok := iConn.(*tls.Conn); ok {
            t = reflect.TypeOf(tlsConn.Conn).Elem()
            p = uintptr(unsafe.Pointer(tlsConn.Conn))
        } else if utlsConn, ok := iConn.(*tls.UConn); ok {
            t = reflect.TypeOf(utlsConn.Conn).Elem()
            p = uintptr(unsafe.Pointer(utlsConn.Conn))
        } else if realityConn, ok := iConn.(*reality.UConn); ok {
            t = reflect.TypeOf(realityConn.Conn).Elem()
            p = uintptr(unsafe.Pointer(realityConn.Conn))
        }
        // 直接定位到 TLS 内部的 bytes.Reader / bytes.Buffer
        i, _ := t.FieldByName("input")     // TLS 未解密数据
        r, _ := t.FieldByName("rawInput")  // TLS 原始输入
        input = (*bytes.Reader)(unsafe.Pointer(p + i.Offset))
        rawInput = (*bytes.Buffer)(unsafe.Pointer(p + r.Offset))
    }
}
```

这段代码通过 `unsafe.Pointer` + `reflect` 直接操作 Go 标准库 `crypto/tls.Conn` 的内部字段。`input` 是已经 TLS 解密但尚未读取的数据，`rawInput` 是 TLS 收到的原始密文。通过操作这两个缓冲区，XTLS 可以从 TLS 连接中"偷"出解密后的数据，直接发送给目标，**无需在用户态再解密一次**。

### 4.3 XTLS Vision 服务端 — `proxy/vless/inbound/inbound.go` 的 `Handler.Process`

服务端做同样的操作，但方向相反：

```go
// 同样使用 unsafe 访问 TLS 内部
if tlsConn, ok := iConn.(*tls.Conn); ok {
    // 要求 TLS 1.3
    if tlsConn.ConnectionState().Version != gotls.VersionTLS13 {
        return errors.New(`failed to use XRV, found outer tls version ...`)
    }
    t = reflect.TypeOf(tlsConn.Conn).Elem()
    p = uintptr(unsafe.Pointer(tlsConn.Conn))
} else if realityConn, ok := iConn.(*reality.Conn); ok {
    t = reflect.TypeOf(realityConn.Conn).Elem()
    p = uintptr(unsafe.Pointer(realityConn.Conn))
}
// 直接操作 TLS 内部缓冲区实现零拷贝
i, _ := t.FieldByName("input")
r, _ := t.FieldByName("rawInput")
input = (*bytes.Reader)(unsafe.Pointer(p + i.Offset))
rawInput = (*bytes.Buffer)(unsafe.Pointer(p + r.Offset))
```

**为什么必须是 TLS 1.3**？TLS 1.3 的 record layer 固定为 1:1 映射（一个 record = 一个应用数据块），这使得可以直接从 TLS 缓冲区提取明文。

### 4.4 零拷贝转发 — `proxy/vless/encoding/encoding.go` 的 `XtlsRead`

```go
func XtlsRead(reader buf.Reader, writer buf.Writer, timer *signal.ActivityTimer,
    conn net.Conn, trafficState *proxy.TrafficState, isUplink bool, ctx context.Context) error {
    for {
        // 检查是否可以直接进行内核态 splice 拷贝
        if isUplink && trafficState.Inbound.UplinkReaderDirectCopy ||
           !isUplink && trafficState.Outbound.DownlinkReaderDirectCopy {
            // 直接拷贝原始连接 → 内核 splice，零拷贝
            return proxy.CopyRawConnIfExist(ctx, conn, writerConn, writer, timer, inTimer)
        }
        // 回退到普通用户态拷贝
        buffer, err := reader.ReadMultiBuffer()
        writer.WriteMultiBuffer(buffer)
    }
}
```

`proxy.CopyRawConnIfExist` 尝试使用 `splice(2)` 系统调用（Linux 内核特性）在两个 socket 之间直接传输数据，**完全不需要经过用户态内存**。这是最高效的零拷贝路径。

### 4.5 VLESS 的 Padding 伪装 — `proxy/vless/outbound/outbound.go` 的 `postRequest`

```go
// 当使用 XTLS Vision 且首次读取超时时，发送空 padding
if requestAddons.Flow == vless.XRV {
    mb := make(buf.MultiBuffer, 1)
    errors.LogInfo(ctx, "Insert padding with empty content to camouflage VLESS header ")
    serverWriter.WriteMultiBuffer(mb)
}
```

**目的**：VLESS 的请求头非常短且固定格式，GFW 可以根据"先发几个字节然后立即发大量数据"的模式识别。插入空 padding 字节使得 VLESS 头部与普通 TLS 应用数据的边界模糊化。

---

## 五、REALITY — 终极流量伪装

`transport/internet/reality/reality.go`

REALITY 解决了一个根本性问题：**你的代理服务器需要 TLS 证书**。传统的 TLS 代理要么用自签名证书（一眼被识别），要么用 Let's Encrypt（多了个你不应该有的域名）。

### 5.1 REALITY 的核心思想

> 你的代理服务器不需要自己的 TLS 证书。它直接使用目标网站（如 `microsoft.com`）的公钥。

```go
// reality.go Server 函数
func Server(c net.Conn, config *reality.Config) (net.Conn, error) {
    realityConn, err := reality.Server(context.Background(), c, config)
    return &Conn{Conn: realityConn}, err
}
```

REALITY 底层的实现来自 `github.com/xtls/reality` 包。它的工作方式：

1. **客户端**：使用目标网站的公钥加密 TLS ClientHello 中的 key_share
2. **服务器**：用私钥解密，如果能解密 → 是合法代理客户端
3. **不能解密** → 直接把原始 TCP 流量转发到目标网站

**效果**：GFW 主动探测时，看到的是正常的 HTTPS 连接（如访问 `microsoft.com`），完全看不出是代理。

### 5.2 REALITY 客户端 — `reality.go UConn`

```go
type UConn struct {
    *utls.UConn        // 使用 uTLS 模拟浏览器指纹
    Config     *Config
    ServerName string
    ...
}
```

`github.com/refraction-networking/utls` 可以模拟 Chrome、Firefox、Safari 等的 TLS 指纹（支持的密码套件、扩展顺序、ALPN 等），**使 TLS ClientHello 与真实浏览器完全一致**。

---

## 六、Fallback 机制 — 抗主动探测

`proxy/vless/inbound/inbound.go` 的 `Handler.Process`：

```go
// 如果请求头解码失败（非法的 VLESS 请求）
if err != nil {
    // 根据 ServerName/ALPN/Path 查找 fallback 配置
    name := ""
    alpn := ""
    if tlsConn, ok := iConn.(*tls.Conn); ok {
        cs := tlsConn.ConnectionState()
        name = cs.ServerName     // 探针发送的 SNI
        alpn = cs.NegotiatedProtocol
    }

    // 查找匹配的 fallback 目的地
    fb := pfb[path]

    // 连接到伪装目标（如 Nginx）
    conn, err = dialer.DialContext(ctx, fb.Type, fb.Dest)

    // 将客户端请求原样转发到伪装目标
    buf.Copy(reader, serverWriter, buf.UpdateActivity(timer))
    // 将伪装目标的响应转发回客户端
    buf.Copy(serverReader, writer, buf.UpdateActivity(timer))
}
```

**典型配置**：同一个端口的连接，如果是合法 VLESS 客户端 → 代理处理；如果是 GFW 探测流量 → 转发到本地 Nginx 的 127.0.0.1:80，返回正常的网页。

---

## 七、Dispatcher — 路由与分发

`app/dispatcher/default.go` 的 `DefaultDispatcher.Dispatch` / `DispatchLink`

两个协议最终都调用这里：

```go
// VLESS 服务端
dispatch.DispatchLink(ctx, request.Destination(), &transport.Link{
    Reader: clientReader,
    Writer: clientWriter,
})

// VMess 服务端
link, err := dispatcher.Dispatch(ctx, request.Destination())
```

`Dispatch` 做了三件事：
1. 根据目标地址查询路由规则（`app/router/router.go`）
2. 选择对应的 outbound handler
3. 创建 pipe（`transport/pipe/pipe.go`），连接 inboud 和 outbound

最终数据通过 **Freedom outbound**（`proxy/freedom/freedom.go`）发出：

```go
// proxy/freedom/freedom.go Handler.Process
// 直接拨号到目标地址并双向拷贝
conn, err := dialer.Dial(ctx, destination)
buf.Copy(reader, writer)
```

---

## 八、总结表格

| 技术 | 文件 | 关键函数/类型 | 作用 |
|---|---|---|---|
| VMess AEAD 加密 | `proxy/vmess/encoding/client.go` | `ClientSession.EncodeRequestHeader` | 加密整个请求头（含目标地址） |
| VMess 防重放 | `proxy/vmess/encoding/server.go` | `SessionHistory.addIfNotExits` | 拒绝重放攻击 |
| VMess 数据加密 | `proxy/vmess/encoding/client.go` | `ClientSession.EncodeRequestBody` | AES-128-GCM / ChaCha20-Poly1305 |
| VMess 大小混淆 | `proxy/vmess/encoding/auth.go` | `ShakeSizeParser.Encode` | SHAKE128 XOR 掩盖数据块大小 |
| VLESS 协议 | `proxy/vless/encoding/encoding.go` | `EncodeRequestHeader` | 极简协议头，加密下沉到传输层 |
| XTLS Vision | `proxy/vless/outbound/outbound.go` | `Handler.Process` | unsafe 访问 TLS 缓冲区，零拷贝 |
| XTLS 零拷贝 | `proxy/vless/encoding/encoding.go` | `XtlsRead` | splice 系统调用，内核态拷贝 |
| REALITY | `transport/internet/reality/reality.go` | `Server` / `UConn` | 用目标网站公钥替代证书 |
| uTLS 指纹伪装 | `transport/internet/tls/tls.go` | `UConn` | 模拟浏览器 TLS ClientHello |
| Fallback | `proxy/vless/inbound/inbound.go` | `Handler.Process` | 非法流量转发到伪装服务 |
| 路由分发 | `app/dispatcher/default.go` | `Dispatch` / `DispatchLink` | 根据路由规则转发到目标 |
| 最终代理 | `proxy/freedom/freedom.go` | `Handler.Process` | 直连目标网站并双向拷贝 |
