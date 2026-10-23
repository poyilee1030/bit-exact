# bit-exact Roadmap：RISC-V 指令集模擬器與 NPU C-Model

## 目的

用 C++ 寫兩個「和硬體逐位元相同」的軟體模型，最後接成一個小 SoC 的模型：

1. **RV32IM 指令集模擬器**（ISS）：以 RISC-V 官方的 `riscv-tests` 為驗收，並和官方參考模擬器 Spike 逐指令比對。
2. **NPU 的 C-Model**：先做逐位元正確的 int8 卷積與量化（功能模型），再做一個 MAC 陣列的週期近似模型（架構模型），兩者結果逐位元相同。
3. **系統整合**：RISC-V 核心經記憶體映射暫存器控制 NPU，一支用 C 寫、交叉編譯成 RV32 的韌體在模擬器上驅動 NPU 跑完一個小 CNN。

目標不是做出最快或最完整的模擬器，而是練出 C-Model 工作的方法。最終希望具備：

- 讀規格寫模型：指令編碼、例外與 CSR、量化公式，都從規格長出測試
- **逐位元正確**的判準與工具：官方測試套件、參考模型差分比對、第一個分歧點的定位
- C++ 的類別設計：匯流排、記憶體、裝置、核心、NPU 各自可替換、可單獨測試
- 功能模型與架構（週期）模型的分工，以及兩者互相驗證
- 一鍵回歸：單元測試、指令測試、差分比對、NPU 向量、端到端，全部由腳本跑完並指出出錯的位置
- 在模擬器上寫韌體：MMIO、輪詢、中斷

這個 repo 的工作方式沿用 `bare-pixels`：一個 Step 一個 PR、測試先紅、驗收寫成腳本、數字與踩坑記在 `docs/baseline.md`。

---

## 文件結構

詳細實作拆成每個 Phase 一個檔案。`ROADMAP.md` 只保留總覽、共通規則與索引。

| Phase | 主題 | Steps | 文件 |
|---:|---|---|---|
| 00 | 環境與專案骨架 | 1, 2 | [phase00.md](phase00.md) |
| 01 | RV32I 指令集模擬器 | 3, 4, 5, 6 | [phase01.md](phase01.md) |
| 02 | 差分驗證、RV32M 與效能 | 7, 8, 9, 10 | [phase02.md](phase02.md) |
| 03 | NPU 功能模型（int8，逐位元） | 11, 12, 13 | [phase03.md](phase03.md) |
| 04 | NPU 架構模型（MAC 陣列，週期近似） | 14, 15, 16 | [phase04.md](phase04.md) |
| 05 | 系統整合：RISC-V 驅動 NPU | 17, 18, 19 | [phase05.md](phase05.md) |

Step 編號跨 Phase 連續。

另有兩份規劃文件（Step 1 建立）：
- `docs/planning-review.md` — 版本查證來源、決策記錄與後備方案
- `docs/baseline.md` — append-only 的驗收數字與踩坑紀錄

## 整體架構

```text
                 ┌──────────────────────── 模擬器（一個 x86 上的 C++ 程式）────────────────────────┐
                 │                                                                                │
 RV32 ELF ──載入──▶  RV32IM 核心 ──▶ 匯流排（位址解碼）──┬──▶ RAM                                  │
（韌體、測試）     │   （解碼、執行、CSR、trap）          ├──▶ tohost／console（結束碼、輸出）       │
                 │                                    └──▶ NPU 暫存器 ──▶ NPU 模型                 │
                 │                                                        ├─ 功能模型（Phase 3）   │
                 │                                                        └─ 架構模型（Phase 4）   │
                 └────────────────────────────────────────────────────────────────────────────────┘
   標準答案：riscv-tests、Spike 的逐指令紀錄（核心）；NumPy 的量化與卷積（NPU）；x86 上跑同一份 C（韌體）
```

## 共通 Step 規則

每個 Step 必須是一個可以獨立 implement、build/run、verify、review 的單位。

```text
單一 Step 預估新增 code <= 800 lines
```

**計算方式**
- 計入：自己寫的 C++／C／組語（含 RV32 的韌體與啟動碼）、CMake 檔，**不含空白行與註解行**。
- 不計入：測試程式、驗收與分析的 Python 腳本、第三方程式碼（GoogleTest、riscv-tests、Spike）、自動產生的檔案。
- 範圍是「這個 Step 新增或修改的行數」，不是檔案總行數。
- 純資料（測試向量、CNN 權重）獨立成檔，另外標示大小，不與程式碼合併計算。

若合理預期超過 800 行，必須在實作前拆成 `Step N-a`、`Step N-b`…，不要先做完大型改動再事後切 commit。
預估超過 600 行時就先想怎麼切；切不開才允許超過，並在 Step 內寫明理由、標示為「例外」。
每個 Step 完成後，把實測行數填回，與預估差超過 50% 就回頭檢討切法。

每個 Step 應清楚描述：

