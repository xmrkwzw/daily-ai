# 即梦 (Jimeng/Dreamina) API 池化架构深度拆解与防封指南

## 第一章：核心黑盒协议穿透原理解析
`jimeng-free-api` 这个容器运行时的背后，不仅是一个简单的代理，而是一个**“双向协议适配器” (Bi-Directional Protocol Adapter)**。

### 1.1 接口报文的动态映射 (Payload Mapping)
当客户端发出的是 OpenAI 格式的请求时：
```json
// 进入网关的 OpenAI 格式
{
  "model": "jimeng-video-s2.0",
  "messages": [{"role": "user", "content": "赛博城市"}]
}
```
**底层转换逻辑：**
网关接收后，会进行结构体拆包。它会在内部硬编码一组**即梦原生 API 的映射表**：
- 它会将 `model: jimeng-video-s2.0` 解析为请求视频生成通道。
- 提取 `messages` 中的 `content`，重组为字节系接口所需的 Payload (例如：`{"prompt": "赛博城市", "style_id": "xxx", "aspect_ratio": "16:9"}`)。

### 1.2 鉴权与防刷体系欺骗 (Auth & Anti-Bot Spoofing)
单纯发送 JSON 是不够的，大厂网关 (WAF) 默认拦截非浏览器流量。网关必须在 HTTP 层面实行大面积伪装：
- **剥夺并替换请求头**：截取您提供的 `sessionid`，将其强行注入到 `Cookie` 键值中，并配对硬编码的 `User-Agent`、`Referer: https://dreamina.capcut.com/`。
- **签名跳过**：网页版有复杂动态 JS 算出来的验证头（如 `a_bogus`）。通常在旧版接口或部分特定接口（如移动端/海外版）存在降级漏洞，API 网关通常利用这些未及时更新的**降级接口**来发包，从而绕过前端 JS 复杂的环境校验。

---

## 第二章：多账号无缝并发引擎体系 (Token WRR Polling)
解决“无限账号”的底层机制是**全异步配额隔离架构**。

### 2.1 状态转移矩阵构建
网关在内存中维护了一张并发路由表。当传入 `Auth: Bearer session1,session2` 时：
1. **健康预检 (Pre-flight Check)**：系统初始化，将 1 号和 2 号压入**可用区 (Active Pool)**。
2. **算力扣除调度 (Quota Deduction)**：当执行生图时（每次耗费约 3 积分），调度器会根据 `Round-Robin`（轮询）选出 `session1`。
3. **死锁与熔断处理 (Circuit Breaker)**：
   - 场景 A：一旦 `session1` 请求得到响应 `403 Quota Exceeded` 或 `响应体包含 '余额不足'`。
   - 动作 A：该线程立刻暂停，将 `session1` 从Active池划入**阵亡区 (Dead Pool)**。
   - 动作 B：当前阻塞的生图请求，无痕重派至 `session2`。此时上方的外部脚本（如 n8n）仍然感受不到任何封号，只觉得接口“反应稍慢了半秒”。

### 2.2 防崩盘隔离策略 (Bulkhead Pattern)
如果两个外接脚本并发请求生图，网关会通过**锁机制 (Mutex Lock)**或**通道 (Channel)**确保同一瞬间，同一个 `session` 不会承担并发任务（以防单账号请求过速触发“极速反爬雪崩”），实现高仿人工频率的操作。

---

## 第三章：工业界级防封与自动化深度实践 (手把手终极版)

如果你真要将其用于微缩规模の全天候生产环境，纯靠 `jimeng-free-api` 是扛不住风控清洗的，下面是完善的高维架构。

### 阶段一：抛弃手工，Headless 云端无头抓取
不要人工复制 `sessionid`了，使用 Python + Playwright 搭建验证码破壁机。
```python
# 架构层级自动化片段：无痕提取
from playwright.sync_api import sync_playwright

def get_fresh_session():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True, proxy={"server": "http://动态高匿住宅IP:端口"})
        context = browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...",
            locale="zh-CN"
        )
        page = context.new_page()
        page.goto("https://dreamina.capcut.com/...")
        # (介入打码平台API跨越滑块验证)
        cookies = context.cookies()
        for cookie in cookies:
            if cookie['name'] == 'sessionid':
                return cookie['value']
```
*这一步的核心：自动扩充“无限”的来源底座。*

### 阶段二：前置流量清洗器 (Traffic Proxy)
在你的 Docker API 下游，再叠一重真实 IP 保护模块。
如果 `jimeng` 服务端发现这 10 个号都在通过你家路由器的 IP 疯狂出图，秒级会把你的出口 IP 封停。
解决方案是通过 `Clash` / `V2ray` 透明代理接管 Docker 的出口流量网，赋予每个 `sessionid` 请求携带不同的底层网络路由。

### 阶段三：低代码编排流 (n8n 高级实战配置)
以搭建一个“自动化自媒体配图机器人”为例：
1. **Trigger**: 定时器每两小时拉取一次 RSS 科技新闻。
2. **Text-Process**: 用大语言模型将新闻核心提取成**一句话 Prompt**。
3. **HTTP-Request (核心对接区)**：
   - Method: POST
   - URL: `http://127.0.0.1:8001/v1/chat/completions`
   - Authentication: 选择 `Generic Credential Type`, 设置好 `Header Auth` -> `Bearer sess1,sess2...`
   - Body Parameters -> JSON 构建 `model` 与 `messages` 结构体。
4. **Download File Node**: 解析返回体中的 `["choices"][0]["message"]["content"]`，匹配其中的 Markdown 图片 URL，自动下载到云盘。

---

## 4. 【深度诊断规程】全能自动化中台架构与风控演进

| 当前技术现状与痛点 | 顶瞻架构级应对策略 (优化建议) | 最终可测算的工程收益 |
| :--- | :--- | :--- |
| **现状痛点：**<br>目前依赖的单机镜像网关请求过于透明。即梦一旦在服务端引入严格验证（如强制验证每一次请求的 Canvas 显卡绘图渲染指纹差异），将直接切断这种低级的纯接口转发请求链。 | **1. AST(抽象语法树) 签名服务器**：分离发包层与加解密层。使用纯 JS 挂载微服务，实时执行官方下发的最新加解密反爬脚本。<br>**2. 软解 VMP(虚拟壳)**：对于最高风控的 App 端抓包协议，利用 Android Unidbg 技术伪装 ARM 寄存器真实跑出 `libcms.so` 的风控头。<br>**3. 流水线云控群控**：告别单纯的 API 调用，采用 Android 集群采用按键精灵方案走物理设备触控。 | 彻底免疫常规的封号锁链。即使是最严苛的风控体系迭代，核心服务网关依然能够快速响应自适应。<br>真正实现无死角的“无限配额白嫖”。 |

```kotlin
// Android/KMM 架构解析：模拟大型端云协同的号池风控过滤架构
// 避免因某一个账号发包时被测出“模拟器”，导致连带账号网段全部熔断
suspend fun allocateSafeTraffic(sessionList: List<String>) {
    supervisorScope {
        sessionList.forEach { session ->
            launch(Dispatchers.IO) {
                // 执行端侧风控模拟，产生合法运行环境上下文
                val mockContext = DeviceFingerprintMockEngine.generateRandomHardware()
                
                // 将上下文同 Token 一同入池监控
                TokenPoolOrchestrator.registerTokenWithContext(session, mockContext)
            }
        }
    }
}
```
