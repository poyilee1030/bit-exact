# Phase 0 — 環境與專案骨架

> Parent roadmap: `ROADMAP.md`

## 目的

在 Ubuntu 24.04（WSL2 或實體機）上把工具備齊、版本查清楚，並建好一個「一個指令就建置、測試、在 CI 上跑同一套」的 C++ 專案骨架。之後每個 Step 只往裡面加東西。

```text
apt 套件 ──┐
Spike（自建）├──▶ check_host_env.sh ──▶ 全過才開工
riscv-tests ┘

CMake + Ninja ──▶ 模擬器、單元測試 ──▶ ctest
       └──────── 同一支 scripts/regress.sh ◀── GitHub Actions（ubuntu-24.04）
```

## 完成標準

- 新機器照 `docs/setup.md` 做完，`scripts/check_host_env.sh` 全過
- `scripts/regress.sh` 在本機與 CI 跑出同樣的結果

---

<!-- STEP_PLAN_START -->
## Step 規劃

### Step 1 — 環境盤點與版本查證

**目標**

把要用的工具裝好、查清楚版本與限制，寫成一支會說實話的環境檢查腳本。

**規模目標**

```text
預估新增 code：0 lines（只有腳本與文件；腳本不計入行數）
```

**實作範圍**

- `scripts/check_host_env.sh` 與它的測試
- `docs/setup.md`、`docs/planning-review.md`、`docs/baseline.md`

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要查證、並寫進 `docs/planning-review.md` 的（附來源與查證日期）：

- Ubuntu 版本（WSL2 的 `lsb_release`）；GitHub Actions `ubuntu-24.04` runner 的映像說明
- GCC、clang、CMake、Ninja、GoogleTest 的版本；GoogleTest 用 apt 還是 `FetchContent` 釘版本（擇一，寫理由）
- `gcc-riscv64-unknown-elf`：
  - 能不能用 `-march=rv32im -mabi=ilp32` 編出 RV32 的 ELF（實際編一個 `.S` 檔，`readelf -h` 看 `Class: ELF32`、`Machine: RISC-V`）
  - 有沒有 RV32 的 C 函式庫（影響 Step 9）
- `riscv-tests`：釘哪個 commit、授權、用 submodule 還是腳本抓
- Spike：從原始碼建置的步驟與相依（`device-tree-compiler` 等）、釘哪個 commit、`--log-commits` 的輸出格式
- QEMU：`qemu-system-riscv32` 的版本
- Python 3、NumPy 的版本與安裝方式

`check_host_env.sh` 的行為（照 `bare-pixels` Step 2 的做法）：

- 每一項印 `PASS`／`FAIL`／`WARN` 與實際版本；缺什麼就明講缺什麼、怎麼裝
- 結束碼 0 = 無 FAIL、1 = 有 FAIL、2 = 用法錯誤
- 不修改系統、不自動安裝

先寫測試：

```text
tests/test_check_host_env.sh（用假工具放在 PATH 前面，不依賴機器現況）:
  - 全部工具都在、版本夠新 → 結束碼 0
  - 少了 riscv64-unknown-elf-gcc → 那一項 FAIL、結束碼 1，訊息寫出怎麼裝
  - 交叉編譯器在，但編不出 ELF32 → FAIL（不是只看指令存在）
  - Spike 不在 → FAIL，訊息指向 docs/setup.md 的建置步驟
```

### 預期檔案

```text
scripts/check_host_env.sh
tests/test_check_host_env.sh
docs/setup.md                # 新機器從零到 check_host_env.sh 全過
docs/planning-review.md      # 版本查證來源、決策、後備方案
docs/baseline.md             # 從這一步開始 append-only
```

### 驗證

- `tests/test_check_host_env.sh` 全過，且每一個情境都紅過一次
- 本機 `check_host_env.sh` 全過；結果（每一項的版本）記進 `docs/baseline.md`

### 這一步不做

- 不寫任何模擬器程式碼
- 不建 CI（Step 2）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
環境檢查腳本的測試全過（每項紅過）
  ↓
本機全過，版本記進 baseline 與 planning-review
```

### Step 2 — 專案骨架、建置、CI

**目標**

一個能建置、能跑測試、CI 跑同一支回歸腳本的空專案。

**規模目標**

```text
預估新增 code：約 80 lines（CMake 與一個最小的 main）
```

**實作範圍**

- `CMakeLists.txt`（C++20、`-Wall -Wextra -Werror`、Debug 開 sanitizer）
- `src/` 的最小程式、`test/` 的 GoogleTest 煙霧測試
- `scripts/regress.sh`：建置、`ctest`，之後的 Step 往裡面加項目
- `.github/workflows/ci.yml`：`ubuntu-24.04`，跑 `check_host_env.sh` 與 `regress.sh`
- `README.md`、`.clang-format`

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

目錄慣例：

```text
src/        模擬器與 NPU 模型（C++）
include/    公開的標頭
test/       GoogleTest 單元測試（不計入行數）
fw/         之後的 RV32 韌體與啟動碼（Phase 2、5）
scripts/    環境、回歸、差分比對、向量產生（Python／shell）
tests/      scripts/ 自己的測試
third_party/ riscv-tests、Spike（submodule 或腳本抓，釘 commit）
docs/       baseline、planning-review、setup、npu-spec、教學章
```

要實際弄懂：

- Debug 建置開 AddressSanitizer 與 UndefinedBehaviorSanitizer：模擬器大量用移位與有號／無號轉換，**C++ 的未定義行為會讓模型「在這台電腦上剛好對」**。之後每個 Step 的單元測試都在 sanitizer 下跑
- `-Werror` 讓 CI 對警告失敗；兩個編譯器（GCC、clang）都建一次

先寫測試：

```text
test/test_smoke.cpp:
  - 一個會失敗的斷言（先紅），再改成通過
tests/test_regress.sh:
  - 建置失敗時 regress.sh 結束碼 1，不是 0
  - 有一個單元測試失敗時結束碼 1
```

### 預期檔案

```text
CMakeLists.txt
src/main.cpp
test/CMakeLists.txt  test/test_smoke.cpp
scripts/regress.sh   tests/test_regress.sh
.github/workflows/ci.yml
README.md  .clang-format  .gitignore
```

### 驗證

- 本機 `scripts/regress.sh` 結束碼 0；故意弄壞一個測試時結束碼 1
- push 之後 CI 綠燈；CI 的輸出和本機一致（記進 baseline）
- GCC 與 clang 都建置成功、無警告

### 這一步不做

- 不寫任何指令的實作（Phase 1）
- 不做程式碼覆蓋率（之後需要時再加）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
本機建置、測試、回歸腳本全過；弄壞時會紅
  ↓
CI 綠燈，跑的是同一支腳本
```
