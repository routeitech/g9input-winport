# 字庫 — `b5_map.txt`

> 大易碼對照表。讓使用者選字後，可透過「轉易」輸出模式，把字碼送到系統大易 IME
> 由它完成最終組字動作。

---

## 一、角色定位

`b5_map.txt` 只在使用者啟用**「轉易」輸出模式**時才會被使用。它回答這個問題：
**「中文字 X 的大易碼是什麼？」**

整體流程和「轉倉」（`cj5_map.txt`）完全相同，只是查的是大易碼、送給系統大易 IME 組字。

```
使用者在九格輸入法選到「你」字
  ↓
輸出模式 = 轉易
  ↓
查 b5_map.txt["你"] = "ad9"
  ↓
送 VK_A, VK_D, VK_9, VK_SPACE → 系統大易 IME
  ↓
游標位置收到「你」
```

**沒有 `b5_map.txt` 的話**：轉易輸出模式無法使用（自動 fallback 到剪貼簿）。

---

## 二、檔案格式

### 2-1. 整體結構

純文字檔案。每行一筆：

```
<char><code>
```

- `<char>` 是一個字素
- `<code>` 是大易碼字串

**沒有分隔符**。第一個字素是 key，剩下是 value。

例：

```
的/d.
是de9
不h9
我vg5
一e
有sj
大v
在s1f
人a
了bj
```

解析：

| char | code | 大易字根／符號 |
|---|---|---|
| 的 | `/d.` | 「/」「d」「.」三個鍵 |
| 是 | `de9` | 「d」「e」「9」 |
| 不 | `h9` | 「h」「9」 |
| 我 | `vg5` | 「v」「g」「5」 |
| 一 | `e` | 「e」 |
| 有 | `sj` | 「s」「j」 |
| 大 | `v` | 「v」 |
| 在 | `s1f` | 「s」「1」「f」 |
| 人 | `a` | 「a」 |
| 了 | `bj` | 「b」「j」 |

### 2-2. 編碼與換行

- **編碼**：UTF-8
- **換行**：`\n` 或 `\r\n`
- **空白行**：略過

### 2-3. 大易字根與符號

和倉頡不同，**大易的碼可以包含符號**，不只是字母：

- **ASCII 小寫字母** `a` ~ `z`（26 個字根）
- **ASCII 數字** `0` ~ `9`（10 個擴展鍵）
- **符號**（`/`、`.`、`,`、`;` 等，實際視你所用的大易 IME 而定）

大易 IME 的完整按鍵表（以最常見的「大易 3.0」為例）：

```
QWERTYUIOP   → q w e r t y u i o p
ASDFGHJKL;'  → a s d f g h j k l ; '
ZXCVBNM,./   → z x c v b n m , . /
數字列 0-9    → 0 1 2 3 4 5 6 7 8 9
```

全部 40 個字根／擴展鍵。

### 2-4. 字根對照（大易 3.0 為例）

這個對照表僅供你製碼時參考。程式本身不在意碼的語意，只負責忠實傳遞：

| 鍵 | 字根 | 鍵 | 字根 | 鍵 | 字根 |
|---|---|---|---|---|---|
| q | 氵 | a | 人 | z | 乙 |
| w | 忄 | s | 日 | x | 丶 |
| e | 十 | d | 土 | c | 口 |
| r | 言 | f | 工 | v | 大 |
| t | 石 | g | 一 | b | 月 |
| y | 立 | h | 山 | n | 弓 |
| u | 田 | j | 女 | m | 門 |
| i | 目 | k | 木 | , | 犬 |
| o | 曰 | l | 火 | . | 糸 |
| p | 禾 | ; | 車 | / | 耳 |
| ' | 鳥 | | | | |

（此表為示意，實際字根略有差異，詳見你所用的大易 IME 說明。）

### 2-5. 特殊字元的處理：送 VK 事件

和 `cj5_map.txt` 一樣，程式要把每個 code 字元轉成 Windows Virtual Key 事件：

