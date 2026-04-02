# hitsuki46 moNa2 風格配列改動說明

## 目錄

- [第一部分：ZMK 韌體基礎知識](#第一部分zmk-韌體基礎知識)
  - [什麼是韌體](#什麼是韌體)
  - [什麼是 ZMK](#什麼是-zmk)
  - [hitsuki46 的硬體架構](#hitsuki46-的硬體架構)
  - [檔案結構](#檔案結構)
  - [核心概念](#核心概念)
- [第二部分：所有改動說明](#第二部分所有改動說明)
  - [改動一：鍵位配置 keymap](#改動一鍵位配置-keymap)
  - [改動二：右半硬體設定 overlay](#改動二右半硬體設定-overlay)
  - [改動三：軌跡球 CPI](#改動三軌跡球-cpi)
  - [改動四：捲動方向修正](#改動四捲動方向修正)
  - [改動五：ws2812_wdg 電量查看](#改動五ws2812_wdg-電量查看)
  - [改動六：BT 層按鍵位置調整](#改動六bt-層按鍵位置調整)
- [第三部分：LED 系統比較 — WS2812 Widget vs RGBLED Widget](#第三部分led-系統比較--ws2812-widget-vs-rgbled-widget)
  - [為什麼 hitsuki46 用不同的 LED 系統](#為什麼-hitsuki46-用不同的-led-系統)
  - [兩種方案的差異比較](#兩種方案的差異比較)
  - [Pros and Cons](#pros-and-cons)
- [第四部分：CI/CD 與刷入韌體](#第四部分cicd-與刷入韌體)

---

## 第一部分：ZMK 韌體基礎知識

### 什麼是韌體

韌體（Firmware）是燒錄在鍵盤微控制器裡面的程式。它負責掃描按鍵、處理層切換、管理藍牙連線、控制 LED 等等。沒有韌體，鍵盤就是一塊死的電路板。

### 什麼是 ZMK

ZMK 是基於 Zephyr RTOS 的開源鍵盤韌體框架，專為無線分離式鍵盤設計。特點：
- 原生藍牙、原生分離式支援
- 低功耗，電池可用數週
- 不需要在本機裝開發環境，GitHub Actions 雲端編譯

### hitsuki46 的硬體架構

```
┌─────────────────────┐        BLE        ┌─────────────────────┐
│      左半（L）        │ ◄──────────────► │      右半（R）        │
│    Peripheral        │                   │    Central           │
│                      │                   │                      │
│  • 按鍵矩陣 (23鍵)   │                   │  • 按鍵矩陣 (23鍵)   │
│  • 軌跡球 PMW3610    │                   │  • 軌跡球 PMW3610    │
│  • WS2812 LED (藍)   │                   │  • WS2812 LED (紅)   │
│  • Ni-MH 電池        │                   │  • Ni-MH 電池        │
│  MCU: XIAO BLE       │                   │  MCU: XIAO BLE       │
└─────────────────────┘                   └─────────────────────┘
```

**跟 FrostOrtho/moNa2 的不同：**
- **雙軌跡球**：左右各一個（左邊預設捲動，右邊預設游標）
- **WS2812 LED**：外接的可定址 LED 燈珠（不是 XIAO 板載的 RGB LED）
- **Ni-MH 電池**：鎳氫電池，需要特殊的電壓管理模組
- **46 鍵**：比 moNa2 (42鍵) 和 FrostOrtho (41鍵) 都多

### 檔案結構

```
config/
├── hitsuki46.keymap              ← 鍵位配置（最常改的檔案）
├── hitsuki46.json                ← 物理配置（給 keymap 編輯器用）
├── west.yml                      ← 依賴模組版本管理
└── boards/shields/hitsuki46/
    ├── hitsuki46.dtsi            ← 共用硬體定義（矩陣、LED、電池）
    ├── hitsuki46_R.overlay       ← 右半：軌跡球、LED、輸入處理器
    ├── hitsuki46_L.overlay       ← 左半：軌跡球、LED
    ├── hitsuki46_R.conf          ← 右半設定：BLE、PMW3610、WS2812 Widget
    ├── hitsuki46_L.conf          ← 左半設定：同上
    └── ...
build.yaml                        ← CI/CD build 目標
.github/workflows/build.yml       ← GitHub Actions 流程
```

### 核心概念

**Layer（層）**：鍵盤有多層按鍵定義，按住特定鍵切換到不同層。像手機切換輸入法。

**Hold-Tap**：一個鍵有「按住」和「點按」兩種功能。例如 `&lt 2 SPACE`：
- 點按 → 輸出空格
- 按住 → 切到 Layer 2（數字層）

**Combo（組合鍵）**：同時按兩個鍵觸發第三個功能。例如 W+E = Tab。

**Auto Mouse Layer（AML）**：軌跡球移動時自動切到滑鼠層，停止後自動退出。你可以在滑鼠層按 J=左鍵、L=右鍵。

**Device Tree（裝置樹）**：描述硬體的設定檔。`.dtsi` 是共用定義，`.overlay` 是左/右半各自的補充。告訴韌體「軌跡球接在哪個腳位」、「LED 用什麼協議」之類的。

**Kconfig（.conf）**：功能開關。`CONFIG_XXX=y` 啟用功能，`CONFIG_XXX=n` 關閉。例如 `CONFIG_WS2812_WIDGET=y` 啟用 LED 顯示功能。

---

## 第二部分：所有改動說明

### 改動一：鍵位配置 (keymap)

#### 層結構（從 7 層改為 9 層）

| 層 | 名稱 | 觸發方式 | 用途 |
|---|------|---------|------|
| 0 | Default | 預設 | QWERTY 打字 |
| 1 | SYM | 按住 LANG2 或 BKSP | 符號 `!@#$%^&*()` |
| 2 | NUM | 按住 SPC / LANG1 / ENTER | 數字 + F 鍵 |
| 3 | MEDIA | 按住 ALT | 音量、亮度、播放控制 |
| 4 | BLUETOOTH | 按住 DEL 或 SQT | 藍牙配對、切換裝置 |
| 5 | MOUSE | 軌跡球移動時自動啟用 | G=左鍵、B=右鍵、左拇指CTRL=捲動 |
| 6 | SCROLL | MOUSE 層按住左拇指 CTRL | 軌跡球變捲輪 |
| 7 | VIM | 按住 Space | H=左 J=下 K=上 L=右 |
| 8 | DESKTOP | 按住 W | macOS 桌面切換 |

#### Default 層配置

```
ESC    Q    W(桌面)  E    R    T    |    Y    U    I    O    P      TAB
LCTRL  A    S        D    F    G    |    H    J    K    L    ;      PIPE
LSHIFT Z    X        C    V    B    |    N    M    ,    .    /      RSHIFT
       CAPS ALT(媒體) CMD  SPC(VIM) CTRL | ENT(NUM) BKSP(SYM) DEL(BT) CAPS SQT(BT)
```

**外側列：**
- 左：ESC / LCTRL / LSHIFT
- 右：TAB / PIPE(|) / RSHIFT

**拇指列（保留 5 鍵 + 可改 5 鍵）：**

| 位置 | 按鍵 | 保留？ | 說明 |
|------|------|--------|------|
| [36] | CAPS | 可改 | 左 Caps Lock |
| [37] | lt(3, ALT) | 可改 | 按住=媒體層，點按=Alt |
| [38] | CMD | 保留 | Command 鍵 |
| [39] | lt(VIM, SPC) | 保留 | 按住=VIM 方向鍵，點按=空格 |
| [40] | CTRL | 保留 | Control 鍵 |
| [41] | lt(2, ENT) | 保留 | 按住=數字層，點按=Enter |
| [42] | lt(1, BKSP) | 保留 | 按住=符號層，點按=Backspace |
| [43] | lt(4, DEL) | 可改 | 按住=藍牙層，點按=Delete |
| [44] | CAPS | 可改 | 右 Caps Lock |
| [45] | lt(4, SQT) | 可改 | 按住=藍牙層，點按=單引號 |

#### Combo

| 組合 | 觸發 | 說明 |
|------|------|------|
| W + E | Tab | 左手頂排兩鍵 |
| J + K | Enter | 右手 home row 兩鍵 |
| S + D | Ctrl+Space | 切換輸入法 |

#### MOUSE 層（Layer 5）

軌跡球移動時自動啟用（2 秒無移動後退出）。在此層：
- **G** [17] = 左鍵（MB1）← 左手
- **B** [29] = 右鍵（MB2）← 左手
- **左拇指 CTRL** [40] = `&mo SCROLL`（按住=捲動模式）

> **為什麼改成左手 G/B？**
> 最初設計是 J/K/L（右手），但實測發現打字時容易誤觸。改成 G/B（左手）後，左手負責點擊、右手負責移動軌跡球，操作更分明，不會跟打字動作衝突。

> **為什麼捲動改到左拇指 CTRL？**
> 原本捲動是按住 K（右手 home row），但同樣有誤觸問題。改到左拇指 CTRL 後，需要刻意用拇指按住才會觸發捲動，不會在打字時意外啟用。在非 MOUSE 層時，這個鍵還是正常的 CTRL。

#### Hold-Tap 參數

```
require-prior-idle-ms = <150>;
```
快速打字時（150ms 內連續按鍵），所有 hold-tap 鍵直接判定為點按。避免打字時意外觸發層切換。

---

### 改動二：右半硬體設定 (overlay)

#### 自動滑鼠層（AML）層號和超時更新

```
zip_temp_layer 1 700  →  zip_temp_layer 5 2000
```

兩個改動：
1. **層號**：從 Layer 1 改為 Layer 5（MOUSE），對應 keymap 的新層結構
2. **超時**：從 700ms 加長到 2000ms。原本 0.7 秒太短，滑鼠操作（定位→點擊）常常來不及就退出了

#### 捲動層層號更新

```
layers = <5>;  →  layers = <6>;
```

軌跡球輸入轉為捲動的層號，從 Layer 5 改為 Layer 6（SCROLL）。

#### 防誤觸設定（excluded-positions + require-prior-idle）

```c
&zip_temp_layer {
    require-prior-idle-ms = <200>;
    excluded-positions = <
        17 // G - MB1（左鍵）
        29 // B - MB2（右鍵）
        40 // 左拇指 CTRL（MOUSE 層中為 SCROLL 觸發）
        12 // LCTRL
        24 // LSHIFT
        35 // RSHIFT
        38 // CMD
    >;
};
```

**`require-prior-idle-ms = <200>`：** 打字後 200ms 內的軌跡球移動會被忽略。防止打字時手掌碰到軌跡球意外切到 MOUSE 層。

**`excluded-positions`：** 在 MOUSE 層按這些鍵不會退出 MOUSE 層。沒有這個設定的話，按 G（左鍵）的瞬間 MOUSE 層就會退出，G 會輸出字母 g 而不是滑鼠左鍵。包含修飾鍵是為了支援 Shift+點擊、Ctrl+點擊等組合操作。

#### 點擊延長超時（mkp_input_listener）

```c
&mkp_input_listener { input-processors = <&zip_temp_layer MOUSE 10000>; };
```

**這是什麼？** 當你按下滑鼠鍵（G 或 B）時，MOUSE 層的超時會被延長到 10 秒。

**為什麼需要？** 沒有這個設定時，MOUSE 層的超時只靠軌跡球移動重置（2 秒）。雙擊操作時：
1. 移動軌跡球 → MOUSE 層啟用
2. 按 G（第一次點擊）→ MB1
3. 鬆開 G → 如果軌跡球沒在動，2 秒計時器在跑
4. 按 G（第二次點擊）→ 如果超時了，MOUSE 層已退出，G 輸出字母 g 而非 MB1

加了 `mkp_input_listener` 後，第一次點擊就會把超時延長到 10 秒，確保後續的雙擊、拖曳等操作都在 MOUSE 層內完成。

> **實測問題：** 在 YouTube 全螢幕雙擊後，單擊會持續變成雙擊。原因就是 MOUSE 層超時太短（700ms）且沒有點擊延長，導致層在操作途中閃退再重入，產生多餘的點擊事件。

---

### 改動三：軌跡球 CPI

```
cpi = <800>;  →  cpi = <1600>;
```

兩邊都改。CPI（Counts Per Inch）= 每移動一英吋感測器回報的位移量。800 太慢，跨多螢幕要滑很多下。1600 大約是原來速度的兩倍。

PMW3610 支援 200-3200，以 200 為一級。如果 1600 還不夠可以再調高。

---

### 改動四：捲動方向修正

右半的捲動處理器原本有 `INPUT_TRANSFORM_Y_INVERT`（Y 軸反轉），導致捲動方向相反。移除後方向正常。

```
// 移除前
&zip_scroll_transform INPUT_TRANSFORM_Y_INVERT,

// 移除後
（整行刪掉）
```

---

### 改動五：ws2812_wdg 電量查看

#### 什麼是 ws2812_wdg？

`&ws2812_wdg` 是 gohanda11/zmk-ws2812-driver 模組提供的行為（behavior），按下後 LED 會依序閃爍顯示：
1. **電量**：綠色(>80%) / 黃色(20-80%) / 橘色(5-20%) / 紅色(<5%)
2. **連線狀態**：藍色(已連線) / 青色(廣播中) / 紅色(斷線) / 綠色(USB)
3. **當前層**：對應層的顏色

#### 上游 Bug 修正

gohanda11 的模組有一個 bug：`ws2812_widget.dtsi` 定義 `#binding-cells = <0>`（不需要參數），但 binding yaml 要求 `#binding-cells = <1>`（需要 1 個參數）。這會導致 build 失敗。

修正方式：不用 `#include` 引入有 bug 的 dtsi，直接在 keymap 裡定義正確版本：

```c
ws2812_wdg: ws2812_widget {
    compatible = "zmk,behavior-ws2812-widget";
    #binding-cells = <1>;        // 修正：yaml 要求 1 個參數
    indicate-battery;
    indicate-connectivity;
    indicate-layer;
};
```

使用時傳入一個參數（0）：`&ws2812_wdg 0`

放在 **BT 層的 B 鍵**位置，進藍牙層按 B 就能查看狀態。

---

### 改動六：BT 層按鍵位置調整

因為安裝軌跡球後，右拇指最外側的鍵 [45] 變成了切層鍵（`lt(4, SQT)` 按住進入 BT 層）。在 BT 層時你正在按住 [45]，所以 [45] 位置的 BT_CLR_ALL 無法觸發。

修正：把 BT_CLR 和 BT_CLR_ALL 各往上移一格：

| 位置 | 改前 | 改後 |
|------|------|------|
| Row 1 右末 [23] | &trans | BT_CLR |
| Row 2 右末 [35] | BT_CLR | BT_CLR_ALL |
| Row 3 右末 [45] | BT_CLR_ALL | &trans（不可按） |

---

## 第三部分：LED 系統比較 — WS2812 Widget vs RGBLED Widget

### 為什麼 hitsuki46 用不同的 LED 系統？

ZMK 生態系中有兩種主流的 LED 狀態顯示方案：

| | hitsuki46 用的 | FrostOrtho / moNa2 用的 |
|---|---|---|
| **模組名稱** | `zmk-ws2812-driver` (gohanda11) | `zmk-rgbled-widget` (caksoylar) |
| **LED 硬體** | WS2812 / SK6812 可定址 LED 燈珠 | MCU 板載 RGB LED（GPIO 直驅） |
| **通訊方式** | SPI 單線協議 | GPIO 直接控制 3 個腳位（R/G/B） |
| **Config 前綴** | `CONFIG_WS2812_WIDGET_*` | `CONFIG_RGBLED_WIDGET_*` |
| **Shield 需求** | 不需要額外 shield（自帶 LED 定義） | 需要 `rgbled_adapter` shield |
| **行為名稱** | `&ws2812_wdg 0` | `&ind_bat` / `&ind_con` |

### 根本差異：LED 硬體不同

**RGBLED Widget（FrostOrtho/moNa2）：**
- 用的是 XIAO BLE 板子上**內建的 RGB LED**
- 這顆 LED 是由 3 個 GPIO 腳位分別控制紅、綠、藍三色
- 優點：不需要額外硬體，板子自帶
- 缺點：亮度低，在不透明外殼裡幾乎看不到
- 需要 `rgbled_adapter` shield 來告訴韌體 GPIO 腳位

**WS2812 Widget（hitsuki46）：**
- 用的是**外接的 WS2812 可定址 LED 燈珠**
- 透過 SPI 單線協議通訊，一條線就能控制多顆 LED
- 優點：亮度高、顏色自訂度高（用 0xRRGGBB hex 色碼）、可串聯多顆
- 缺點：需要額外的 LED 硬體和電源控制電路
- 不需要 `rgbled_adapter`，在 `.dtsi` 裡自己定義 LED 節點

### 兩種方案的差異比較

| 特性 | WS2812 Widget (hitsuki46) | RGBLED Widget (FrostOrtho) |
|------|--------------------------|---------------------------|
| **顏色精度** | 24-bit RGB (0x000000-0xFFFFFF) | 8 色 (0-7 色碼對應固定顏色) |
| **亮度控制** | 透過 hex 值微調（如 0x7F0000 = 50% 紅） | 無法調整，固定亮度 |
| **每層獨立顏色** | 支援（Layer 0-31 各自設定色碼） | 支援（但只有 8 色可選） |
| **持續顯示層色** | 支援（`SHOW_LAYER_COLORS_PERSIST`） | 只閃一下 |
| **電量分級** | 4 級（高/中/低/危險） | 4 級（高/中/低/危險） |
| **連線狀態顯示** | 4 種（USB/已連/廣播/斷線） | 3 種（已連/廣播/斷線） |
| **手動觸發行為** | `&ws2812_wdg 0`（一鍵顯示全部） | `&ind_bat`、`&ind_con`（分開觸發） |
| **外部電源控制** | 有（P-MOSFET 控制 LED 供電） | 無 |
| **LED 數量** | 可串聯多顆 | 僅 1 顆板載 |
| **ZMK 分支相容性** | ZMK main branch | 需要 v0.3-branch 版本 |

### Pros and Cons

#### WS2812 Widget（hitsuki46 用的）

**Pros：**
- 顏色更豐富，24-bit 自由調色
- 亮度可控，不會太刺眼也不會看不到
- 支援持續亮燈顯示當前層（不用等它閃）
- 可以串多顆 LED，做更複雜的顯示
- 有獨立電源控制，睡眠時完全斷電省電
- 跟 ZMK main branch 相容，不需要特殊分支

**Cons：**
- 需要額外的 LED 硬體（WS2812 燈珠 + 電源電路）
- 驅動較複雜（SPI 協議）
- 模組有 bug（`#binding-cells` 不一致），需要手動修正
- 手動觸發只有一個 `&ws2812_wdg`，無法分開查電量和連線
- 模組是較小的社群維護，更新頻率不確定

#### RGBLED Widget（FrostOrtho/moNa2 用的）

**Pros：**
- 不需要額外硬體，XIAO BLE 板載就有
- 設定簡單，加個 `rgbled_adapter` shield 就能用
- 有獨立的 `&ind_bat`、`&ind_con`、`&ind_lyr` 行為，可以分開觸發
- caksoylar 是知名的 ZMK 社群開發者，模組維護穩定
- 文件完善，使用者多

**Cons：**
- 只有 8 色可選（0-7），無法自訂 RGB 色碼
- 亮度固定，不透明外殼裡幾乎看不到
- 只有 1 顆板載 LED，無法擴展
- 層色只閃一下就滅，不會持續顯示
- 需要特定版本（v0.3-branch），舊版 v0.3.0 功能不全
- 需要 `rgbled_adapter` shield，忘了加就完全不亮（我們在 FrostOrtho 上踩過這個坑）

#### 總結

| 適用場景 | 推薦方案 |
|---------|---------|
| 鍵盤自帶 WS2812 LED | WS2812 Widget |
| 只有 XIAO 板載 LED | RGBLED Widget |
| 想要持續顯示當前層顏色 | WS2812 Widget |
| 想要最簡單的設定 | RGBLED Widget |
| 需要精確的顏色調整 | WS2812 Widget |
| 使用 ZMK main branch | WS2812 Widget（RGBLED 也可用但功能可能不同） |

兩者本質上做的事一樣（用 LED 顯示電量/連線/層），差異主要在 LED 硬體和顏色控制精度。選哪個取決於你的鍵盤用什麼 LED。

---

## 第四部分：CI/CD 與刷入韌體

### 流程

```
改設定檔 → git push → GitHub Actions 自動編譯 → 下載 UF2 → 刷入鍵盤
```

GitHub Actions 會讀取 `build.yaml` 知道要 build 什麼，讀取 `west.yml` 下載依賴模組，然後編譯出 `.uf2` 韌體檔案。

### 刷入步驟

1. USB 接**右半**（Central）
2. 快速**雙擊 RST** → 出現 XIAO-SENSE 磁碟
3. 拖入 `hitsuki46_R` 的 UF2
4. 重複步驟刷左半（`hitsuki46_L`）

### 何時需要 settings_reset

- 左右半無法連線
- 要清除所有藍牙配對
- 刷完 reset 後：先右半再左半刷正常韌體
- 電腦上**移除舊的 hitsuki46 藍牙**，重新配對
