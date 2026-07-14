# Day 25 · 安全 —— authn vs authz、OAuth2/OIDC、密钥与加密

> 对应文档：`Day-25-security-authn-authz.html`
> 日期：2026-06-25

## 今日思考题

> 文档结尾「带着这两个问题回来讨论」。先写下自己的初步想法，再来 Claude Code 聊；聊完我会在每条下面补点评（像 Day 01–24 那样）。

1. QuoteFlow 后台你会先用 RBAC 还是 ABAC？给「pricing 工程师」「broker」「核保员」「审计员」各自划一条最小权限的边界，哪里最容易划过头（给多了）？

   - 我的想法：我会先用RBAC，因为简单+好审计，有单独需要特殊权限的内容，可以用ABAC补全。对哪里容易划过头的问题，pricing工程师容易多分给修改用户数据的权限，实际上他们只需要能修改费率表，重跑定价流程的权限，或是根据已有数据去验证定价模型正确与否，但不应该能修改他们使用的用户的数据（有些PII数据甚至应该只能看到mask过的信息，而不是原始信息）。Broker容易划多给可以看别的broker的保险和保费信息，每个broker应该只能看自己的。核保员容易划多的就是允许看定价信息，因为核保其实只需要看用户信息来确定风险等问题，不需要看具体定价，导致定价信息影响决策。审计员容易划多的就是给他们写入的权限，他们只需要读，不需要也不应该有写权限（比如为了审计成功去修改记录）。

   - **点评（答得很实战，补三刀）：** RBAC 打底 + ABAC 补充 ✅；四个角色的过界点抓得都对。

     ⚠ **但你自己的 broker 例子，其实推翻了一半的 RBAC——这点你没意识到，很关键**：「每个 broker 只能看**自己的**保单」——**RBAC 表达不了这句话**。RBAC 只能说「broker 这个角色能读 policy」，它没有任何位置去写「自己的」。这是**行级（row-level）**限制，条件形如 `policy.broker_id == user.broker_id` → 天然是 **ABAC**（也叫 ReBAC / 多租户隔离）。所以你的结论可以更有力：**「RBAC 定动词（能不能读 policy），ABAC 定范围（能读哪些 policy）」**——broker 那条就是最典型的证据。

     **核保员那条——理由很精算，但要 nuance：** 你的理由是「定价信息会影响核保决策」，这不是安全理由，是 **segregation of duties（职责分离）+ 避免 bias**，直觉很好。但现实里核保员往往需要看到 loading %（才能做 counter-offer）。更精确的划法：**看得到风险评分和 loading，看不到费率表本身和定价模型内部**。而核保员真正最容易划过头的是——**能看全部 case，而不是「分配给他的 case」**（又是一个行级限制）。

     **审计员那条——「只读」还有两个更深的坑：** ①「审计员能读**全部数据（含原始 PII）**」这件事本身就是巨大的权限 → 最小权限是常态只读 audit log + 元数据，要看原始 PII 走 **break-glass（临时提权，且这次提权本身也被审计）**。②「不能有写权限」要延伸到——**连审计员也不能改 audit log 本身** → audit log 必须 **append-only / 不可变**（直接接 Day 24）。

     **你漏掉的、现实中最大的一个「划过头」：不是人，是 service account。** 夜间批量重定价那个 job，最常见的偷懒就是直接给它 DB admin。人的权限大家会认真评审，**机器的权限没人管**——这是真实世界里权限泄漏的头号入口。附带一条：权限最容易失控的时刻是「**生产救火时临时给的权限没收回**」→ 所以要有定期 **access review / recertification**。

