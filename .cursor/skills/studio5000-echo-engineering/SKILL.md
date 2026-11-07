---
name: studio5000-echo-engineering
description: >-
  Studio 5000 Logix Designer v35 program editing and FactoryTalk Logix Echo testing
  via the studio5000-mcp tools. Use when working with ACD/L5X projects, ladder or ST
  edits, Logix Designer GUI export/import/verify, Echo download, or online tag
  read/write/test against an Echo emulator.
---

# Studio 5000 + Logix Echo Engineering

## Safety

- Prefer timestamped working copies of ACD/L5X. Do not overwrite the user's original project unless asked.
- CIP writes (`echo_write`, `echo_run_test` writes) only target Echo emulators. Do not set `STUDIO5000_MCP_ALLOW_REAL_PLC` unless the user explicitly requests it.
- Call `echo_snapshot` on the target Echo controller before `echo_download` overwrites it.
- GUI automation leaves unexpected dialogs untouched and aborts. Never invent clicks through licensing, save-prompts, or error dialogs.
- Never go online or download to a real PLC through this skill unless the user explicitly directs it.

## Pre-flight: no-EDS devices (before any export/import)

Third-party / generic Ethernet modules **without a usable EDS on this PC** lose their IO tags on L5X export↔import. That breaks verify/download. Treat this as a **hard gate** — do not start `gui_export_l5x` / `gui_import_l5x` until the user chooses.

1. Inspect the project modules (ACD tree or L5X `Modules`) for known/likely no-EDS devices.  
   This machine’s known set: `weight_plc`, `Festo`, `ArrangleIO`, `PressureF`, `Host`, `SecondCount` (and any other non-Rockwell Ethernet node without local EDS).
2. **Stop and ask the user** (do not auto-strip):

   > 项目里有无 EDS 设备（列出名称）。L5X 导入/导出后 IO 标签会丢失。请选择：  
   > **A)** 为 Echo 临时删除这些模块及依赖程序/引用（可还原的工作副本）  
   > **B)** 你先在本机手动安装/导入对应 EDS，装好后告诉我再继续导入导出  
   > **C)** 取消本次导入导出

3. Only after the user answers:
   - **A** → work on a timestamped copy; remove modules + delete/NOP dependent programs/rungs; convert consumed tags (e.g. `ReceiveFrom170`) to Base if needed; then continue the toolchain.  
     JOYO/Damin refs that typically go with the set above: drop or NOP `EM13_SpaceFeeder01` / `EM30_IO_Status`; PF525 AOI types may also need cleanup.
   - **B** → wait; do not call export/import until the user confirms EDS are installed.
   - **C** → stop.

Never silently strip no-EDS content “to make Echo work.”

## Default toolchain (no SDK/COM licence)

Assume the license-free path unless `logix_backend_status` proves COM/SDK works. Run the **no-EDS pre-flight** first when the project has Ethernet third-party modules.

1. `gui_export_l5x` — ACD → L5X (Logix Designer v35 on a background virtual desktop).
2. L5X tools — `l5x_summary`, `l5x_get_routine`, `l5x_list_tags`, `l5x_validate_rung`, `l5x_edit_rung`, `l5x_add_tag`, `l5x_add_routine`, `l5x_set_st_routine`, …
3. `l5x_set_controller_type` — retarget full-project L5X to the Echo catalog (5580 emulator → `1756-L85E` major 35 unless verify/download says otherwise).
4. `gui_import_l5x` — L5X → new ACD.
5. `gui_verify` — Logic > Verify > Controller; fix errors before download.
6. `echo_snapshot` → `echo_download` → `echo_list_programs` / `echo_read` / `echo_run_test`.

If a needed L5X structure change has no tool (for example scheduling a new program on a continuous task), apply a minimal XML edit, then re-verify.

After `gui_import_l5x`, Logix Designer may show `Logix Designer - <ControllerName> [catalog]` without `in <file>.ACD`. The GUI client matches that form to the ACD basename stem so verify/download can proceed on the open window.

`gui_verify` reads the Errors RICHEDIT even when that pane is not visible; the status bar also shows `Verify complete with no errors or warnings.`

For Echo proof of a free-running wrap, write near the threshold (e.g. `Cnt=4995`) and `expect` a low band — do not try to watch the climb from 0 over CIP.

## Ladder notes

- Neutral text must end with `;`. Branches use `[A,B]`.
- Validate with `l5x_validate_rung` before `l5x_edit_rung` (unknown tags rejected unless `force=true`).
- Program-scope CIP names: `Program:Name.Tag`. Controller tags: bare name.
- Large L5X files may exceed editor patch limits — use a short Python `read → replace-in-EM-section → write` for targeted rung edits, then `gui_import_l5x` / `gui_verify`.

