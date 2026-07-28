# AIQuota Widget 開發決策紀錄：動態字體折行、App Group 警告與模擬器憑證信任

#ios #swiftui #widgetkit #dynamic-type #app-group #debugging #ios-simulator #decision-record

## 這是什麼

開發 AIQuota（iPhone App + Home Screen Widget，顯示 Codex、Claude、AGY 額度）時遇到的三個問題的排查與決策紀錄：Widget 文字動態字體折行、App Group 偏好設定的警告訊息判讀、以及模擬器信任自簽憑證的操作方式。整理成可回查的知識，避免下次遇到類似狀況重新排查一次。

## 核心結論

- **WidgetKit 內固定寬度欄位裡的文字，不要用會隨系統字體縮放的字體**（`.footnote`、`.body` 等），否則使用者調大系統字體後會被擠壓折行或溢出；要嘛用固定 `.system(size:)` 字級，要嘛把欄位寬度也跟著動態字體走。
- `.fixedSize(horizontal: true)` 會讓同時掛著的 `.minimumScaleFactor` 失效，兩者不要同時用在同一個 `Text`；只需要「不折行、超出就縮小」時，`lineLimit(1) + minimumScaleFactor` 已經足夠。
- `Couldn't read values in CFPrefsPlistSource ... kCFPreferencesAnyUser with a container is only allowed for System Containers` 這類訊息，在**第一次**存取 App Group 的 `UserDefaults(suiteName:)` 時幾乎必然出現，是系統框架探測不允許層級留下的雜訊，**不代表 App Group 沒設好**；要判斷是否真的有問題，去查 entitlements 有沒有正確嵌入、`simctl get_app_container <bundle-id> groups` 能不能解析出路徑，這比看 log 準。
- Widget Extension 進程被系統 SIGKILL（`exit code 9`）在渲染完 timeline 後是正常生命週期，不是崩潰。
- 模擬器信任自簽憑證最快的方式是 `simctl keychain booted add-root-cert`，一行指令，不用進模擬器設定畫面手動點。

## 背景 / 問題

### 問題一：Widget 上 Provider 名稱「Claude」被擠成兩行

Widget 用一個固定寬度（50pt）的欄位顯示 Provider 名稱，字體用的是 `.footnote`：

```swift
Text(provider.displayName)
    .font(.footnote)
    .fontWeight(.bold)
    .lineLimit(1)
    .minimumScaleFactor(0.5)
    .fixedSize(horizontal: true, vertical: false)
    .frame(width: 50, alignment: .leading)
```

`.footnote` 是動態字體（Dynamic Type），會隨系統「文字大小」設定放大。使用者把字體調大一點，「Claude」六個字母在放大後的字級下超過 50pt 欄寬，就會被擠壓折成兩行。更糟的是，`.fixedSize(horizontal: true)` 會讓 SwiftUI 以「文字的理想寬度」佈局，這跟 `.minimumScaleFactor` 想做的「超出可用空間就縮小」互相矛盾——實測結果是 `fixedSize` 贏，`minimumScaleFactor` 完全不生效，文字直接以理想寬度溢出欄位，蓋住旁邊的內容（例如 "5h" 標籤）。

### 問題二：console 出現 CFPrefs 警告，App Group 是不是沒設好？

執行時 console 印出：

```
Couldn't read values in CFPrefsPlistSource<0xa71071c80> (Domain: group.com.hom.AIQuota, User: kCFPreferencesAnyUser, ByHost: Yes, Container: (null), Contents Need Refresh: Yes): Using kCFPreferencesAnyUser with a container is only allowed for System Containers, detaching from cfprefsd
[S:1] Error received: Connection invalidated.
Debug session ended with code 9: killed
```

看起來像是 App Group 沒設定成功，但實際上這是 Apple framework 的已知雜訊。

### 問題三：Collector 走 HTTPS 自簽憑證，模擬器連不上

App 只讀取 HTTPS endpoint 產生的 `quota.json`，不允許停用 TLS 驗證（安全邊界要求），所以自簽憑證需要讓模擬器「真的信任」，而不是繞過驗證。

## 架構 / 流程 / 實作方式

### 修正一：Provider 名稱欄改用固定字級

```swift
// Provider Name - 固定字級（不隨動態字體放大，與同列 5h/7d、百分比一致），
// 超出欄寬時以 minimumScaleFactor 縮小而非折行
Text(provider.displayName)
    .font(.system(size: 13, weight: .bold, design: .rounded))
    .lineLimit(1)
    .minimumScaleFactor(0.6)
    .frame(width: 52, alignment: .leading)
```

