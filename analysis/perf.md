# FHIR 驗證器效能瓶頸分析（聚焦 Bundle 寫入／驗證）

## TL;DR
目前程式架構的慢點不是單一函式，而是「**Bundle 驗證路徑上的高複雜度掃描 + 大量物件建立 + 規則/術語檢查串接**」疊加出來。

若觀察大型 Bundle（例如數千 entry），效能惡化通常是：

1. 先在 `InstanceValidator` 進入 bundle special validation。
2. `BundleValidator` 針對 entry、reference、interlink 做大量走訪。
3. 每個 entry 仍可能觸發額外 profile 規則驗證與 `checkSpecials`。
4. 在輸出（寫入）階段若用 canonical/pretty 或涉及簽章 canonicalization，會再做一次完整序列化成本。

---

## 1) 主要瓶頸在哪裡

### A. Bundle interlink/reference 驗證的整體複雜度偏高
`BundleValidator.checkAllInterlinked(...)` 的流程是：

- 先收集每個 entry 的 resource。
- 對每個 entry 遞迴掃描 `findReferences(...)`。
- 對每個 reference 呼叫 `resolveInBundle(...)`。
- 解析出 target 後，再透過 `entryForTarget(...)` 反查 summary。

這條路徑很容易從 O(N) 變成接近 O(N×R) 甚至更差（N = entry 數、R = 每個資源內 reference 數），尤其在文件型 / message 型 Bundle 的關聯驗證上最明顯。

### B. `findReferences` 去重方式是線性查找，參考數大時會放大
`hasReference(ref, references)` 用 list 線性檢查重複。當單一資源內有大量 URL/reference 時，去重成本會接近 O(R²)。

### C. `entryForTarget` 反查 target 也是線性
`entryForTarget(entryList, tgt)` 逐筆比對 entry 指標，對每個 resolve 出來的 target 都掃一遍 list，會造成大量重複掃描。

### D. 每個 entry 都可能跑額外 rule/profile 驗證
在 `validateBundle(...)` 裡：

- 逐 entry 處理 fullUrl/id/type 規則。
- 套用 `validator().getBundleValidationRules()`，每次可能 `context.fetchResource(...)` profile。
- 然後又呼叫 `checkSpecials(...)`。

這使得 bundle entry 數量一大時，CPU 時間成長很快。

### E. Validator object churn（暫時性物件過多）
例如在 `InstanceValidator.checkSpecials(...)` 中，每種資源型別都可能 `new XxxValidator(...)`；Bundle 每個 entry 都建 `NodeStack`、list、set。GC 壓力在高併發或大 Bundle 下會明顯。

### F. 術語/外部解析延遲會被「放大感知」
雖然你觀察到「都卡在 validator」，但其中有一塊其實是 validator 內呼叫 terminology / profile lookup 的時間。如果 cache 命中率不高，延遲會被記到 validator 身上。

---

## 2) 為什麼「寫入 Bundle」體感很爛？

你提到「寫入 bundle 效能很爛」，在這套架構裡通常有三層原因：

### 2.1 寫入前已經被驗證流程拖慢（最常見）
業務端看到的是「我寫一個 bundle 很慢」，但實際主時間可能在 validate（尤其 interlink + profile + terminology）。

### 2.2 寫入（序列化）本身在大資源圖上有遞迴與多次掃描
`JsonParser.compose(...)` 會遞迴走 element tree，對 list/children 進行處理；在 canonical/pretty 模式下還有額外格式化與輸出成本。

### 2.3 簽章相關流程會強制 canonical compose
`BundleValidator.makeSignableBundle(...)` 會建立 parser、canonical filter，再把整個 bundle compose 成 bytes；若是 XML 還會做 canonicalization。這對大 Bundle 是昂貴步驟。

> 換句話說：你感受到的「寫入慢」，有很高機率是「寫入 + 驗證 +（可能）簽章 canonicalization」一起慢。

---

## 3) 重構方向（依優先級）

## P0：先做可量測化（不先量就很難改善）

1. 在 `BundleValidator.validateBundle`、`checkAllInterlinked`、`findReferences`、`resolveInBundle`、`InstanceValidator.startInner/checkSpecials` 加分段計時。
2. 每次驗證輸出：entry 數、reference 總數、resolve 命中率、terminology 呼叫次數/耗時、profile fetch 次數。
3. 用這些指標建立 95/99 percentile dashboard。

