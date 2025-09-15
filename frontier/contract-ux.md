# 29.2 合约交互、签名与安全 UX 🧾🔐

> 这章解决三件事：**怎么调合约**、**怎么签消息**、**怎么把风险讲明白再让用户点确定**。技术栈以 **EIP-1193 + viem/wagmi** 为主，配足“防坑与 UX 规约”。

---

## 0）工程心法（先立规矩）

- **读写分离**：`publicClient` 只读、`walletClient` 落写；所有写操作一律**先 simulate 再发送**。
- **显式链**：所有请求必须显式 `chainId(hex)`，拒绝“当前链猜测”。
- **最小授权**：`approve` 只批**用量上限**，能用 **permit(EIP-2612)** 就不用“无限大授权”。
- **可解释**：在确认框里展示 **函数名/代币流向/授权对象**，必要时提供**模拟结果**与“可撤销”入口。
- **事务排队**：前端**串行分配 nonce**，避免多事务同时发送的“nonce 竞态”。

---

## 1）viem 基线：客户端与链配置

```ts
// web3/client.ts
import { createPublicClient, createWalletClient, http, custom } from 'viem';
import { base, mainnet } from 'viem/chains';

export const publicClient = createPublicClient({
  chain: base,
  transport: http('https://base-rpc.publicnode.com'), // 显式 RPC
});

export const walletClient = createWalletClient({
  chain: base,
  transport: custom((window as any).ethereum),          // EIP-1193 provider
});
```

---

## 2）只读调用与写入：**先 simulate 后 write**

### 2.1 只读（`eth_call`）

```ts
// 读取 ERC20 总量
import erc20 from './abi/erc20.json' assert { type: 'json' };
export async function totalSupply(token: `0x${string}`) {
  return publicClient.readContract({
    address: token,
    abi: erc20,
    functionName: 'totalSupply',
  });
}
```

### 2.2 写入（`simulateContract` → `writeContract` → `waitForReceipt`）

```ts
import erc20 from './abi/erc20.json' assert { type: 'json' };

// 精确授权（不是无限大）
export async function approveExact(params: {
  token: `0x${string}`,
  spender: `0x${string}`,
  amount: bigint,
  account: `0x${string}`,
}) {
  const { request } = await publicClient.simulateContract({
    address: params.token,
    abi: erc20,
    functionName: 'approve',
    args: [params.spender, params.amount],
    account: params.account,
  }); // ✅ 预执行，拿到 gas/nonce/链一致的安全参数

  const hash = await walletClient.writeContract(request);           // 发送交易
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  return receipt.status; // 'success' | 'reverted'
}
```

> 规范：所有写操作**必须**经过 `simulateContract`，把 **gasLimit / value / calldata** 都敲定。确认框展示 **函数签名 + args（脱敏 / 代币符号化）**。

---

## 3）错误与重试：**把 JSON-RPC 错误翻成人话**

```ts
// web3/errors.ts
export function explainEvmError(e: any) {
  const code = e?.code ?? e?.error?.code;
  const msg  = (e?.shortMessage || e?.message || '').toLowerCase();

  if (code === 4001) return '已取消签名/交易（用户拒绝）';
  if (code === 4902) return '当前钱包未识别该网络，请允许添加后重试';
  if (msg.includes('insufficient funds')) return '账户余额不足（含 gas 费）';
  if (msg.includes('replacement transaction underpriced')) return '加速/撤销失败：手续费太低';
  if (msg.includes('nonce too low')) return '交易序号冲突，请等待前一笔入池/确认';
  if (msg.includes('execution reverted')) return '合约拒绝：请检查参数或余额/授权';
  return '区块链交互失败：' + (e?.shortMessage || e?.message || '未知错误');
}
```

> UX：错误弹窗永远配 **“排查建议”** 与**复制原始错误**按钮。重试策略：**幂等写**（相同 nonce）只允许“加速/取消”，不生成新事务。

---

## 4）签名：`personal_sign` / **EIP-712** / **SIWE**

### 4.1 个人消息（不推荐做授权）

```ts
import { toHex } from 'viem';

export async function signPlain(account: `0x${string}`, msg: string) {
  return walletClient.signMessage({
    account,
    message: msg, // 建议携带 domain/链/过期时间
  });
}
```

### 4.2 **EIP-712 TypedData**（强烈推荐）

```ts
import { walletClient } from './client';

export async function signTypedPermit(params: {
  account: `0x${string}`;
  token: `0x${string}`;
  tokenName: string;               // 需链上读取 name()
  version?: string;                // 多为 '1'
  chainId: number;
  spender: `0x${string}`;
  value: bigint;
  nonce: bigint;                   // ERC20Permit 的 nonces(owner)
  deadline: bigint;                // 秒级时间戳
}) {
  const domain = {
    name: params.tokenName,
    version: params.version ?? '1',
    chainId: params.chainId,
    verifyingContract: params.token,
  } as const;

  const types = {
    Permit: [
      { name: 'owner',   type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value',   type: 'uint256' },
      { name: 'nonce',   type: 'uint256' },
      { name: 'deadline',type: 'uint256' },
    ]
  } as const;

  const message = {
    owner: params.account,
    spender: params.spender,
    value: params.value,
    nonce: params.nonce,
    deadline: params.deadline,
  } as const;

  const signature = await walletClient.signTypedData({
    account: params.account,
    domain, types, primaryType: 'Permit', message
  });
  return signature;
}
```

