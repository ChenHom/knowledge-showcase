# Claude Design 系統提示詞（中文整理）

## 說明
這份筆記把老闆提供的內容，依照**段落功能**與**提示詞類型**重新拆開存放，方便之後閱讀、引用與對照。

## 檔案索引
- `01-background-and-reading-lens.md`
  - 老闆對 Claude / OpenAI 系統提示詞的閱讀動機
  - 對提示詞結構安排的觀察
  - 對「如何引導 AI 問好問題」這段的重點感受
- `02-role-and-confidentiality-boundary.md`
  - Claude Design 的角色、媒介、禁止洩漏事項、能力描述邊界
- `03-workflow-questions-and-inputs.md`
  - 工作流程
  - 如何閱讀文件 / 提供資源
  - 如何提問、什麼時候該問、怎麼問好問題
- `04-output-and-frontend-conventions.md`
  - HTML 輸出規範
  - React/Babel、動畫、幻燈片、speaker notes、Tweaks
  - 技術實作上的硬性限制
- `05-design-methodology-and-validation.md`
  - 設計探索流程
  - 高保真設計的上下文要求
  - 變體、tweak、驗證、done / verifier 的使用原則
- `06-content-principles-and-cross-project-rules.md`
  - GitHub 匯入
  - 跨專案讀取
  - 內容指南
  - 固定尺寸內容 / starter components / 版權邊界
- `07-claude-design-vs-huashu-design.md`
  - 電腦王阿達文章整理
  - Claude Design 的外部產品脈絡
  - Huashu Design 如何把 Claude Design 的核心工作流 skill 化
  - 對原 prompt 的再解讀：prompt 不只是文字，而是 workflow / protocol / product contract
- `08-general-system-prompt-template.md`
  - 從 Claude Design / Huashu Design 抽出的通用骨架
  - 不綁平台、不綁工具、不限單一 AI 的 system prompt 模板
  - 包含完整版、精簡版、模組化版本與使用方式

## 分類方式
這次拆分主要依照兩條軸線：
1. **段落主題**：例如角色、工作流、提問、輸出、驗證、內容治理。
2. **提示詞功能**：例如禁止事項、操作流程、格式規格、品質約束、互動策略。

## 後續可延伸的閱讀角度
- 哪些段落屬於**安全邊界**（不要說什麼 / 不要做什麼）
- 哪些段落屬於**工作流 orchestration**（先做什麼、後做什麼）
- 哪些段落屬於**提問設計**（如何教模型問出高價值問題）
- 哪些段落屬於**輸出品質控制**（格式、元件、驗證、持久化）
- 哪些段落屬於**產品哲學**（不要做 AI slop、不要亂補內容、要先有上下文）
