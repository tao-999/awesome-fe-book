# 29.1 钱包连接：EIP-1193 / EIP-6963 🔌🦊🦝

> 把“我是谁（账户）”“我在哪（链）”“我能干啥（签名/交易）”变成稳定、可观测的**连接协议**。本章聚焦两大基石：
>
> - **EIP-1193**：浏览器注入 Provider 的统一接口（`.request()` + 事件）。
> - **EIP-6963**：多钱包并存时代的**发现/选择**规范（announce/request 事件）。
>
> 给你能直接抄走的 TS 适配层、Modal 选择器、链切换与常见坑位清单。🚀

---

## 0）速查心法

- **1193** = “怎么跟钱包说话”：`provider.request({method, params})`，再监听 `accountsChanged / chainChanged / connect / disconnect`。
- **6963** = “怎么**找到**一堆钱包并让用户选”：页面 `dispatchEvent('eip6963:requestProvider')`，各钱包通过 `eip6963:announceProvider` 回报 `{ info, provider }`。
- **不要自动连接**：首次必须用户手势触发 `eth_requestAccounts`；空闲用 `eth_accounts` 查看授权状态。
- **链 ID 一律十六进制字符串**（如 `0x1`、`0x2105`）。切链先 `wallet_switchEthereumChain`，报 `4902` 再 `wallet_addEthereumChain`。

---

## 1）最小可用 1193/6963 适配层（可直接搬）

> 文件：`src/wallet/bridge.ts` —— 原生、零依赖，Svelte/React/Vue 都能用。🎯