2. 「JWT 无需回查、扩展性好」和「audit 要求每一次敏感访问都留痕、且能即时封停某个被盗身份」——这两个诉求冲突吗？你会怎么设计 token 生命周期 + 撤销机制，既快又可控？

   - 我的想法：不冲突，只要“即时”不意味着当下立刻封停，audit的要求就能和JWT共存。首先每次敏感访问都留痕是信息记录机制，跟JWT无需回查可以共存，只要在audit.log里如实记录访问信息就好。对于被盗的情况，这个需要用到文档里提到的短命access token和长命refresh token的设计，把access token的生命周期设的很短，例如只有五分钟。然后refresh token可以设相对长命。这样在正常情况下，access token短期expire之后，去auth server重新验证refresh token，如果存在就签发新的access token。如果发现出了问题，那么立刻切断当前的refresh token，这样出问题的client端就只有短命token到expire之前剩余的这些时间保留access，expire之后立马失去权限。如果真的有必要即时撤销，还可以引入黑名单/网关强制回查，但这样就丢失了JWT的性能优势。

   - **点评（骨架基本是标准答案，补两个缺口）：** 题眼你抓住了——「只要『即时』不意味着**当下立刻**封停，两者就不冲突」✅。**撤销延迟 = access token 的剩余寿命**，这个取舍说清楚了；黑名单 / 网关强制回查作为「代价是丢掉 JWT 性能优势」的兜底，也对。

     **缺口 1（留痕那半答得太轻）：** 要说清**为什么**正交——**留痕发生在资源服务器 / 网关侧**：它验完签名后把 `sub` / `scope` / `trace_id` 写进 audit log，**压根不需要回 Auth Server**。⚠ 但这里有个坑：**JWT 里的 claims 是签发那一刻的快照**。若 audit log 只记「role=pricing_engineer」这个字符串，而该 role 15 分钟前刚被撤销，你记下的就是一份**过期的权限视图**。所以 audit 要记 **`jti`（JWT ID）+ `sub` + 签发时间**——事后才能对上「这次访问用的是哪一个 token、那个 token 当时凭什么被签出来」。

     **缺口 2（撤销的前提是「知道被盗了」）：** 你写的是「**如果发现出了问题**，就切断 refresh token」——但**怎么发现**？这就是 **refresh token rotation + reuse detection**：旧 refresh token 用一次即废，一旦一个**已作废的** refresh token 又被使用 → 只可能是被盗 → Auth Server **自动**把整条 **token family** 全部吊销。这是撤销机制自带的「**检测**」那一半。

     **两个措辞：** 吊销时要吊销该用户的**所有 token family / session**，不只当前那一个；「退休 refresh token」→ 标准词是 **revoke（吊销）**。

## 提问 & 解答

> 讨论时冒出来的问题记这。每条：先记自己的问题，再记讨论后的结论（用自己能看懂的话）。

### Q1. PKCE 是什么？SPA 又是什么？

**问：** 文档说「配合 PKCE 用于 App / SPA」——这两个缩写是什么？

**答：**

**SPA = Single Page Application（单页应用）**：React / Vue 那类前端应用，浏览器只加载一次 HTML 外壳，之后页面切换全在前端 JS 里渲染，数据靠调 API。关键性质：**代码全在用户浏览器里，F12 就能看**。所以它和手机 App 一样属于 **public client（公开客户端）——没有任何地方能藏住 `client_secret`**。相对的，有服务端的传统 Web 应用是 **confidential client**（secret 存服务器上）。

**PKCE = Proof Key for Code Exchange**（念 "pixy"）：给授权码流程加的一道保险，**本质是「每次登录临时造一个一次性的 client_secret」**。

它要解决的问题：换 token 那一步本来靠 `client_secret` 证明「我是合法客户端」，可 public client 没有 secret → 变成「**光有 code 就能换 token**」→ code 一旦在回跳途中被截走（恶意 App 抢注同一 URL scheme、referer / history 泄漏），攻击者直接换到 token。

做法：

```
1. App 发起登录前，本地随机生成 code_verifier（只留在自己内存里）
2. 算 code_challenge = SHA256(code_verifier)，把 challenge（哈希，不是原文）随授权请求发出去
3. Auth Server 记下 challenge，正常发回 authorization code
4. App 拿 code 换 token 时，附上原始 code_verifier
5. Auth Server 验 SHA256(code_verifier) == 存下的 challenge？相等才发 token
```

为什么安全：攻击者截到的只有 `code` 和 `code_challenge`（哈希）。**`code_verifier` 的原文从头到尾只出现在最后那次换 token 的请求里**，哈希又不可逆 → 他倒推不出 verifier，拿着 code 也换不到 token。