```csharp
// 簡化版
foreach (char c in code)
{
    VirtualKey vk = c switch
    {
        >= 'a' and <= 'z' => (VirtualKey)(c - 'a' + 0x41),  // VK_A..VK_Z
        >= '0' and <= '9' => (VirtualKey)(c - '0' + 0x30),  // VK_0..VK_9
        '/'  => VirtualKey.OemSlash,
        '.'  => VirtualKey.OemPeriod,
        ','  => VirtualKey.OemComma,
        ';'  => VirtualKey.OemSemicolon,
        '\'' => VirtualKey.OemQuotes,
        _    => VirtualKey.None
    };
    SendKey(vk);
}
SendKey(VirtualKey.Space);
```

所以**任何 ASCII 可印字元**（除了空白）都可以出現在大易碼中，但效果取決於系統大易 IME
如何解讀這些按鍵。

### 2-6. 重複 key

後到覆蓋（`Dictionary` 的 indexer 賦值行為）。

---

## 三、key 設計原則

和 `cj5_map.txt` 完全一樣：

### 3-1. 至少涵蓋 `char_map.txt` 的所有字

否則使用者在轉易模式下會遇到字被 fallback 到剪貼簿，體驗不一致。

### 3-2. 不需收錄 PUA 字元

PUA 字會先被 `b5_uni.txt` 轉成標準 Unicode，才查 `b5_map.txt`。

### 3-3. 擴展 CJK（B~F 區）的支援

大易 IME 通常**不支援**擴展區 B 以後的字元（U+20000 以上）。這些字元即使你寫了碼，
送出後系統 IME 也組不出字，使用者只會看到按鍵被打到文字框。

**建議**：對擴展 CJK 字元**不要**收錄倉頡碼，讓它們自然 fallback 到剪貼簿輸出。

---

## 四、value 設計原則

### 4-1. 碼長度

大易碼通常是 1~3 個字元（絕大多數字），極少數達到 4 個。如果你的字庫出現 5+ 字的 code，
值得檢查一下是不是有誤。

### 4-2. 驗證碼的正確性

和 `cj5_map.txt` 一樣，用系統內建的大易 IME 手動抽查：

1. 切到大易 IME（Win + Space）
2. 從 `b5_map.txt` 抽樣 30 字
3. 逐一打 code + Space，看 IME 能否組出字
4. 組不出的記下，查原因

---

## 五、載入行為與實作細節

### 5-1. 載入時機

啟動時載入，`FrozenDictionary<string, string>`，重啟生效。

### 5-2. 查詢時機

只在**轉易**輸出模式時被查詢：

```
OutputService.Send("你")
  ↓ OutputMode == DayiRelay
查 B5xpMap["你"]
  ↓ 有結果（如 "ad9"）
送 VK_A, VK_D, VK_9, VK_SPACE
  ↓ 沒結果
fallback 到 Clipboard
```

### 5-3. 為什麼屬性叫 `B5xpMap`？

`b5_map.txt` 在程式內的 `CharTables` 類別裡叫 `B5xpMap`（取自舊名「B5xp」或
「大易 Beta 5 XP」版本）。這是歷史遺留的命名，外部使用者只需要關心檔名就好。

---

## 六、與其他字庫的關係

### 6-1. 與 `char_map.txt`

建議字集一致。缺字導致 fallback 到剪貼簿，體驗下降。

### 6-2. 與 `cj5_map.txt`

平行的角色，一個是倉頡、一個是大易。使用者可依偏好啟用其中一個輸出模式。**兩者並存不衝突**。

通常同一套核心字集，兩張表都要提供，讓使用者可自由選擇輸出模式。

### 6-3. 與 `b5_uni.txt`

`b5_uni.txt` 和 `b5_map.txt` **名字相似但功能完全不同**：

- `b5_uni.txt` = Big5 編碼 ↔ Unicode 碼點對照（用於 HKSCS PUA 字元自動轉換）
- `b5_map.txt` = 中文字 → **大易碼**對照（用於轉易輸出）

命名很容易搞混。請記住：
- `b5_uni` = Big5（編碼）↔ **Uni**code
- `b5_map` = Big5（大易碼名稱歷史）**map** = 大易碼對照

---

## 七、實例分析

DEMO `b5_map.txt` 的開頭（前 10 字是全文前十個高頻字）：

