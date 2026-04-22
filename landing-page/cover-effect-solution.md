# Dark → Light 覆蓋效果技術方案

landing page hero 區塊「白色內容區從下往上蓋住黑色 hero」的技術方案。需求：
- dark hero 內容**動態高度**（PM / 文案可自由加減段落）
- 用戶先**完整看完 dark 所有內容**，再觸發 cover 動畫
- cover 期間 dark 在原地「後退成卡片」（scale / blur / radius / opacity），不被推走

---

## 一、核心模型：三個獨立零件

| 零件 | 作用 | CSS 機制 |
|---|---|---|
| ① **Pin**（釘住） | 讓 dark 在一段滾動期間停留在視窗底 | `position: sticky; top: calc(100vh - var(--dark-h))` |
| ② **Overlap**（重疊） | 讓 light 進到 dark 的視覺空間 | `margin-top: -100vh` |
| ③ **Stack**（疊序） | 讓 light 渲染在 dark 上方 | `z-index: 2` vs `1` |

三者獨立可調。只有 ① 需要知道 dark 實際高度，② ③ 永遠是常數。

---

## 二、完整 CSS

```css
:root {
  --dark-h: 100vh;                     /* fallback；JS 會覆寫成實際像素 */
}

.stage {
  position: relative;
  height: calc(var(--dark-h) + 100vh); /* dark 自身高度 + 100vh cover 滾動範圍 */
  z-index: 1;
}

.dark {
  position: sticky;
  top: calc(100vh - var(--dark-h));    /* 負 top：dark 先自然滾完再釘在視窗底 */
  min-height: 100vh;                   /* 內容少於一屏時的保險 */
  /* height: auto，由內容決定 */
}

.light {
  position: relative;
  z-index: 2;
  margin-top: -100vh;                  /* 常數，跟 dark 高度無關 */
}
```

---

## 三、唯一要靠 JS 的部分：同步 dark 實際高度

```js
const dark = document.getElementById('dark');
const syncDarkHeight = () => {
  document.documentElement.style.setProperty('--dark-h', dark.offsetHeight + 'px');
};
syncDarkHeight();
new ResizeObserver(syncDarkHeight).observe(dark);
```

三行。`ResizeObserver` 會在以下情況自動觸發：
- 內容變動（PM 改文案、CMS 替換段落）
- 圖片 / 字體載入完成（late layout shift）
- 視窗縮放（影響 `100vh` 與 `min-height: 100vh`）
- 嵌入媒體（影片、Base 預覽元件）尺寸回報

都不用手動處理，公式自動重算。

---

## 四、滾動時序

假設 viewport = V（如 800px），`dark.offsetHeight` = H（由內容決定）：

| scroll 區間 | 狀態 | 畫面 |
|---|---|---|
| `0 → H − V` | dark 自然滾動 | 用戶看到 dark 內容從頂到底依序出現 |
| `H − V` | sticky 啟動，`dark.bottom` 接到視窗底 | dark 最後一屏完整可見（含 tagline） |
| `H − V → H` | sticky 維持，light 從視窗底爬上來蓋 | cover 動畫進行（scale / blur / radius / shadow 漸變） |
| `H` | cover 完成 | light 全屏覆蓋，dark 完全藏在後面 |
| `> H` | dark 解除 sticky，跟著頁面滾走 | 用戶繼續瀏覽 light 區塊 |

**關鍵**：所有時機點都自動跟著 H 變化。H 變 1000 → 2000，cover 啟動點從 `200` 自動移到 `1200`，無需手動調整。

---

## 五、cover 動畫的視覺參數（JS 驅動）

ScrollTrigger 方式或手寫 rAF + `light.getBoundingClientRect()` 都可：

```js
const t = clamp01((vh - light.rect.top) / vh);  // 0 → 1 的進度
// 然後把 t 餵給以下 CSS 變數做差值：
//   --ai-scale:        1   → 0.93
//   --ai-opacity:      1   → 0.55
//   --ai-blur:         0px → 4px
//   --ai-radius:       0px → 20px     (dark 自己的圓角)
//   --light-radius:    28px → 0px     (light 頂部圓角拉平)
//   --shadow-y:        0   → -28px    (light 向上拋的陰影)
//   --shadow-blur:     0   → 90px
//   --shadow-opacity:  0   → 0.55
```

