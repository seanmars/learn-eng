# Sync Report: sentence-list-playback

## Summary

Conclusion: PASS

13 條 Requirement 全部逐條對照過實作: 12 條 MATCH, 1 條 SPEC-UPDATED, 0 條 CODE-BUG.

## Requirements

### 新增句子

- Implementation: 純邏輯區塊的 addSentence (去頭尾空白, 空字串就回原陣列), 由表單的 submit handler 呼叫, 送出後 reset 表單
- Verdict: MATCH
- Notes: 三個 scenario (有內容, 空白, 重複) 都由自我檢查的 assert 與瀏覽器驗收情境覆蓋

### 清單顯示

- Implementation: render 依陣列順序產生每一項, 每項含勾選控制, 句子文字, 播放鈕與刪除鈕; 空陣列時改顯示提示列
- Verdict: MATCH
- Notes: 顯示順序即陣列順序, 無額外排序

### 句子選取

- Implementation: 選取是一組索引, 勾選狀態由 render 依這組索引決定; 新增只在尾端附加所以索引不動, 刪除呼叫 remapSelection, 清空一併歸零
- Verdict: MATCH
- Notes: 五個 scenario 全數覆蓋, 其中「刪除後選取仍對應原本的句子」是這條的關鍵路徑, 由 remapSelection 的 assert 與瀏覽器情境雙重驗證

### 單句刪除

- Implementation: 刪除鈕經事件委派進入 del, 確認後以 removeAt 依索引移除
- Verdict: MATCH
- Notes: removeAt 為純函式, 不改動其餘元素順序

### 清空全部

- Implementation: 清空鈕的 click handler, 確認後把句子與選取都設為空; render 依清單長度設定按鈕的停用狀態
- Verdict: MATCH
- Notes: 空清單時按鈕為 disabled, 瀏覽器不會對 disabled 的按鈕觸發 click, 因此「按下不跳 dialog」自然成立

### 刪除前二次確認

- Implementation: 單句刪除與清空全部都在動資料之前呼叫原生 confirm, 回傳 false 就直接 return
- Verdict: MATCH
- Notes: 取消路徑不做任何變更, 包含不中止正在進行的朗讀

### 清單與選取的本機保存

- Implementation: save 同時寫入句子與選取兩個 key, 由 commit 與勾選 handler 呼叫; 啟動時以 parseList 與 parseSelection 還原
- Verdict: MATCH
- Notes: 四個 scenario 都覆蓋, 含損毀字串與超出清單長度兩道防護

### 單句播放

- Implementation: 播放鈕經事件委派取得索引, 先中止目前朗讀再朗讀該句; 建立朗讀時讀取當下的語速與音色選擇
- Verdict: MATCH
- Notes: 不參考選取狀態, 因此有勾選時單句播放仍只唸被按的那一句

### 依序播放的播放範圍

- Implementation: 純邏輯區塊的 playQueue, 有勾選時只取勾選的索引, 沒有勾選時取全部, 兩種情況都依清單順序
- Verdict: MATCH
- Notes: 「依清單順序而非勾選順序」由 assert 直接覆蓋 (勾選順序 [2,0] 得到佇列 [0,2])

### 依序播放的逐句朗讀

- Implementation: 按下時算出佇列並固定在閉包裡, 以朗讀結束事件串接下一句
- Verdict: MATCH
- Notes: 佇列在按下當下快照, 因此播放中變更勾選不影響進行中的序列

### 依序播放鈕標示播放範圍

- Implementation: syncPlayAll 依播放中與否, 以及選取數量決定按鈕文字, 由 render 與勾選 handler 呼叫
- Verdict: SPEC-UPDATED (updated to match code)
- Notes: 原本的 Requirement 只描述了兩個狀態 (有勾選 / 沒勾選), 但實作有第三個: 播放進行中按鈕改為標示停止. 這不是缺陷 - 該鈕在播放中兼任停止鈕, 是「中止播放」那條 Requirement 的再按一次即停止直接推出來的結果, 只是原本的措辭沒涵蓋. 已把 Requirement 改寫成區分未播放與播放中兩種情形, 並補上「播放進行中的按鈕標示」scenario. 由於本 change 的 delta 是新建 capability (tospec/specs/ 尚無對應的主 spec), 修正直接寫在 ADDED 區塊內而非新增 MODIFIED 區塊 - 對一個尚未存在的 Requirement 下 MODIFIED 會在合併時指向不存在的目標. 更新後 tospec validate 通過 (1 passed, 0 failed)

### 中止播放

- Implementation: stop 先遞增播放序號讓還在路上的結束事件失效, 再取消目前朗讀; 依序播放鈕在播放中再按一次, 單句播放, 刪除與清空都會經過它
- Verdict: MATCH
- Notes: 這是本 change 最容易失效的路徑 - 取消朗讀時瀏覽器仍會對被中斷的那句觸發結束事件, 序號比對是讓中止確定的關鍵. 續播判斷已由 assert 覆蓋, 四種中止情境也都有瀏覽器驗收

### 沿用語速與音色設定

- Implementation: 建立朗讀時即時讀取兩個選擇器的當下值; 兩者的本機記憶沿用本次改動前既有的程式碼, 未變動
- Verdict: MATCH
- Notes: 單句與依序播放走同一個建立朗讀的函式, 因此兩者行為一致

## Unrequested Behavior

以下是實作裡沒有任何 Requirement 要求的行為. 依規則只在此回報, 不寫進 spec, 也不阻擋封存 - 要保留, 拿掉還是正式規劃由使用者決定.

- **句子文字做 HTML escape**: 句子以 innerHTML 產生, 因此對 `&` `<` `>` 做了跳脫. 使用者可見的差別是含有角括號的句子會照字面顯示; 沒有跳脫的話那段內容會被當成標記解讀
- **勾選時不重繪整份清單**: 其餘所有變動都走「整份重繪」, 但勾選只更新狀態與按鈕標籤. 原因是重繪會重建 checkbox, 用鍵盤操作的焦點會跟著消失. 這是刻意的無障礙取捨, 但它是 design.md D3 的例外, 目前只以程式碼註解記錄, 沒有對應的決策條目
- **送出後把焦點移回輸入框**: 讓連續新增句子不必再點一次輸入框
- **勾選控制帶 aria-label**: 勾選方塊本身沒有可見文字, 補了無障礙標籤
- **刪除確認的訊息帶出該句內容**: Requirement 只要求跳出確認, 訊息內容是實作細節