**QuoteFlow 落地：** broker 的服务端后台 = confidential client（可以用 `client_secret`）；QuoteFlow 的 iOS App / 消费者投保 SPA / 内部 React 控制台 = public client → 一律 **授权码 + PKCE**。

> 行业方向：**OAuth 2.1 要求所有客户端都用 PKCE**（连 confidential client 也要——它防的是 code 截获，和有没有 secret 是两层保护）。旧的 **implicit flow**（token 直接从 URL fragment 返回给 SPA）已正式废弃。

### Q2. 授权码流程里的 ⑤ 和 ⑥ 到底在干什么？

**问：** 图里第 5 步（回跳带 authorization code）和第 6 步（后端用 code 换 token）能展开讲讲吗？

**答：** 先立一个概念，⑤ ⑥ 必须分成两步的**全部理由**都在这：

| | 走哪条路 | 可信吗 |
|---|---|---|
| **Front channel（前端通道）** | Auth Server ↔ **用户浏览器** ↔ App（302 重定向、参数在 URL 里） | ❌ 不可信：进浏览器历史、可能被 referer / 恶意扩展 / 抢注 scheme 的 App 看到 |
| **Back channel（后端通道）** | App 后端 ↔ Auth Server（服务器直连、TLS） | ✅ 可信：不经过用户设备 |

**⑤ 走 front channel，⑥ 走 back channel。**

**⑤ 回跳并带 authorization code**——Auth Server 不是把 code 直接送到后端，而是给**浏览器**一个 302：

```http
HTTP/1.1 302 Found
Location: https://app.quoteflow.example/callback?code=SplxlOBeZQQ...&state=xyz123
```

所以 code 是**经用户浏览器「手递手」传过来的**。正因为这条路不可信，这里只能传「**半张票**」——authorization code 有四条性质让它敢走这条脏路：

- **短命**（30 秒 ~ 10 分钟）
- **一次性**（换过即废；同一 code 被用第二次 → 强烈暗示被截获重放 → 好的 Auth Server 会把已签发的 token 一并吊销）
- **绑死**（绑 `client_id`、`redirect_uri`、有 PKCE 时绑 `code_challenge`）
- **本身没有任何权限**（拿 code 去调 `/quotes` 是废纸，它只能在 token endpoint 换票）

顺带：图里没画的 `state` —— ② 发出去时带一个随机串，⑤ 必须原样带回且和会话里的一致 → 防 **CSRF / code injection**（防止攻击者把**他自己的** code 塞进你的回调，把你的账号绑到他身上）。

**⑥ 后端用 code 换 token**——QuoteFlow 后端**自己**向 token endpoint 发直连 POST，**完全不经浏览器**：

```http
POST /oauth2/token
grant_type=authorization_code
&code=SplxlOBeZQQ...
&redirect_uri=https://app.quoteflow.example/callback
&client_id=quoteflow-web
&client_secret=<只有服务器知道>     ← confidential client 走这条
&code_verifier=<PKCE 原文>          ← public client (App/SPA) 走这条
```

Auth Server 逐条校验：① code 存在、没过期、**没被用过**；② 绑的 `client_id` 一致；③ `redirect_uri` 与授权时**完全一致**（防 code 被诱导换到攻击者的回调）；④ `client_secret` 对得上 —— 或 `SHA256(code_verifier) == code_challenge`。全过才发 token。

**为什么非拆两步？多一次往返换了三样东西：**

1. **token 永不出现在浏览器 URL 里**——值钱的东西只走 back channel。
2. **多一道「客户端身份证明」**——⑤ 只证明了「**用户同意了**」；⑥ 才证明「**来换票的确实是 QuoteFlow 本人**」。**这就是为什么攻击者光截到 code 也换不到 token**：他过不了第 ④ 关。
3. **refresh_token 有了安全的发放渠道**——它更值钱，只能在 back channel 给。

> 被废弃的 **implicit flow** 干的正是「省掉 ⑥，在 ⑤ 就把 token 塞进 URL fragment」：省一个往返，代价是 token 暴露在 front channel + 完全无法证明客户端身份。

