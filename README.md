# Chinese Dark Chess — Parallel MCTS Engine

暗棋（Chinese Dark Chess）對局引擎，以蒙地卡羅樹搜尋（MCTS）處理翻棋帶來的隨機性，
搜尋以 OpenMP 平行化。透過 MGTP 協定與對局伺服器溝通。

> 這是以 MCTS 探索暗棋的實驗版本。ICGA 電腦奧林匹亞與 TCGA 電腦對局競賽的參賽引擎
> 使用的是 NegaScout 搜尋，不是這一版。

---

## Why MCTS

暗棋與一般完全資訊棋類不同：棋子初始全部蓋著，翻開哪一顆是隨機的。
這讓傳統 Alpha-Beta 系列的搜尋難以直接套用——分支不只來自玩家選擇，還來自機率事件。

MCTS 的模擬式評估天然適合這種隨機賽局樹：不需要人工設計精確的評估函數，
用大量隨機對局的統計結果來近似局面價值。

---

## Implementation

### 搜尋

以 UCT（Upper Confidence bounds applied to Trees）平衡探索與利用：

$$\text{UCT}(n) = \frac{w_n}{v_n} + c\sqrt{\frac{\ln v_{parent}}{v_n}}$$

預設探索常數 $c = 1.41$（$\approx \sqrt{2}$）。

### 平行化

搜尋以 OpenMP 平行展開。共享的 `visits` 與 `wins` 以 `#pragma omp atomic read`
保護讀取、`#pragma omp critical` 保護葉節點判定，避免展開階段的競爭條件。

```
Selection ──► Expansion ──► Simulation ──► Backpropagation
     ▲                                            │
     └────────────────────────────────────────────┘
              （多執行緒同時走訪同一棵樹）
```

### 對局介面

實作 MGTP（Multi-Game Text Protocol）指令集，可直接接上競賽用的對局伺服器。
`MyAI` 維護盤面狀態、各棋種的暗子計數與雙方時間。

---

## Project Structure

```text
.
├── include/
│   ├── MCTS.h        # MCTSNode 樣板類別、UCT、四階段搜尋
│   ├── DarkChess.h   # 暗棋規則、走步產生、狀態表示
│   ├── libchess.h    # 棋種、盤面常數定義
│   └── MyAI.h        # 引擎主體與 MGTP 狀態
├── src/
│   ├── main.cpp      # MGTP 指令迴圈
│   ├── MyAI.cpp      # 盤面初始化、指令處理、決策進入點
│   └── DarkChess.cpp # 規則實作與模擬
└── Makefile
```

---

## Build & Run

需求：`g++`（C++14）與 OpenMP。

```bash
make
bin/cdc
```

編譯選項為 `-O3 -fopenmp`。執行後程式從標準輸入讀取 MGTP 指令。

---

## Notes

棋力主要取決於單位時間內的模擬次數，
因此平行化的效率與模擬階段的成本是兩個主要調校方向。
