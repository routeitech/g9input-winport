# 字庫 — `cj5_map.txt`

> 倉頡五代碼對照表。讓使用者選字後，可透過「轉倉」輸出模式，把字碼送到系統倉頡 IME
> 由它完成最終組字動作。

---

## 一、角色定位

`cj5_map.txt` 只在使用者啟用**「轉倉」輸出模式**時才會被使用。它回答這個問題：
**「中文字 X 的倉頡五代碼是什麼？」**

流程範例：

```
使用者在九格輸入法選到「華」字
  │
  ▼
輸出模式 = 轉倉
  │
  ▼
查 cj5_map.txt["華"] = "tmj"
  │
  ▼
模擬鍵盤送出 T、M、J、Space（三個倉頡字根鍵 + 一個完字 Space）
  │
  ▼
作業系統的倉頡 IME 攔截這 4 個按鍵，組出「華」字
  │
  ▼
游標所在的文字欄位收到「華」
```

**關鍵**：這個流程把最終的字元寫入交給系統 IME，而不是我們自己寫入。用途是：

1. **相容性**：某些老舊應用只接受特定 IME 的輸入（例如某些銀行網頁的表單、舊版 Office
    某些欄位），直接送 Unicode 會被過濾掉
2. **不改變使用者習慣**：使用者的系統已經安裝了倉頡 IME，用這個模式可以保留該 IME 的
    詞庫、候選字記憶等

**沒有 `cj5_map.txt` 的話**：轉倉輸出模式無法使用（會自動 fallback 到剪貼簿模式）。

---

## 二、檔案格式

### 2-1. 整體結構

純文字檔案。**每行**一筆對應：

```
<char><code>
```

- `<char>` 是**一個字素**（grapheme），通常是單一 CJK 字元
- `<code>` 是倉頡碼字串，由 ASCII 小寫字母組成（`a`~`z`），1~5 個字母

**沒有分隔符**。程式以第一個字素作為 key，剩下全部作為 value。

例：

```
日a
曰a
昌aa
昍aa
晶aaa
暘aamh
晹aaph
暍aapv
曝aate
暻aayf
```

解析：

| char | code | 倉頡字根 |
|---|---|---|
| 日 | `a` | 日 |
| 曰 | `a` | 日 |
| 昌 | `aa` | 日日 |
| 昍 | `aa` | 日日 |
| 晶 | `aaa` | 日日日 |
| 暘 | `aamh` | 日日一竹 |
| 晹 | `aaph` | 日日心竹 |
| 暍 | `aapv` | 日日心女 |
| 曝 | `aate` | 日日廿水 |
| 暻 | `aayf` | 日日卜火 |

### 2-2. 編碼與換行

- **編碼**：UTF-8
- **換行**：`\n` 或 `\r\n`
- **空白行**：略過
- **行首／行尾空白**：避免

### 2-3. 倉頡字根對應表

倉頡字根共 26 個，對應 ASCII 字母 `a`~`z`。完整對照：

| 字母 | 字根名稱 | 字形 |
|---|---|---|
| a | 日 | 日 |
| b | 月 | 月 |
| c | 金 | 金 |
| d | 木 | 木 |
| e | 水 | 水 |
| f | 火 | 火 |
| g | 土 | 土 |
| h | 竹 | 竹 |
| i | 戈 | 戈 |
| j | 十 | 十 |
| k | 大 | 大 |
| l | 中 | 中 |
| m | 一 | 一 |
| n | 弓 | 弓 |
| o | 人 | 人 |
| p | 心 | 心 |
| q | 手 | 手 |
| r | 口 | 口 |
| s | 尸 | 尸 |
| t | 廿 | 廿 |
| u | 山 | 山 |
| v | 女 | 女 |
| w | 田 | 田 |
| x | 難 | 難 |
| y | 卜 | 卜 |
| z | 重 | 重（特殊碼） |

這是**倉頡五代**（亦稱「蒼頡/仓颉第五代」，1992 年朱邦復先生釋出公版的版本）的字根定義。
DEMO 版的 `cj5_map.txt` 遵循此版本。

### 2-4. 同一字多個倉頡碼

有些字在倉頡系統裡有**多個合法碼**。本表**只支援單一碼**——每個字在檔案裡只應出現一次。
如果重複：

```
華tmj
華mtj
```

後一行會**覆蓋**前一行（`Dictionary` 的 indexer 賦值是後到覆蓋）。選取哪個碼是你的決定，
建議選**最通用／最標準**的版本。

### 2-5. char 可以是多字元嗎？

**不行**。程式用 `StringInfo.GetNextTextElement` 解析第一個字素作為 char，剩下全部當 code。

但如果是 surrogate pair 表示的單一字素（如 U+20000「𠀀」），**可以**：