**一句话记：** ⑤ 是「经浏览器递过来的一张**一次性、限时、绑死的取票凭证**」；⑥ 是「后端**亲自跑一趟柜台、出示自己的身份证**，把凭证换成真票」。

### Q3. refresh token 的换发规则——它是「变成」新的 access token 吗？

**问：** refresh token 是不是用它去替换原本的短命 JWT，然后它自己变成新的短命 JWT，应用再在后台生成一个新的长命 refresh token？

**答：** 有**两个错位**：

- ❌ **refresh token 不会「变成」access token。** 它不是原料，是**凭证**——你拿它去 Auth Server **换回一个全新的** access token，它自己不被消耗成 token。
- ❌ **应用（Client）永远不能自己生成任何 token。** 它没有 Auth Server 的私钥，签不出合法 JWT。**所有 token 一律由 Auth Server 签发。**

**实际是这样：** ⑦ 那一步 Auth Server 一次性发回两个东西——

```json
{
  "access_token":  "eyJhbGci...",   // JWT, 短命 (如 15 分钟)
  "refresh_token": "v1.MRq8...",    // 通常是不透明随机串, 长命 (如 30 天)
  "expires_in": 900
}
```

**两者分工是硬性的：**

| | 发给谁 | 每次用要回查吗 |
|---|---|---|
| **access_token** | 每次调 API 时发给**资源服务器**（`Authorization: Bearer …`） | ❌ 不用——验签就行（JWT 快的原因） |
| **refresh_token** | **只发给 Auth Server 的 token endpoint**，从不发给业务 API | ✅ **每次必须回 Auth Server**（有状态、存在库里） |

access token 过期（API 返 401）→ 客户端走 back channel `grant_type=refresh_token` → Auth Server **回查**（这个 refresh token 还有效吗？用户还在职吗？角色变了吗？）→ 签发**新的 access_token**。用户无感（静默换发）。

**我直觉里那个「新的长命 refresh token」——确实存在，叫 Refresh Token Rotation（轮换）：** 每次换发时 Auth Server **连带发一个新的 refresh token，旧的立刻作废**（生成者是 Auth Server，不是应用）。它的杀手锏是 **reuse detection**：既然旧的用一次即废，**一个已作废的 refresh token 又被使用 = 只可能是被偷了** → Auth Server 把整条 **token family** 全部吊销，强制重新登录。对 SPA / App 这种 public client（refresh token 只能放浏览器/设备里），rotation + reuse detection 基本是必须的。

**为什么这套设计解掉了「JWT 撤销不了」的死结**（= 思考题 ② 的骨架）：

- access token：**短命 + 无需回查** → 快、可扩展，但撤销不了
- refresh token：**长命，但每次使用都必须回 Auth Server** → **撤销的钩子就挂在这一跳上**

所以「即时封停一个被盗身份」= **吊销他的 refresh token**。他手上那个 access token 最多再撑 15 分钟，之后再也换不到新的。

> **撤销延迟 = access token 的剩余寿命。** 这才是「短命 access token」的真正意义——**它不是为了防泄漏，是为了把撤销的空窗压到可接受**。

### Q4. JWT 的「自包含」是什么意思？

**问：** 说 JWT 只是 token 的一种「格式」（自包含 + 带签名）——自包含指什么？

**答：** **自包含（self-contained）= token 自己就装着「判断所需的全部信息」，验证方不用再去问任何人。**

| | 长什么样 | 验证方要做什么 |
|---|---|---|
| **Opaque token（不透明令牌）** | `v1.MRq8xK2p...` 随机串，**本身没有信息**，只是数据库里一行的**指针** | **必须回查**：问 Auth Server「这串是谁的？有什么权限？还有效吗？」 |
| **JWT（自包含）** | `eyJhbGci...` Base64 解开就是一段 JSON，**信息写在里面** | **只需验签名**：用公钥确认「没被改过、确实是 Auth Server 签的」，然后直接读 |

**比喻：** Opaque token = 寄存处的**号码牌「37 号」**（牌子上什么都看不出来，必须回柜台问）；JWT = 一张**写满字、盖了骑缝章的通行证**（门卫只要确认章是真的，当场放行，不用打电话回总部）。