## Reference recipe: free-running counter

Controller tag `Cnt` (`DINT`). Program `CounterTest`, RLL `MainRoutine`:

```text
ADD(Cnt,1,Cnt);
GRT(Cnt,5000)MOV(0,Cnt);
```

Schedule `CounterTest` on the continuous task. After download to Echo:

1. `echo_write({"Cnt": 4990})` (every-scan ADD is too fast to watch from 0).
2. `echo_run_test` / `echo_read` until `Cnt` wraps through clear into a low band (e.g. `< 100`).

## PackML machine projects (JOYO / Damin-style)

Do **not** prove a new EM by forcing motion tags alone. Start PackML, get the master running, then judge the new logic.

### Required online order

1. Complete **no-EDS pre-flight** (user chooses strip vs install EDS) → retarget → `gui_import_l5x` → `gui_verify` (0 errors) → `echo_snapshot` → `echo_download`.
2. Echo-only: keep `EM00` (master) + the EM under test `Selected`; clear other `EMxx_Conditions.Selected` (many Selected modules block EnableDone / Resetting).
3. Mode auto: `HMI_I_MachineMode=1` → `Unit01_Auto` true; clear `HoldingEvent1/2`, `Stopevent`.
4. PackML: Clear → Stop → Reset until `Sts_Idle` (watch `Wrk_SEQ_Resetting` / Enable→Home→Prepare Done — leftover `SEQ=1000` can look like Idle without a real Resetting pass).
5. Start → wait `Sts_Starting` → `Sts_Execute` with master `CommandVelocity` / position wrapping.
6. Only then set operator gates (e.g. `G_SealAllowRun`) and sample the EM under test.

### GUI / import pitfalls (already fixed in MCP client; do not re-break)

- Import window title may be `Logix Designer - <ControllerName> [catalog]` without `in <file>.ACD` — match ACD stem.
- Errors pane can be invisible; read cloaked RICHEDIT + status bar “Verify complete…”.
- After import, wait ≥3s settle before re-verify so “Verifying Controller” is not treated as stuck.

### Platform / size

- Damin bring-up failures on Echo were **5069 ↔ 5580 platform** (retarget to `1756-L85E`), not “project too large”.
- No-EDS handling is gated in **Pre-flight** above — never invent a silent strip mid-import.

### New EM clone checklist (e.g. EM16_SealingDown from EM14)

- Clone module + axis + program; schedule on the same continuous task as siblings.
- PackML Done bits must not form cycles. Classic deadlock:  
  `StartExecutionDone` ← `Sts_Started` ← `Wrk_EM_Executing` ← `StartExecutionDone`.  
  For non-cam EMs use pass-through: `XIC(Inp_Commands.StartExecution)OTE(Out_EMConditions.StartExecutionDone);`
- Resetting: home-only if that is the design; `PrepareExecutionDone` may be immediate. `EnableDone` needs real axis enable (`Sts_EnableDone`), not CIP overwrite of Done bits.
- Motion trigger for MAM: rising edge must **`OTL`** the trig bit; clear on `PC`. `OTE` + `ONS` only enables MAM for one scan — unreliable for time/distance moves.
- Angle window: `LIM(master±Win)` on `Ref_MasterAxis.CommandPosition` with Auto + allow-run + HMI active gates.
- “MATC-like” time+distance: use `MAM(..., Seconds, ...)` when the project has no MATC opcode.
- CIP sampling: one-scan bits (`ONS`/`OTE` trig) are easy to miss — also watch `MI_MAM_*.IP` / `.PC` and axis `ActualPosition` / `CommandPosition`.

### Echo bring-up tags (typical)

- State: `UN01_MachineStateModel.Sts_*`, `Program:OP01_Production.Wrk_SEQ_Resetting|Starting`
- Commands/Done: `UN01.Commands.*`, `UN01.EMConditions_ALL.*`, `EMxx_Conditions.*`
- Master: `EM00_MachineMaster.ActualPosition`, `CommandVelocity`
- Gate: `Unit01_Auto`, `G_SealAllowRun` (often OTU’d outside Execute — rewrite while sampling if needed)

## Completion

- Report which MCP tools ran, verify/download outcomes, and live tag evidence.
- For PackML EMs: state evidence must include Idle → Execute + master motion **before** claiming the new EM logic is correct.
- Note any GUI dialogs, catalog/firmware mismatches, or missing MCP capabilities discovered during the run.