```
𠀀hmv
```

這樣 `<char>` = `"𠀀"`（佔 2 個 UTF-16 char 但算 1 個字素），`<code>` = `"hmv"`。

### 2-6. code 只能是字母嗎？

**必須是 ASCII 小寫字母** `a`~`z`。不接受：

- 大寫字母（`A`~`Z`）
- 數字
- 符號

這是因為程式要把 code 字元轉成對應的 Windows Virtual Key 事件送給作業系統，轉換邏輯是：

```csharp
// 簡化版
foreach (char c in code)
{
    VirtualKey vk = c switch
    {
        >= 'a' and <= 'z' => (VirtualKey)(c - 'a' + 0x41),  // 'a'=0x41=VK_A, etc.
        _ => VirtualKey.None  // 不支援
    };
    SendKey(vk);
}
SendKey(VirtualKey.Space);  // 倉頡用 Space 完字
```

大寫字母和數字在 VK 空間有不同的值，需要另外的處理，本程式目前不支援。

---

## 三、key 設計原則（哪些字要收）

### 3-1. 最小必要集合

**把 `char_map.txt` 裡所有的字都收進 `cj5_map.txt`**。否則使用者在轉倉模式下會遇到
「某字找不到倉頡碼」而自動 fallback 到剪貼簿，體驗不一致。

### 3-2. 擴充集合

超出 `char_map.txt` 的字沒必要收——因為使用者不可能選中這些字（它們不在候選字表裡）。

### 3-3. HKSCS PUA 字元

**不建議**給 PUA 字元寫倉頡碼。原因：

1. 大部分作業系統的倉頡 IME 不認得 PUA 字，即使送對的碼，IME 也組不出字
2. 程式會在輸出前先做 HKSCS → Unicode 轉換（詳見 `b5_uni.txt`），PUA 字早就被換成
    標準 Unicode
3. 如果該標準 Unicode 字元不在 `cj5_map.txt`，就會 fallback 剪貼簿

---

## 四、value 設計原則

### 4-1. 字碼長度

倉頡碼最長 5 個字母。超過 5 字母的 code 程式不會報錯，但系統倉頡 IME 會拒絕組字，
使用者只會看到幾個字母被打到文字框而沒有實際的中文字出現。

### 4-2. 選擇「標準」的倉頡碼

當一個字有多個合法碼時，建議選：

1. **朱邦復官方版（倉頡第五代 1992 年公版）**的取碼
2. 如果你是用 Windows 內建倉頡 IME，優先選該 IME 接受的碼（可用該 IME 自己試打驗證）
3. 民間流通的「倉頡 6」「蒼頡簡易」等分支不建議採用——相容性較差

### 4-3. 驗證方式

對你維護的 `cj5_map.txt`，可以用 Windows 內建的倉頡 IME（或 OpenVanilla 的 CJ5）
做抽樣驗證：

1. 打開系統倉頡 IME（Win + Space 切換）
2. 從 `cj5_map.txt` 隨機抽 30 字
3. 手動輸入每個字的 code + Space，檢查是否能組出該字
4. 不行的就有錯，查原因

---

## 五、載入行為與實作細節

### 5-1. 載入時機

程式啟動時載入，`FrozenDictionary<string, string>`，重啟生效。

### 5-2. 查詢時機

只在使用者觸發**轉倉輸出**時查詢。具體路徑：

```
使用者選字（例：華）
  ↓
OutputService.Send("華")
  ↓
檢查 OutputMode == CangjieRelay
  ↓ Yes
  查 Cj5Map["華"]
    ↓ 有結果（如 "tmj"）
    送 VK_T, VK_M, VK_J, VK_SPACE → 系統 IME
    ↓ 沒結果
    fallback 到 Clipboard 模式（剪貼簿 + Ctrl+V）
```

### 5-3. Unicode 輸出 vs VK 事件的差別

本程式的其他輸出模式（直出、剪貼簿）送的是 **Unicode 碼點**（經由 `KEYEVENTF_UNICODE`
或 Ctrl+V），目標程式不需要 IME 就能接收。

轉倉模式送的是 **VK 事件**（模擬鍵盤按鍵），目標程式和作業系統之間的 IME 需要介入。
這是為什麼：

1. 轉倉模式需要**系統倉頡 IME 被啟用**（語言列切到「中文（繁體）— 倉頡」或相容的 IME）
2. 送 VK 前後**不要切換 IME**，否則組字會失敗

---

## 六、與其他字庫的關係

### 6-1. 與 `char_map.txt`

強烈建議兩張表的**字集一致**。缺字會導致轉倉模式下該字 fallback 到剪貼簿，使用者體驗
不一致（有時候是轉倉 IME 幫忙組字，有時候是直接貼上）。

### 6-2. 與 `b5_map.txt`