payload 解开长这样：

```json
{
  "iss": "https://auth.quoteflow.example",  // 谁签的
  "sub": "u_8213",                          // 谁
  "aud": "quoteflow-api",                   // 给谁用的
  "exp": 1751002500,                        // 什么时候过期
  "jti": "t_9f2c4a...",                     // 这个 token 的唯一 ID
  "scope": "quotes:read rates:write",       // 能干什么
  "roles": ["pricing_engineer"]
}
```

网关要做的全部事：**Base64 解开 + 公钥验签 + 检查 `exp`/`aud`/`iss`** —— 全在本地内存完成，**0 次网络调用、0 次 DB 查询**。

> 所以「**无需回查**」不是 JWT 的另一个独立优点，**它是「自包含」的直接推论**：信息已经在手里了，还查什么？

**自包含的三个代价（同一枚硬币的背面）：**

1. **信息是「签发那一刻的快照」** → 15 分钟内角色被撤了，token 里还写着旧权限。**这就是「JWT 难以即时撤销」的根源**——想即时就得回查，一回查就退化成 opaque token。
2. **payload 明文可读**（只是 Base64，**不是加密**）→ 别塞密码、完整 PII（「签名 ≠ 加密」那条陷阱）。
3. **会变胖**——权限越细 token 越大，而它**每个请求都要带**（HTTP header 有几 KB 上限）。

> 第 3 点顺带回答了一个设计问题：**为什么 ABAC 那种细粒度规则不该塞进 JWT。** JWT 里只放**粗粒度属性**（`roles` / `scope` / `broker_id`）；「这个 broker 能不能看**这一张**保单」是资源服务器拿着 `broker_id` **当场判断**的事，规则本身留在服务端。

**一句话记：opaque token 是「问题的编号」（答案在服务器上）；JWT 是「答案本身」（只需验一下章是不是真的）。**

## 我的理解（用自己的话复述今天的核心）

> 不看文档，凭记忆写 2–4 句。能写清楚 = 真的懂了。

- 网络安全看重两个概念：authentication和authorization。authn负责“你是谁”，是优先被验证的东西，之后authz负责验证这个身份的用户有权限做哪些事。像OAuth2就属于授权委托框架，是authz的范畴，而OIDC就是搭在OAuth2上的身份验证功能。

- 身份验证通过JWT来自动更新access token，并通过短命access token和长命refresh token的机制来享受JWT带来的免重复登录验证的好处，同时保证出问题的access可以通过退休对应refresh token的方式被及时撤销。而授权要走一个授权码流程，保证用户密码不被share给第三方服务的同时，可以让应用代用户去call第三方的服务。

- Access Control有两种方式，一种是Role Based，根据给定的角色分配权限，优点是简单，易审计；另一种是Attribute Based，好处是可以给出更细致的规则去分配权限（比如地理位置，所在时间的要求），但规则会变得很复杂。实际应用当中一般先做RBAC，然后再用ABAC去补充一些更具体的权限分配要求。

- 对于网络安全来说，要保证密钥永远不出现在代码和git里，要用专门的KMS，secret manager去保存密钥信息，并且仅在运行时按需拉取。再就是数据加密要在in transit和at rest时都做，有敏感信息的话，还要有额外的字段级加密。

> **点评：** 第 1、3、4 段（authn/authz 的先后、OAuth2 是授权委托框架 + OIDC 架在其上、RBAC/ABAC 的取舍与「先 RBAC 再 ABAC 补」、KMS/secrets manager + in-transit/at-rest/字段级加密）——**全对，不改**。
>
> ⚠ **第 2 段有一处概念错位**：你隐含建立了 **JWT ↔ authn**、**授权码流程 ↔ authz** 的对应，两条都不成立：
>
> | 你的映射 | 实际是什么 |
> |---|---|
> | ~~JWT = 身份验证机制~~ | **JWT 只是 token 的一种「格式」**（自包含 + 带签名）。`access_token` 和 `id_token` **都可以**是 JWT。它是**编码方式**，不是 authn。 |
> | ~~授权码流程 = 授权(authz)专用~~ | **授权码流程 = 「怎么安全地拿到 token」的流程**。它一趟**同时**返回 `access_token`（authz 的产物）**和** `id_token`（authn 的产物，来自 OIDC）。它两边都服务。 |
>
> **四件事各归其位：授权码流程 = 怎么安全地拿到 token；JWT = token 的一种格式；id_token = authn 的产物（你是谁）；access_token = authz 的产物（你能干什么）。**
>
> 另一处措辞：「JWT 带来的**免重复登录验证**的好处」——JWT 真正省掉的是「**每次请求回查 session 库**」（stateless 验证）；「免重复登录」是 session / refresh token 的功劳，两回事。
>
> （已收进 `confusable-pairs.html`，「JWT vs OAuth2」和「id_token vs access_token」两对标 ⚠「你栽过」。）

