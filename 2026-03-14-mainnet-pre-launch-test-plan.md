# SuperGuild 主网上线前完整测试方案

> **版本**: v1.0
> **日期**: 2026-03-14
> **适用阶段**: Phase 11.5 完成后 → 主网部署前
> **环境**: Arbitrum Sepolia 测试网（代码逻辑等价于主网）
> **目标**: 全系统功能验证 + 安全边界验证 + 主网清单核对

---

## 目录

1. [测试前准备](#一测试前准备)
2. [测试账户角色矩阵](#二测试账户角色矩阵)
3. [模块 A — 身份认证 & JWT Auth 流程](#模块-a--身份认证--jwt-auth-流程)
4. [模块 B — NFT 门控安全边界](#模块-b--nft-门控安全边界)
5. [模块 C — 协作全链路（自行管理模式）](#模块-c--协作全链路自行管理模式)
6. [模块 D — DirectPay 链上支付结算](#模块-d--directpay-链上支付结算)
7. [模块 E — VCP 积分 & 反作弊系统](#模块-e--vcp-积分--反作弊系统)
8. [模块 F — Resolver Bot 验证](#模块-f--resolver-bot-验证)
9. [模块 G — Supabase RLS 安全边界](#模块-g--supabase-rls-安全边界)
10. [模块 H — API 路由安全验证](#模块-h--api-路由安全验证)
11. [模块 I — 万能中后台（服务购买）](#模块-i--万能中后台服务购买)
12. [模块 J — 星火广场 & DAO 治理](#模块-j--星火广场--dao-治理)
13. [模块 K — 通知系统](#模块-k--通知系统)
14. [模块 L — 个人档案 & BadgeWall](#模块-l--个人档案--badgewall)
15. [模块 M — i18n 覆盖验证](#模块-m--i18n-覆盖验证)
16. [模块 N — Admin 面板功能](#模块-n--admin-面板功能)
17. [模块 O — 仲裁庭（Arbitration）](#模块-o--仲裁庭arbitration)
18. [安全渗透测试清单](#十八安全渗透测试清单)
19. [主网环境切换核对清单](#十九主网环境切换核对清单)
20. [上线放行标准](#二十上线放行标准)
21. [Bug 报告模板](#二十一bug-报告模板)
22. [合约 & 工具速查](#二十二合约--工具速查)

---

## 一、测试前准备

### 1.1 测试人员工具箱

| 工具 | 用途 | 备注 |
|------|------|------|
| MetaMask 浏览器插件 | 钱包签名、交易确认 | 建议同时开 ≥ 2 个浏览器 Profile |
| Arbitrum Sepolia ETH | 支付 Gas | 见下方水龙头 |
| MockUSDC | 协作支付测试 | 从 `/admin/faucet` 领取 |
| Supabase Dashboard | 数据库直查 | 需要 service role key |
| Arbiscan (Sepolia) | 交易/事件验证 | https://sepolia.arbiscan.io |
| curl / Postman | API 安全测试 | 手动发送 HTTP 请求 |
| Node.js ≥ 18 | 运行 resolver-bot | `npx tsx scripts/resolver-bot.ts` |

### 1.2 网络配置

**Arbitrum Sepolia（协作 & VCP）**

| 参数 | 值 |
|------|-----|
| Chain ID | 421614 |
| RPC | https://sepolia-rollup.arbitrum.io/rpc |
| 区块浏览器 | https://sepolia.arbiscan.io |

**Sepolia ETH（Privilege NFT）**

| 参数 | 值 |
|------|-----|
| Chain ID | 11155111 |
| RPC | https://rpc.sepolia.org |
| 区块浏览器 | https://sepolia.etherscan.io |

### 1.3 测试 Token 领取

| Token | 获取方式 |
|-------|----------|
| Sepolia ETH (Arbitrum) | https://faucet.triangleplatform.com/arbitrum/sepolia |
| MockUSDC | 登录平台 → `/admin/faucet`（需 Token #3 或管理员代领） |
| Sepolia ETH | https://sepoliafaucet.com |

---

## 二、测试账户角色矩阵

测试需要至少 **5 个独立钱包地址**，建议提前部署：

| 编号 | 角色 | 需持有 NFT | 主要参与模块 |
|------|------|-----------|-------------|
| W1 | 发布者 A（Publisher） | 无要求 | C / D / E / K |
| W2 | 承接人 A（Worker） | 无要求 | C / D / E / K |
| W3 | 管理员（Admin） | Token #3 (Sepolia ETH) | N / D-03 / I |
| W4 | 仲裁员（Hand of Justice） | Token #4 (Sepolia ETH) | O |
| W5 | Pioneer（星火广场）| Token #5 (Sepolia ETH) | J |
| W6 | 无权限账户（攻击者模拟）| 无 | B / G / H（安全测试） |

> W3-W5 需要在 Sepolia ETH 网络上持有对应 Privilege NFT（`0x46486Aa0aCC327Ac55b6402AdF4A31598987C400`）。
> 如无法获取，向现有持有者申请转移或使用测试账户。

---

## 模块 A — 身份认证 & JWT Auth 流程

> **优先级: P0** — 所有写操作依赖此基础

### A-01：钱包连接基本流程

```
前置: MetaMask 已安装，已切换到 Arbitrum Sepolia
步骤:
  1. 访问平台首页，点击右上角"连接钱包"
  2. 在 MetaMask 弹窗中选择账户，点击授权
预期:
  ✅ 右上角显示钱包地址缩写（0x...XXXX）
  ✅ 导航栏出现功能性入口（我的工作台/通知等）
  ✅ 无报错信息
```

### A-02：JWT 签发流程（AuthProvider）

```
前置: 钱包已连接
步骤:
  1. 打开浏览器 DevTools → Application → Cookies
  2. 连接钱包后观察 Cookie 变化
  3. 检查 Network Tab 中 /api/auth/wallet 请求
预期:
  ✅ /api/auth/wallet 返回 200，body 包含 { token: "..." }
  ✅ Supabase session cookie 被写入（sb-{project}-auth-token）
  ✅ 后续 Supabase 请求 Authorization Header 携带 JWT
```

### A-03：JWT 过期后重新认证

```
步骤:
  1. 手动清除 Cookie（DevTools → Application → Clear Site Data）
  2. 刷新页面
  3. 尝试执行写操作（如申请协作）
预期:
  ✅ 系统提示"请先连接钱包"或自动触发重连流程
  ✅ 无 500 错误，无数据泄露
```

### A-04：跨账户切换

```
步骤:
  1. 用 W1 登录，进入个人档案页面
  2. 在 MetaMask 中切换到 W2
  3. 观察页面状态变化
预期:
  ✅ 页面数据自动更新为 W2 的档案
  ✅ 不显示 W1 的私有数据
  ✅ W2 的 JWT 重新签发（无遗留 W1 的 session）
```

---

## 模块 B — NFT 门控安全边界

> **优先级: P0** — 门控失效 = 权限泄漏

### B-01：Admin 面板门控（Token #3）

```
步骤:
  A. 用 W6（无 NFT）访问 /admin
     预期: ❌ 拒绝访问 — 显示"需要 The First Flame NFT"提示

  B. 用 W3（Token #3 持有者）访问 /admin
     预期: ✅ 正常进入 Admin 面板

  C. W6 尝试直接访问 /admin/services、/admin/bulletins
     预期: ❌ 同样被拒，不可绕过
```

### B-02：仲裁庭门控（Token #4）

```
步骤:
  A. 用 W6 访问 /council/arbitration
     预期: ❌ 拒绝访问

  B. 用 W4（Token #4）访问 /council/arbitration
     预期: ✅ 可查看争议案卷

  C. W6 直接 POST /api/council/arbitration/vote（携带伪造签名）
     预期: ❌ 返回 403 Forbidden
```

### B-03：Pioneer 发帖门控（Token #5）

```
步骤:
  A. W6 在 /bulletin 页面尝试发帖
     预期: ❌ 发帖按钮不可见，或点击后被阻止

  B. W5（Token #5）正常发帖
     预期: ✅ 公告出现在列表

  C. W6 直接 POST /api/bulletin/pioneer（无签名）
     预期: ❌ 返回 401/403
```

### B-04：双源验证回退机制

```
步骤:（需要在测试网 RPC 不稳定时观察，或手动通过 DevTools 模拟网络失败）
  1. 关闭主 RPC 连接（DevTools → Network → Block URL: "sepolia-rollup.arbitrum.io"）
  2. 刷新 /admin（使用 W3）
预期:
  ✅ 系统切换到 /api/nft/verify（Alchemy REST API 后备）
  ✅ W3 仍能正常通过门控
  ✅ W6 仍被拒绝（Fail-closed，不在错误时放行）
  ✅ 无控制台 CORS 错误
```

---

## 模块 C — 协作全链路（自行管理模式）

> **优先级: P0** — 核心业务流程，每条必须全部通过

### C-01：创建协作

```
角色: W1（发布者）
步骤:
  1. 访问 /collaborations/create
  2. 填写标题、描述（≥ 50 字）
  3. 选择等级 D（100 USDC，40 VCP）
  4. 支付模式选择"自行管理"
  5. 添加 2 个里程碑：[70%, 30%]
  6. 提交
预期:
  ✅ 协作出现在 /collaborations 列表，状态 = OPEN
  ✅ collaborations 表记录正确（payment_mode = self_managed）
  ✅ milestones 表有 2 条记录
  ✅ 发布者自己无法"申请"自己创建的协作（申请按钮隐藏）
```

### C-02：申请承接（防重复申请）

```
角色: W2（承接人）
步骤:
  1. 访问 C-01 创建的协作详情页
  2. 点击"申请承接"，填写申请理由
  3. 提交成功后，再次点击"申请"
预期:
  ✅ 第一次申请：状态变为 PENDING_APPROVAL，W1 收到通知
  ✅ 第二次申请：被阻止（"您已申请过此协作"提示）
  ✅ collaboration_applications 表中 W2 只有 1 条记录
```

### C-03：批准申请

```
角色: W1（发布者）
步骤:
  1. 切换到 W1，进入协作详情页
  2. 在申请列表中找到 W2，点击"批准"
预期:
  ✅ 协作状态变为 ACTIVE
  ✅ 其他申请者被自动拒绝（如有）
  ✅ W2 收到"申请已通过"通知
  ✅ W1 不能再批准其他申请者（防重复选人）
```

### C-04：提交凭证

```
角色: W2（承接人）
步骤:
  1. 进入 ACTIVE 协作的里程碑 #1
  2. 点击"提交凭证"
  3. 填写说明 + 添加外部链接（GitHub PR URL）
  4. 确认提交
预期:
  ✅ 里程碑状态变为 PENDING（前端显示"待确认"）
  ✅ proofs 表新增记录（只能 INSERT，不会覆盖）
  ✅ W1 收到"凭证待审核"通知
  ✅ W2 无法对同一里程碑重复提交（或提交后按钮变灰）
```

### C-05：发布者审核凭证 & 拒绝

```
角色: W1
步骤:
  1. 查看 W2 提交的凭证
  2. 点击"拒绝"，填写拒绝原因
预期:
  ✅ 里程碑状态退回（可重新提交）
  ✅ W2 收到拒绝通知
  ✅ 拒绝记录保存在 proofs 表（不删除，只新增）
```

### C-06：取消协作

```
角色: W1（在 ACTIVE 阶段）
步骤:
  1. 在协作详情页点击"取消协作"
预期:
  ✅ 状态变为 CANCELLED
  ✅ W2 收到取消通知
  ✅ 取消后协作从任务大厅消失（不再展示）
  ✅ W2 可从"我的工作台"删除此已取消记录（RLS 允许 provider DELETE）
  ✅ W1 无法删除（initiator_id DELETE 限制）
```

### C-07：状态机非法跳转防护

```
步骤:
  1. 对 OPEN 状态的协作，W2 尝试直接提交凭证（无 ACTIVE 状态）
  2. 对 CANCELLED 状态的协作，W1 尝试批准申请
预期:
  ✅ 两种情况均被前端按钮状态阻止，无法执行
  ✅ 即使绕过前端，数据库层面 Supabase RLS 拒绝非法写入
```

---

## 模块 D — DirectPay 链上支付结算

> **优先级: P0** — 涉及真实资金流向（测试网用 MockUSDC）

### D-01：USDC Approve + Pay（完整双笔交易）

```
前置: W1 持有 ≥ 70 MockUSDC，W2 是 ACTIVE 协作的 provider
      W2 已提交里程碑 #1 凭证（状态 PENDING）
步骤:
  1. W1 进入里程碑 #1，点击"确认并支付"
  2. MetaMask 弹出第一笔交易 → Approve USDC 授权给 DirectPay
  3. 确认 Approve
  4. MetaMask 弹出第二笔交易 → 调用 DirectPay.pay()
  5. 确认 Pay
预期:
  ✅ 两笔交易均在 Arbiscan 可查
  ✅ W2 MockUSDC 余额增加 70 USDC（里程碑 #1 = 70%）
  ✅ 里程碑状态变为 SETTLED
  ✅ Arbiscan 上 DirectPay 合约有 Paid 事件（collabId, publisher, worker, amount）
  ✅ 如只有一个里程碑，协作状态变为 SETTLED
```

### D-02：EIP-1559 Gas 估算

```
步骤: 执行 D-01 的 Pay 交易
观察点:
  - MetaMask 中 Gas 费用字段是否有效（非 0，非 undefined）
  - 交易是否在合理时间内（≤ 30 秒）被矿工确认
预期:
  ✅ Gas 费用显示正常（基于 baseFeePerGas × 2 动态计算）
  ✅ 不出现 "gas fee too low" 或 "underpriced" 错误
```

### D-03：重复支付防护

```
步骤:
  1. D-01 支付成功后，W1 再次点击"确认并支付"对同一里程碑
预期:
  ✅ 按钮处于禁用状态（里程碑已 SETTLED）
  ✅ 即使强行发起交易，Supabase 不会重复更新状态
```

### D-04：多里程碑分批支付

```
步骤:
  1. 创建一个含 3 个里程碑的协作 [40%, 30%, 30%]
  2. W2 依次提交凭证，W1 依次逐笔支付
  3. 全部支付后观察协作总状态
预期:
  ✅ 每笔支付后对应里程碑变为 SETTLED
  ✅ 第三笔支付后协作整体状态变为 SETTLED
```

---

## 模块 E — VCP 积分 & 反作弊系统

> **优先级: P1** — 依赖 Resolver Bot 运行

### E-01：DirectPay 触发 VCP 铸造（0.5x）

```
前置: D-01 已完成（DirectPay.Paid 事件已上链）
      resolver-bot.ts 已运行（携带正确 RESOLVER_PRIVATE_KEY + MINTER_ROLE）
步骤:
  1. 等待 Resolver Bot 下一个轮询周期（默认 60 秒）
  2. 查看 bot 控制台日志
  3. 刷新 W2 的 /profile 页面
预期:
  ✅ 控制台日志: "[directPay] Minted X VCP (grade D, 0.5x) to 0x..."
  ✅ W2 的 VCP 余额增加（D 级 = 40 × 0.5 = 20 VCP）
  ✅ vcp_settlements 表新增记录:
     - settlement_key = "direct-{txHash}-{logIndex}"
     - anti_cheat_passed = true
     - vcp_amount = 20
     - submitted_at IS NOT NULL
     - confirmed_at IS NOT NULL
  ✅ profiles 表 vcp_cache 字段同步更新
```

### E-02：幂等性验证

```
步骤: 重启 resolver-bot，让它重新扫描相同的区块范围
预期:
  ✅ 控制台: "[directPay] Already processed {key}, skipping"
  ✅ VCP 不会重复铸造
  ✅ vcp_settlements 无重复记录
```

### E-03：7 天冷却期

```
前置: E-01 已完成（W1-W2 之间有一条已结算记录）
步骤:
  1. W1 和 W2 立即创建第二个协作，完成支付
  2. 等待 bot 处理
预期:
  ✅ 第二次铸造被跳过
  ✅ vcp_settlements 新记录:
     - anti_cheat_passed = false
     - skip_reason = "cooldown_7d"
     - vcp_amount = 0
```

### E-04：月度上限（1000 VCP/月）

```
（此测试为压测，可用模拟数据替代）
步骤:
  1. 在 vcp_settlements 表中手动插入该 worker 已达 990 VCP 的历史记录（当月）
  2. 完成一笔新的结算（约产生 20 VCP）
预期:
  ✅ bot 检测到月度上限，跳过铸造
  ✅ skip_reason = "monthly_cap"
```

### E-05：快速确认冻结

```
（此测试需要构造数据）
步骤:
  1. 手动在 vcp_settlements 中插入同一 worker 最近 10 条记录，
     其中 ≥ 5 条 confirmed_at - submitted_at < 120 秒
  2. 完成下一笔结算
预期:
  ✅ bot 检测到快速确认模式，冻结 30 天
  ✅ skip_reason = "fast_confirm_freeze"
```

---

## 模块 F — Resolver Bot 验证

> **优先级: P1** — 运维层面，需要实际运行脚本

### F-01：启动检查

```
步骤:
  1. 配置 .env 中 RESOLVER_PRIVATE_KEY / SUPABASE_URL / SUPABASE_SERVICE_ROLE_KEY
  2. 运行: npx tsx scripts/resolver-bot.ts
预期启动日志:
  ✅ 显示 "SuperGuild Resolver Bot v1.1"
  ✅ 显示 Chain / Escrow / DirectPay / VCP 地址
  ✅ 显示 "[startup] MINTER_ROLE verified on VCPTokenV2"
     （如显示 WARN 则需要 grantRole）
```

### F-02：MINTER_ROLE 验证

```
如 F-01 出现 WARN，执行:
  1. 用 Admin 账户调用 VCPTokenV2.grantRole(MINTER_ROLE, resolverAddress)
  2. 重启 bot，确认 "[startup] MINTER_ROLE verified"
```

### F-03：AutoRelease（仅公会托管模式可测）

```
（MVP 阶段 self_managed 无 AutoRelease，此测试适用于 guild_managed 开放后）
如已开放 guild_managed:
  1. 创建一个 guild_managed 协作，W2 提交凭证
  2. 手动将链上里程碑的 deadline 设为过去时间（测试网可用合约调试工具）
  3. 等待 bot 触发 autoRelease
预期:
  ✅ bot 日志: "[autoRelease] Releasing..."
  ✅ USDC 转入 W2 钱包
  ✅ 里程碑状态更新为 SETTLED
```

### F-04：进程崩溃恢复

```
步骤:
  1. 让 bot 正常运行 3 个轮询周期
  2. 强制终止（Ctrl+C）
  3. 重启 bot
预期:
  ✅ 重启后 lastProcessedBlock / lastProcessedBlockDP 从零重扫最近 1000 个区块
  ✅ 已处理记录不重复（幂等性保证）
  ✅ 新事件正常处理
```

---

## 模块 G — Supabase RLS 安全边界

> **优先级: P0** — 数据安全红线，每条必须验证

### G-01：跨用户读取私有数据

```
工具: Supabase JS SDK（或 Postman，携带 W6 的 anon JWT）
步骤:
  1. 用 W6 的 session 查询 W1 的 collaborations 数据
  2. 尝试读取 profiles 表中其他用户的 contact_email
预期:
  ✅ 协作数据：只返回 W6 作为 initiator/provider 参与的记录
  ✅ contact_email：W6 无法读取 W1 的联系方式
     （RLS 策略：只有协作双方才能读取对方联系方式）
```

### G-02：跨用户写入防护

```
步骤:
  1. W6 尝试 UPDATE collaborations SET status='SETTLED' WHERE id='{W1 的协作 ID}'
  2. W6 尝试 INSERT INTO proofs (collab_id, ...) VALUES ('{别人的协作}', ...)
  3. W6 尝试 DELETE FROM notifications WHERE user_address='{W1 地址}'
预期:
  ✅ 三种操作均返回 RLS 错误（0 行受影响 或 403）
  ✅ 数据未被修改
```

### G-03：proofs 表不可删除/覆盖

```
步骤:
  1. W2（自己的凭证）尝试 DELETE FROM proofs WHERE id='{自己的凭证 ID}'
  2. W2 尝试 UPDATE proofs SET content_hash='0x...' WHERE id='{自己的凭证 ID}'
预期:
  ✅ DELETE 被 RLS 拒绝（无 DELETE 策略）
  ✅ UPDATE 被 RLS 拒绝（无 UPDATE 策略）
```

### G-04：vcp_settlements 表防篡改

```
步骤:
  1. W2 尝试 INSERT INTO vcp_settlements（直接写入，绕过 bot）
  2. W2 尝试修改已有记录的 vcp_amount
预期:
  ✅ 只有 service_role（bot）可写，anon/authenticated 均被拒绝
```

### G-05：Service Role 隔离

```
验证: vcp_settlements / services / pioneer_codes / service_access 等敏感表
步骤: 用普通 authenticated JWT 尝试写入上述表
预期:
  ✅ 全部 RLS 拒绝（只有 service_role 可写）
```

---

## 模块 H — API 路由安全验证

> **优先级: P1**

### H-01：无签名请求被拒

```
步骤: 对以下路由发送无签名 POST 请求（仅携带 body，无 X-Wallet-Signature）
  - POST /api/admin/bulletins
  - POST /api/council/arbitration/vote
  - POST /api/bulletin/pioneer
预期:
  ✅ 全部返回 401 Unauthorized
```

### H-02：签名伪造检测

```
步骤:
  1. 用 W6 的私钥签名 W3 的消息（冒充 Admin）
  2. 发送到 /api/admin/services（header: x-wallet-address: W3, x-wallet-signature: W6 签的）
预期:
  ✅ 返回 403 — 签名与 x-wallet-address 不匹配
```

### H-03：Nonce 重放攻击防护

```
步骤:
  1. 正常登录，截获一次有效的 /api/auth/wallet 请求（nonce + signature）
  2. 1 分钟后用相同的 nonce + signature 重放请求
预期:
  ✅ 第一次请求成功（返回 JWT）
  ✅ 重放请求返回 401（nonce 已使用或已过期）
```

### H-04：Rate Limiting

```
步骤: 在 10 秒内对 /api/nft/verify 发送 ≥ 20 次请求
预期:
  ✅ 触发限流，返回 429 Too Many Requests
  ✅ 限流解除后正常响应
```

### H-05：上传文件安全（大文件 + 非法 MIME）

```
步骤:
  1. 尝试上传 6MB 文件到 /api/upload
  2. 尝试上传 .exe 文件（MIME: application/octet-stream）
  3. 尝试上传 .php 文件改后缀为 .jpg（内容为 PHP 代码）
预期:
  ✅ 6MB 文件被拒（限制 5MB）
  ✅ .exe 被拒（不在 MIME 白名单）
  ✅ .php 伪装为 .jpg 被拒（服务端 MIME 嗅探）
```

---

## 模块 I — 万能中后台（服务购买）

### I-01：浏览三频道

```
步骤: 访问 /services，依次进入三个频道
  - /services/infrastructure（基础设施）
  - /services/core（核心服务）
  - /services/consulting（专家咨询）
预期:
  ✅ 服务列表正常加载
  ✅ 价格、标签、描述显示正确
  ✅ 语言切换后内容同步翻译
```

### I-02：服务购买（链上 USDC 支付）

```
前置: W2 持有足够 MockUSDC
步骤:
  1. 选择一个基础设施服务（如 50 USDC）
  2. 点击购买，MetaMask 弹出 USDC Transfer 交易
  3. 确认交易
预期:
  ✅ USDC 从 W2 转至 SERVICE_TREASURY 地址
  ✅ 服务内容解锁（文档/联系方式/下载链接）
  ✅ service_access 表新增记录
  ✅ Arbiscan 可查到对应 ERC-20 Transfer 事件
```

### I-03：重复购买检测

```
步骤: I-02 购买成功后，再次点击购买同一服务
预期:
  ✅ 显示"已购买"状态，无法重复购买
```

### I-04：Admin 上架/编辑服务

```
角色: W3（Admin）
步骤:
  1. 访问 /admin/services
  2. 新建一个频道 2 服务，填写名称、描述、价格
  3. 保存后访问 /services/core 验证
  4. 编辑刚创建的服务，修改价格
  5. 删除该服务
预期:
  ✅ 创建/编辑/删除操作均需 Token #3 NFT 门控（W6 无法操作）
  ✅ 变更实时反映在服务列表
```

---

## 模块 J — 星火广场 & DAO 治理

### J-01：公告浏览

```
步骤: 访问 /bulletin
预期:
  ✅ 公告列表正常加载，带分页
  ✅ 公告内容（Markdown）正确渲染
  ✅ 附件链接可访问
```

### J-02：Pioneer 发布公告（Token #5）

```
角色: W5
步骤:
  1. 点击"发布公告"
  2. 填写标题、正文，附加外链
  3. 提交
预期:
  ✅ 公告出现在列表（作者显示 W5 的 username 或地址缩写）
  ✅ bulletins + bulletin_attachments 表各有对应记录
```

### J-03：DAO 提案全流程

```
步骤:
  1. W2 在 /council/proposals 点击"发布提案"
  2. 填写提案标题、内容、类型
  3. 提交（状态: 联署中 GATHERING_COSIGNS）
  4. W1 对该提案联署
  5. 达到联署门槛后，状态变为 ACTIVE_VOTING
  6. W1、W2 分别投赞成/反对票
  7. 投票期结束（或手动调用 finalize）
预期:
  ✅ 每个状态跳转正确显示
  ✅ 投票权重 = VCP 余额（链上镜像）
  ✅ W2 可撤回自己的提案（状态 = GATHERING_COSIGNS 时）
  ✅ 撤回后 proposals 表状态变为 WITHDRAWN
```

### J-04：提案投票签名验证

```
步骤:
  1. W6 直接 POST /api/council/arbitration/vote，伪造 voter_address = W4
预期:
  ✅ 返回 403（签名验证失败）
```

---

## 模块 K — 通知系统

### K-01：通知触发完整性验证

以下操作后，对应方必须收到通知：

| 触发操作 | 通知接收方 |
|---------|----------|
| W2 申请 W1 的协作 | W1 |
| W1 批准 W2 的申请 | W2 |
| W1 拒绝 W2 的申请 | W2 |
| W2 提交里程碑凭证 | W1 |
| W1 确认里程碑 | W2 |
| W1 取消协作 | W2 |

```
预期:
  ✅ 通知铃出现红点 badge
  ✅ 打开通知面板可看到对应消息（标题/正文正确）
  ✅ 点击通知可导航到相关页面（如果有 metadata.collab_id）
```

### K-02：通知已读 & 删除

```
步骤:
  1. 点击单条通知 → 标记为已读（红点消失）
  2. 点击"全部已读"→ 所有 badge 清除
  3. 对一条已读通知点击删除
预期:
  ✅ 单条已读：is_read 变为 true，unread count 减 1
  ✅ 全部已读：unread count = 0
  ✅ 删除：通知从列表消失，数据库记录删除
  ✅ 删除未读通知：被阻止（RLS 只允许删除 is_read=true 的通知）
```

### K-03：通知实时性

```
步骤: W2 发送申请，W1 在另一浏览器窗口等待
预期:
  ✅ W1 在 ≤ 60 秒内（轮询间隔）看到新通知（或更快，如果 Supabase Realtime 启用）
```

---

## 模块 L — 个人档案 & BadgeWall

### L-01：个人资料编辑

```
步骤:
  1. 访问 /profile，点击编辑
  2. 修改 username、bio、portfolio
  3. 至少填写一种联系方式（Email 或 Telegram）
  4. 保存
预期:
  ✅ 数据保存，页面刷新后持久化
  ✅ 空 username → 显示错误提示（不允许保存）
  ✅ 无联系方式 → 显示错误提示
```

### L-02：头像上传

```
步骤:
  1. 上传 ≤ 2MB 的 JPG/PNG 头像
  2. 上传 > 5MB 文件
  3. 上传 .gif 文件
预期:
  ✅ 正常头像：上传成功，显示新头像，URL 为 Supabase Storage CDN 地址
  ✅ 大文件：被前端/后端拒绝（显示错误提示）
  ✅ .gif：根据 MIME 白名单决定是否允许
```

### L-03：BadgeWall NFT 展示

```
步骤: 用持有不同 NFT 的账户查看 /profile
预期:
  ✅ 持有 Token #1 → Pioneer Memorial 3D 模型显示
  ✅ 持有 Token #2 → Lantern Keeper 3D 模型显示
  ✅ 未持有的 Token → 对应位置完全隐藏（不显示"未持有"占位）
  ✅ 3D 模型可鼠标旋转，不出现渲染错误
  ✅ GLB 文件从 Supabase Storage CDN 加载（非 /public 本地路径）
```

### L-04：VCP 余额同步

```
步骤:
  1. Resolver Bot 铸造 VCP 成功后
  2. 刷新 /profile 页面
预期:
  ✅ VCP 显示值与链上 balanceOf 一致
  ✅ profiles.vcp_cache 同步更新（在链上 mint 后触发更新）
```

### L-05：公开档案页（他人查看）

```
步骤: W1 访问 W2 的公开档案 URL（/profile/[address]）
预期:
  ✅ 显示 W2 的 username、bio、portfolio、BadgeWall、VCP
  ✅ 不显示 W2 的 contact_email 和 contact_telegram（私密字段）
  ✅ 不显示编辑按钮
```

---

## 模块 M — i18n 覆盖验证

> **优先级: P1** — 主网上线面向国际用户

### M-01：全页面扫描

```
步骤: 切换到英文（EN），依次浏览以下页面，截图或手动检查是否有遗漏翻译
  页面列表:
  - / (Landing)
  - /collaborations
  - /collaborations/create
  - /collaborations/[id]
  - /profile
  - /bulletin
  - /council/proposals
  - /council/arbitration
  - /services
  - /services/infrastructure
  - /services/core
  - /services/consulting
  - /admin（Token #3 账户）
  - 所有弹窗（创建协作/申请/提交凭证/创建提案等）
  - 所有 Toast 通知
  - 所有错误提示
预期:
  ✅ 无中文遗漏（不出现汉字或中文标点）
  ✅ 英文语义准确，无机翻感
```

### M-02：中英切换无刷新

```
步骤: 在各页面中点击语言切换，不刷新页面
预期:
  ✅ 文字即时切换
  ✅ 无布局错位（英文字符串长度与中文差异不引起 UI 崩溃）
```

---

## 模块 N — Admin 面板功能

> **前提: 使用 W3（Token #3 持有者）登录**

### N-01：公告管理

```
步骤:
  1. /admin/bulletins → 创建新公告（填写标题/正文）
  2. 编辑已有公告
  3. 删除公告
预期:
  ✅ 创建：公告出现在 /bulletin
  ✅ 编辑：内容更新反映在前台
  ✅ 删除：公告从列表消失
  ✅ W6 访问 /admin/bulletins → 403 拒绝
```

### N-02：服务管理

```
步骤: 参考 I-04
预期: ✅ Admin API 签名验证正常工作（参考 H-01/H-02）
```

### N-03：Faucet 页面（仅测试网）

```
步骤:
  1. W3 访问 /admin/faucet
  2. 点击 Mint 10,000 MockUSDC
预期:
  ✅ MockUSDC 铸入 W3 钱包
  ✅ 页面有明确提示"仅限测试网"（IS_MAINNET 守卫正确工作）
  ✅ 主网部署时该页面应显示"Not available on mainnet"或直接 404
```

---

## 模块 O — 仲裁庭（Arbitration）

> **前提: W4（Token #4 持有者），需要有 DISPUTED 状态的协作**

### O-01：构建争议场景

```
步骤:（需要 guild_managed 模式，MVP 阶段可跳过或标记 BLOCKED）
  1. 发布者对承接人的凭证提交发起争议（Dispute）
  2. 协作状态变为 DISPUTED
```

### O-02：仲裁投票

```
步骤（仅 guild_managed 模式）:
  1. W4 访问 /council/arbitration，找到 DISPUTED 案卷
  2. 投票"支持 Worker"
  3. 其他仲裁员同样投票（达到 3 票门槛）
预期:
  ✅ dispute_votes 表记录投票，voter_address = W4
  ✅ 达到门槛后 Resolver Bot 触发 resolveDispute() 链上交易
  ✅ GuildEscrow 执行 10% 罚没（workerWon=false 时）
```

### O-03：API 安全（当前 self_managed 阶段）

```
步骤: 对 /api/council/arbitration/vote 执行 H-01 ~ H-02 安全测试
预期:
  ✅ 无签名请求被拒
  ✅ 无 Token #4 地址被拒
```

---

## 十八、安全渗透测试清单

> 以下测试由熟悉 Web3 安全的测试人员执行，模拟恶意攻击者视角

### SEC-01：IDOR（越权直接访问）

```
目标: 尝试读取/修改不属于自己的资源
方法:
  1. 枚举协作 UUID，用未参与账户访问详情 API
  2. 修改请求体中的 collab_id 为其他协作
预期: 全部被 RLS 拦截，返回空数据或 403
```

### SEC-02：签名重放 + 跨方法复用

```
方法:
  1. 截取 /api/bulletin/pioneer 的签名，用同一签名调用 /api/admin/services
  2. 截取 5 分钟前的过期签名重放
预期:
  ✅ 跨路由重放失败（消息体不同，签名无效）
  ✅ 过期签名失败（timestamp 超时校验）
```

### SEC-03：NFT 门控 Mock 环境变量

```
检查: 确认 NEXT_PUBLIC_DEV_MOCK_NFTS 未被设置为 "true"
方法: 在页面控制台执行 console.log(process.env.NEXT_PUBLIC_DEV_MOCK_NFTS)
     或查看 .env.local 文件
预期:
  ✅ 该变量不存在或为空值
  ✅ 主网部署前必须移除，否则所有人绕过 NFT 门控
```

### SEC-04：XSS 注入

```
方法: 在以下字段输入: <script>alert('xss')</script>
  - 协作标题/描述
  - 用户 username/bio
  - 公告内容
  - 凭证说明
预期:
  ✅ 内容正确转义显示（不执行脚本）
  ✅ Markdown 渲染不注入原始 HTML（使用安全的 Markdown 解析器）
```

### SEC-05：SQL 注入（Supabase SDK）

```
方法: 在搜索框、过滤字段输入 SQL 特殊字符
  例如: ' OR 1=1 --  ; DROP TABLE collaborations;
预期:
  ✅ Supabase SDK 使用参数化查询，输入被视为字面值
  ✅ 无数据泄露，无 SQL 错误
```

---

## 十九、主网环境切换核对清单

> **切换前逐条打勾，每条缺失都可能导致生产事故**

### 🔴 必须关闭（否则安全漏洞）

- [ ] `.env.local` 中 `NEXT_PUBLIC_DEV_MOCK_NFTS` → **删除或设为空值**
  ⚠️ 存在此变量 = 所有人绕过 NFT 门控
- [ ] Admin Faucet 页面 `/admin/faucet` → **确认 IS_MAINNET 守卫生效**
  ⚠️ 主网无 MockUSDC.mint，页面必须不可访问

### 🔴 必须替换

- [ ] `HOT_WALLET_PRIVATE_KEY` → 使用全新主网专用热钱包（测试网私钥已暴露）
- [ ] MockUSDC 地址 → 替换为 Circle USDC 主网地址
  `constants/nft-config.ts` 中的 USDC_ADDRESS

### 🔴 必须切换（链 & 合约）

- [ ] `PRIMARY_CHAIN_ID` → `42161` (Arbitrum One)
  文件: `constants/chain-config.ts`
- [ ] `PRIVILEGE_CHAIN_ID` → `1` (Ethereum Mainnet)
  文件: `constants/chain-config.ts`
- [ ] 全部合约地址 → 主网部署地址
  文件: `constants/contract-address.ts` / `nft-config.ts`
- [ ] `DIRECT_PAY_ADDRESS` in `resolver-bot.ts` → 主网 DirectPay 地址
  当前占位: `0x0000000000000000000000000000000000000000`
- [ ] Alchemy RPC API Key → 主网版本（带速率限制配置）

### 🟡 必须验证（功能）

- [ ] Supabase Auth JWT 端到端流程验证
  - `SUPABASE_JWT_SECRET` 匹配 Supabase 项目 JWT Secret
  - `/api/auth/wallet` 签发的 JWT 中 `wallet_address` claim 被 RLS 正确读取
  - `auth.wallet_address()` DB 函数在生产环境可用
- [ ] RLS 全表覆盖已在主网 Supabase 项目中执行
  migration: `20260311000000_rls_mainnet_hardening.sql`
- [ ] Resolver Bot 热钱包已在主网 VCPTokenV2 被授予 MINTER_ROLE
- [ ] GuildEscrow resolver 地址 & treasury 地址已更新为主网值
- [ ] Alchemy Webhook 已切换到主网监听 DirectPay.Paid 事件

### 🟢 强烈建议（上线前完成）

- [ ] 联系专业机构完成 GuildEscrow + VCPTokenV2 合约安全审计
- [ ] contact_email / contact_telegram PII 字段加密（明文存储 TODO）
- [ ] 在 next.config.js 中启用完整 CSP（Content-Security-Policy）
- [ ] 启用 HSTS（Strict-Transport-Security）
- [ ] 不稳定依赖项升级至稳定版（eslint-config-next RC 版）

---

## 二十、上线放行标准

### 必须全部通过（P0 阻断项）

| 检查项 | 通过标准 |
|-------|---------|
| 模块 A — Auth 流程 | A-01 ~ A-04 全部通过 |
| 模块 B — NFT 门控 | B-01 ~ B-04 全部通过，B-04 Fail-closed 验证通过 |
| 模块 C — 协作全链路 | C-01 ~ C-06 全部通过，C-07 防护有效 |
| 模块 D — DirectPay 支付 | D-01 ~ D-04 全部通过 |
| 模块 G — RLS 安全 | G-01 ~ G-05 全部通过，无数据泄露 |
| 安全 SEC-03 | NEXT_PUBLIC_DEV_MOCK_NFTS 确认不存在 |
| 安全 SEC-04/05 | XSS & SQL 注入测试通过 |
| 主网清单 🔴 项 | 全部勾选完成 |
| 合约审计 | GuildEscrow + VCPTokenV2 通过专业审计 |

### 建议通过（P1，可带问题上线但需记录）

| 检查项 | 通过标准 |
|-------|---------|
| 模块 E — VCP 反作弊 | E-01 ~ E-03 通过（E-04/E-05 可测试环境验证） |
| 模块 F — Resolver Bot | F-01 ~ F-02 通过（F-03 仅 guild_managed 开放后） |
| 模块 H — API 安全 | H-01 ~ H-05 全部通过 |
| 模块 M — i18n | 无遗漏硬编码文本 |
| 模块 N — Admin | N-01 ~ N-03 通过 |

---

## 二十一、Bug 报告模板

发现问题时，按以下格式提交：

```
【Bug 报告】

模块: [例如：D-01 DirectPay 支付]
严重程度:
  🔴 P0-崩溃  — 流程完全无法走通 / 资金安全问题
  🟠 P1-严重  — 核心功能受阻，有临时绕过方法
  🟡 P2-一般  — 功能降级，有较好替代
  🟢 P3-轻微  — UI/文案/体验问题，不影响功能

测试网络: Arbitrum Sepolia / Sepolia ETH
测试账户（角色）: W1-Publisher / W2-Worker / W3-Admin / W4-Arbitrator / W5-Pioneer / W6-NoNFT
钱包地址: 0x...

复现步骤:
  1.
  2.
  3.

实际结果:

预期结果:

附件（截图/交易 Hash/控制台报错）:
  - Arbiscan TX: https://sepolia.arbiscan.io/tx/0x...
  - 控制台错误截图:
  - Supabase Dashboard 截图（如涉及数据库）:

是否阻断上线: 是 / 否
```

---

## 二十二、合约 & 工具速查

### 合约地址（Arbitrum Sepolia 测试网）

| 合约 | 地址 | Arbiscan |
|------|------|---------|
| GuildEscrow | `0x8828c3fe2f579a70057714e4034d8c8f91232a60` | [查看](https://sepolia.arbiscan.io/address/0x8828c3fe2f579a70057714e4034d8c8f91232a60) |
| DirectPay | `0xDc0f7BF5c7C026f8000e00a40d0f93a28c04bf65` | [查看](https://sepolia.arbiscan.io/address/0xDc0f7BF5c7C026f8000e00a40d0f93a28c04bf65) |
| MockUSDC | `0xdd0a2bf984d690c9cdd613603094d7455fc63e06` | [查看](https://sepolia.arbiscan.io/address/0xdd0a2bf984d690c9cdd613603094d7455fc63e06) |
| VCPTokenV2 | `0xcDD2b15fEFC2071339234Ee2D72104F8E702f63C` | [查看](https://sepolia.arbiscan.io/address/0xcDD2b15fEFC2071339234Ee2D72104F8E702f63C) |
| MedalNFT | `0xef96bE9fFf59B5653085C11583beaC0D16450F1a` | [查看](https://sepolia.arbiscan.io/address/0xef96bE9fFf59B5653085C11583beaC0D16450F1a) |
| Privilege NFT (Sepolia ETH) | `0x46486Aa0aCC327Ac55b6402AdF4A31598987C400` | [查看](https://sepolia.etherscan.io/address/0x46486Aa0aCC327Ac55b6402AdF4A31598987C400) |

### Privilege NFT Token ID 映射

| Token ID | 名称 | 门控功能 |
|----------|------|---------|
| #1 | Pioneer Memorial | 创始成员纪念 |
| #2 | Lantern Keeper's Withered Lamp | 社区守护者标识 |
| #3 | The First Flame | Admin 面板 |
| #4 | Hand of Justice | 仲裁庭 |
| #5 | Beacon of the Forerunner | Pioneer 发帖 |

### Resolver Bot 运行命令

```bash
# 测试网
RESOLVER_PRIVATE_KEY=0x... \
SUPABASE_URL=https://xxx.supabase.co \
SUPABASE_SERVICE_ROLE_KEY=eyJ... \
npx tsx scripts/resolver-bot.ts

# 主网（切换后）
CHAIN_ID=42161 \
RESOLVER_PRIVATE_KEY=0x... \
npx tsx scripts/resolver-bot.ts
```

### 关键数据库表查询（Supabase Dashboard → SQL Editor）

```sql
-- 查看最近 VCP 结算记录
SELECT settlement_key, worker_address, grade, vcp_amount,
       anti_cheat_passed, skip_reason, submitted_at, confirmed_at, evaluated_at
FROM vcp_settlements
ORDER BY evaluated_at DESC LIMIT 20;

-- 查看协作状态分布
SELECT status, COUNT(*) FROM collaborations GROUP BY status;

-- 查看 RLS 是否启用
SELECT tablename, rowsecurity FROM pg_tables
WHERE schemaname = 'public' ORDER BY tablename;
```

---

*本测试方案基于 Phase 11.5 完成后的代码状态（2026-03-14）。guild_managed 模式开放后，需补充 GuildEscrow 全链路测试用例。*
