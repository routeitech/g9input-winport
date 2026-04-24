# 字庫 — `b5_uni.txt`

> Big5 ↔ Unicode 對照表。讓程式能自動把 HKSCS 私人使用區（PUA）的字元轉換成標準
> Unicode 碼點，確保輸出在不支援 HKSCS 的應用中仍然可見。

---

## 一、角色定位

`b5_uni.txt` 回答這個問題：
**「某個 Big5 編碼的位元組對應到哪一個 Unicode 碼點？」（反向亦然）**

它的核心用途：**HKSCS PUA 字元的自動轉換**。

### 1-1. 背景：為什麼需要這張表

在 Unicode 普及之前，香港曾擴展 Big5 編碼以容納粵語方言字和罕用漢字，形成了
**HKSCS**（Hong Kong Supplementary Character Set，香港增補字符集）。這批字元在
當時被映射到 Big5 的空白區域，並在電腦顯示時放在 Unicode 的 PUA 區
（**U+E000** ~ **U+F8FF**，Private Use Area）。

隨著 Unicode 的 CJK 擴展 B、C、D、E、F 逐步推出，原本放在 PUA 的 HKSCS 字元**陸續**
都有了**標準**的 Unicode 碼點。例如粵語常用字「嘅」：

- 在某些舊系統中：**U+F14B**（PUA 區）
- 在現代 Unicode 中：**U+55CE**（BMP 區的 CJK 統一漢字）

### 1-2. 輸出時的轉換流程

本程式在**送字**前會做這件事：

```
候選字 UI 選中「嘅」（U+F14B，PUA 區）
   ↓
OutputService.Send(char)
   ↓
偵測到字元在 U+E000 ~ U+F8FF 範圍內
   ↓
用 Big5 編碼反查：字元 → Big5 兩位組（例 0x8831）
   ↓
查 b5_uni.txt["8831"] = "55CE"
   ↓
把 U+F14B 替換成 U+55CE，送出「嘅」（現代 Unicode）
   ↓
目標應用顯示為「嘅」，即使該應用沒有 HKSCS 字型
```

### 1-3. 在 UI 中的提示

候選字 UI 會把 PUA 區的字元**以紅色**渲染，視覺上提醒使用者：「這個字會經過自動轉換」。

**沒有 `b5_uni.txt` 的話**：PUA 字元會**原樣**送出，在不支援 HKSCS 的應用中顯示為豆腐
方塊或亂碼。

---

## 二、檔案格式

### 2-1. 整體結構

純文字檔案。每行一筆對應：

```
<Big5Hex>,<UnicodeHex>
```

- `<Big5Hex>` 是 Big5 編碼的 4 位十六進位數（大寫，例 `8831`、`FB40`）
- `<UnicodeHex>` 是 Unicode 碼點的 4~5 位十六進位數（大寫，例 `55CE`、`20000`）
- 中間用**半形逗號 `,`** 分隔

例：

```
8831,55CE
8839,9EDE
883A,9EBB
883B,9EC1
883E,A012
8840,9FB4
8841,9FB5
8842,9FB6
8843,9FB7
8845,9FB8
```

### 2-2. 編碼與換行

- **檔案編碼**：UTF-8（儘管內容都是 ASCII 十六進位）
- **換行**：`\n` 或 `\r\n`
- **空白行**：略過

### 2-3. Big5Hex 的範圍

HKSCS 的 PUA 字元通常落在下列 Big5 範圍：

- `8140` ~ `A0FE`（HKSCS 擴展區 1）
- `FA40` ~ `FEFE`（HKSCS 擴展區 2）
- `8E40` ~ `A0FE`（HKSCS 擴展區 3）

完整列表請參閱香港特別行政區政府官方的 HKSCS-2016 規格。

### 2-4. UnicodeHex 的範圍

目標 Unicode 碼點可能落在：

- **BMP 區**（`0020` ~ `FFFD`，4 位十六進位）：最常見，如 `55CE`、`9EDE`
- **SMP 區**（`10000` ~ `1FFFD`，5 位十六進位）：CJK 擴展 B、C、D 等，如 `20000`

