# v2rayN config.json 生成全过程详解

> 本文分析 v2rayN 如何将用户选中的代理节点转换为 Xray 可执行的 `config.json`，涵盖从用户点击"启动"到 config 落盘的完整链路。

---

## 目录

1. [总体流程图](#1-总体流程图)
2. [触发入口：CoreManager.LoadCore](#2-触发入口coremanagerloadcore)
3. [上下文构建：CoreConfigContextBuilder.Build](#3-上下文构建coreconfigcontextbuilderbuild)
4. [Config 分发：CoreConfigHandler.GenerateClientConfig](#4-config-分发coreconfighandlergenerateclientconfig)
5. [模板加载](#5-模板加载)
6. [各节填充](#6-各节填充)
   - [6a. GenLog](#6a-genlog)
   - [6b. GenInbounds](#6b-geninbounds)
   - [6c. GenOutbounds](#6c-genoutbounds)
   - [6d. GenRouting](#6d-genrouting)
   - [6e. GenDns](#6e-gendns)
   - [6f. GenStatistic](#6f-genstatistic)
   - [6g. 后处理](#6g-后处理)
7. [模板合并：ApplyFullConfigTemplate](#7-模板合并applyfullconfigtemplate)
8. [写入磁盘](#8-写入磁盘)
9. [最终输出示例](#9-最终输出示例)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 总体流程图

```
CoreManager.LoadCore (用户点击启动)
    │
    ▼
CoreConfigContextBuilder.Build
    │  ┌─────────────────────────┐
    │  │ 解析路由规则引用的节点   │
    │  │ 解析代理链 prev/next     │
    │  │ 节点有效性校验           │
    │  │ 构建 AllProxiesMap       │
    │  └─────────────────────────┘
    ▼
CoreConfigHandler.GenerateClientConfig
    │
    ├── Custom 类型 → 直接复制文件
    ├── sing_box  → CoreConfigSingboxService
    └── Xray      → CoreConfigV2rayService  ← 本文追踪
                        │
                        ▼
              CoreConfigV2rayService.GenerateClientConfigContent
                        │
                        ├── 1. 加载嵌入模板 SampleClientConfig
                        ├── 2. GenLog()       → log 节
                        ├── 3. GenInbounds()  → inbounds 节
                        ├── 4. GenOutbounds() → outbounds 节 (核心)
                        │       ├── BuildProxyOutbound()
                        │       │   ├── FillOutbound() → 协议字段
                        │       │   └── FillBoundStreamSettings() → 传输+TLS
                        │       └── (可选) GenObservatory + GenBalancer
                        ├── 5. GenRouting()   → routing 节
                        ├── 6. GenDns()       → dns 节
                        ├── 7. GenStatistic() → stats/metrics 节
                        ├── 8. ApplyOutboundFragment (可选)
                        ├── 9. ApplyOutboundBindInterface
                        ├── 10. ApplyOutboundSendThrough
                        └── 11. ApplyFullConfigTemplate → 合并用户模板
                                    │
                                    ▼
                          返回最终 JSON 字符串
                                    │
                                    ▼
                        File.WriteAllTextAsync(config.json)
                                    │
                                    ▼
                        Xray 进程启动，读取 config.json
```

---

## 2. 触发入口：CoreManager.LoadCore

**文件**: `ServiceLib/Manager/CoreManager.cs`

```csharp
// :63
public async Task LoadCore(CoreConfigContext? mainContext, CoreConfigContext? preContext)
{
    // :71-73 生成 config.json
    var fileName = Utils.GetBinConfigPath(Global.CoreConfigFileName);  // => "config.json"
    var result = await CoreConfigHandler.GenerateClientConfig(mainContext, fileName);

    // :83 停止旧核心
    await CoreStop();

    // :92-93 启动主核心 + 可选 PreService（链式/SOCKS前置）
    await CoreStart(mainContext);
    await CoreStartPreService(preContext);
}
```

- `CoreStart`（`:176-189`）→ 从 `CoreInfo` 获取核心可执行文件路径 → `RunProcess`
- `CoreStartPreService`（`:191-209`）→ 如果存在 `preContext`（链式前置代理），为其生成 `configPre.json` 并启动第二个核心进程

---

## 3. 上下文构建：CoreConfigContextBuilder.Build

**文件**: `ServiceLib/Handler/Builder/CoreConfigContextBuilder.cs`

### 3.1 构建 CoreConfigContext（`:34-52`）

```csharp
public static async Task<CoreConfigContextBuilderResult> Build(Config config, ProfileItem node)
{
    var context = new CoreConfigContext()
    {
        Node = node,                        // 用户选择的节点
        RunCoreType = runCoreType,          // Xray / sing_box
        AllProxiesMap = [],                 // 路由规则引用的 outbound 节点
        AppConfig = config,                 // 应用全局配置
        FullConfigTemplate = ...,           // 用户自定义全量配置模板
        IsTunEnabled = config.TunModeItem.EnableTun,
        SimpleDnsItem = config.SimpleDNSItem,
        RawDnsItem = await AppManager.Instance.GetDNSItem(coreType),
        RoutingItem = await ConfigHandler.GetDefaultRouting(config),
        IsWindows = Utils.IsWindows(),
        IsMacOS = Utils.IsMacOS(),
        ProtectDomainList = [],
    };
}
```

### 3.2 解析节点（`:54-59`）

```csharp
var (actNode, nodeValidatorResult) = await ResolveNodeAsync(context, node);
```

### 3.3 ResolveNodeAsync（`:158-188`）

节点解析流程：

```
ResolveNodeAsync(node)
    │
    ├── BuildSubscriptionChainNodeAsync(node) → 检查订阅的 prev/next profile
    │   如果存在 → 创建 EConfigType.ProxyChain 虚拟节点，包含所有链式节点
    │
    └── RegisterNodeAsync(node)
            │
            ├── 如果是 Group 类型 → RegisterGroupNodeAsync
            │       └── TraverseGroupNodeAsync → 递归遍历所有子节点
            │
            └── 如果是单节点 → RegisterSingleNodeAsync
                    ├── NodeValidator.Validate(node, runCoreType) → 校验
                    ├── context.AllProxiesMap[node.IndexId] = node
                    └── 记录 ProtectDomainList (地址/ECH/xhttp)
```

### 3.4 解析路由规则引用的 outbound（`:61-93`）

对于 `RoutingItem.RuleSet` 中每条启用的规则：
- 如果 `OutboundTag` 不是内置标签（direct/block/proxy），认为是用户自定义的节点备注名
- 从数据库查找该节点 → 递归 `ResolveNodeAsync` → 加入 `AllProxiesMap`

### 3.5 BuildAll（`:103-126`）

`BuildAll` 额外构建 `PreSocksContext`（用于 TUN 保护模式或前置 SOCKS 链式代理）。

---

## 4. Config 分发：CoreConfigHandler.GenerateClientConfig

**文件**: `ServiceLib/Handler/CoreConfigHandler.cs`

```csharp
// :10-42
public static async Task<RetResult> GenerateClientConfig(CoreConfigContext context, string? fileName)
{
    if (node.ConfigType == EConfigType.Custom)
    {
        // :18-22 Custom 类型：直接复制用户提供的配置文件
        result = node.CoreType switch
        {
            ECoreType.mihomo => new CoreConfigClashService(config).GenerateClientCustomConfig(node, fileName),
            _ => GenerateClientCustomConfig(node, fileName) // 直接 File.Copy
        };
    }
    else if (context.RunCoreType == ECoreType.sing_box)
    {
        // :24-27 sing-box
        result = new CoreConfigSingboxService(context).GenerateClientConfigContent();
    }
    else
    {
        // :28-31 Xray ← 本文追踪
        result = new CoreConfigV2rayService(context).GenerateClientConfigContent();
    }

    // :36-39 写入磁盘
    if (fileName.IsNotEmpty() && result.Data != null)
        await File.WriteAllTextAsync(fileName, result.Data.ToString());
}
```

---

## 5. 模板加载

**文件**: `ServiceLib/Services/CoreConfig/V2ray/CoreConfigV2rayService.cs`

### 5.1 GenerateClientConfigContent（`:13-83`）

```csharp
// :33 加载嵌入资源模板
var result = EmbedUtils.GetEmbedText(Global.V2raySampleClient);

// :40 反序列化为 C# 对象模型
_coreConfig = JsonUtils.Deserialize<V2rayConfig>(result);
```

### 5.2 数据模型（`ServiceLib/Models/CoreConfigs/V2rayConfig.cs`）

```csharp
public class V2rayConfig        // :3
{
    public Log4Ray log;                     // 日志
    public object dns;                      // DNS（运行时类型: Dns4Ray / JsonObject）
    public List<Inbounds4Ray> inbounds;     // 入站
    public List<Outbounds4Ray> outbounds;   // 出站
    public Routing4Ray routing;             // 路由
    public Metrics4Ray? metrics;            // 度量
    public Policy4Ray? policy;              // 策略
    public Stats4Ray? stats;                // 统计
    public Observatory4Ray? observatory;    // 健康观测
    public BurstObservatory4Ray? burstObservatory;  // 突发观测
}
```

**Outbounds4Ray**（`:114-129`）:
```csharp
public class Outbounds4Ray
{
    public string tag;                      // "proxy"
    public string protocol;                 // "vmess"
    public string? sendThrough;
    public string? targetStrategy;
    public Outboundsettings4Ray settings;   // vnext[] / servers[]
    public StreamSettings4Ray streamSettings;
    public Mux4Ray mux;
}
```

### 5.3 默认模板（`ServiceLib/Sample/SampleClientConfig`）

```json
{
    "log": { "access": "Vaccess.log", "error": "Verror.log", "loglevel": "warning" },
    "inbounds": [],
    "outbounds": [
        { "protocol": "freedom", "tag": "direct" },
        { "protocol": "blackhole", "tag": "block" }
    ],
    "routing": {
        "domainStrategy": "IPIfNonMatch",
        "rules": [ { "inboundTag": ["api"], "outboundTag": "api", "type": "field" } ]
    }
}
```

### 5.4 Outbound 模板（`ServiceLib/Sample/SampleOutbound`）

```json
{
    "tag": "proxy", "protocol": "vmess",
    "settings": {
        "vnext": [{ "address": "v2ray.cool", "port": 10086, "users": [{ "id": "...", "security": "auto" }] }],
        "servers": [{ "address": "v2ray.cool", "method": "chacha20", ... }]
    },
    "streamSettings": { "network": "tcp" },
    "mux": { "enabled": false }
}
```

---

## 6. 各节填充

`GenerateClientConfigContent` 按固定顺序调用（`:47-64`）：

```
GenLog() → GenInbounds() → GenOutbounds() → GenRouting() → GenDns() → GenStatistic()
```

### 6a. GenLog

**文件**: `V2rayLogService.cs:5-28`

```csharp
private void GenLog()
{
    if (_config.CoreBasicItem.LogEnabled)
    {
        _coreConfig.log.loglevel = _config.CoreBasicItem.Loglevel;  // "warning"
        _coreConfig.log.access = Utils.GetLogPath($"Vaccess_{dtNow:yyyy-MM-dd}.txt");
        _coreConfig.log.error = Utils.GetLogPath($"Verror_{dtNow:yyyy-MM-dd}.txt");
    }
    else
    {
        _coreConfig.log.loglevel = _config.CoreBasicItem.Loglevel;
        _coreConfig.log.access = null;
        _coreConfig.log.error = null;
    }
}
```

### 6b. GenInbounds

**文件**: `V2rayInboundService.cs:5-101`

- **SOCKS inbound**（`:12`）：端口 = `AppManager.Instance.GetLocalPort(EInboundProtocol.socks)`（默认 10808）
- **HTTP inbound**（`:19-23`）：如果启用了 `SecondLocalPortEnabled`，端口 +1（默认 10809）
- **LAN 访问**（`:25-47`）：`AllowLANConn` → 监听地址改为 `0.0.0.0`；可选用户名密码认证
- **TUN inbound**（`:50-69`）：如果 `IsTunEnabled`，加载 `Global.V2raySampleTunInbound` 模板添加 TUN 虚拟网卡 inbound

**BuildInbound**（`:78-100`）：
```csharp
var inbound = JsonUtils.Deserialize<Inbounds4Ray>(result);  // 嵌入模板
inbound.port = inItem.LocalPort + (int)protocol;
inbound.protocol = EInboundProtocol.mixed.ToString();  // "mixed" (同时支持 SOCKS+HTTP)
inbound.settings.udp = inItem.UdpEnabled;
inbound.sniffing = { enabled, destOverride, routeOnly };
```

### 6c. GenOutbounds（核心）

**文件**: `V2rayOutboundService.cs:5-884`

#### 6c.1 总入口（`:5-19`）

```csharp
private void GenOutbounds()
{
    var proxyOutboundList = BuildAllProxyOutbounds();
    _coreConfig.outbounds.InsertRange(0, proxyOutboundList);  // 插入到模板 outbounds 最前面

    // 如果生成了多个 proxy outbound（PolicyGroup 多节点），添加负载均衡
    if (proxyOutboundList.Count(n => n.tag.StartsWith(Global.ProxyTag)) > 1)
    {
        var multipleLoad = _node.GetProtocolExtra().MultipleLoad ?? EMultipleLoad.LeastPing;
        GenObservatory(multipleLoad);  // 健康观测器
        GenBalancer(multipleLoad);     // 均衡器
    }
}
```

#### 6c.2 BuildAllProxyOutbounds（`:21-33`）

```csharp
private List<Outbounds4Ray> BuildAllProxyOutbounds(string baseTagName = Global.ProxyTag)
{
    if (_node.ConfigType.IsGroupType())
        return BuildGroupProxyOutbounds(baseTagName);  // PolicyGroup / ProxyChain
    else
        return [BuildProxyOutbound(baseTagName)];       // 单节点
}
```

#### 6c.3 BuildProxyOutbound（`:51-58`）

```csharp
private Outbounds4Ray BuildProxyOutbound(string baseTagName = Global.ProxyTag)
{
    var txtOutbound = EmbedUtils.GetEmbedText(Global.V2raySampleOutbound);  // 加载 outbound 模板
    var outbound = JsonUtils.Deserialize<Outbounds4Ray>(txtOutbound);
    FillOutbound(outbound);      // 填入协议字段
    outbound.tag = baseTagName;  // "proxy"
    return outbound;
}
```

#### 6c.4 FillOutbound（`:60-290`）— 协议填充

按 `_node.ConfigType` 分发：

| 协议 | 行号 | settings 填充 | 清空 |
|---|---|---|---|
| **VMess** | `:68-109` | `vnext[0].{address, port}` + `users[0].{id, alterId, security, email}` | `servers = null` |
| **Shadowsocks** | `:111-136` | `servers[0].{address, port, password, method, uot}` | `vnext = null` |
| **SOCKS / HTTP** | `:138-172` | `servers[0].{address, port}`，可选 `users[{user, pass}]` | `vnext = null` |
| **VLESS** | `:174-213` | `vnext[0].{address, port}` + `users[0].{id, encryption, flow}` | `servers = null` |
| **Trojan** | `:215-237` | `servers[0].{address, port, password}` | `vnext = null` |
| **Hysteria2** | `:239-249` | `settings.{version=2, address, port}` | `vnext/servers = null` |
| **WireGuard** | `:251-275` | `settings.{secretKey, address, peers[0].{publicKey, endpoint, preSharedKey}}` | `vnext/servers = null` |

最后（`:279-283`）：
```csharp
outbound.protocol = Global.ProtocolTypes[_node.ConfigType];
FillBoundStreamSettings(outbound);  // 传输层 + TLS
```

**FillOutboundMux**（`:292-315`）：
```csharp
if (enabledTCP) {
    mux.enabled = true;
    mux.concurrency = _config.Mux4RayItem.Concurrency;
} else if (enabledUDP) {
    mux.enabled = true;
    mux.xudpConcurrency = _config.Mux4RayItem.XudpConcurrency;
    mux.xudpProxyUDP443 = _config.Mux4RayItem.XudpProxyUDP443;
}
```

#### 6c.5 FillBoundStreamSettings（`:317-672`）— 传输层 + TLS + Reality

**确定 network**（`:322-327`）：
```csharp
var network = _node.GetNetwork();           // ws / tcp/raw / kcp / grpc / xhttp / httpupgrade
if (_node.ConfigType == EConfigType.Hysteria2) network = "hysteria";
streamSettings.network = network;
```

**读取传输参数**（`:328-371`）：
```csharp
var transport = _node.GetTransportExtra();
// 根据 network 读取对应的 Host / Path / headerType / kcpSeed / GrpcServiceName 等
```

**TLS 设置**（`:377-423`）：
```csharp
streamSettings.security = "tls";
tlsSettings = {
    allowInsecure = ...,
    alpn = _node.GetAlpn(),
    fingerprint = ...,
    serverName = sni ?? host,
    echConfigList = ...,
    certificates = CertPemManager.ParsePemChain(_node.Cert),  // 自定义证书
    pinnedPeerCertSha256 = _node.CertSha,
    disableSystemRoot = true    // 当有自定义证书时
}
```

**Reality 设置**（`:427-443`）：
```csharp
streamSettings.security = "reality";
realitySettings = {
    fingerprint = ...,
    serverName = sni,
    publicKey = _node.PublicKey,
    shortId = _node.ShortId,
    spiderX = _node.SpiderX,
    mldsa65Verify = _node.Mldsa65Verify
}
```

**传输层配置**（`:446-661`）：

| network | 行号 | 关键设置 |
|---|---|---|
| **kcp** | `:448-490` | `kcpSettings.{mtu, tti, uplinkCapacity, downlinkCapacity, cwndMultiplier}` + `finalmask.udp[]`（header type + mkcp-original/aes128gcm seed） |
| **ws** | `:492-510` | `wsSettings.{path, host, headers.User-Agent}` |
| **httpupgrade** | `:512-530` | `httpupgradeSettings.{path, host, headers.User-Agent}` |
| **xhttp** | `:532-556` | `xhttpSettings.{path, host, mode, extra}` |
| **grpc** | `:558-571` | `grpcSettings.{authority, serviceName, multiMode, idle_timeout, health_check_timeout, permit_without_stream, initial_windows_size, user_agent}` |
| **hysteria** | `:573-627` | `hysteriaSettings.{version=2, auth=password}` + `finalmask.quicParams.{congestion, brutalUp, brutalDown, udpHop}` + 可选 `finalmask.udp[]`（salamander obfs） |
| **raw/tcp** | `:629-661` | 若 `headerType=http` → `rawSettings.header` + 从模板 `SampleHttpRequest`/`SampleHttpResponse` 读取 HTTP 请求/响应伪装 |

**finalmask 覆盖**（`:663-666`）：
```csharp
if (!_node.Finalmask.IsNullOrEmpty())
    streamSettings.finalmask = JsonUtils.ParseJson(_node.Finalmask);
```

#### 6c.6 Group Outbound

**BuildOutboundsList**（`:674-706`）— `PolicyGroup`：为每个子节点生成独立的 outbound，tag 为 `proxy-{n}-{remarks}`

**BuildChainOutboundsList**（`:708-792`）— `ProxyChain`：反转顺序后，通过 `streamSettings.sockopt.dialerProxy` 串联各节点，形成链式代理

#### 6c.7 BuildDnsOutbound（`:815-819`）

TUN 模式下添加 DNS outbound：`{ "tag": "dns", "protocol": "dns" }`

### 6d. GenRouting

**文件**: `V2rayRoutingService.cs:5-267`

**入口**（`:5-80`）：
```csharp
private void GenRouting()
{
    // :9-34 TUN 模式专用规则：DNS 直连 + 核心进程直连
    if (context.IsTunEnabled) { ... }

    // :37 设置 domainStrategy
    _coreConfig.routing.domainStrategy = _config.RoutingBasicItem.DomainStrategy;

    // :39-61 遍历用户定义的路由规则
    foreach (var item in rules) {
        if (!item.Enabled) continue;
        GenRoutingUserRule(item2);
    }

    // :63-72 将 balancer 应用到对应规则
}
```

**GenRoutingUserRule**（`:82-175`）：
- 将 domain / ip / process 分别拆成独立的 `RulesItem4Ray` 条目
- 如果某个字段为空数组，设为 `null`（避免 Xray 报错）
- 每个条目 `type = "field"`

**GenRoutingUserRuleOutbound**（`:177-209`）：
- 如果 outboundTag 引用的是合法但非内置的节点，递归调用 `BuildAllProxyOutbounds` 生成 outbound 并加入 `_coreConfig.outbounds`

**BuildFinalRule**（`:211-233`）— 兜底规则：
```csharp
{ "type": "field", "network": "tcp,udp", "outboundTag": "proxy" }
// 如果 domainStrategy == IPIfNonMatch:
{ "type": "field", "ip": ["0.0.0.0/0", "::/0"], "outboundTag": "proxy" }
```

### 6e. GenDns

**文件**: `V2rayDnsService.cs:5-472`

**入口**（`:5-109`）：

```
路由 1: RawDnsItem.Enabled == true
    → GenDnsCustom() 使用自定义 DNS JSON
    → 如果 domainStrategy == IPIfNonMatch，添加 DNS 模块路由标签

路由 2: SimpleDNSItem 配置
    → 设置 Freedom outbound 的 domainStrategy
    → 设置代理 outbound 的 targetStrategy
    → FillDnsServers → 将域名分类(直连/代理)分配给不同 DNS server
    → FillDnsHosts → 添加 hosts 映射
    → 添加 DNS 内置路由规则
```

**FillDnsServers**（`:111-331`）— DNS server 详细分配逻辑：
```
remoteDNSAddress + proxyDomains        → DNS server (使用远程 DNS)
directDNSAddress + directDomains       → DNS server (标记为 direct-dns，使用直连 DNS)
remoteDNSAddress + proxyGeosites       → DNS server (远程 + geosite)
directDNSAddress + directGeosites      → DNS server (直连 + geosite)
directDNSAddress + expectedDomains     → DNS server (直连 + 期望 IP)
bootstrapDNS + dnsServerDomains        → 引导 DNS (DNS 服务器的域名)
```

### 6f. GenStatistic

**文件**: `V2rayStatisticService.cs:5-50`

如果启用了「启用统计」或「显示实时速度」（`:7`）：
```csharp
_coreConfig.stats = new Stats4Ray();
_coreConfig.metrics = new Metrics4Ray { tag = "api" };
_coreConfig.policy = new Policy4Ray { system = { statsOutboundUplink = true, statsOutboundDownlink = true } };
// 添加 API inbound (dokodemo-door)
// 添加 API 路由规则
```

### 6g. 后处理

在 `GenerateClientConfigContent`（`:59-64`）中顺序执行：

```csharp
if (_config.CoreBasicItem.EnableFragment)
    ApplyOutboundFragment();         // fragment/noise 分片
ApplyOutboundBindInterface();        // 网络接口绑定
ApplyOutboundSendThrough();          // 源 IP 绑定
```

**ApplyOutboundFragment**（`V2rayOutboundService.cs:821-883`）：
- 对所有启用了 TLS/Reality 的 outbound，在 `finalmask.tcp[]` 添加 fragment 掩码，`finalmask.udp[]` 添加 noise 掩码

**ApplyOutboundBindInterface**（`V2rayConfigTemplateService.cs:137-173`）：
- 配置 `streamSettings.sockopt.Interface = bindInterface`

**ApplyOutboundSendThrough**（`V2rayConfigTemplateService.cs:175-186`）：
- 配置 `outbound.sendThrough = sendThrough`（指定出口 IP）

---

## 7. 模板合并：ApplyFullConfigTemplate

**文件**: `V2rayConfigTemplateService.cs:5-135`

如果用户启用了「自定义全量配置模板」（`:8`）：
1. 解析用户提供的 JSON 模板
2. 合并 balancer（`:26-69`）、observatory（`:71-83`）、burstObservatory（`:85-97`）
3. 将生成的代理 outbound 注入到用户模板的 `outbounds` 数组中（`:99-132`）
4. 如果 `AddProxyOnly == true`，跳过内置的 blackhole/dns/freedom
5. 如果设置了 `ProxyDetour`，为代理 outbound 添加 `dialerProxy` 转发

否则直接 `JsonUtils.Serialize(_coreConfig)`（`:10`）。

---

## 8. 写入磁盘

**文件**: `CoreConfigHandler.cs:36-39`

```csharp
if (fileName.IsNotEmpty() && result.Data != null)
    await File.WriteAllTextAsync(fileName, result.Data.ToString());
```

- 默认路径：`{bin}/config.json`（由 `Utils.GetBinConfigPath("config.json")` 决定）
- 默认 `bin` 路径：`{AppData}/v2rayN/bin/`

---

## 9. 最终输出示例

以 **VMess + WebSocket + TLS** 为例的最终 `config.json`：

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "tag": "socks",
      "port": 10808,
      "listen": "127.0.0.1",
      "protocol": "mixed",
      "settings": { "udp": true },
      "sniffing": { "enabled": true, "destOverride": ["http", "tls"], "routeOnly": false }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess",
      "settings": {
        "vnext": [{
          "address": "server.example.com",
          "port": 443,
          "users": [{ "id": "uuid-here", "alterId": 0, "security": "auto", "email": "t@t.tt" }]
        }]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "serverName": "example.com",
          "allowInsecure": false,
          "alpn": ["h2", "http/1.1"],
          "fingerprint": "chrome"
        },
        "wsSettings": {
          "path": "/websocket",
          "headers": { "Host": "example.com", "User-Agent": "Mozilla/5.0..." }
        }
      },
      "mux": { "enabled": false, "concurrency": -1 }
    },
    { "protocol": "freedom", "tag": "direct" },
    { "protocol": "blackhole", "tag": "block" }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      { "type": "field", "outboundTag": "direct", "domain": ["geosite:cn"] },
      { "type": "field", "outboundTag": "direct", "ip": ["geoip:cn", "geoip:private"] },
      { "type": "field", "network": "tcp,udp", "outboundTag": "proxy" }
    ]
  },
  "dns": {
    "servers": [
      { "address": "1.1.1.1", "port": 53, "skipFallback": true, "domains": ["geosite:google"] },
      { "address": "223.5.5.5", "port": 53, "skipFallback": true, "domains": ["geosite:cn"] }
    ]
  }
}
```

---

## 10. 关键文件索引

### 核心逻辑

| 文件 | 作用 | 关键方法 |
|---|---|---|
| `ServiceLib/Manager/CoreManager.cs` | 触发 config 生成 + 启动核心进程 | `LoadCore(:63)`, `CoreStart(:176)`, `CoreStartPreService(:191)` |
| `ServiceLib/Handler/CoreConfigHandler.cs` | 生成 + 写盘分发 | `GenerateClientConfig(:10)` |
| `ServiceLib/Handler/Builder/CoreConfigContextBuilder.cs` | 构建上下文、路由引用、节点校验 | `Build(:34)`, `ResolveNodeAsync(:158)`, `RegisterSingleNodeAsync(:274)` |
| `ServiceLib/Services/CoreConfig/V2ray/CoreConfigV2rayService.cs` | 主控：模板→Gen*序列→合并 | `GenerateClientConfigContent(:13)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayOutboundService.cs` | outbound 协议填充 + streamSettings | `GenOutbounds(:5)`, `BuildProxyOutbound(:51)`, `FillOutbound(:60)`, `FillBoundStreamSettings(:317)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayRoutingService.cs` | 路由规则生成 | `GenRouting(:5)`, `GenRoutingUserRule(:82)`, `BuildFinalRule(:211)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayInboundService.cs` | inbound 生成（SOCKS/HTTP/TUN） | `GenInbounds(:5)`, `BuildInbound(:78)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayDnsService.cs` | DNS 配置生成 | `GenDns(:5)`, `FillDnsServers(:111)`, `FillDnsHosts(:333)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayLogService.cs` | 日志配置 | `GenLog(:5)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayStatisticService.cs` | 流量统计配置 | `GenStatistic(:5)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayBalancerService.cs` | 负载均衡 observatory + balancer | `GenObservatory(:5)`, `GenBalancer(:84)` |
| `ServiceLib/Services/CoreConfig/V2ray/V2rayConfigTemplateService.cs` | 用户自定义模板合并 + bindInterface + sendThrough | `ApplyFullConfigTemplate(:5)`, `ApplyOutboundBindInterface(:137)`, `ApplyOutboundSendThrough(:175)` |

### 数据模型

| 文件 | 作用 |
|---|---|
| `ServiceLib/Models/CoreConfigs/V2rayConfig.cs` | 所有 config.json 的 C# 数据模型（551行） |
| `ServiceLib/Models/Entities/ProfileItem.cs` | 节点/服务器数据实体 |
| `ServiceLib/Models/Entities/ProtocolExtraItem.cs` | 协议额外参数（AlterId/Flow/SsMethod 等） |
| `ServiceLib/Models/Entities/TransportExtraItem.cs` | 传输层额外参数（Host/Path/GrpcServiceName 等） |
| `ServiceLib/Models/CoreConfigs/CoreConfigContext.cs` | Config 生成上下文 |

### 模板文件

| 文件 | 作用 |
|---|---|
| `ServiceLib/Sample/SampleClientConfig` | 默认 client 配置 JSON 模板 |
| `ServiceLib/Sample/SampleOutbound` | 默认 outbound 配置 JSON 模板 |
| `ServiceLib/Sample/SampleInbound` | 默认 inbound 配置 JSON 模板 |
| `ServiceLib/Sample/SampleTunInbound` | TUN 模式 inbound 模板 |
| `ServiceLib/Sample/SampleTunRules` | TUN 模式路由规则模板 |
| `ServiceLib/Sample/SampleHttpRequest` | TCP 伪装 HTTP 请求模板 |
| `ServiceLib/Sample/SampleHttpResponse` | TCP 伪装 HTTP 响应模板 |

### 配置常量

| 文件 | 作用 |
|---|---|
| `ServiceLib/Global.cs` | 全局常量: 模板路径、协议映射、默认值 |
| `ServiceLib/Enums/EConfigType.cs` | 协议类型枚举 |

### 订阅解析

| 文件 | 作用 |
|---|---|
| `ServiceLib/Handler/SubscriptionHandler.cs` | 订阅更新总控 |
| `ServiceLib/Handler/ConfigHandler.cs` | `AddBatchServers` 批量解析订阅内容 |
| `ServiceLib/Handler/Fmt/FmtHandler.cs` | URI 路由派发 |
| `ServiceLib/Handler/Fmt/VmessFmt.cs` | VMess:// 解析 |
| `ServiceLib/Handler/Fmt/VLESSFmt.cs` | VLESS:// 解析 |
| `ServiceLib/Handler/Fmt/ShadowsocksFmt.cs` | SS:// 解析 |
| `ServiceLib/Handler/Fmt/TrojanFmt.cs` | Trojan:// 解析 |
| `ServiceLib/Handler/Fmt/Hysteria2Fmt.cs` | Hysteria2:// 解析 |
| `ServiceLib/Handler/Fmt/TuicFmt.cs` | TUIC:// 解析 |
| `ServiceLib/Handler/Fmt/WireguardFmt.cs` | WireGuard:// 解析 |
| `ServiceLib/Handler/Fmt/ClashFmt.cs` | Clash YAML 解析 |
| `ServiceLib/Handler/Fmt/V2rayFmt.cs` | V2ray JSON 解析 |
| `ServiceLib/Handler/Fmt/SingboxFmt.cs` | sing-box JSON 解析 |
| `ServiceLib/Handler/Fmt/InnerFmt.cs` | v2rayn:// 内部 URI 解析 |
| `ServiceLib/Services/DownloadService.cs` | HTTP 下载订阅内容 |