## 待深挖 / 存疑

> 还没想透、或想之后回头补的点。

- [ ] **Day 26（部署与发布）**：密钥/证书怎么随 CI-CD 流水线安全地注入到容器里（secret manager vs 环境变量 vs 挂载），轮换（rotation）时怎么不停机。
- [ ] **Day 27（可靠性工程）**：认证/授权服务本身就是 single point of failure——IdP 挂了整个平台是不是全体登不进？降级策略是什么。
- [ ] **Day 28（架构决策 ADR）**：「token 有效期定多久」「RBAC 还是 ABAC」都是典型的取舍型 ADR——安全 vs 可用性 vs 复杂度。
- [ ] **Day 30（毕业作业）**：端到端设计 QuoteFlow 时，authn/authz 这条线（broker API 用什么、内部服务间怎么互信、审计员的只读边界）要能一口气讲清。
- [ ] **service account 的最小权限**（今天点评里补的洞）：夜间批量那个 job 该拿什么权限？「机器身份」怎么认证（mTLS 证书？短命 workload token？）、怎么轮换、怎么审计——人的权限有人评审，机器的没人管。
- [ ] **break-glass（紧急提权）怎么落地**：临时给审计员/救火工程师提权，本身要被审计、要自动到期回收。和「定期 access review / recertification」一起，是最容易被忘掉的一环。

## 一句话串联

> 今天的内容和我的 pricing engine / 之前学过的哪天，能怎么联系起来？

- 今天内容主要分析了QuoteFlow在设计时需要考虑的安全问题，如何进行身份验证和权限分配，这个内容相对独立，但是也和auditability有关系，因为对于寿险来说，什么身份做了什么事，也需要被audit log记录下来，成为审计记录的一部分。

> **点评 + 补全（「相对独立」这句要反驳——安全是这 25 天里连接度最高的一天之一）：**
>
> - **接 Day 23（API 网关 / Service Mesh）——最直接的接续**：Day 23 说「网关集中做认证、鉴权」，今天填上了内容——**网关就是那个用公钥验 JWT 签名的地方**（在边缘验一次，内部服务信任它）。而 service mesh 的 **mTLS 就是「服务之间」的 authn**——**人有 JWT，服务有证书**，同一个「先证明你是谁」的问题，两种身份载体。
>
> - **接 Day 24（可观测性）——你只写到这一条，但可以更锋利**：Day 24 记录「**发生了什么**」，Day 25 决定「**谁被允许让它发生**」。同一条 structured log 里，`trace_id` 来自 Day 23、`sub` / `jti` 来自 Day 25——**合起来才是一条合规的 audit 记录**。
>
> - **接 Day 5（API 设计）**：Bearer token / API key / OAuth 本来就是 **API 契约的一部分**——「broker API 用哪种认证」，是 Day 5 那张设计表上没填的一格。
>
> - **接 Day 12 / Day 16（append-only 日志、不可变存储）**：「连审计员也不能改 audit log」= append-only、不可篡改——今天给了它一个**授权层面的理由**（不是「技术上做不到改」，而是「授权上就不该有人能改」）。
>
> - **接你的 pricing engine**：reproducibility 要回答「**用哪版费率、算出什么、谁在什么时候触发的**」——前两个是 Day 24 的版本指针，**最后那个「谁」，只有 authn 能给**。少了 Day 25，你的审计记录里**永远缺一列主语**。