這些參數可以按設計再微調，跟 cover 機制本身無關。

---

## 五·五、light 內容的進場漸入（reveal）

cover 動畫的「後半段」通常還要配一組 light 裡面的元素依序 fade-in + y 位移，避免白色區域「光溜溜地蓋上去」。這部分跟 cover 機制完全解耦，但用同一個 `t` 就能驅動。

### 實作

HTML 在要進場的元素上標 `data-reveal`：

```html
<div class="light">
  <h2 data-reveal>Built for every kind of work</h2>
  <p data-reveal>...</p>
  <div class="cards">
    <div class="card" data-reveal>...</div>
    <div class="card" data-reveal>...</div>
    <div class="card" data-reveal>...</div>
  </div>
</div>
```

CSS 給一個初始態，避免 JS 第一次跑之前元素閃現一下：

```css
[data-reveal] {
  opacity: 0;
  transform: translate3d(0, 28px, 0);
  will-change: opacity, transform;
}
```

JS 用同一個 cover 進度 `t` 分派給每個元素，加 stagger 和緩出曲線：

```js
const CONFIG = {
  revealStart:    0.2,   // t 到這個進度才開始 reveal（cover 先跑 20%）
  revealStagger:  0.05,  // 元素之間的間隔
  revealDuration: 0.3,   // 每個元素的淡入長度（在 t 軸上）
  revealY:        28,    // 初始 y 位移（px）
};

// 對齊 GSAP 的 power3.out
const easeOutPower3 = (t) => 1 - Math.pow(1 - t, 3);

const revealEls = Array.from(light.querySelectorAll('[data-reveal]'));

for (let i = 0; i < revealEls.length; i++) {
  const start  = CONFIG.revealStart + i * CONFIG.revealStagger;
  const raw    = clamp01((t - start) / CONFIG.revealDuration);
  const eased  = easeOutPower3(raw);
  const y      = lerp(CONFIG.revealY, 0, eased);
  revealEls[i].style.transform = `translate3d(0, ${y}px, 0)`;
  revealEls[i].style.opacity   = String(eased);
}
```

### 為什麼要 easing（不能用純 lerp）

線性差值會讓 opacity 和 y 同速變化，感覺生硬——像一個匀速從下面滑到定位的方塊。`power3.out`（`1 - (1-t)^3`）頭快尾慢：前半段快速出現，後半段「減速穩落到位」，整體感覺像真的有重量落地。

這是跟 GSAP timeline + `power3.out` 的效果對齊——用手寫 rAF 時必須顯式把緩出函數寫進去，不然會失去 index.html 那版本的質感。

### 時序

假設 `revealStart=0.2, revealStagger=0.05, revealDuration=0.3`，5 個元素：

| cover 進度 t | 元素狀態 |
|---|---|
| 0 → 0.2 | 所有元素 opacity=0（cover 先自己跑一段）|
| 0.2 → 0.5 | el[0] 淡入（完成於 0.5）|
| 0.25 → 0.55 | el[1] 淡入 |
| ... | 依 stagger 0.05 遞進 |
| 0.4 → 0.7 | el[4] 淡入（最後一個完成於 0.7）|
| > 0.7 | 全部穩定 opacity=1, y=0 |

所以 cover 後 30% 的滾動距離裡 reveal 都是全部到位的狀態，讓白色區自然銜接到後續的正文滾動。

### 可調參數對照

| 變數 | 小值 | 大值 |
|---|---|---|
| `revealStart` | 0（立刻開始） | 0.5（等 cover 跑一半才開始）|
| `revealStagger` | 0.02（齊刷刷）| 0.1（明顯一個個來） |
| `revealDuration` | 0.2（俐落）| 0.5（從容）|
| `revealY` | 0（純 fade）| 40+（明顯上升感）|