1. 誰和誰互動（核心、匯流排、裝置、NPU、韌體、host 腳本）。
2. 誰是 master／initiator，誰是 slave／target。
3. 資料與控制如何流動（附圖）。
4. 要新增哪些類別、模組與工具。
5. 依據哪一份規格的哪一節（RISC-V 非特權／特權規格、量化規格、本 repo 的 NPU 規格）。
6. 實際驗收方法，以及**標準答案從哪裡來**。
7. 這一步明確不做什麼。

## 測試規則

先寫測試再實作。這個專案的測試分四層：

| 層 | 做法 | 標準答案 |
|---|---|---|
| 單元測試 | GoogleTest（`ctest`），每個類別單獨測 | 規格的表與例子（指令編碼、立即數、量化公式） |
| 一致性測試 | RISC-V 官方 `riscv-tests`（rv32ui、rv32mi、rv32um） | 測試自己判定，寫 `tohost` 回報 |
| 差分比對 | 同一個程式在本模擬器與參考模型上跑，逐步比對 | Spike 的逐指令紀錄；NumPy 的逐層輸出；x86 上跑同一份 C 的輸出 |
| 端到端 | RV32 韌體在模擬器上驅動 NPU 跑完一個 CNN | Phase 3 的功能模型與 NumPy |

**模型對模型的比對要兩邊獨立寫。** NumPy 的標準答案不能從 C++ 的實作翻譯過去，要從規格自己寫；否則兩邊會抄到同一個錯。
**比對失敗時要指出第一個分歧點**（第幾條指令、哪個暫存器；第幾層、哪個座標），不是只印「不一致」。

## 工作流程規則（每個 Step 都適用）

後續的 Step 執行者只讀這份 ROADMAP（配 `step-execution` skill），所以執行期規則寫在這裡。

1. **版本與相容性宣稱要有當日查證來源。** 寫進規劃或文件的版本號，要標「(YYYY-MM 時點)」並附來源；訓練資料裡的版本知識一律當過期處理。查證結果集中記在 `docs/planning-review.md`，每個 Step 開工當天重新查，並更新該文件。
2. **測試先於實作，而且要真的紅過一次。** 沒看過它紅的測試不算數。事後補的測試會照著實作長，抄到同一個漏洞；跨模組的契約要從**對方的規格或原始碼**去長測試。
3. **修正前行為沒有量化基準時，先補量基線再動手。** 驗收數字（通過的測試數、指令每秒、週期數、誤差）與對照方法（含容差）追加到 `docs/baseline.md`，**不回頭改舊數字**；踩坑寫成「原本以為什麼」。
4. **驗收要量「下游」。** 模型的下游是用它的人與對照它的人：指令的正確性由 `riscv-tests` 與 Spike 判定，不是由自己寫的單元測試判定；NPU 的正確性由 NumPy 的標準答案與 CNN 的推論結果判定；系統的正確性由韌體在模擬器上跑出的結果判定。下游還不存在時，退而驗自己，但必須有後面的 Step 在真下游收口。
5. **腳本入庫、冪等、fail loud、自我終結。**
   - 驗收、回歸與環境腳本放 `scripts/`，不寫拋棄式；踩到的坑若屬腳本缺陷，修進腳本本體。
   - shell 用 `set -euo pipefail`；禁止 `A && B || echo 跳過` 這類把真失敗吞成跳過的寫法。「已完成」要用明確條件判斷後才准跳過。
   - 結束碼：0 = 全過、1 = 有項目沒過、2 = 無法量測（工具不在、檔案不在）。量不出來就是 2，不留 `--skip`。
   - 會開背景程序的腳本（Spike、QEMU、長時間的回歸）要有 timeout 與 `trap` 清理，退出前確認沒有殘留程序才准結束碼 0。
6. **範圍紀律。** 實作中發現的本 Step 範圍外缺陷，記成清單、排進後續 Step，不順手修。基線 Step 與行為修改 Step 分開，出問題才能二分。
7. **合併前 review 的發現項要分流。** PR 開出後由使用者在乾淨的 session 跑 `/code-review`；範圍內的當場修並重驗，行為級的排進後續 Step。修完後基線紀錄、坑單、程式註解與實測結果不留矛盾。
8. **文件隨 PR 同步。** ROADMAP 打勾、`baseline.md` 追加、README 照實寫（哪段驗證過、哪段還沒）、教學章（`docs/stepNN.html`，做法同 `bare-pixels`）一併交付。
9. **可重建性。** 標準：「這台電腦今晚壞掉，新機器照 repo 能重建開發環境」。第三方程式碼釘死版本（`riscv-tests` 與 Spike 用 git submodule 或腳本抓固定的 commit；apt 套件記下版本）；環境步驟寫在 `docs/setup.md`，由 `scripts/check_host_env.sh` 驗證。
10. **CI 與本機同一個環境。** GitHub Actions 用和本機相同的 Ubuntu 版本；CI 跑的回歸就是 `scripts/` 裡的同一支腳本，不另寫一套。

## 版本查證（待 Step 1 查證）