程式讀取時用 `int.Parse(hex, NumberStyles.HexNumber)` 解析，兩種長度都能處理。

### 2-5. 雙向查詢

程式啟動時會把這張表建立成**兩張** `Dictionary`：

```
b5_to_uni: { 0x8831 → U+55CE, 0x8839 → U+9EDE, ... }
uni_to_b5: { U+55CE → 0x8831, U+9EDE → 0x8839, ... }
```

雙向查詢的用途：

- **b5_to_uni**：從候選字 UI 的 Unicode 字元（可能是 PUA）查到標準 Unicode
- **uni_to_b5**：「編碼過濾」模式（Big5 only）下，判斷某字是否能被 Big5 編碼表示

### 2-6. 重複 key

```
8831,55CE
8831,9000
```

只保留**最後一筆**（`Dictionary` 的 indexer 賦值，後到覆蓋）。但建議**不要**讓同一個
`Big5Hex` 出現多次——代表資料重複，應該清理。

---

## 三、key 設計原則

### 3-1. 完整性優先

這張表**不是**給你手動編寫的——它是一份**標準資料**，應該從官方／第三方的 HKSCS
mapping 檔案轉換而來。

建議來源：

1. **香港政府數碼政策辦公室** 的 HKSCS-2016 官方規格（可下載 `HKSCS-2016.zip`）
2. **Unicode Consortium** 的 [Big5-HKSCS mapping](https://www.unicode.org/Public/MAPPINGS/OBSOLETE/EASTASIA/OTHER/BIG5.TXT)
    （注意：此檔已標為 OBSOLETE，但仍是歷史參考）
3. **HKSCS 2008** 或更早版本，如果你的使用者環境較舊

### 3-2. 不要手動增減

和其他字庫不同，這張表的條目**嚴謹對應到正式字符集規範**。如果你隨意增刪：

- 新增一個不存在的映射 → 送字時把 PUA 字元換成不相干的 Unicode，顯示錯誤
- 刪除一個必要的映射 → 該 PUA 字元被原樣送出，在現代應用中顯示方塊

如果你發現使用者會用到**額外的**香港或台灣方言字，且已知它的標準 Unicode 碼點，
可以自行加入一行。但前提是你能確認對應關係。

### 3-3. 不涵蓋基本 Big5

注意：**這張表只涵蓋 PUA 映射**，不是完整的 Big5 ↔ Unicode 對照表。對於普通的繁體
字（例如「我」在 Big5 是 `A7DA`、Unicode 是 `6211`），這裡**不會**也**不應該**出現。
原因：

- 程式在候選字 UI 裡顯示的就已經是 Unicode
- 一般字元的 Big5↔Unicode 映射使用 .NET 的 `Encoding.GetEncoding(950)`（Big5） API
    即可處理
- 只有 PUA 區的 HKSCS 字元需要另外維護，因為它們的映射是**約定俗成**而非由標準
    編碼函式庫決定

---

## 四、value 設計原則

### 4-1. 選擇正確的 Unicode 碼點

如果一個 HKSCS PUA 字元在 Unicode 中有**多個候選**碼點（罕見但存在），選原則：

1. **BMP 優先**：能用 BMP（`0000-FFFF`）就不用 SMP（`10000+`）——BMP 的字型覆蓋較廣
2. **正式 CJK 區優先**：`4E00-9FFF` > `3400-4DBF`（擴展 A）> `20000-2A6DF`（擴展 B）
3. **相容性字元最後**：CJK Compatibility Ideographs（`F900-FAFF`）和兼容區補充
    （`2F800-2FA1F`）通常有對應的正規形式，除非有特殊理由否則避免

### 4-2. 跟官方標準走

如果你用 HKSCS-2016 官方映射，它已經幫你做過這些取捨。跟著官方走是最安全的。

---

## 五、載入行為與實作細節

### 5-1. 載入時機

程式啟動時載入，建立**兩張** `FrozenDictionary`（b5→uni 和 uni→b5）。兩張表都凍結後
供後續查詢。

### 5-2. 查詢時機

**時機 1：送字前的 PUA 轉換**

```csharp
// 簡化版邏輯
public string NormalizeForOutput(string text)
{
    var sb = new StringBuilder();
    foreach (var grapheme in ParseGraphemes(text))
    {
        if (grapheme.Length == 1 && grapheme[0] >= 0xE000 && grapheme[0] <= 0xF8FF)
        {
            // PUA 區，嘗試轉換
            int puaPoint = grapheme[0];
            if (UniToB5.TryGetValue(puaPoint, out int big5)
                && B5ToUni.TryGetValue(big5, out int stdPoint)
                && stdPoint != puaPoint)
            {
                sb.Append(char.ConvertFromUtf32(stdPoint));
                continue;
            }
        }
        sb.Append(grapheme);
    }
    return sb.ToString();
}
```

**時機 2：編碼過濾模式**

```csharp
public bool CanEncodeInBig5(string text)
{
    // 嘗試把 text 用 Big5 編碼
    // 如果失敗，檢查 uni_to_b5 中是否有該字的 HKSCS 映射
    // 都失敗 → 該字不能用 Big5 表示
}
```

### 5-3. UI 渲染的紅色標示

候選字格渲染時會檢查 `IsInPua(char)`，為 true 則套用紅色前景色。這個檢查是**純程式碼
邏輯**，不查 `b5_uni.txt`——只要字元落在 `U+E000~U+F8FF` 就渲染紅色，不管有沒有實際的
HKSCS 映射。

---

## 六、與其他字庫的關係

### 6-1. 與 `char_map.txt`

如果 `char_map.txt` 裡放了 PUA 字元，它們在 UI 顯示為紅色，輸出時會經過轉換。

**建議**：優先在 `char_map.txt` 裡放 **標準 Unicode 碼點**（如「嘅」`U+55CE`）而不是
PUA 碼點（`U+F14B`），這樣：

- 使用者看到的是正常顏色（非紅色）的字
- 送字不需要轉換，效能稍好
- 輸出行為不依賴 `b5_uni.txt` 的正確性

但如果你繼承了舊字庫（可能混有 PUA 碼點），`b5_uni.txt` 就是讓舊資料仍然正常運作的
橋樑。

### 6-2. 與 `homo_map.txt`

粵語同音字表常包含方言字，這些字可能是 PUA 也可能是標準 Unicode。同樣建議存成
標準 Unicode，避免額外轉換。

### 6-3. 與 `cj5_map.txt` / `b5_map.txt`

**重要時序**：PUA 轉換發生在**送字時**，但轉倉／轉易模式的查詢發生在**轉換後**。
流程是：

```
1. 使用者選字：候選格顯示的字元（可能是 PUA）
2. 轉換：PUA → 標準 Unicode
3. 查碼：用標準 Unicode 查 cj5_map.txt 或 b5_map.txt
4. 送 VK 事件
```

因此 `cj5_map.txt` 和 `b5_map.txt` 的 key 應該用**標準 Unicode**，不要用 PUA 碼點。

---

## 七、實例分析

DEMO `b5_uni.txt` 的開頭片段：

```
8831,55CE
8839,9EDE
883A,9EBB
883B,9EC1
883E,A012
8840,9FB4
8841,9FB5
8842,9FB6
8843,9FB7
8845,9FB8
8846,9FB9
8847,9FBA
8848,9FBB
884B,3EAC
884C,3EB8
...
FB40,7631
FB41,8B75
...
```

逐行分析：

| Big5 | Unicode | 字 | 說明 |
|---|---|---|---|
| 8831 | 55CE | 嘅 | 粵語助詞 |
| 8839 | 9EDE | 點 | 異體字 |
| 883A | 9EBB | 麻 | 異體字 |
| 883B | 9EC1 | 黁 | 罕用字 |
| 884B | 3EAC | 㺬 | CJK 擴展 A |

觀察：

- **BMP 區為主**：大部分映射到 `55xx`、`9Exx`、`9Fxx` 等 BMP 漢字區
- **CJK 擴展 A**：`3400-4DBF` 區（如 `3EAC`）也有映射，這是補上香港字的 Unicode 正規形式
- **Big5 位址**是雙位元組值（8 位元 + 8 位元 = 16 位元十六進位 = 4 字元）
- **Unicode** 可能是 4 位（BMP）或 5 位（SMP）

---

## 八、作者的實務建議

### 8-1. 直接用官方 HKSCS-2016 mapping

**最推薦**的作法是：

1. 從香港政府網站下載 `HKSCS-2016.csv` 或類似的 mapping 檔
2. 用腳本轉成本程式的格式：

```python
import csv

with open("HKSCS-2016.csv", encoding="utf-8") as f_in, \
     open("b5_uni.txt", "w", encoding="utf-8") as f_out:
    reader = csv.DictReader(f_in)
    for row in reader:
        big5 = row["Big5"].upper()  # 如 "8831"
        unicode_hex = row["Unicode"].upper()  # 如 "55CE"
        # 跳過非 PUA 映射（如果檔案裡有雜項）
        f_out.write(f"{big5},{unicode_hex}\n")
```

### 8-2. 驗證 mapping

取一些已知對應關係的字測試：

| 字 | PUA 碼點 | 標準 Unicode | Big5 |
|---|---|---|---|
| 嘅 | U+F14B (某版本) | U+55CE | 0x8831 |
| 嘢 | U+F14C | U+55E2 | 0x8832 |
| 冇 | U+F14D | U+5187 | 0x8833 |

把這些字作為 `char_map.txt` 的候選字，實際在記事本、Word、瀏覽器各試一次輸出，
看是否正常。

### 8-3. 當 mapping 不存在時

如果一個 HKSCS PUA 字元**沒有**對應的標準 Unicode（例如某些罕見方言字還沒被收入 CJK
擴展），怎麼辦？

**方案 A**：在 `b5_uni.txt` **不加入**該字的映射。輸出時原樣送 PUA 字元，在不支援
HKSCS 字型的應用會顯示方塊。使用者要自己安裝 HKSCS 字型。

**方案 B**：用近似字替代（例如沒有標準碼的「𪘲」用「齒」替代）。這會讓輸出在任何
應用都能看到字，但字不是原字。**不推薦**，因為會造成語義錯誤。

一般採用方案 A，並在使用者文件中提示：「如果出現方塊，請安裝 HKSCS 字型」。

---

## 九、常見錯誤診斷

| 症狀 | 可能原因 |
|---|---|
| PUA 字輸出到記事本變方塊 | 記事本字型不支援 HKSCS；`b5_uni.txt` 缺少該字映射 |
| PUA 字輸出顯示完全不同的字 | `b5_uni.txt` 的對應錯誤（人為編輯失誤）；重新從官方檔產生 |
| 某字在候選格顯示紅色但輸出正常 | 正常——紅色只是「會經過轉換」的提示 |
| 某字在候選格顯示紅色但輸出方塊 | `b5_uni.txt` 缺該字映射，字被原樣送出 |
| 檔案載入失敗 | 檢查格式：是否有不是十六進位的字元、逗號是否半形、是否有意外的空白 |

---

## 十、快速參考卡

```
格式：<Big5Hex>,<UnicodeHex>
編碼：UTF-8（內容為 ASCII 十六進位）
分隔符：半形逗號（0x2C）
Big5Hex：4 位大寫十六進位（如 8831）
UnicodeHex：4 或 5 位大寫十六進位（如 55CE 或 20000）
載入：啟動時建立 b5→uni 和 uni→b5 兩張查表
觸發：字元落在 U+E000~U+F8FF 時自動查轉
資料來源：HKSCS-2016 官方 mapping，不宜手動編輯
```

---

**接下來請看：[`字庫_t2s.md`](字庫_t2s.md)**