> 沒有這層，你無法知道是「reference 掃描慢」還是「術語服務慢」。

## P1：把 O(N×R) 路徑改成索引驅動

1. **Reference 去重改用 HashSet**
   - `findReferences` 的 `hasReference` list 線性檢查改成 set。
2. **建立 entry 反查 map**
   - 目前 `entryForTarget` 線性掃描，改成 `Map<Element, EntrySummary>`。
3. **Bundle reference index 一次建好，多處共用**
   - 現在 `resolveInBundle` 會把 map 存 userData，方向是對的；可進一步把「fullUrl->entry」「relative->entry」「fragment->entry」整合為不可變索引物件，供整個驗證週期共用。
4. **避免重複遞迴掃描相同資源**
   - 對 `findReferences(resource)` 結果做 memoization（key 可用 element identity + session id）。

## P2：降低每 entry 額外驗證成本

1. **BundleValidationRule profile 預先解析與快取**
   - 進入 bundle 驗證前就把 rule 對應的 `StructureDefinition` 取好；避免每 entry fetch。
2. **Validator 實例重用**
   - `checkSpecials` 中常見 validator 改為可重用欄位（thread-confined 或 pool），減少 object churn。
3. **NodeStack 輕量化**
   - 對熱路徑避免過深 push/pop 造成字串路徑組裝成本，可用 lazy path 或共享 prefix。

## P3：把寫入成本從請求主路徑拆出去

1. **區分 Validate 與 Persist pipeline**
   - 寫入流程先做「輕量同步檢查（schema/必填/基本 reference）」；深度語意驗證改 async 或延後。
2. **簽章 canonicalization 非必要不做同步**
   - 若業務允許，將簽章相關 canonical compose 改成背景任務。
3. **輸出風格預設 NORMAL**
   - 避免預設 PRETTY/CANONICAL；僅在 human-read 或簽章時啟用。

---

## 4) 建議的新架構藍圖

### 4.1 Validation 分層

- **Layer 1（Fast Gate）**：
  - JSON/XML 可解析
  - 必填欄位
  - 基本 type/cardinality
  - 基本 reference 形式檢查
  - 目標：< 50~100ms（中型 bundle）

- **Layer 2（Semantic Core）**：
  - profile 驗證
  - interlink/document/message 關聯
  - 可配置 strictness

- **Layer 3（Terminology/External）**：
  - code 驗證、外部服務查詢
  - 強 cache + timeout + circuit breaker

### 4.2 Bundle Index 作為一級公民
提供單一 `BundleValidationIndex`：

- `fullUrl -> entry`
- `type/id -> entries`
- `entry element -> summary`
- `resource -> extracted references`

所有 bundle 規則共享這個 index，避免各 validator 重複掃描。

### 4.3 可中斷與分批
對超大 bundle：

- 支援分批驗證（chunk）
- 支援 early-exit（達到 error threshold 就停止深層檢查）
- 提供 degraded mode（先可寫入、後補完整驗證結果）

---

## 5) 具體落地順序（4 週示意）

- **Week 1**：加 metrics + flame graph + 基準資料集（100/1k/10k entry）
- **Week 2**：`findReferences` set 去重、`entryForTarget` map 化、rule profile 預載
- **Week 3**：reference extraction memoization、validator 實例重用
- **Week 4**：拆分 validate/persist pipeline，將 heavy terminology/簽章改 async 選項

預期可先拿到 30%~60% 改善（視 bundle 結構與 terminology 命中率而定）；若目前大量耗時在 interlink + reference 掃描，改善幅度通常更高。

---

## 6) 風險與注意事項

1. FHIR 規則非常細，快取要注意「版本、profile、validation mode、session」維度，不可錯共用。
2. 異步化會影響一致性語義，需要清楚定義「寫入成功」時保證到哪個驗證層級。
3. canonical/簽章流程屬合規需求，不能為了效能直接移除，只能做「條件化」或「延後化」。

---

## 7) 結論

你目前的痛點判斷是合理的：效能主要卡在 validator，而且「寫入 bundle 很慢」多半是被 validator 的深度驗證與關聯解析拖住。

最有效的重構策略不是只微調單一函式，而是：

1. 先把熱點量測清楚；
2. 把 bundle 驗證改成索引驅動；
3. 降低每 entry 的重複工作；
4. 把重型語意/術語/簽章步驟從同步寫入主路徑拆開。

這樣才會同時改善吞吐、延遲與可預期性。