```ts
// src/wallet/bridge.ts
// —— 类型：简化版 EIP-1193 / EIP-6963 ——
// 你也可以引入社区 d.ts，这里内联方便开箱即用

export type EIP1193RequestArgs = { method: string; params?: unknown[] | object };
export interface EIP1193Provider {
  request<T = unknown>(args: EIP1193RequestArgs): Promise<T>;
  on(event: string, handler: (...args: any[]) => void): void;
  removeListener(event: string, handler: (...args: any[]) => void): void;
  isMetaMask?: boolean;
  isRabby?: boolean;
  isTrust?: boolean;
  // ...其他标志
}
export type EIP6963ProviderDetail = {
  info: { uuid: string; name: string; icon: string; rdns: string };
  provider: EIP1193Provider;
};

// —— 发现：按 6963 收集所有注入钱包 ——
// 也兼容老式 window.ethereum / ethereum.providers
export async function discoverProviders(timeoutMs = 150): Promise<EIP6963ProviderDetail[]> {
  const found: EIP6963ProviderDetail[] = [];

  const onAnnounce = (e: Event) => {
    const detail = (e as CustomEvent).detail as EIP6963ProviderDetail;
    if (!found.some(x => x.info.uuid === detail.info.uuid)) found.push(detail);
  };
  window.addEventListener('eip6963:announceProvider', onAnnounce as any);

  // 触发广播请求
  window.dispatchEvent(new Event('eip6963:requestProvider'));

  // 兜底：老注入
  const legacy = (window as any).ethereum as EIP1193Provider | undefined;
  if (legacy) {
    const candidates: EIP1193Provider[] = (legacy as any).providers?.length
      ? (legacy as any).providers
      : [legacy];

    for (const p of candidates) {
      const name =
        (p as any).isMetaMask ? 'MetaMask' :
        (p as any).isRabby    ? 'Rabby'    :
        (p as any).isTrust    ? 'Trust'    : 'Injected';
      found.push({
        info: {
          uuid: crypto.randomUUID(),
          name,
          icon: '', // 可在 UI 层按 name 映射本地图标
          rdns: 'injected.legacy'
        },
        provider: p
      });
    }
  }

  // 等一小会儿收齐广播
  await new Promise(r => setTimeout(r, timeoutMs));
  window.removeEventListener('eip6963:announceProvider', onAnnounce as any);
  // 去重（按 provider 引用/uuid）
  return dedupe(found);
}

function dedupe(list: EIP6963ProviderDetail[]) {
  const seen = new Set<string>();
  return list.filter(it => (seen.has(it.info.uuid) ? false : (seen.add(it.info.uuid), true)));
}

// —— 连接/断开/切链/资产观察 ——
// 状态管理建议交给你自己的 store（pinia/zustand/svelte store）
export type WalletState = {
  provider?: EIP1193Provider;
  accounts: string[];
  chainId?: `0x${string}`;
};

export async function connect(detail: EIP6963ProviderDetail): Promise<WalletState> {
  const provider = detail.provider;
  const accounts = await provider.request<string[]>({ method: 'eth_requestAccounts' });
  const chainId = await provider.request<`0x${string}`>({ method: 'eth_chainId' });
  return { provider, accounts, chainId };
}

export async function disconnect(state: WalletState) {
  // 1193 没有强制的 disconnect；部分钱包有自定义方法或通过 connector（如 WalletConnect）断开
  state.provider = undefined;
  state.accounts = [];
  state.chainId = undefined;
}

export async function switchChain(provider: EIP1193Provider, chainIdHex: `0x${string}`, addIfMissing?: { chainName: string; rpcUrls: string[]; nativeCurrency: { name: string; symbol: string; decimals: number }; blockExplorerUrls?: string[] }) {
  try {
    await provider.request({ method: 'wallet_switchEthereumChain', params: [{ chainId: chainIdHex }] });
  } catch (e: any) {
    // 4902: Unrecognized chain
    if (e?.code === 4902 && addIfMissing) {
      await provider.request({
        method: 'wallet_addEthereumChain',
        params: [{ chainId: chainIdHex, ...addIfMissing }],
      });
      // 再切一次
      await provider.request({ method: 'wallet_switchEthereumChain', params: [{ chainId: chainIdHex }] });
    } else {
      throw e;
    }
  }
}

export async function watchAsset(provider: EIP1193Provider, token: { address: `0x${string}`; symbol: string; decimals: number; image?: string }) {
  return provider.request({
    method: 'wallet_watchAsset',
    params: { type: 'ERC20', options: token } as any
  });
}

export function onCoreEvents(provider: EIP1193Provider, handlers: {
  accountsChanged?: (accs: string[]) => void;
  chainChanged?: (hex: `0x${string}`) => void;
  connect?: (info: { chainId: `0x${string}` }) => void;
  disconnect?: (err: any) => void;
}) {
  if (handlers.accountsChanged) provider.on('accountsChanged', handlers.accountsChanged as any);
  if (handlers.chainChanged)   provider.on('chainChanged',   handlers.chainChanged   as any);
  if (handlers.connect)        provider.on('connect',        handlers.connect        as any);
  if (handlers.disconnect)     provider.on('disconnect',     handlers.disconnect     as any);
}

// —— 小工具 —— 
export const hex = (n: number) => ('0x' + n.toString(16)) as `0x${string}`;
```

---

## 2）极简“连接弹窗”示例（原生 + 任意框架可用）

> 文件：`src/wallet/connect-modal.ts` —— 用 6963 收集钱包 → 渲染选择 → 点击连接。  
> UI 自行替换成你家的 Design System，这里重逻辑轻样式。✨

