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

本版本直接用 rAF + `getBoundingClientRect()` 推導進度，不依賴外部動畫 library：

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

如果白色區域「整塊光溜溜地蓋上去」看起來會很單薄——在 cover 的後半段讓 light 裡面的元素依序 fade-in + y 位移，畫面才會有節奏。這部分跟 cover 機制完全解耦（獨立調、不影響 cover 本身），但用同一個 `t` 進度就能驅動，不需要另外綁 scroll listener。

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

CSS 給一個初始態，避免 JS 第一次執行之前這些元素以完整樣式閃現一幀：

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
  revealStart:    0.2,   // t 到這個進度才開始 reveal（前 20% 讓 cover 自己先跑）
  revealStagger:  0.05,  // 相鄰元素之間的進入時差
  revealDuration: 0.3,   // 每個元素從 0 到完成花費的 t 長度
  revealY:        28,    // 元素初始位置相對終點的 y 偏移（px）
};

// 緩出三次方曲線：頭快尾慢
const easeOutCubic = (x) => 1 - Math.pow(1 - x, 3);

const revealEls = Array.from(light.querySelectorAll('[data-reveal]'));

for (let i = 0; i < revealEls.length; i++) {
  const start  = CONFIG.revealStart + i * CONFIG.revealStagger;
  const raw    = clamp01((t - start) / CONFIG.revealDuration);  // 0 → 1
  const eased  = easeOutCubic(raw);
  const y      = lerp(CONFIG.revealY, 0, eased);
  revealEls[i].style.transform = `translate3d(0, ${y}px, 0)`;
  revealEls[i].style.opacity   = String(eased);
}
```

### 為什麼要 easing（不能用純 lerp）

線性差值（`opacity = raw`, `y = lerp(28, 0, raw)`）會讓兩個屬性等速變化——每個元素看起來像一塊匀速從下方滑到定位的方塊，沒有「落地」的感覺。

`easeOutCubic`（`1 - (1-x)^3`）頭快尾慢：

```
linear          :  0 ─────────────── 1     （匀速）
easeOutCubic    :  0 ──────────── 1        （前段快速出現 → 後段減速穩落）
```

前半段進度條快速向前，元素很快達到 ~75% 的可見度；後半段減速，像真有重量般緩緩落到定位。同樣 0.3 的 t 長度，視覺上感覺像是「一瞬間出現 + 穩穩落地」，而不是「勻速爬完一段距離」。

### 時序

假設 `revealStart=0.2, revealStagger=0.05, revealDuration=0.3`，5 個元素：

| cover 進度 t | 元素狀態 |
|---|---|
| 0 → 0.2 | 全部 opacity=0（cover 先自己跑一段，白色區從底往上爬）|
| 0.2 → 0.5 | el[0] 進場 |
| 0.25 → 0.55 | el[1] 進場 |
| ... | 依 stagger 0.05 遞進 |
| 0.4 → 0.7 | el[4] 進場（最後一個完成於 t=0.7）|
| 0.7 → 1.0 | 全部穩定 opacity=1, y=0 |

後 30% 的滾動距離裡所有元素都已到位，讓白色區自然銜接到後續的正文滾動。

### 可調參數對照

| 變數 | 小值 | 大值 |
|---|---|---|
| `revealStart` | 0（cover 一開始就 reveal）| 0.5（等 cover 跑一半才開始）|
| `revealStagger` | 0.02（幾乎同時出現）| 0.1（明顯一個一個來）|
| `revealDuration` | 0.2（俐落）| 0.5（從容）|
| `revealY` | 0（純 fade，無位移）| 40+（明顯上升感）|

---

## 五·六、light 背景轉色（黑 → 白）

如果 light 一開始就是純白，cover 前半段畫面會有明顯的黑白對比——白色區從底部爬上來像切割一刀。如果想讓過渡更絲滑，可以讓 light 初始也是深色（甚至和 dark 同色），隨 cover 進度漸變到白。視覺上像「從黑暗中顯影」而不是「白色蓋上去」。

這層跟 cover、reveal 都解耦，用同一個 `t` 驅動就好。

### 實作

CSS 把 light 的背景和前景色改成用 RGB 通道 CSS 變數驅動，保留原本的 alpha 設計：

```css
:root {
  --light-bg-r: 255; --light-bg-g: 255; --light-bg-b: 255;
  --light-fg-r:  17; --light-fg-g:  17; --light-fg-b:  17;
}
.light {
  background: rgb(var(--light-bg-r), var(--light-bg-g), var(--light-bg-b));
  color:      rgb(var(--light-fg-r), var(--light-fg-g), var(--light-fg-b));
}
.light p {
  color: rgba(var(--light-fg-r), var(--light-fg-g), var(--light-fg-b), 0.62);
}
.card {
  border: 1px solid rgba(var(--light-fg-r), var(--light-fg-g), var(--light-fg-b), 0.08);
}
```

JS 根據 `bgStart / bgDuration` 切出一段獨立進度，`easeInOutCubic` 讓中段灰色快速通過：

```js
const CONFIG = {
  bgEnabled:   true,
  bgStart:     0.1,   // t 到這個進度才開始轉色
  bgDuration:  0.2,   // 轉色走多長（0.1 → 0.3 完成）
  bgFrom:      { r: 10, g: 10, b: 18 },     // 起始色（建議和 dark body 一致）
  bgTo:        { r: 255, g: 255, b: 255 },  // 結束色
  shadowFollowsBg: true,   // 陰影 opacity 跟著 bg 進度出現（黑底時不可見，白底時到位）
};

