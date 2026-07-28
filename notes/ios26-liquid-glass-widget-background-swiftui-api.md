# iOS 26 Liquid Glass 筆記：Widget 玻璃背景系統機制與 SwiftUI API

#ios26 #liquid-glass #swiftui #widgetkit #glasseffect #design-system

## 這是什麼

iOS 26 導入 Liquid Glass 設計語言後，「Home Screen Widget 要怎麼做出跟原生 Widget 一樣的玻璃背景」這個問題的調查筆記。結論跟直覺不同：**Widget 的玻璃背景不是 App 端寫程式碼指定出來的，而是系統在特定使用者設定下自動套用的**。順帶整理 App 內一般畫面（非 Widget）可用的 Liquid Glass SwiftUI API。查證依據是 Xcode 內附 iOS SDK 的 swiftinterface 定義（`grep` swiftmodule 檔案的 `@available` 標註），不是憑印象或文件片段猜的。

## 核心結論

- **Widget 沒有任何 iOS API 可以在程式碼裡把背景設成 Liquid Glass。** WidgetKit 唯一的玻璃相關 API `widgetTexture(.glass)`，在 SDK 裡明確標記只支援 `visionOS 26.0`、`iOS unavailable`，iOS 專案引用了編譯直接失敗。
- **原生玻璃外觀是系統套用的，不是 App 決定的**：使用者把主畫面外觀切到「透明（Clear）」或「有色（Tinted）」時，系統會自動移除 Widget 用 `containerBackground(for: .widget)` 指定的背景，換成跟系統原生 Widget 完全相同的 Liquid Glass。
- App 端要做的不是「換成玻璃」，而是滿足**兩個前提條件**讓系統願意套用：背景可被移除（預設就是，除非自己呼叫了 `containerBackgroundRemovable(false)`）、內容要能適應 accented 渲染模式（重點元素標 `.widgetAccentable()`）。
- **自己把背景拿掉或換成 `Color.clear` 不會產生玻璃**，在預設（非透明）模式下只會失去面板，甚至讓系統顯示錯誤佔位——因為 Widget 是離線渲染後由 SpringBoard 合成的靜態畫面，程式碼**無法取樣或模糊桌布**，市面上「透明小工具」App 的玻璃感其實是拿桌布截圖手動合成的假象，不是真的即時玻璃。
- `glassEffect()` / `GlassEffectContainer` 這組 API 是給**一般 App 介面**（畫面、按鈕、卡片）用的，跟 Widget 背景是兩回事，不要混用。

## 背景 / 問題

需求是「Widget 背景要用原生玻璃材質，跟系統內建的 Widget 看起來一樣」。第一直覺會去找一個類似 `.background(.glass)` 或 `glassEffect()` 的 modifier 直接套在 Widget 的 `containerBackground` 上，但實際查證後發現思路錯了：iOS 上不存在這種 API，玻璃是系統依使用者的外觀設定自動套用的視覺層，不是內容層可以指定的材質。

## 架構 / 流程 / 實作方式

### 1. Widget 玻璃背景的實際機制

Widget 視圖照常宣告 `containerBackground`：

```swift
var body: some View {
    VStack {
        // widget 內容
    }
    .containerBackground(.ultraThinMaterial, for: .widget)
}
```

系統行為分兩種情境：

- **主畫面外觀為預設**：`containerBackground` 指定的材質（如 `.ultraThinMaterial`）正常顯示，這是不透明底色 + 霧面效果，不會透出桌布。
- **主畫面外觀切成透明／有色**：系統忽略/移除你指定的背景，換上原生 Liquid Glass（跟時鐘、天氣等系統 Widget 用的是同一套渲染）。

App 端要確保系統願意套用玻璃，需要滿足：

```swift
// 1. 不要關掉「背景可移除」——這是預設值，不要主動呼叫：
//    .containerBackgroundRemovable(false)
//    （呼叫了會被排除在 iPad 鎖定畫面、StandBy 之外，且玻璃模式也套用不了）

// 2. 重點內容標記 accentable，讓內容在有色/清晰模式下正確去飽和：
ProgressBarView(...)
    .widgetAccentable()
```

要親眼看到效果，操作路徑是：**長按主畫面空白處 → 編輯 → 自訂 → 外觀選「透明」**。

### 2. 為什麼程式碼做不到：SDK 查證過程