---

## 六、走過的坑（踩過才寫下來）

### 坑 1：`margin-top: 0vh`（研發原版的 bug）
沒有負 margin，light 和 stage 完全不重疊。scroll 到 light 出場時 dark 已經滾走，看起來是「推」上去而不是覆蓋。
**正解**：`margin-top: -100vh`。

### 坑 2：用 `padding-bottom: 100vh` 延伸 stage
padding 讓 stage 外觀變高了，**但 sticky 的 containing block 只看 content-box，padding 不算**。sticky 實際可活動範圍還是等於 dark.height，等於完全沒有 sticky 空間。
**正解**：要真正的 block-level content（例如 `.stage::after { content:''; display:block; height: 100vh }`），或直接把 `height` 寫成 `calc(var(--dark-h) + 100vh)`。

### 坑 3：想用 `position: sticky; bottom: 0` 免 JS
這個語意只適用於「從下方進入視窗」的元素（例如文章末尾的 sticky footer）。對於「在頂部、往上滾出」的 dark，`bottom: 0` 不會把它釘在視窗底——它只會自然滾走。方向反了。
**實測**：拿掉 transform / filter / will-change 之後 sticky 也沒啟動，確認是語意問題不是干擾問題。
**正解**：動態高度還是得靠 JS 算 `top`（`calc(100vh - var(--dark-h))`）。

### 坑 4：`margin-top = -dark.height` 不是通用公式
那個公式只對「dark 從 scroll=0 就 sticky、dark 底部永遠看不到」的版本成立。如果要「看完再蓋」，`margin-top` 永遠是 `-100vh`（常數）。
記住：`margin-top` 控制的是 **cover 動畫的滾動距離 = 一屏**，跟 dark 多高無關。

### 坑 5：dark 內容少於一屏時，sticky 行為異常
如果 dark 內容只有 400px（小於視窗），`top: calc(100vh - 400px)` 算出來是正值，sticky 會把 dark 往下推，漂在視窗中央。
**正解**：加 `min-height: 100vh`。這樣 dark 最少 100vh，負 top 保底為 0，退化成「從 scroll=0 就 sticky」的經典模式。

---

## 七、驗證

用 playwright 做的自動化驗證，把同一份 `dev-cover-dynamic.html` 在兩種不同內容長度下跑過：

| 場景 | dark 高度 | sticky 啟動點 | cover 完成點 |
|---|---|---|---|
| 少量內容 | 984px | scroll=184 | scroll=984 |
| 追加內容 | 1524px | scroll=724 | scroll=1524 |

兩種場景下：
- `dark.layout.bottom` 在 sticky 期間都被釘在 viewport.bottom = 800 ✓
- `light.rect.top` 從 800 遞減到 0，動畫進度對齊 ✓
- `ResizeObserver` 自動把 `--dark-h` 同步成新值，sticky 觸發點、cover 完成點全部自動重算 ✓
- un-scaled layout 還原算回去完全符合預期 ✓

---

## 八、檔案對照

| 檔案 | 用途 |
|---|---|
| `dev-cover-dynamic.html` | 隔離的 dark→light cover 原型，動態高度版，可直接交付研發 |
| `index.html` | 目前官網落地的完整 concept（使用 GSAP + ScrollTrigger + Lenis 做 4 段 cover）|

---

## 九、交付給研發時的重點

1. **三個零件獨立可調**，不要把 pin 的負 top 和 light 的 margin-top 視為同一公式
2. **JS 部分只有 3 行**（`ResizeObserver` + 寫變數），不需要額外 library
3. **不要用 `padding-bottom` 當 sticky spacer**，要用 `height: calc(...)` 或 `::after` 偽元素
4. **不要用 `sticky; bottom: 0` 想偷懶免 JS**，語意不對，實測不會釘住
5. `min-height: 100vh` 是 edge case 保險絲，不是可選項
6. cover 動畫的視覺參數（scale / blur 等）可以跟 PM / 設計調整，跟 cover 機制解耦