const easeInOutCubic = (x) => x < 0.5 ? 4*x*x*x : 1 - Math.pow(-2*x + 2, 3) / 2;

let bgProgress = 1;
if (CONFIG.bgEnabled) {
  const rawBg = clamp01((t - CONFIG.bgStart) / CONFIG.bgDuration);
  bgProgress  = easeInOutCubic(rawBg);
  const c = lerpRgb(CONFIG.bgFrom, CONFIG.bgTo, bgProgress);
  root.style.setProperty('--light-bg-r', c.r);
  root.style.setProperty('--light-bg-g', c.g);
  root.style.setProperty('--light-bg-b', c.b);
}
```

### 三個要配合處理的視覺細節

**① 陰影要跟背景同步出現**
`box-shadow: 0 -28px 90px rgba(0,0,0, 0.55)` 是黑色陰影，壓在黑底上等於沒效果、看不到邊界。解法是讓陰影 opacity 依 `bgProgress` 而不是 `t`——背景是黑時陰影不可見，背景變白時陰影同步增強，始終維持可見的分界線。

```js
const shadowT = (CONFIG.bgEnabled && CONFIG.shadowFollowsBg) ? bgProgress : t;
root.style.setProperty('--shadow-opacity', String(lerp(0, CONFIG.shadowOpacity, shadowT)));
```

**② 文字色策略**（黑底時文字的處理）
背景還是黑時，如果文字預設是深色（`#111`），那它會沉到背景裡看不見。三種策略：

| `fgMode` | 機制 | 優缺點 |
|---|---|---|
| `opacity` | 文字始終是終色 `#111`，只靠 reveal 的 `opacity` 淡入 | 最簡單；過渡中段元素「從黑中模糊顯形」，像曝光照片 |
| `reverse` | 文字色隨 `bgProgress` 從白漸變到黑 | 戲劇性最強，但中段會經過灰色，可讀性差 |
| `delay` | 文字始終是 `#111`，但整組 reveal 的 `revealStart` 自動推遲到背景快轉完才開始 | 背景先乾淨變白、文字再進場，分工清楚不會打架。**推薦預設** |

delay 模式的推遲公式：

```js
let effectiveRevealStart = CONFIG.revealStart;
if (CONFIG.bgEnabled && CONFIG.fgMode === 'delay') {
  effectiveRevealStart = Math.max(
    CONFIG.revealStart,
    CONFIG.bgStart + CONFIG.bgDuration * 0.8  // 背景走到 80% 時 reveal 起步
  );
}
```

