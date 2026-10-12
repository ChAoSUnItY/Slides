---
marp: true
theme: default
paginate: true
math: katex
style: |
  section { font-family: "Noto Sans CJK TC", "Microsoft JhengHei", sans-serif; font-size: 26px; }
  h1 { color: #1f4e79; }
  h2 { color: #1f4e79; border-bottom: 3px solid #1f4e79; padding-bottom: 6px; }
  table { font-size: 22px; }
  blockquote { background: #fff6e0; border-left: 6px solid #f0a500; padding: 6px 16px; }
  .small { font-size: 20px; color: #555; }
  section.lead { text-align: center; }
---

<!-- _class: lead -->
<!-- _paginate: false -->

# QUBO 如何解「含時間窗的取送貨問題」?

QUBO Formulation of the Pickup and Delivery Problem with
Time Windows for Quantum Annealing
Curuliuc & Leon, *Applied Sciences* 2026, 16, 1690

<!-- 開場約 30 秒:本次報告介紹一篇把 PDPTW 完整編碼成 QUBO,並與 MILP、GA 以及 IBM 量子硬體比較的論文。 -->

---

## Table Of Contents

1. What's VRPPD & PDPTW?
2. How to build QUBO from PDPTW?
3. MILP vs GA vs QUBO
4. IBM Heron r3 + QAOA
5. Conclusion

<!-- 時間分配提醒。 -->

---

## What's VRPPD & PDPTW?

```
Vehicle Routing Problem with Pickup and Delivery (VRPPD)
└── Pickup and Delivery Problem with Time Windows (PDPTW)
```

- 每筆請求 = 一個**取貨點 + 一個送貨點**，必須**同一台車**服務
- **先取後送** (precedence)
- 車輛有**載重上限**；每個點有**時間窗** $[e_v, l_v]$
  - $e_v$: ready time
  - $l_v$: due time
- Goal：最小化總行駛成本
- 目的是方法論比較，而非貼近完整的真實物流。

<!-- 類比:外送員接單,先到餐廳取餐再送到客人,要在時間內完成,背包有容量。TSP 的進階版。 -->

---

## Why transform it into QUBO?

$$\min_{x \in \{0,1\}^n} \; x^\top Q x$$

- **量子退火機**直接吃 QUBO (Ising Hamiltonian)
- **QAOA** 也以同一個 Hamiltonian 為輸入
- 因此：**一個模型，多種求解器** (D-Wave、IBM、模擬退火)

Challenge：QUBO 是 **unconstrained**，但 PDPTW 充滿 **constraints**

<!-- 銜接背景知識,接下來三頁是論文核心。 -->

---

## Constraints in QUBO

| PDPTW 限制 | 轉換 |
|---|---|
| **等式**<br>每個點恰好被拜訪一次 | 平方懲罰 $\lambda(\sum_{i=1}^{n} a_i x_i - C)^2$ |
| **不等式**<br>載重 $\le$ 容量 | 加鬆弛變數 $s \ge 0$，轉成等式再平方 |
| **整數量**<br>時間、載重、鬆弛 | 二進位展開 $s=\sum_b 2^b s_b$ |

> 代價：**變數(qubit)數量增加**，矩陣變稠密。

---

## Variable Designs in Paper

| 組別 | 符號 | 意義 | 數量 |
|---|---|---|---|
| 路徑 | $x^k_{uv}$ | 車 $k$ 是否走弧 $(u,v)$ | $\|V\|(\|V\|-1)\|K\|$ |
| 時間 | $T^{k,b}_v$ | 服務開始時間的位元 | $\|V\| \cdot B_T$ |
| 載重 | $Q^{k,b}_v$ | 離開節點時的載重位元 | $\|V\|\|K\| B_Q$ |
| 期限鬆弛 | $S^{D,b}_i$ | 期限/時間不等式的鬆弛位元，每筆請求一組 | $\|R\| B_{SD}$ |
| 容量鬆弛 | $S^{C,k,b}_v$ | 容量不等式的鬆弛位元，每個節點、每輛車一組 | $\|V\|\sum_k B^k_{SC}$ |

懲罰項涵蓋：**拜訪一次、載重、時間一致、時間窗、先取後送、同車配對**

<!-- 路徑變數隨 |V|^2 |K| 成長,這是後面規模擴大時 qubit 數增加的主因。 -->

---

## Qubit Reduction: Scaling Parameter $\rho$

借用 Verma & Lewis 的概念：不等式兩邊同除以自訂正整數 $\rho$

$$\forall \rho : \mathbb{N}^+, \frac{Ax}{\rho} \le \frac{b}{\rho}$$

- 鬆弛變數從 $O(\log(b+1))$ 降到 $O(\log\frac{b+1}{\rho})$，節省 $O(\log(\rho))$ 個鬆弛變數
- $\rho = 10$ 讓總變數**減少 8 – 18%** (例：High Complexity 492 → 408)

**Lemma 1 (鬆弛變數減少下的解法保持性)**：設 $x^*$ 為基礎問題 $\min x^\top Qx$ (受限於 $Ax \le b$) 的最優解。若最優解的 $\frac{Ax^*}{\rho}$ 與 $\frac{b}{\rho}$ 為整數，最優解保持不變。

實驗驗證 (Medium Complexity)：
$$\rho = 10 \text{(滿足整除)} \Rightarrow \text{Cost} = 431.5$$
$$\rho = 13 \text{(不滿足整除)} \Rightarrow \text{Cost} = 448.20$$

---

## Penalty Weights & Automatic Weight Adjustment Algorithm

$$\min_{z \in \{0,1\}^\text{N}}f(z) = C_{route}(z) + \sum_{j\in C} \lambda_j P_j(z)$$

- $\lambda$ 太小 → 求解器偏好「便宜但違規」的解
- $\lambda$ 太大 → 壓過原本的成本項

### **自動調整權重演算法**

- $\theta$（設定參數）包含：$T_\text{max}$（最大迭代數）、$\alpha$（增加係數）、$\beta$（減少係數）

1. **解 QUBO**：檢查哪些限制被違反，沒有違反就直接回傳結果
2. 違反者 $\lambda \times \alpha$ ($\alpha \gt 1$，通常是 1.5 – 2.0)；滿足者 $\lambda \times \beta$ ($\beta \in (0,1)$，通常是 0.9 – 0.95)
3. 重複至可行，或達最大迭代數 $T_\text{max}$

在不可行案例 (Capacity-Infeasible) 中，容量權重 $w_3$ 持續暴增直至 (+10,920%)，雖然最後回傳失敗的結果，但仍顯示該算法有正確找出違反的條件。

<!-- 作者定位為概念設計與初步分析。 -->

---

## Control Groups: MILP & GA

**MILP (GLPK + Pyomo)**
- 使用 branch-and-bound，能得出**全域最佳解**，作為**最優解基準**
- 運行時間會因問題大小以指數方式成長
- 有最優性保證(若有成功解出)

**GA (基因演算法)**
- 啟發式取樣，可能得到**局部最佳解**
- 可以處理大型的問題
- 無法保證找到全域最佳解
- GA 以 Optuna 調出 Initial / A / B / C 四組參數；Set C 表現最佳，High Complexity 只有它找到可行解

---

## Datasets

| 情境 | 設計目的 | 可行性？ | 客戶點 | 請求數 | 車輛數 | QUBO 變數<br>($\rho=10$) |
|---|---|---|---|---|---|---|
| Easily Verifiable | 正確性檢查 | 是 | 4 | 2 | 1 | 90 |
| Capacity-Infeasible | 違規檢查 (容量不足) | 否 | 4 | 2 | 1 | 45 |
| Medium Complexity | 中等難度 | 是 | 10 | 5 | 2 | 182 |
| Realistic Heterogeneous | 真實模擬 (車隊不足) | 否 | 12 | 6 | 2 | 264 |
| High Complexity | 壓力測試 | 否 | 20 | 10 | 3 | 408 |

---

## Experimental Study

| 情境 | MILP | GA (Set C) | QUBO + SA |
|---|---|---|---|
| Easily Verifiable | 279.80 | 100% 可行，最優 | 可行，最優 |
| Capacity-Infeasible | 不可行 | 不可行 | 不可行 |
| Medium Complexity | 412.70 | 100% 可行，</br>1.5% 誤差（$418.92\pm8.4$） | 可行，4.6% 誤差 ($431.50$) |
| Realistic Heterogeneous | 不可行 | 40% 可行（有違規） | 不可行 |
| High Complexity | 逾時 | *80% 可行*$^\text{[註1]}$ | 不可行 |

- MILP：小、中型最優，大型逾時
- GA：唯一解出最複雜情境的方法，但可能有違規的問題
- QUBO + SA：2 / 5 情境找到可行解

*註1：High Complexity 於論文附錄標為不可行，但 GA 回報 80% 可行；MILP 逾時，無法確認。*

---

## Effect of $\rho$

27 個合成實例 (n5 / n8 / n10 / n15 / n20 / n25，叢集與隨機分布) 各跑 10 次，共 270 次執行。

| 指標 | $\rho = 1$ | $\rho = 10$ | 改善 |
|---|---|---|---|
| 整體可行率 | 56.3% | 70.4% | **+14.1pp** |
| 平均最優解差距 | 9.8% | 6.2% | −3.6pp |
| 找到最優解比例 | 12.2% | 21.5% | +9.3pp |
| 平均執行時間(秒) | 8.42 | 7.58 | -10pp |

- Wilcoxon signed-rank test：可行率 $p<0.001$、差距 $p<0.01$
- 變數少 → 搜尋空間小 → SA 更容易找到可行區域。

---

## Scaling Results (With SA)

| 類別 | QUBO 變數 ($\rho=10$) | 平均執行時間(秒) | 可行率 |
|---|---|---|---|
| n5 | 85 – 95 | 0.39 | ~100% |
| n10 | 180 – 200 | 4.0 | 77 – 87% |
| n15 | 325 – 355 | 10.8 | 43% |
| n20 | 455 – 495 | 24.2 | 17% |
| n25 | 625 – 680 | 45.6 | **3%** |

### **作者解釋**
狀態空間隨變數數指數成長 (High Complexity 約 $2^{408}$)，限制式耦合緊密，SA 難以找到窄小的可行區域

---

## IBM Heron r3 + QAOA

- 156 physical qubits；實務上約 130 – 140 個以內可用
- 流程：QUBO → Ising → QAOA 電路 → transpile → COBYLA 優化 → 4096 shots 取樣

| 情境 | Qubits | 結果 |
|---|---|---|
| Easily Verifiable | 90 | 可行，最優 |
| Capacity-Infeasible | 45 | 不可行 |
| n5 instance | 88–92 | 可行率 96.7 – 100%，與 MILP 最優解差距 2.8 – 4.2%<br>(古典 SA 為 1.4 – 2.1%) |

其餘 3 個主要情境 (182 – 408 變數) 超出硬體上限。單次耗時約 185s (古典 SA 約 0.4 s)。

---

## Conclusion

**論文貢獻**
- 完整、明確的 PDPTW → QUBO 建模範例
- 利用 $\rho$ 縮放降低 qubit 數並提升 SA 表現
- 同一模型同時跑 MILP / GA / QUBO / IBM 實機

**比較結論**
- **MILP**：中、小型問題最優，大型問題時常逾時
- **GA**：擴展性最好，但對參數敏感，偶有大型問題違規
- **QUBO + SA**：對懲罰權重極度敏感，規模一大難以找到可行解
- **QUBO + QAOA**：目前為概念驗證，受限於 qubit 數與雜訊

**未來方向**：GA 自適應控制；QUBO 叢集分解的混合量子-古典流程；將自動權重調整整合進完整流程。

---

<!-- _class: lead -->
<!-- _paginate: false -->

# Q & A