```
的/d.
是de9
不h9
我vg5
一e
有sj
大v
在s1f
人a
了bj
```

觀察：

- **包含符號**：「的」的碼 `/d.` 用了「/」和「.」兩個符號鍵，這是大易的特色
- **極短碼**：「一」碼只有 `e`，這是極高頻字給予的「一鍵打成」優待
- **數字碼**：「我」的碼 `vg5` 用了數字 `5`，表示第 5 種變體（大易用數字區分同根但不同字）

### 7-1. 其他片段

```
么5/
九fbj
九3/
了jn
了c3/
```

「九」和「了」都有多個可能的寫法。本表處理方式：**只保留最後一筆**。範例中：

- 「九」出現 2 次，最後一筆是 `九3/` → 最終 key `九` → value `3/`
- 「了」出現 3 次，最後一筆是 `了c3/` → 最終 key `了` → value `c3/`

這是 `Dictionary` 的預設行為。如果你希望保留特定版本的碼，把那一筆**放在檔案最後面**。

---

## 八、作者的實務建議

### 8-1. 直接採用現有大易碼表

和倉頡類似，大易的碼表有公開版本：

- **Unicode 官方 Unihan** 的 kJapanese... 欄位**不含**大易碼（大易是台灣特有輸入法）
- **Open source**：[gcin](https://github.com/dougrich/gcin) 等 Linux IME 專案有 `.cin` 格式的大易表
- **台灣各大中文輸入法論壇**也有完整表的 CSV/TXT 分享

建議取一份現成表，轉成本程式需要的 `<char><code>` 格式即可。

### 8-2. 產生工具範例

假設你拿到一份 `.cin` 格式的原始大易表：

```
# dayi.cin 範例
%chardef begin
一 e
二 e9
三 ec
...
%chardef end
```

轉換腳本：

```python
import re

with open("dayi.cin", encoding="utf-8") as f:
    lines = f.readlines()

out = []
in_def = False
for line in lines:
    line = line.rstrip("\n")
    if line.strip().startswith("%chardef begin"):
        in_def = True
        continue
    if line.strip().startswith("%chardef end"):
        in_def = False
        continue
    if not in_def:
        continue
    # 格式：<code> <TAB> <char>  或  <code> <space> <char>
    m = re.match(r"^(\S+)\s+(\S+)$", line)
    if m:
        code, char = m.groups()
        # 注意：.cin 通常是「code 在前，char 在後」，本程式是「char 在前」
        out.append(f"{char}{code}")

with open("b5_map.txt", "w", encoding="utf-8") as f:
    f.write("\n".join(out) + "\n")
```

### 8-3. 字集對齊

```python
# 從 char_map.txt 抽出所有字
chars_in_char_map = set(...)

# 找出 b5_map.txt 中多餘的行（在 char_map 外的字）
extra_lines = []
with open("b5_map.txt", encoding="utf-8") as f:
    for line in f:
        line = line.rstrip("\n")
        if line and line[0] not in chars_in_char_map:
            extra_lines.append(line)

print(f"Can remove {len(extra_lines)} unused lines from b5_map.txt")
```

---

## 九、常見錯誤診斷

| 症狀 | 可能原因 |
|---|---|
| 啟用轉易後打字無反應 | 系統 IME 沒切到大易 |
| 某些字 fallback 到剪貼簿 | 該字不在 `b5_map.txt` |
| 某字送出的 code 打出別的字 | 碼對應大易 IME 裡另一個字，或 IME 版本不同（3.0 vs 6.0） |
| 符號被奇怪解讀 | `/`、`.` 等符號鍵的 VK 對應在某些鍵盤布局下不同（例如日系 JIS 鍵盤），考慮提醒使用者使用標準 US 鍵盤 |

---

## 十、快速參考卡

```
格式：<char><code>（無分隔符）
編碼：UTF-8
char：一個字素
code：1~N 個 ASCII 可印字元（字母、數字、符號皆可）
重複 key：後到覆蓋
建議長度：1~4 字元
```

---

**接下來請看：[`字庫_homo_map.md`](字庫_homo_map.md)**