**③ 起始色建議和 dark body 一致**
如果 `bgFrom` 跟 body 的 `background` 用同色（例如都是 `#0a0a12`），cover 前段 light 爬上來的瞬間用戶看不到分界，會誤以為 dark 自己在「長高」——這正是本方案要的「從暗中顯影」效果。如果分界太明顯會破壞沉浸感。

### 時序（預設 `bgStart=0.1, bgDuration=0.2, fgMode='delay'`）

| cover 進度 t | 背景 | 文字（delay 模式）|
|---|---|---|
| 0 → 0.1 | 全黑（`bgFrom`）| 不可見（opacity=0）|
| 0.1 → 0.3 | 快速從黑過渡到白 | 持續不可見，等待 |
| 0.3 → 0.46 | 已穩定在白 | 依 stagger 0.05 開始進場（`effectiveRevealStart = 0.26`）|
| 0.46 → 1.0 | 白 | 全部到位 |

這樣分工：cover 前 30% 是「黑色空間在後退」，中段 16% 是「空間亮起來」，後半才是「內容浮現」——每階段視覺焦點明確。

### 可調參數對照

| 變數 | 小值 | 大值 |
|---|---|---|
| `bgStart` | 0（cover 一開始就轉色）| 0.4（很晚才開始，前段維持全黑）|
| `bgDuration` | 0.1（刷過去）| 0.6（慢慢亮）|
| `bgFrom` | 和 dark 同色（無縫）| 明顯不同的深色（有邊界感）|
| `shadowFollowsBg` | false（陰影全程出現，黑底上失效）| true（陰影只在白底時可見）|

---

## 六、走過的坑（踩過才寫下來）

### 坑 1：漏了 `margin-top: -100vh` 的負 margin
沒有負 margin，light 和 stage 完全不重疊。scroll 到 light 出場時 dark 已經滾走，看起來是「推」上去而不是覆蓋。
**正解**：`margin-top: -100vh`。

### 坑 2：用 `padding-bottom: 100vh` 延伸 stage
padding 讓 stage 外觀變高了，**但 sticky 的 containing block 只看 content-box，padding 不算**。sticky 實際可活動範圍還是等於 dark.height，等於完全沒有 sticky 空間。
**正解**：要真正的 block-level content（例如 `.stage::after { content:''; display:block; height: 100vh }`），或直接把 `height` 寫成 `calc(var(--dark-h) + 100vh)`。

### 坑 3：想用 `position: sticky; bottom: 0` 免 JS
這個語意只適用於「從下方進入視窗」的元素（例如文章末尾的 sticky footer）。對於「在頂部、往上滾出」的 dark，`bottom: 0` 不會把它釘在視窗底——它只會自然滾走。方向反了。
**實測**：拿掉 transform / filter / will-change 之後 sticky 也沒啟動，確認是語意問題不是干擾問題。
**正解**：動態高度還是得靠 JS 算 `top`（`calc(100vh - var(--dark-h))`）。

### 坑 4：把 `margin-top` 寫成 `-dark.height`（直覺錯誤）
直覺上「light 要蓋住 dark → margin-top 應該等於 dark 的高度」，但這只對應一種特殊情況：dark 從 scroll=0 就 sticky、其底部內容永遠看不到。本方案要「用戶看完 dark 再蓋」，所以 `margin-top` 永遠是 `-100vh`（常數）。
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

## 八、參考原型

`dev-cover-dynamic.html` — 本文件對應的完整可運行原型（單檔，無依賴）。HTML / CSS / JS 全部內嵌，直接在瀏覽器打開即可驗證。

---

## 九、交付給研發時的重點

1. **三個零件獨立可調**，不要把 pin 的負 top 和 light 的 margin-top 視為同一公式
2. **JS 部分只有 3 行**（`ResizeObserver` + 寫變數），不需要額外 library
3. **不要用 `padding-bottom` 當 sticky spacer**，要用 `height: calc(...)` 或 `::after` 偽元素
4. **不要用 `sticky; bottom: 0` 想偷懶免 JS**，語意不對，實測不會釘住
5. `min-height: 100vh` 是 edge case 保險絲，不是可選項
6. cover 動畫的視覺參數（scale / blur 等）可以跟 PM / 設計調整，跟 cover 機制解耦