> 细节：EIP-2612 的 `nonces` 部分代币实现为 `nonces(address)`（函数名同但 ABI 不总是标准），**先读链上**确认；`deadline` 别给极长，常用 10–30 分钟。

### 4.3 **SIWE 登录**（EIP-4361）

**前端：构造并签名**

```ts
// login.ts
import { walletClient } from './client';

export async function signInWithEthereum(params: {
  account: `0x${string}`;
  domain: string;                  // 站点域名
  statement?: string;              // 提示文案
  uri: string;                     // 站点 URL
  chainId: number;
  nonce: string;                   // 服务端下发，一次性
}) {
  const now = new Date().toISOString();
  const msg = [
    `${params.domain} wants you to sign in with your Ethereum account:`,
    `${params.account}`,
    '',
    params.statement ?? 'Authenticate to this app.',
    '',
    `URI: ${params.uri}`,
    `Version: 1`,
    `Chain ID: ${params.chainId}`,
    `Nonce: ${params.nonce}`,
    `Issued At: ${now}`,
  ].join('\n');

  const signature = await walletClient.signMessage({ account: params.account, message: msg });
  return { message: msg, signature };
}
```

**后端：校验并颁发会话**

```ts
// api/siwe-verify.ts (Node)
import { recoverMessageAddress, isAddressEqual } from 'viem';

export async function verifySiwe({ message, signature, expectedDomain, expectedUri }: {
  message: string; signature: `0x${string}`; expectedDomain: string; expectedUri: string;
}) {
  const addr = await recoverMessageAddress({ message, signature });
  // 基础校验：域名、URI、过期、nonce（服务端缓存去重）
  if (!message.includes(`URI: ${expectedUri}`) || !message.startsWith(`${expectedDomain} wants you to sign`)) {
    throw new Error('SIWE 域/URI 不匹配');
  }
  // TODO: 验证 Nonce & TTL
  return addr;
}
```

> UX：登录弹窗必须说明 **用途/权限/有效期**；会话采用 **短 Token + 滚动刷新**，并与地址/链绑定。

---

## 5）授权与 **Permit**：把“无限大”变成“有限且可撤”

### 5.1 经典 `approve` 与撤销

```ts
// 将授权改为 0 即撤销
await approveExact({ token, spender, amount: 0n, account });
```

### 5.2 **EIP-2612 Permit** + 一次性执行（免两次弹窗）

很多路由合约支持 `permit + action` 一次打包（先在链下签名，再在合约里 `permit` 生效后执行）。  
**确认框必须展示：**“本次将授权 X 代币给 Y，额度 Z，截止时间 T”。

> 进阶：一些协议支持 **Permit2**（通用许可），或 **Permit for ERC721/1155**（如 `permitForAll` 变体）——都要清晰标注**范围**与**截止**。

---

## 6）事务排队：**前端非阻塞串行发交易**

```ts
// web3/tx-queue.ts
import { walletClient, publicClient } from './client';

type Key = `${string}:${number}`; // address:chainId
const inflight = new Map<Key, bigint>();

export async function sendTxSequential(req: Parameters<typeof walletClient.sendTransaction>[0]) {
  const key = `${req.account}:${req.chain?.id ?? (await publicClient.getChainId())}` as Key;

  // 计算 pending nonce（串行分配）
  let base = inflight.get(key);
  if (base == null) {
    base = await publicClient.getTransactionCount({ address: req.account as `0x${string}`, blockTag: 'pending' });
  }
  const nonce = base;
  inflight.set(key, base + 1n);

  try {
    const hash = await walletClient.sendTransaction({ ...req, nonce });
    const receipt = await publicClient.waitForTransactionReceipt({ hash });
    return receipt;
  } finally {
    // 成功/失败都释放（下一次会以 pending count 重新对齐）
    inflight.delete(key);
  }
}
```

> 说明：队列确保“用户瞬间点了三次确认”也不会 nonce 互撞。若你用 wagmi 的 `writeContract`，同理封装一层串行。

---

## 7）安全确认框：**把风险翻译成中文**

确认弹框**必须包含**：

- **目标链**（图标 + 十六进制链 ID）：不匹配时先切链。
- **调用对象**：合约徽章（已验证/未验证）、`address`、`functionName(args)`（脱敏格式化）。
- **代币流向**：`- 你将花费 X TOKEN`、`+ 你将收到 Y TOKEN`（估算/模拟）。
- **授权变化**（若有）：授权对象、额度、过期时间；是否覆盖现有额度。
- **费用**：`maxFeePerGas / priority` 与估计法币等值。
- **撤销路径**：一键跳转“授权管理”页面（你自己的或者链上第三方）。

> 设计小贴士：对 `setApprovalForAll(true)`、`approve(MAX_UINT)` 使用 **红色高危样式** 与长按确认；默认改为**精确额度**。

