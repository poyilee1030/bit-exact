# Phase 1 — RV32I 指令集模擬器

> Parent roadmap: `ROADMAP.md`
> 規格：RISC-V 非特權規格（RV32I、Zicsr）、特權規格（M-mode）；驗收：`riscv-tests` 的 rv32ui、rv32mi

## 目的

寫出一個能通過官方 RV32I 一致性測試的指令集模擬器。這是 C-Model 最典型的入門題：規格公開、範圍明確、有官方的標準答案。

```text
ELF 檔 ──▶ ELF 載入器 ──▶ RAM
                            ▲
                            │ 讀寫
  ┌─────────┐  取指令   ┌────┴────┐
  │  Hart   │─────────▶│  Bus    │──▶ RAM
  │ x0–x31  │◀─────────│（位址    │──▶ tohost（測試回報、結束）
  │  PC     │  讀寫資料 │  解碼）  │
  │  CSR    │          └─────────┘
  └─────────┘
  取指令 → 解碼（純函式）→ 執行 → 更新 PC；例外 → trap 到 mtvec
```

## 必須理解

- RV32I 的六種指令格式（R、I、S、B、U、J）與立即數的位元拼法：B、J 格式的立即數是打散的
- `x0` 永遠是 0；寫入 `x0` 要被丟掉
- 有號與無號：`SLT` 與 `SLTU`、`SRA` 與 `SRL`、`LB` 與 `LBU`；**C++ 對有號數的右移與溢位的規定**（C++20 起有號右移是算術右移，但溢位仍是未定義行為）
- M-mode 的 trap：`mepc`、`mcause`、`mtval`、`mtvec`、`mstatus`，`ecall` 與 `mret`
- `riscv-tests` 怎麼判定通過：測試把結果寫到 `tohost`

## 完成標準

- rv32ui-p 與 rv32mi-p 的測試全部通過（數量以實跑為準，記進 baseline）
- 每條指令有從規格長出來的單元測試

---

<!-- STEP_PLAN_START -->
## Step 規劃

### Step 3 — 指令解碼器

**目標**

把 32 位元的指令字解成「哪一條指令、哪些暫存器、立即數」，純函式、不碰狀態。

**規模目標**

```text
預估新增 code：約 250 lines
```

**實作範圍**

- `include/bitexact/decode.h`、`src/decode.cpp`
- 支援 RV32I 的 40 條指令與 `FENCE`、`ECALL`、`EBREAK`、Zicsr 的 6 條；不認得的編碼回傳「非法指令」

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

```cpp
struct Decoded {
    Op op;              // enum：ADD、ADDI、BEQ…、ILLEGAL
    uint8_t rd, rs1, rs2;
    int32_t imm;        // 已經符號延伸
    uint16_t csr;
};
Decoded decode(uint32_t insn);
```

要實際弄懂：

- 立即數的拼法要逐位元照規格的圖：B 格式是 `imm[12|10:5]` 與 `imm[4:1|11]`，J 格式是 `imm[20|10:1|11|19:12]`
- 「非法指令」的範圍：保留的 funct3／funct7 組合、全 0 的指令字（規格定義為非法）

先寫測試（**標準答案從規格與組譯器來，不從自己的實作來**）：

```text
test_decode:
  - 每條指令至少一個例子：用 riscv64-unknown-elf-as 組譯一份 .S，
    scripts/gen_decode_vectors.py 把 objdump 的結果轉成「指令字 → 期望欄位」的表
  - 立即數的邊界：最大、最小、負數、B／J 格式的每一個位元單獨為 1
  - x0 當目的暫存器時照樣解碼（丟掉寫入是執行階段的事）
  - 非法：全 0、全 1、保留的 funct7
```

### 預期檔案

```text
include/bitexact/decode.h  src/decode.cpp
test/test_decode.cpp
scripts/gen_decode_vectors.py  test/vectors/decode.txt   # 資料檔，另外標示大小
```

### 驗證

- 單元測試全過（在 sanitizer 下），每條指令的測試紅過
- 向量表由組譯器產生、入庫，產生腳本可重跑且結果相同

### 這一步不做

