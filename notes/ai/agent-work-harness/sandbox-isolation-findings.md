# Sandbox 隔離的實測邊界：codex 與 bubblewrap

#ai #agent #sandbox #security #bubblewrap #isolation #knowledge

## 這是什麼

在實作 Agent Work Harness 時，設計文件要求「以實際 probe 證明隔離成立，不可只依賴文件假設」。
照做之後發現：**三個安全假設裡有兩個不成立**。

這篇記錄實測結果與可重用的結論。適用於任何要把不可信程式碼放進沙箱執行的場景，
不限於 Agent。

## 核心結論

> 沙箱模式的名字（read-only / workspace-write）只描述**檔案系統**邊界。
> 網路、HOME、暫存目錄都要另外處理，而且預設值往往比你以為的寬鬆。

## 實測一：沙箱模式不等於網路隔離

只指定 `workspace-write` 沙箱模式時：

| 探測 | 結果 |
|---|---|
| workspace 內寫檔 | 允許（預期） |
| workspace 外寫檔 | 拒絕（預期） |
| `curl https://example.com` | **HTTP 200 — 網路完全沒被擋** |
| `printenv HOME` | **操作者的真實 HOME** |
| `ls -a $HOME` | **看得到 `.ssh`、`.aws`、各種工具的 token** |

檔案系統邊界是對的，其他兩項完全開放。

如果只在 prompt 裡寫「你不可以連網」，那不是隔離，是請求。
不可信的程式碼（或被 prompt injection 影響的 Agent）不會照做。

### 修正方式

網路要在 runtime 設定裡顯式關閉，不能依賴沙箱模式的預設：

```toml
[sandbox_workspace_write]
network_access = false
```

HOME 要用最小環境變數啟動子行程，指向一個專用目錄：

```bash
env -i PATH=... HOME=/path/to/isolated-home TERM=dumb <command>
```

修正後同一組探測：網路 DNS 解析失敗、`ls -a $HOME` 只剩 `.` 與 `..`。

## 實測二：`--ro-bind / /` 會把整個 HOME 帶進沙箱

用 bubblewrap 做隔離執行時，第一版寫法是：

```bash
bwrap --unshare-all --ro-bind / / ... -- <command>
```

看起來很安全（唯讀掛載），實際上沙箱裡讀得到：

```text
$ cat /home/<user>/.secrets
export OPENAI_KEY=sk-proj-...
```

**唯讀不等於不可見。** 憑證檔案唯讀也一樣會被讀走。

### 修正方式

改用白名單 bind，並且明確遮蔽 `/home`：

```bash
bwrap --unshare-all --die-with-parent --new-session \
  --ro-bind /usr /usr --ro-bind-try /bin /bin --ro-bind-try /lib /lib \
  --ro-bind-try /lib64 /lib64 --ro-bind /etc /etc \
  --proc /proc --dev /dev --tmpfs /tmp --tmpfs /run \
  --tmpfs /home \                       # 先遮蔽，再單獨 bind 需要的
  --ro-bind-try <toolchain> <toolchain> \
  --bind <workspace> <workspace> \
  --setenv HOME <isolated-home> --chdir <workspace> -- <command>
```

`--tmpfs /home` 是關鍵：先把整個 `/home` 蓋掉，再單獨掛回真正需要的路徑。

toolchain 路徑（node、python 的安裝位置）要能設定，但**應該由執行方的全域設定提供，
不能由被驗證的 repo 自己指定** —— 否則等於讓不可信的一方決定沙箱能看到什麼。

## 實測三：network namespace 有自己的 loopback

`--unshare-all` 包含 `--unshare-net`，會建立全新的 network namespace。這裡有個容易誤判的細節：

| 情境 | 結果 |
|---|---|
| 程式自己起 server 再連自己（純 loopback） | **可以** — 新 namespace 自帶 `lo` 介面 |
| 連主機上已在跑的服務（`127.0.0.1:<port>`） | **Connection refused** |
| 連外網 | DNS 解析失敗 |

第二行是重點：沙箱裡的 `127.0.0.1` 是**該 namespace 自己的** loopback，
跟主機的 `127.0.0.1` 不是同一個。

實務影響：整合測試如果自己啟動依賴服務就跑得起來；如果假設「DB 已經在 localhost 跑著」就不行。
這兩種寫法在沙箱外表現一樣，在沙箱內差別巨大。

**分類比統稱有用。** 遇到「測試需要網路」時，先分清楚是哪一種：

```text
A  測試自己 spawn 服務        → 沙箱內可行
B  連 localhost 既有服務      → 不可行
C  連 Docker network 服務名   → 不可行（連 DNS 都沒有）
D  連主機服務                 → 不可行
E  需要外網 API               → 不可行
```

這五種的解法完全不同。太早統稱成「需要網路」，會導致設計出錯誤的例外機制。

## 實測四：`execFile` 的 maxBuffer 是殺行程，不是截斷

Node 的 `child_process.execFile` 有 `maxBuffer` 選項。直覺以為超過就截斷輸出，實際上：

```text
err.code = ERR_CHILD_PROCESS_STDIO_MAXBUFFER   ← 字串，不是數字
行程被殺掉
保留的是輸出的「開頭」，結尾丟失
```

三個後果：

1. `err.code` 是字串，如果程式碼寫 `typeof code === 'number' ? code : null`，
   會得到 `exitCode = null`，然後被當成失敗 —— 但原因完全看不出來。
2. 測試框架的摘要在**結尾**，必然丟失，所以任何依賴解析摘要的邏輯都會失效。
3. 如果拿「保留下來那段的末端」當作 log tail 顯示給使用者，
   看起來會像「程式跑到一半卡住」，極具誤導性。

正確處理是把它視為**執行沒跑完**（語意接近 timeout），而不是「輸出有點長」。

## 一個結構性的教訓

實作到後期才發現：Agent 的執行環境和驗證的執行環境**不一致**。

```text
Agent 執行:  /tmp 唯讀（為了防止它把東西藏在 workspace 外）
驗證執行:    /tmp 是可寫的 private tmpfs
```

後果是 Agent 無法預演驗證會做的事。它跑測試時因為寫不了暫存檔而中止，
於是回報「測試沒過」，而驗證那邊實際上是通過的。**兩邊說法矛盾，使用者不知道該信誰。**

這裡要區分兩個概念，否則會下錯判斷：

```text
capability  ≠  isolation
```

驗證那邊的 `/tmp` 雖然可寫，但那是 private tmpfs，主機的 `/tmp` 對兩邊同樣不可見。
從 host security boundary 看**沒有誰比誰弱**，所以這不是「驗證的隔離比 Agent 鬆」，
而是**執行語意不一致**（execution parity）。

判斷「這是安全問題還是一致性問題」會導向完全不同的修法，不能混。

## 適用性

以上都在 Linux + bubblewrap 環境實測。結論中可移植的部分：

- 沙箱模式的名字只描述它明確宣稱的那一項，其他一律要自己驗
- 唯讀掛載不能保護憑證，要用遮蔽（tmpfs）而不是唯讀
- 網路隔離要先分類需求形態，再決定例外機制
- 執行環境的差異要當一等公民處理，不是實作細節