在本機 Xcode 附帶的 iOS SDK 裡直接找 WidgetKit 的 swiftinterface 定義：

```bash
SDK=$(xcrun --sdk iphoneos --show-sdk-path)
IFACE="$SDK/System/Library/Frameworks/WidgetKit.framework/Modules/WidgetKit.swiftmodule/arm64e-apple-ios.swiftinterface"
grep -n -i "texture\|glass" "$IFACE"
```

找到：

```swift
@available(visionOS 26.0, *)
@available(iOS, unavailable)
@available(watchOS, unavailable)
@available(macOS, unavailable)
@available(tvOS, unavailable)
public struct WidgetTexture : Swift.Sendable, Swift.Hashable {
  public static let glass: WidgetKit.WidgetTexture
  public static let paper: WidgetKit.WidgetTexture
}

@available(visionOS 26.0, *)
@available(iOS, unavailable)
...
extension SwiftUI.WidgetConfiguration {
  public func widgetTexture(_ material: WidgetKit.WidgetTexture) -> some SwiftUI.WidgetConfiguration
}
```

`@available(iOS, unavailable)` 是硬性排除，不是「iOS 版本太舊」，而是**這個 API 從設計上就不給 iOS 用**，只服務 visionOS 的空間介面材質。這比在網路上找二手教學文章可靠——教學文章常把 visionOS 專屬 API 誤植成 iOS 通用寫法。

### 3. App 內一般畫面可用的 Liquid Glass API（跟 Widget 無關）

這組 API 是給 App 內普通 SwiftUI 畫面用的（導覽列、按鈕、卡片），SDK 內確認存在：

```swift
// 對單一 view 套玻璃，預設 .regular 玻璃 + capsule 形狀
someView.glassEffect(.regular, in: .capsule)

// 多個玻璃 view 需要「共享同一片玻璃、彼此融合」時用容器包起來
GlassEffectContainer(spacing: 20) {
    HStack {
        iconView.glassEffect()
        labelView.glassEffect()
    }
}

// 按鈕樣式
Button("Action") { }.buttonStyle(.glass)
Button("Primary") { }.buttonStyle(.glassProminent)
```

`glassEffect` / `GlassEffectContainer` 在 SDK 的 `SwiftUI.swiftmodule` 裡查不到直接對應符號（可能是 `_SwiftUI_Glass` 或私有模組拆分，本次查證未深入 disassemble），但按鈕樣式 `static var glass: GlassButtonStyle` / `static var glassProminent: GlassProminentButtonStyle` 已在 SDK 中確認存在且可用。

## 注意事項 / 限制

- 不要把「Widget 沒有玻璃 API」跟「App 內沒有玻璃 API」搞混——兩者是完全不同的兩組能力，適用範圍不同。
- 常見的 `glassEffect` 使用錯誤（社群整理，非本次 SDK 查證）：
  - 不要在 `glassEffect` 上再疊 `.blur` / `.opacity` / `.background`。
  - 不要在玻璃 view 背後放實色底（`Color.white` / `Color.black`），會蓋掉折射效果。
  - 不要對 `glassEffect` view 用 `.clipShape`，形狀要用 `glassEffect(_:in:)` 的 shape 參數指定。
  - `GlassEffectContainer` 不可巢狀（一個 container 裡面不能再放另一個 container 包住的 glassEffect）。
- Widget 背景材質的選擇（例如 `.ultraThinMaterial` vs `.fill.tertiary`）在非透明模式下純粹是視覺口味問題，跟能不能套用系統玻璃無關；系統玻璃只在使用者主動切換外觀模式時出現，App 端無法強制開啟。
- 查證方法本身值得記錄：遇到「這個 API 到底支不支援 iOS」的疑問，直接 `grep` 本機 SDK 的 `.swiftinterface` 檔案看 `@available` 標註，比查網路文章準確，因為文章常把 beta 版本或跨平台 API 的可用範圍寫錯或寫得不夠精確。

## 後續可擴充方向

- 如果之後要在 App 內（非 Widget）畫面導入 Liquid Glass 視覺，可以針對 `glassEffect` / `GlassEffectContainer` 的 shape 融合行為、與現有 `.ultraThinMaterial` 等舊 API 的差異，另外整理一篇更完整的 SwiftUI 實作筆記。
- `WidgetTexture.paper` 目前查到同樣是 visionOS-only，用途待查（可能是空間介面的另一種材質選項），非本次調查重點。
