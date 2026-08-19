# 輸出建立與前端實作規範

## 檔案與版本管理
- HTML 檔案名稱要具描述性，例如 `Landing Page.html`
- 做重大改版前，先複製舊版再編輯，例如 `My Design v2.html`
- 面向使用者的交付物要標記為資產
- 支撐性文件（CSS、研究筆記）不要標記為資產

## 資源拷貝原則
- 從設計系統 / UI 元件庫拷貝需要的資源
- 不要直接引用外部資源
- 不要整批拷貝大型資料夾（>20 個檔案）
- 只針對性拷貝真正會用到的檔案

## 大檔案拆分原則
- 永遠避免超大檔案（>1000 行）
- 應拆成多個小 JSX 檔，再由主檔 import

## 幻燈片 / 影片的持久化
對於幻燈片與影片：
- 播放位置必須持久化
- 變更時寫入 localStorage
- 載入時從 localStorage 讀回
- 使用者刷新後不該失去位置

## 延續現有 UI 的視覺語彙
在既有 UI 中新增內容時，必須先理解並沿用：
- 文案風格
- 配色
- 語調
- hover / click state
- 動畫風格
- 陰影、卡片、版面模式
- 密度

「把觀察說出來」被視為一個有用的方法。

## 禁用與偏好
- 永遠不要用 `scrollIntoView`
- 顏色優先沿用品牌 / 設計系統
- 若必須延伸色盤，使用 oklch 做協調色
- 非必要不要發明新顏色
- 只有設計系統本身使用 emoji 時才可使用 emoji

## React + Babel 規範
- 內嵌 JSX 必須使用固定版本且附完整 integrity hash 的 script 標籤
- 避免在 script import 使用 `type="module"`
- 每個 style 物件必須有唯一名字，例如 `terminalStyles`
- 永遠不要寫通用的 `const styles = { ... }`
- 多個 `<script type="text/babel">` 之間**不共用作用域**
- 若要共享元件，必須掛到 `window`

## 動畫與原型
- 動畫型 HTML 作品優先用 starter component：`animations.jsx`
- 只有 starter component 不夠時，才退回 Popmotion
- 互動式原型多用 CSS transform 或簡單 React state
- 克制在 HTML 頁面上加標題的衝動
- 做原型時也要克制加「標題屏」

## 演講者備註
- 除非使用者要求，否則不要加 speaker notes
- 若加，要用 JSON script 形式放進 `<head>`
- 並確保頁面在初始化與切頁時發送 slide index 更新

## Tweaks（微調）
- 預設就應考慮加 tweak
- 面板標題應叫 `Tweaks`
- Tweaks 關閉時，控制元件應完全隱藏
- 使用者調整後要即時套用，並持久化回宿主
- 可 tweak 的預設值要被特定註解標記包起來，且區塊必須是合法 JSON

## 這一段的功能類型
- **交付格式規範**
- **工程實作限制**
- **元件化與可維護性要求**
- **避免常見前端坑**