---

## 8）模拟与预估：**先看会发生什么再签**

- **链上模拟**：已经用 `simulateContract` 做基本执行校验；  
- **协议模拟**：对复杂 swap/路由，可先 **静态 call** 获取返回值（例如 `previewRedeem / getQuote`），在确认框显示“**最差可得**”与 **滑点**。
- **失败预判**：常见 revert 原因（余额不足/授权不足/过期）在确认前即检查，避免“签了也白签”。

---

## 9）Gas 费与 EIP-1559

```ts
import { publicClient } from './client';
const fee = await publicClient.estimateFeesPerGas(); 
// { maxFeePerGas, maxPriorityFeePerGas } 可用于 UI “标准/快速/极速”档
```

- 对新手：默认“标准”，且**不允许大幅加价**（设置上限）。
- 对专家：开启“自定义费率”，但配**风险提示**（过高费率=被夹的风险与浪费）。

---

## 10）账户抽象（ERC-4337）与 Session Keys（实验性）

- **4337**：通过 **bundler** 提交 `UserOperation`，实现“批量动作、代付 gas、社交恢复”。前端 UX：**动作列表**一次确认，**支付方**选择（自付/赞助）。
- **Session Keys/授权指令**：为短期/特定合约授予“会话密钥”，免重复签（需钱包/协议支持）。  
- **提示**：这些能力的**钱包支持**与**生态一致性**差异较大，UI 上要明确标注“实验性”，并提供**快速失效/吊销**入口。

---

## 11）代币与授权管理页（你自己的“安全中心”）

- **持仓**：列出用户常用代币余额、价格（如果有）。
- **授权**：按代币罗列所有 spender 的 allowance（`allowance(owner, spender)` / 721/1155 的 `isApprovedForAll`），提供**一键撤销**。
- **最近交互**：展示近 N 条交易（哈希、函数、结果、gas 花费）。
- **风控**：命中黑名单/风险合约时标红（维护你自己的警示列表）。

---

## 12）测试与可观测性

- E2E 场景：**余额不足/授权不足/过期**，**用户拒绝/切链失败**，**模拟失败**，**nonce 冲突**。
- 指标：`connect_success/failure`、`tx_sent/confirmed/failed`、`avg_confirm_time`、`reject_rate`。
- 采样保存**签名前后的模拟摘要**（不含私密数据），用于回放与复盘。

---

## 13）“最小可用”整合示例（读/写/签名/确认）

```ts
// example/transfer-with-confirm.ts
import erc20 from '../abi/erc20.json' assert { type: 'json' };
import { publicClient, walletClient } from '../web3/client';
import { explainEvmError } from '../web3/errors';

export async function transferToken({
  token, to, amount, account
}: { token:`0x${string}`; to:`0x${string}`; amount:bigint; account:`0x${string}` }) {
  try {
    // 1) 前置校验
    const bal = await publicClient.readContract({ address: token, abi: erc20, functionName:'balanceOf', args:[account] });
    if (bal < amount) throw new Error('余额不足');

    // 2) 模拟
    const { request, result } = await publicClient.simulateContract({
      address: token, abi: erc20, functionName:'transfer', args:[to, amount], account
    });

    // 3) 安全确认 UI（伪）：展示 to/amount/预估 gas/result
    await openSafetyConfirm({
      title: '发送代币',
      lines: [
        `从 ${short(account)} → ${short(to)}`,
        `数量：${format(amount)} TOKEN`,
        `预计结果：${String(result)}`
      ]
    });

    // 4) 发送 & 等待
    const hash = await walletClient.writeContract(request);
    const receipt = await publicClient.waitForTransactionReceipt({ hash });
    return receipt;
  } catch (e) {
    throw new Error(explainEvmError(e));
  }
}

const short = (a:string)=>a.slice(0,6)+'...'+a.slice(-4);
const format = (n:bigint)=>Number(n)/1e18;
async function openSafetyConfirm(_: any){ /* 弹框实现留给你 */ }
```

---

## 14）安全 UX 清单（落地版）✅

- [ ] 链/账户明确显示，变更时弹条提示。  
- [ ] 写操作必经 `simulate`，失败原因在签名前抛出。  
- [ ] 确认框包含：**函数名/对象/代币流向/授权变更/费用/撤销入口**。  
- [ ] `approve` 默认**精确额度**，`MAX_UINT` 需要长按二次确认。  
- [ ] 事务串行分配 nonce，提供 **加速/撤销** 按钮。  
- [ ] 错误翻译 + 原始错误可复制。  
- [ ] 登录用 **SIWE**，绑定域/链/过期/nonce。  
- [ ] 提供“授权管理”页：列出并一键撤销。  
- [ ] 观测埋点与回放（不存私密数据）。  

---

### 结语

合约交互不是“点了就行”，而是**协议 + 模拟 + 解释 + 授权边界**的一整套组合拳。把不确定性搬到**签名前**搞清楚，把**撤销与自救**入口放在用户眼前，你的 DApp 会既顺手、又安心。🛡️✨