- 不執行指令（Step 4）
- 不做 M 擴充（Step 8）、不做壓縮指令（選修）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
組譯器產生的向量表全部解對
  ↓
立即數邊界與非法編碼都有測試、都紅過
```

### Step 4 — 執行核心、匯流排與記憶體

**目標**

能一條一條執行 RV32I 指令：暫存器、ALU、分支、跳躍、載入儲存，經一個可替換的匯流排存取記憶體。

**規模目標**

```text
預估新增 code：約 350 lines
```

**實作範圍**

- `Hart`（暫存器檔、PC、`step()`）
- `Bus` 介面（`read(addr, size)`、`write(addr, size, value)`）與 `Ram` 裝置、位址解碼
- 非對齊存取、超出範圍存取回報成例外（實際 trap 在 Step 5）

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

誰和誰互動：

```text
Hart（initiator）──read/write──▶ Bus（位址解碼）──▶ Device（target）：Ram、之後的 tohost、NPU
```

類別設計（C++ 的 OOP 重點在這裡）：

- `Device` 是抽象介面；`Bus` 持有「位址範圍 → Device」的表。之後加 tohost（Step 5）、console（Step 9）、NPU（Step 17）都只是新的 `Device`
- `Hart` 只認識 `Bus`，不知道後面接了什麼：單元測試可以接一個記錄所有存取的假匯流排
- 小端序；`LB`／`LH` 的符號延伸在 `Hart`，不在 `Bus`

要實際弄懂：

- `x0` 的寫入在哪裡丟掉（一處，不要散在每條指令裡）
- `JALR` 的目標位址要把最低位元清掉
- 有號運算在 C++ 裡怎麼寫才不踩未定義行為：先轉成 `uint32_t` 運算，再轉回 `int32_t`

先寫測試：

```text
test_hart（接一個假匯流排）:
  - 每條 ALU 指令的一般值與邊界值（0x80000000、0x7FFFFFFF、-1、移位量 31、移位量的高位被忽略）
  - SLT／SLTU、SRA／SRL 的有號無號差異
  - 分支往前、往後、不跳；JAL／JALR 的連結暫存器與最低位元
  - LB／LBU／LH／LHU 的符號延伸；SB／SH 只改到該寫的位元組
  - 寫 x0 之後讀回仍是 0
test_bus:
  - 位址解碼到正確的裝置；沒有裝置的位址回報存取錯誤
```

### 預期檔案

```text
include/bitexact/{hart,bus,device,ram}.h
src/{hart,bus,ram}.cpp
test/test_hart.cpp  test/test_bus.cpp
```

### 驗證

- 單元測試全過（sanitizer 下）、每類指令的測試紅過
- 手寫一段組語（加總 1 到 10）組譯後放進 RAM，執行到某個位址停下，`x10 = 55`

### 這一步不做

- 不做 CSR 與 trap（Step 5）
- 不載入 ELF（Step 6）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
每條 RV32I 指令的單元測試通過、紅過
  ↓
手寫組語在模擬器上算出正確結果
```

### Step 5 — CSR、M-mode trap 與 tohost

**目標**

實作 `riscv-tests` 需要的最小特權架構：M-mode 的 CSR、例外與 `mret`，以及 `tohost` 回報。

**規模目標**

```text
預估新增 code：約 250 lines
```

**實作範圍**

- Zicsr 的 6 條指令；CSR：`mstatus`、`misa`、`mhartid`、`mtvec`、`mepc`、`mcause`、`mtval`、`mscratch`、`mcycle`／`minstret`（及高半部）
- 例外：非法指令、非對齊、存取錯誤、`ecall`、`ebreak`；`mret`
- `ToHost` 裝置：寫入時判定通過或失敗、記下測試編號、停止模擬

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- `riscv-tests` 的 `env/p` 怎麼設定 trap、怎麼寫 `tohost`（讀它的原始碼當規格）：寫 1 是通過，其他值的高位是失敗的測試編號
- 不支援的 CSR 要回非法指令，不能默默讀成 0
- `mepc` 指向造成例外的那條指令；`ecall` 之後韌體自己加 4

先寫測試：