```ts
// src/wallet/connect-modal.ts
import { discoverProviders, connect, onCoreEvents, switchChain, hex } from './bridge';

export function openConnectModal(root: HTMLElement, opts?: { targetChain?: number; chainMeta?: Parameters<typeof switchChain>[2] }) {
  root.innerHTML = `
    <div class="modal">
      <h3>选择钱包</h3>
      <div class="grid" id="wallets"></div>
      <style>
        .modal{padding:16px; width:360px}
        .grid{display:grid; gap:8px}
        button{display:flex; align-items:center; gap:8px; padding:10px 12px; border:1px solid #eee; border-radius:8px; cursor:pointer}
        img{width:20px;height:20px}
      </style>
    </div>
  `;

  const grid = root.querySelector<HTMLDivElement>('#wallets')!;

  discoverProviders().then(list => {
    if (!list.length) {
      grid.innerHTML = `<div>未检测到浏览器钱包，建议安装 MetaMask/Rabby，或改用 WalletConnect。</div>`;
      return;
    }
    grid.innerHTML = '';
    for (const d of list) {
      const btn = document.createElement('button');
      btn.innerHTML = `<img src="${d.info.icon || fallbackIcon(d.info.name)}" alt=""><span>${d.info.name}</span>`;
      btn.onclick = async () => {
        try {
          const state = await connect(d);
          onCoreEvents(state.provider!, {
            accountsChanged: (a) => console.log('accountsChanged', a),
            chainChanged:    (c) => console.log('chainChanged', c),
            disconnect:      (e) => console.log('disconnect', e),
          });

          // 可选：要求特定链（例如 Base 主网 8453）
          if (opts?.targetChain) {
            await switchChain(
              state.provider!,
              hex(opts.targetChain),
              opts.chainMeta // 缺链时的链信息
            );
          }

          console.log('connected:', state);
          root.dispatchEvent(new CustomEvent('connected', { detail: state }));
        } catch (e) {
          console.error(e);
          alert('连接失败：' + (e as any)?.message);
        }
      };
      grid.appendChild(btn);
    }
  });
}

function fallbackIcon(name: string) {
  // 你可以在此映射本地图标
  return 'data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="20" height="20"><circle cx="10" cy="10" r="9" fill="%23eee"/></svg>';
}
```

**在页面使用：**

```ts
import { openConnectModal } from '@/wallet/connect-modal';

const mount = document.getElementById('connect-modal')!;
openConnectModal(mount, {
  targetChain: 8453, // Base 主网（示例）
  chainMeta: {
    chainName: 'Base',
    rpcUrls: ['https://base-rpc.publicnode.com'],
    nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
    blockExplorerUrls: ['https://basescan.org']
  }
});

mount.addEventListener('connected', (e: any) => {
  const { accounts, chainId } = e.detail;
  // 更新你的全局状态管理（Zustand/Pinia/Svelte store）
  console.log('✅ 已连接', accounts[0], '链', chainId);
});
```

---

## 3）在 React / Svelte 里怎么落（示意）

**React**

```tsx
// ConnectButton.tsx
import { useState } from 'react';
import { discoverProviders, connect, onCoreEvents } from '@/wallet/bridge';

export function ConnectButton() {
  const [addr, setAddr] = useState<string | null>(null);

  const onClick = async () => {
    const providers = await discoverProviders();
    // 简化：若只有一个，就直接用；否则弹你自己的 Modal
    const picked = providers[0];
    const st = await connect(picked);
    setAddr(st.accounts[0] ?? null);
    onCoreEvents(st.provider!, {
      accountsChanged: (a) => setAddr(a[0] ?? null),
      disconnect: () => setAddr(null)
    });
  };

  return <button onClick={onClick}>{addr ? short(addr) : '连接钱包'}</button>;
}
const short = (a: string) => a ? a.slice(0, 6) + '...' + a.slice(-4) : '';
```

**Svelte**

```svelte
<script lang="ts">
  import { discoverProviders, connect, onCoreEvents } from '@/wallet/bridge';
  let addr: string | null = null;

  async function doConnect() {
    const list = await discoverProviders();
    const picked = list[0];
    const st = await connect(picked);
    addr = st.accounts[0] ?? null;
    onCoreEvents(st.provider!, {
      accountsChanged: (a) => addr = a[0] ?? null,
      disconnect: () => addr = null
    });
  }
</script>

<button on:click={doConnect}>{addr ? addr.slice(0,6)+'...'+addr.slice(-4) : '连接钱包'}</button>
```

---

## 4）Wagmi/Viem 一把梭（可与 6963 共存）

> 你已用过 wagmi，这里给一个**稳态配置**：注入 + WalletConnect + Coinbase。  
> wagmi 近两个大版本已支持 6963/多注入的自动发现；若遇到“多个钱包覆盖”的奇怪行为，记得更新到近版本并开启注入 shim。

```ts
// web3/config.ts
import { http, createConfig } from 'wagmi';
import { base, mainnet } from 'wagmi/chains';
import { injected, walletConnect, coinbaseWallet } from '@wagmi/connectors';

export const wagmiConfig = createConfig({
  chains: [base, mainnet],
  multiInjectedProviderDiscovery: true,   // 多注入发现（配合 6963）
  connectors: [
    injected({ shimDisconnect: true }),   // 浏览器钱包（MetaMask/Rabby/Trust...）
    walletConnect({ projectId: 'YOUR_WC_ID' }),
    coinbaseWallet({ appName: 'YourApp' }),
  ],
  transports: {
    [base.id]: http('https://base-rpc.publicnode.com'),
    [mainnet.id]: http()
  }
});
```

**常见错误排查：**

- `Provider not found`：多半是 connectors 未配置/版本过旧；或在非安全上下文（file://）运行。
- **Trust 扩展**：部分环境只在移动端内置浏览器，与桌面扩展行为不同；桌面推荐用 Injected + WalletConnect 兼容。
- **链不匹配**：wagmi 已有 `switchChain` action；失败码 `4902` 时补链信息与上文相同。

---

## 5）高频 API 速查（1193）

| 目的 | `method` | 说明 |
|---|---|---|
| 请求账户 | `eth_requestAccounts` | 首次必须用户手势触发 |
| 获取现有账户 | `eth_accounts` | 未授权返回 `[]` |
| 当前链 | `eth_chainId` | 例如 `"0x2105"`（Base） |
| 切换链 | `wallet_switchEthereumChain` | `{ chainId: '0x...' }` |
| 添加链 | `wallet_addEthereumChain` | 不认识的链（报 4902）时 |
| 加入资产 | `wallet_watchAsset` | 添加 ERC-20 代币到钱包 |
| 发交易 | `eth_sendTransaction` | DApp 构造交易参数 |
| 签消息 | `personal_sign` / `eth_signTypedData_v4` | 用于登录/凭证 |

**事件**：`accountsChanged`, `chainChanged`, `connect`, `disconnect`（统一在适配层订阅）。

---

## 6）安全与 UX 清单 🛡️✨

- **不自动弹授权**：首次连接必须按钮触发；刷新仅用 `eth_accounts` 探测。
- **短地址显示**：`0x1234…abcd`，并提供“复制”。
- **链指示**：显式展示链名 + 切换入口；切换失败要有退路（禁用按钮+提示）。
- **权限与风险文案**：签名/交易前说明用途；签名使用 **EIP-4361 SIWE**（如有登录）。
- **持久化**：将所选钱包 `uuid` 与最后链写入 `localStorage`，下次优先渲染它在列表前排，不要**静默连接**。
- **手机端**：内置浏览器/钱包 App 内 WebView 优先用 **深链/WalletConnect**，PC 外链再提示安装扩展。

---

## 7）常见坑位与修复 🕳️🩹

- **十进制链 ID 乱用** → 一律用 hex 字符串（`0x…`），否则 `switchEthereumChain` 报错。
- **多钱包注入覆盖** → 启用 6963；不要只读 `window.ethereum`。
- **账户数组为空** → 用户锁定/未授权；正确处理空数组，不要误判为“连接成功”。
- **签名大小写问题** → 地址规范化为小写比较，显示再做 checksum（EIP-55）。
- **跨标签页状态不同步** → 监听 `storage` 事件或使用 BroadcastChannel 同步 UI。

---

## 8）集成路线图（团队版）

1. **桥接层**：落地本章 `bridge.ts`（1193/6963）。
2. **UI**：统一“连接 Modal + 当前状态 Chip + 切链 Dropdown”三个组件。
3. **状态**：统一 store（address/chainId/provider/detail），订阅核心事件更新。
4. **降级**：无注入 → WalletConnect；移动端优先深链/扫码。
5. **观测**：埋点 `connect_success/failure`、`switch_chain_*`、`sign/tx_*` 时延与错误码。

> 连接这件事别整花活：**发现 → 选择 → 授权 → 切链 → 使用**，每一步都给用户一个“能理解的反馈”，你的 DApp 就稳得像老狗。🫡