`b5_map.txt` 是大易碼版的平行表。兩張表通常字集相同，只是碼不同。維護兩張表時可
共用同一份「字清單」：

```python
# 從 char_map.txt 抽出所有字
chars = set()
with open("char_map.txt", encoding="utf-8") as f:
    for line in f:
        if "," in line:
            chars.update(c for c in line.split(",", 1)[1] if c != "*")

# 驗證 cj5_map.txt 有覆蓋所有字
cj5_chars = set()
with open("cj5_map.txt", encoding="utf-8") as f:
    for line in f:
        line = line.rstrip()
        if line:
            cj5_chars.add(line[0])

missing = chars - cj5_chars
print(f"{len(missing)} chars missing in cj5_map.txt: {''.join(sorted(missing))}")
```

### 6-3. 與 `b5_uni.txt`

PUA 字會**先**被 `b5_uni.txt` 轉換成標準 Unicode，**再**查 `cj5_map.txt`。所以你不用
在 `cj5_map.txt` 裡寫 PUA 字的倉頡碼。

---

## 七、實例分析

DEMO `cj5_map.txt` 的開頭片段：

```
日a
曰a
昌aa
昍aa
晶aaa
暘aamh
晹aaph
暍aapv
曝aate
暻aayf
㬙aayf
㫼aatc
暱aayv
旧aal
晤aamr
晧aahgr
暉abjj
...
```

觀察：

- **按倉頡碼字母順序排**——先是所有 `a` 開頭的碼，再是 `aa`，然後 `aaa`…
- **部分字有相同碼**：「日」和「曰」碼都是 `a`。在倉頡中，使用者要靠出字順序或翻頁
    選擇這兩者。
- **擴展字元也有**：「㬙」「㫼」（U+3B19、U+2BEC5 等區）都可在倉頡 IME 裡打出

此外有一個特殊情況要注意：**按照倉頡的規則，有些字沒有 5 個字根**（會使用「x」作為
「難字」標記）。程式不必特別處理——把碼原樣送出去就對了。

---

## 八、作者的實務建議

### 8-1. 直接用既有的倉頡碼表

坊間已有多個公開的倉頡碼表：

- **Unicode 官方**的 [Unihan Database](https://www.unicode.org/Public/UNIDATA/Unihan.zip)
    裡有 kCangjie 欄位，含倉頡 5 代碼
- **Open Source IME 專案**（例如 rime-cangjie5、IBus-Table 的倉頡表）都有現成資料
- **朱邦復先生網站**（[倉頡輸入法世界](http://cj5.cbflabs.com/)）有官方版本

取得後需要把格式轉換成本程式的 `<char><code>` 格式，用 Python/Awk 一行指令就能搞定。

### 8-2. 與 `char_map.txt` 對齊的工作流

```python
# 從完整倉頡字集，過濾出 char_map.txt 裡有的字
chars_in_char_map = set(...)  # 由前面的腳本抽出

with open("full_cangjie.txt") as f_in, \
     open("cj5_map.txt", "w", encoding="utf-8") as f_out:
    for line in f_in:
        line = line.rstrip()
        if not line:
            continue
        char = line[0]
        if char in chars_in_char_map:
            f_out.write(line + "\n")
```

### 8-3. Windows 內建倉頡 vs 第三方 IME 的碼差異

如果你的使用者主要用 Windows 內建倉頡，建議用 **Windows 內建倉頡的碼**做驗證——
部分字在不同 IME 有碼差（例如朱邦復原始版 vs Microsoft 版）。你的 `cj5_map.txt` 要和
使用者實際用的 IME 一致才能組字成功。

---

## 九、常見錯誤診斷

| 症狀 | 可能原因 |
|---|---|
| 啟用轉倉後，打字不出字，只出現幾個字母 | 系統 IME 沒切到倉頡，或該字的 code 在該 IME 下不合法 |
| 某些字在轉倉模式下變成貼上 | 該字不在 `cj5_map.txt`，程式 fallback 到剪貼簿 |
| 某字的倉頡碼打出後出現別的字 | 多字共用同一碼，系統 IME 預設選了第一個。這是倉頡 IME 的行為，本程式無法控制 |
| 符號也會被轉倉 | 啟用了「轉倉符號輸出」選項。關閉則標點直接貼上 |
| 程式變慢 | 字庫太大？`cj5_map.txt` 105 KB 算正常，>1 MB 會影響啟動時間 |

---

## 十、快速參考卡

```
格式：<char><code>（無分隔符）
編碼：UTF-8
char：一個字素
code：1~5 個 ASCII 小寫字母（a~z），對應倉頡字根
重複 key：後到覆蓋
空行：允許，略過
```

---

**接下來請看：[`字庫_b5_map.md`](字庫_b5_map.md)**