- 字級改為固定 `.system(size:)`，不受動態字體影響，跟同一列的「5h/7d」（8pt）、百分比（10pt）等固定字級保持一致的設計語言。
- 移除 `.fixedSize`，只留 `lineLimit(1) + minimumScaleFactor(0.6)`：名稱超出欄寬時縮小而不是折行或溢出。
- 欄寬微調到 52pt。

**驗證方式**：WidgetKit 的 `#Preview` 在 Xcode Canvas 之外不好批次驗證多種字級組合，改用 CLI 渲染工具驗證——把 widget extension 的實際原始碼與 `Shared` 目錄複製到暫存資料夾（用 `awk '/^#Preview/{exit} {print}'` 去掉尾端的 `#Preview` 區塊，避免 target 參照編譯失敗），寫一個小 `main.swift` 用 `ImageRenderer` 在不同 `dynamicTypeSize`（`.large` / `.xxxLarge` / `.accessibility1`）與不同 widget 尺寸（329×155、291×141）下渲染 `MediumQuotaWidgetView`，輸出 PNG：

```bash
xcrun -sdk iphonesimulator swiftc \
  -target arm64-apple-ios18.0-simulator \
  -O -o render-harness main.swift src/*.swift

xcrun simctl spawn <booted-sim-udid> ./render-harness <outdir> <prefix>
```

`ImageRenderer.uiImage` 是 `@MainActor` 隔離的，harness 裡的渲染函式要標 `@MainActor`，否則編譯期報 actor isolation 錯誤。跑完直接讀 PNG 檢查五種尺寸/字級組合是否都維持單行、不溢出。

### 判讀二：CFPrefs 警告是雜訊，用這三個檢查代替看 log

不要用「console 有沒有印錯誤」判斷 App Group 是否正常，改查：

```bash
# 1. entitlements 是否正確嵌入（模擬器 build 存在 __TEXT,__entitlements section，
#    codesign 簽章顯示空字典是正常現象，要看 section 內容才準）
xcrun segedit "<App二進位路徑>" -extract __TEXT __entitlements /dev/stdout \
  | strings | grep -A3 "application-groups"

# 2. App Group 容器能否解析出實際路徑
xcrun simctl get_app_container <sim-udid> <bundle-id> groups

# 3. App 啟動後群組容器內 Library/Preferences 是否有被建立
ls "<上面查到的路徑>/Library/Preferences"
```

三項都正常，代表 App Group 沒問題，可以無視那則 CFPrefs 警告。`Connection invalidated` / `exit code 9 (killed)` 也是同理：Widget Extension 在 timeline 渲染完成後被系統回收是正常生命週期，在 Xcode 按停止鍵同樣會得到 exit code 9。

### 修正三：模擬器信任自簽憑證

```bash
xcrun simctl keychain booted add-root-cert /path/to/rootCA.pem
```

支援 `.pem` / `.cer` / `.der`。裝完後把 App 砍掉重開即可，不用重開模擬器；可到「設定 → 一般 → 關於本機 → 憑證信任設定」確認。

## 注意事項 / 限制

- 憑證本身要符合條件才有用，就算根憑證已信任，缺以下任一項照樣連不上：
  - **SAN（Subject Alternative Name）必須包含連線用的主機名稱或 IP**（iOS 不看 CN；例如連 `192.168.x.x` 要在 SAN 放 `IP:192.168.x.x`），錯誤訊息通常是 `NSURLErrorServerCertificateUntrusted`。
  - 有效期 ≤ 825 天、RSA ≥ 2048、簽章 SHA-256 以上。
  - 加入的必須是**簽發鏈的根憑證**（`CA:TRUE`），或直接把自簽 leaf 憑證當根加入。
- 自己手動產 openssl 憑證常常漏掉 SAN，建議直接用 [mkcert](https://github.com/FiloSottile/mkcert)（`mkcert 192.168.x.x localhost`），產出的憑證天生符合上述所有要求；根憑證路徑用 `mkcert -CAROOT` 查。
- 若 Collector endpoint 是純 HTTP（非 HTTPS），跟憑證信任無關，那是 ATS（App Transport Security）本地網路例外的問題，不要混為一談。
- `minimumScaleFactor` 只在**沒有** `fixedSize` 搶走佈局決定權時才會生效，這個坑不限於 WidgetKit，任何固定寬度容器裡的動態字體 `Text` 都可能踩到。

## 後續可擴充方向

- 如果之後 Widget 支援更多 Provider 或更長的名稱，可以把 CLI 渲染驗證工具收斂成一個可重複執行的 script，納入開發流程而不是每次臨時寫。
- App Group 容器路徑與偏好設定的三項檢查，可以包成一個小型診斷指令（例如 `./scripts/verify-app-group.sh <bundle-id>`），用在遇到類似「看起來像錯誤但其實正常」的情境時快速排除。