以下是預計使用的工具，**版本號一律在 Step 1 當天查證後填入** `docs/planning-review.md`，這裡不寫憑記憶的版本。

| 項目 | 預計 | 要查的事 |
|---|---|---|
| 作業系統 | Ubuntu 24.04 LTS（本機用 WSL2 或實體機；CI 用 `ubuntu-24.04` runner） | WSL2 與 runner 的實際版本 |
| C++ 編譯器 | apt 的 GCC（C++20） | 版本、C++20 支援度；clang 當第二個編譯器 |
| 建置 | CMake + Ninja | 版本 |
| 單元測試 | GoogleTest（apt 的 `libgtest-dev` 或 CMake `FetchContent` 釘版本） | 兩種取得方式擇一，記下理由 |
| RISC-V 交叉編譯器 | apt 的 `gcc-riscv64-unknown-elf`（用 `-march=rv32im -mabi=ilp32`） | 能否編出 RV32 的裸機程式、有沒有 RV32 的 C 函式庫 |
| 一致性測試 | `riscv-tests`（GitHub） | 釘一個 commit；授權 |
| 參考模擬器 | Spike（`riscv-isa-sim`，從原始碼建置） | 釘一個 commit；`--log-commits` 的輸出格式 |
| 第二參考 | QEMU（`qemu-system-riscv32`） | 版本；裸機程式怎麼跑 |
| Python | Ubuntu 附的 Python 3 + NumPy | 版本；NumPy 用 apt 或 venv |
| CNN 權重 | PyTorch（只在電腦端訓練與量化一次，結果存成資料檔） | 是否值得裝；替代：純 NumPy 訓練一個很小的網路 |

## 規格來源

| 規格 | 用在 | 備註 |
|---|---|---|
| RISC-V 非特權規格（Unprivileged ISA） | RV32I、RV32M、Zicsr | 公開，記下版本日期 |
| RISC-V 特權規格（Privileged ISA） | M-mode CSR、trap、`mret` | 只做 M-mode |
| `riscv-tests` 的環境（`env/p`） | `tohost` 協定、測試怎麼回報 | 讀它的原始碼當規格 |
| int8 量化規格 | Phase 3 的量化與再量化 | 選一份公開的整數量化規格（例如 TensorFlow Lite 的 int8 量化規格）當「硬體規格」，Step 11 決定並寫下 |
| `docs/npu-spec.md`（本 repo 自己寫） | Phase 4、5 的 NPU 架構、暫存器表 | Step 14 寫成，之後的模型與韌體都照它 |

**不複製第三方實作的程式碼**（Spike、QEMU、TFLite 的 kernel）：可以讀、可以拿來對答案，模型要自己寫。

## 風險與限制

- **RV32 的 C 函式庫**：Ubuntu 的 RISC-V 裸機工具鏈可能沒有 RV32 的 newlib。`riscv-tests` 不需要 C 函式庫；Step 9 的 C 程式若需要，改用自寫的最小啟動碼與 `printf` 替代品，或另裝工具鏈（xPack）。Step 1 查證。
- **Spike 在 RV32 下的設定**：要用 `--isa=rv32im` 之類的參數，記憶體配置要和本模擬器一致才比得了。Step 7 查證。
- **週期模型不是真的硬體**：Phase 4 的 NPU 是本 repo 自己定義的玩具架構，週期數只和「自己的規格與解析公式」比，不代表任何真實晶片。
- **這台 Windows 電腦**：主要開發在 Ubuntu（WSL2）；`bare-pixels` 的工具（PlatformIO 等）在 Windows 那一側，兩個 repo 互不相依。

## 選修主題

| 主題 | 前置 | 備註 |
|---|---|---|
| 和 `bare-pixels` 對照：STM32 上用 CMSIS-NN 跑同一層卷積，與本 repo 的功能模型逐位元比對 | Phase 3；`bare-pixels` 要另加一個 Step | 兩個 repo 只交換資料檔（輸入、權重、預期輸出） |
| 在模擬器上跑 FreeRTOS（RISC-V port） | Phase 5；需要計時器中斷（CLINT 的 `mtime`／`mtimecmp`） | 對應職缺的 RTOS 加分項 |
| C 擴充（壓縮指令） | Phase 2 | 編譯器預設會產生，關掉要 `-march` 不含 `c` |
| 真晶片三方對照（ESP32-C3、WCH CH32V 等） | Phase 2 | 要買板子；模擬器、Spike、真晶片同一份程式結果相同 |
| 指令層級的週期模型（管線、分支預測） | Phase 2 | 和 Phase 4 同一種「功能 vs 週期」的分工 |

## 建議執行方式

一次只執行一個 Step。完成、驗證、review 並修正後，再進入下一個 Step。
一個 Step 對應一個 branch（`step-N`，拆分的用 `step-12a`）與一個 PR，PR 標題用 `step-N: <一句話目標>`。
驗收數字與踩到的坑記在 `docs/baseline.md`，下一個 Step 開工前先讀它。
