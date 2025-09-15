# 7.2 原生 Web Components / Custom Elements 🧩

本章目标：在**不依赖框架**的前提下，用 **Custom Elements + Shadow DOM + Slots + ElementInternals** 写出**可复用、可无障碍（a11y）、可测试、可主题**的组件原语；顺手讲透 **属性/属性反射**、**样式隔离**、**表单联动**、**事件契约**、**跨框架集成** 与 **性能/安全**。

---

## 0) 核心心法（先立规矩）

- **HTML 优先**：组件 = 语义标签 + 无障碍角色 + 最少 JS。
- **状态内聚**：组件内部状态与 DOM 一一对应；对外只暴露**清晰的属性/方法/事件**。
- **样式可定制**：Shadow DOM 隔离，但**给出口**（CSS 自定义属性、`::part`、`exportparts`）。
- **框架中立**：对 React/Vue/Svelte/Angular 都只是一个 DOM 节点——**别绑死**某个框架的运行时。
- **渐进增强**：无 JS 时仍能显示基本内容（尤其表单与数据列表）。

---

## 1) Custom Elements：从“会动的标签”开始

### 1.1 定义一个最小组件
```ts
// x-badge.ts
class XBadge extends HTMLElement {
  static observedAttributes = ['type', 'text']; // 监听这两个属性变化
  #root = this.attachShadow({ mode: 'open' });

  constructor() {
    super();
    this.#root.innerHTML = `
      <style>
        :host { display: inline-flex; font: 12px/1.6 system-ui; }
        .badge { border-radius: 999px; padding: .2em .6em; }
        .badge[data-type="info"]  { background:#e0f2fe; color:#0369a1; }
        .badge[data-type="warn"]  { background:#fef3c7; color:#92400e; }
        .badge[data-type="error"] { background:#fee2e2; color:#991b1b; }
      </style>
      <span class="badge" part="badge" data-type="info"><slot></slot></span>
    `;
  }

  attributeChangedCallback(name: string, _old: string | null, v: string | null) {
    const el = this.#root.querySelector('.badge') as HTMLElement;
    if (name === 'type') el.dataset.type = v ?? 'info';
    if (name === 'text' && v != null) el.textContent = v; // 支持 <x-badge text="...">
  }

  // 属性 <-> 属性反射（Property/Attribute Reflection）
  get type() { return this.getAttribute('type') ?? 'info'; }
  set type(v: string) { this.setAttribute('type', v); }
}
customElements.define('x-badge', XBadge);

// 用法：
// <x-badge type="warn">即将维护</x-badge>
// document.querySelector('x-badge')!.type = 'error';
```

**要点速记**
- **`static observedAttributes`**：列出需要监听的 HTML 属性。  
- **属性反射**：让 `el.type = 'warn'` 与 `el.setAttribute('type','warn')` 等价。  
- **`part="badge"`**：供外部使用 `::part(badge)` 定制样式。

---

## 2) Shadow DOM & Slots：隔离与组合

### 2.1 Shadow DOM 模式
- `mode: 'open'`：可通过 `el.shadowRoot` 访问（调试/样式注入更方便）。  
- `mode: 'closed'`：完全封闭（更安全，但不易调试/测试）。

### 2.2 插槽（Slots）
```ts
// x-card.ts
class XCard extends HTMLElement {
  #root = this.attachShadow({ mode: 'open' });
  constructor() {
    super();
    this.#root.innerHTML = `
      <style>
        :host { display:block;border:1px solid #e5e7eb;border-radius:12px;padding:12px }
        .title ::slotted(*) { font-weight:600; }
      </style>
      <div class="title"><slot name="title"></slot></div>
      <div class="content"><slot></slot></div>
      <div class="extra"><slot name="extra"></slot></div>
    `;
  }
}
customElements.define('x-card', XCard);

/* 用法：
<x-card>
  <h3 slot="title">标题</h3>
  正文……
  <button slot="extra">操作</button>
</x-card>
*/
```

> `::slotted(selector)` 只能选中**插槽的直接子节点**；深层需要组件提供 `part` 或 CSS 变量。

### 2.3 Declarative Shadow DOM（DSD，声明式）
```html
<x-card>
  <template shadowrootmode="open">
    <style>/* 与 attachShadow 一样 */</style>
    <slot></slot>
  </template>
  这是投影内容
</x-card>
```
> DSD 让**服务端渲染**时就能有 Shadow DOM；现代浏览器已普遍支持。

---

## 3) 可主题化：CSS 变量 + `::part` + `exportparts`

```ts
// x-button.ts（开放样式钩子）
this.#root.innerHTML = `
  <style>
    :host { --btn-bg:#111827; --btn-fg:#fff; --btn-radius:8px; --btn-pad:.6em 1em; }
    button {
      background: var(--btn-bg); color: var(--btn-fg);
      border-radius: var(--btn-radius); padding: var(--btn-pad); border:0;
    }
  </style>
  <button part="button"><slot></slot></button>
`;
```

外部定制：
```css
x-button::part(button) { font-weight: 600; }
x-button { --btn-bg:#2563eb; --btn-fg:#fff; }
```

如果内部还有子组件，需要把它们的 `part` 往外层转发：
```html
<!-- 父组件模板 -->
<div part="wrapper">
  <x-icon part="icon"></x-icon>
</div>
<!-- 父元素标签上 -->
<host exportparts="icon:button-icon"></host>
```
> 这样外部可以写 `x-parent::part(button-icon)` 直接控制深层子组件的样式。

---

## 4) 表单联动：**Form-associated Custom Elements**（FACE）

用 `ElementInternals` 让自定义组件像原生 `<input>` 一样参与表单提交/重置/约束校验。

```ts
// x-switch.ts
class XSwitch extends HTMLElement {
  static formAssociated = true; // 开启 FACE
  #root = this.attachShadow({ mode: 'open' });
  #internals = this.attachInternals();
  #on = false;

  constructor() {
    super();
    this.#root.innerHTML = `
      <style>:host{display:inline-block;}</style>
      <button part="control" role="switch" aria-checked="false"></button>
    `;
    this.#root.querySelector('button')!.addEventListener('click', () => this.toggle());
  }

  connectedCallback() { this.#reflect(); }

  // 表单接口
  formAssociatedCallback(_form: HTMLFormElement | null) {/* 可感知挂接 */}
  formDisabledCallback(disabled: boolean) {
    (this.#root.querySelector('button') as HTMLButtonElement).disabled = disabled;
  }
  formResetCallback() { this.#on = false; this.#reflect(); }
  formStateRestoreCallback(state: any) { this.#on = !!state; this.#reflect(); }

  toggle() {
    this.#on = !this.#on;
    this.#reflect();
    // 把值塞进表单（提交时会带上该值）
    this.#internals.setFormValue(this.#on ? 'on' : null);
  }

  #reflect() {
    const btn = this.#root.querySelector('button')!;
    btn.setAttribute('aria-checked', String(this.#on));
    this.#internals.ariaChecked = this.#on ? 'true' : 'false'; // 可通过 Internals 反射 ARIA
    this.toggleAttribute('checked', this.#on);
  }
}
customElements.define('x-switch', XSwitch);

/* 用法：
<form>
  <x-switch name="agree"></x-switch>
  <button>提交</button>
</form>
*/
```

> 其它回调：`checkValidity()`、`reportValidity()`、`setValidity()` 也通过 `ElementInternals` 提供。

---

## 5) 事件契约：`CustomEvent` 与“受控/非受控”

```ts
// x-counter.ts
class XCounter extends HTMLElement {
  #root = this.attachShadow({ mode: 'open' });
  #value = 0;

  constructor(){
    super();
    this.#root.innerHTML = `<button id="dec">-</button><span id="v">0</span><button id="inc">+</button>`;
    this.#root.getElementById('inc')!.addEventListener('click', () => this.setValue(this.#value + 1));
    this.#root.getElementById('dec')!.addEventListener('click', () => this.setValue(this.#value - 1));
  }

  get value(){ return this.#value; }
  set value(v: number){ this.setValue(Number(v)); }

  setValue(v: number){
    if (v === this.#value) return;
    this.#value = v;
    this.#root.getElementById('v')!.textContent = String(v);
    // 像原生 input 一样派发可冒泡、可组合的事件
    this.dispatchEvent(new CustomEvent('change', { detail: { value: v }, bubbles: true }));
  }
}
customElements.define('x-counter', XCounter);

/* 用法：
<x-counter id="c"></x-counter>
<script>
  c.addEventListener('change', e => console.log(e.detail.value));
  // 受控模式
  c.value = 10;
</script>
*/
```

**建议**
- 事件名使用**小写**与原生对齐（`change`/`input`/`toggle`），并在 `detail` 传结果。  
- **不要**在 `constructor` 里触发事件（元素尚未连接到文档）；放在 `connectedCallback` 或交互后。

---

## 6) 性能与生命周期

- **升级（upgrade）**：当你调用 `customElements.define()` 时，已在 DOM 的同名标签会被“升级”为你的类实例。**避免**在构造函数做重活；把 I/O 放到 `connectedCallback`。  
- **`connectedCallback`/`disconnectedCallback`**：在此挂接/清理监听器、计时器、观察器（IntersectionObserver/ResizeObserver）。  
- **变化监听**：  
  - 属性变更 → `attributeChangedCallback`（谨慎监听，减少触发频率）。  
  - 内容变更 → `MutationObserver`（用于观测 slot 内容变化）。  
- **样式与布局**：优先 `transform/opacity`；避免强制同步布局（读写分离）。  
- **Constructable Stylesheets**（可共享 CSS 实例）：
  ```ts
  const sheet = new CSSStyleSheet();
  sheet.replaceSync(`:host{display:block}`);
  shadowRoot.adoptedStyleSheets = [sheet];
  ```

---

## 7) 无障碍（a11y）与焦点管理 ♿

- 选择正确的**语义元素**（按钮就用 `<button>`；对话框配 `role="dialog"`、`aria-modal="true"`）。  
- 焦点循环：弹层内可加 **FocusTrap**；关闭时把焦点还给触发器。  
- 名称与关系：`aria-labelledby`/`aria-describedby`；`aria-controls` 关联控制与面板。  
- 使用 **ElementInternals** 的 `aria*` 反射与 `states` 暴露开关/选中等状态。  
- 支持 **`prefers-reduced-motion`**：对重动画降级，见 5.3。

---

## 8) 与框架集成（React/Vue/Svelte/Angular）

- **React**：  
  - 属性是**字符串反射**；非字符串/函数 props（对象、数组）请**通过 `ref` 直接赋值**。  
  - 自定义事件**不要指望 `onFoo`** 自动绑定；在 `useEffect` 里 `el.addEventListener('change', handler)`。  
  - 受控值：通过 `ref.current.value = ...` 或属性反射保持同步。  
- **Vue 3**：  
  - 默认支持自定义元素；在构建工具里将 `isCustomElement` 白名单（`compilerOptions.isCustomElement`）。  
  - 自定义事件可用 `v-on` 监听（`@change="..."`）。  
- **Svelte**：原生支持，`on:change` 监听自定义事件。  
- **Angular**：需要在 `schemas` 里加入 `CUSTOM_ELEMENTS_SCHEMA`。

> 组件要**框架无关**：对外暴露**标准 DOM API**（属性/方法/事件），别依赖宿主框架的运行时。

---

## 9) 安全与隔离

- **严禁**把不可信内容直接塞 `innerHTML`；必要时用可信模板或可靠的 sanitizer。  
- Shadow DOM 隔离**样式**，不隔离**脚本**——XSS 规则一视同仁。  
- `closed` shadow 不是安全边界，只是调试隐藏。

---

## 10) 进阶示例：Headless Tabs（可换皮）

```ts
// x-tabs.ts（行为 + a11y，外观留给 ::part）
class XTabs extends HTMLElement {
  static observedAttributes = ['value'];
  #root = this.attachShadow({ mode: 'open' });
  #value = '';

  constructor(){
    super();
    this.#root.innerHTML = `
      <style>
        :host { display:block; }
        [part="list"] { display:flex; gap:.5rem; }
        [part="tab"][aria-selected="true"] { font-weight:600; }
        [part="panel"] { padding:.5rem 0; }
      </style>
      <div part="list" role="tablist"><slot name="tab"></slot></div>
      <div part="panels"><slot></slot></div>
    `;
    this.addEventListener('click', e => {
      const tab = (e.target as HTMLElement).closest('[slot="tab"]') as HTMLElement | null;
      if (tab && tab.id) this.value = tab.id;
    });
    this.addEventListener('keydown', e => {
      const tabs = this.#tabs();
      const idx = tabs.findIndex(t => t.id === this.#value);
      if (e.key === 'ArrowRight') tabs[(idx + 1) % tabs.length]?.focus();
      if (e.key === 'ArrowLeft')  tabs[(idx - 1 + tabs.length) % tabs.length]?.focus();
    });
  }

  connectedCallback(){ if (!this.value) this.value = this.#tabs()[0]?.id ?? ''; this.#reflect(); }
  attributeChangedCallback(n: string, _o: string | null, v: string | null){ if (n === 'value') { this.#value = v ?? ''; this.#reflect(); } }

  get value(){ return this.#value; }
  set value(v: string){ this.setAttribute('value', v); this.dispatchEvent(new CustomEvent('change', { detail:{ value:v }, bubbles:true })); }

  #tabs(){ return [...this.querySelectorAll<HTMLElement>('[slot="tab"]')]; }
  #panels(){ return [...this.querySelectorAll<HTMLElement>('[role="tabpanel"]')]; }

  #reflect(){
    const tabs = this.#tabs(), panels = this.#panels();
    for (const t of tabs) {
      t.setAttribute('role','tab');
      t.setAttribute('tabindex', t.id === this.#value ? '0' : '-1');
      t.setAttribute('aria-selected', String(t.id === this.#value));
    }
    for (const p of panels) {
      p.hidden = p.getAttribute('aria-labelledby') !== this.#value;
    }
  }
}
customElements.define('x-tabs', XTabs);

/* 用法（皮肤可自定义）：
<x-tabs value="t1" id="tabs">
  <button id="t1" slot="tab" part="tab">概览</button>
  <button id="t2" slot="tab" part="tab">设置</button>

  <section role="tabpanel" aria-labelledby="t1" part="panel">内容 1</section>
  <section role="tabpanel" aria-labelledby="t2" part="panel">内容 2</section>
</x-tabs>

# 外部样式：
x-tabs::part(tab){ padding:.4rem .8rem; border-radius:8px; }
x-tabs [slot="tab"][aria-selected="true"]{ background:#eef2ff; }
*/
```

---

## 11) 测试与发布

- **单测**：`@web/test-runner` 或 Vitest + `happy-dom`/JSDOM。  
- **可访问性**：Playwright + Axe；或在 Storybook 里跑 a11y 插件（见 4.1）。  
- **打包**：使用 ESM 输出（`"type":"module"`），避免编译成私有运行时；如需 polyfill，**按需引入**（尤其旧浏览器的 `:focus-visible`、`adoptedStyleSheets`）。

---

## 12) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 构造器做重活 / 发请求 | 首次渲染卡顿、升级阻塞 | 把 I/O 放到 `connectedCallback`，按需懒加载 |
| 把内部实现细节暴露到 Light DOM | 后续无法维护 | 对外只暴露属性/方法/事件；样式走 `::part`/变量 |
| 不做属性反射 | 跨框架控制困难 | `get/set` + `observedAttributes` 打通 |
| 自定义事件名五花八门 | 难以记忆/绑定 | 复用原生语义（`change`/`input`/`toggle`） |
| Shadow 封闭但又要外部改样式 | 最终全用 `!important` | 预留 CSS 变量与 `::part`/`exportparts` |
| 把不可信 HTML 塞进组件 | XSS | 严格消毒或仅文本渲染 |

---

## 13) 提交前检查清单 ✅

- [ ] 对外 API：**属性/方法/事件**三清楚；有文档。  
- [ ] Shadow DOM + Slots 正确使用；提供 `::part` 与 CSS 变量作为主题接口。  
- [ ] a11y：语义、焦点、键盘、ARIA 全覆盖；`prefers-reduced-motion` 友好。  
- [ ] 表单组件使用 **ElementInternals** 接入 FACE。  
- [ ] 生命周期与性能：重活延后；监听器在 `disconnectedCallback` 清理。  
- [ ] 测试：单测覆盖状态迁移；E2E 验证交互与样式钩子。

---

## 14) 练习 🏋️

1. 写一个 `x-dialog`：`role="dialog"`、焦点陷阱、`Escape` 关闭、`::part(backdrop|panel)` 主题接口，支持 `open` 属性与 `show()/close()` 方法。  
2. 写一个 `x-rating`（星级评分）：键盘左右选择、`change` 事件、`value` 属性反射，导出 `::part(star)` 供外部换皮。  
3. 把 `x-tabs` 改造成 **表单可联动**：切换页签后写入隐藏输入，表单提交带上当前 tab 值。  

---

**小结**：Web Components 不是“与框架对抗”，而是**做底层原语**。当你的组件以标准 DOM 协议（属性/方法/事件）说话，外面套 React/Vue/Svelte 都没压力；Shadow DOM + FACE + `::part` 让它既安全、可主题，又能无障碍上生产。🛠️🚀