```text
test_csr:
  - CSRRW／CSRRS／CSRRC 與立即數版本：讀回舊值、寫入新值；rs1 = x0 時 CSRRS 不寫
  - 唯讀 CSR（mhartid）寫入 → 非法指令
  - 不存在的 CSR → 非法指令
test_trap:
  - 非法指令：mepc = 該指令位址、mcause = 2、mtval = 指令字、PC 跳到 mtvec
  - ecall：mcause = 11；mret 回到 mepc
  - 非對齊的載入：mcause = 4、mtval = 位址
test_tohost:
  - 寫 1 → 通過；寫 (n << 1) | 1 → 失敗於第 n 項
```

### 預期檔案

```text
include/bitexact/{csr,trap,tohost}.h
src/{csr,trap,tohost}.cpp
test/test_csr.cpp  test/test_trap.cpp  test/test_tohost.cpp
```

### 驗證

- 單元測試全過、紅過
- 手寫組語：故意執行非法指令，trap handler 讀 `mcause` 寫到 `tohost`，模擬器回報正確的值

### 這一步不做

- 不做 S-mode、U-mode、虛擬記憶體、中斷（中斷在 Step 18）
- 不做 PMP
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
CSR、trap、tohost 的單元測試通過、紅過
  ↓
手寫的 trap 範例在模擬器上回報正確的 mcause
```

### Step 6 — ELF 載入與官方一致性測試

**目標**

載入 ELF、跑完 `riscv-tests` 的 rv32ui-p 與 rv32mi-p，全部通過。

**規模目標**

```text
預估新增 code：約 200 lines
```

**實作範圍**

- 自寫的 ELF32 載入器（只讀 program header，載入 `PT_LOAD` 段；找 `tohost` 符號的位址）
- 命令列：`bitexact run <elf> [--max-insns N]`，結束碼 0 = 通過
- `scripts/run_riscv_tests.sh`：建出 `riscv-tests`、逐個跑、列出結果，加進 `regress.sh`

<!-- STEP_DETAIL_START -->
### 這一步到底要做什麼

要實際弄懂：

- `riscv-tests` 預設編 64 位元；RV32 要用它的設定編出 `rv32ui-p-*`（Step 1 查過的工具鏈參數）
- 每個測試要有指令數上限：跑進無窮迴圈時要結束、回報「逾時」，不是卡住整個回歸
- 失敗時要印出**哪一個測試、第幾項**（`tohost` 的編號），之後 Step 7 再定位到指令
- 非對齊存取：規格允許「硬體直接支援」或「trap」兩種。Step 4、5 先選了 trap；`riscv-tests` 裡和非對齊有關的測試（例如 `ma_data`）期待哪一種，開工時讀它的原始碼確認，和 Spike 的預設行為一致，選擇寫進 baseline

先寫測試：

```text
test_elf:
  - 用交叉編譯器產生的小 ELF：段落載到正確位址、入口點正確、tohost 符號找得到
  - 不是 ELF32／不是 RISC-V → 拒收並說明
tests/test_run_riscv_tests.sh:
  - 用一個故意失敗的測試（自己寫的 .S）確認腳本結束碼 1、指出測試名與編號
  - 用一個無窮迴圈的測試確認會逾時、結束碼 1
```

**先紅**：在全部實作好之前就把整套 `riscv-tests` 接進回歸，記下第一次跑的通過數（之前的 Step 若有漏洞，這裡會現形）。

### 預期檔案

```text
include/bitexact/elf.h  src/elf.cpp  src/main.cpp（命令列）
test/test_elf.cpp
scripts/run_riscv_tests.sh  tests/test_run_riscv_tests.sh
third_party/riscv-tests（submodule，釘 commit）
```

### 驗證

- rv32ui-p 與 rv32mi-p **全部通過**；通過數與測試清單記進 baseline
- CI 跑同一支腳本、同樣全過

### 這一步不做

- 不做 rv32ui-v（虛擬記憶體版）
- 不做 RV32M（Step 8）
<!-- STEP_DETAIL_END -->

**驗收條件**

```text
ELF 載入器的測試通過
  ↓
riscv-tests rv32ui-p、rv32mi-p 全部通過（本機與 CI）
```
