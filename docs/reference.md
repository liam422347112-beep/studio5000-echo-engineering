# Reference — Studio 5000 + Echo engineering notes

Companion detail for `studio5000-echo-engineering`. Prefer `SKILL.md` as the agent source of truth; this page is for humans and deeper context.

## Toolchain map

| Step | Tool / action | Notes |
|------|----------------|-------|
| 0 | No-EDS pre-flight | Ask user A=strip / B=install EDS / C=cancel |
| 1 | `gui_export_l5x` | ACD → L5X on background virtual desktop |
| 2 | L5X edits | `l5x_*` tools or minimal XML patch |
| 3 | `l5x_set_controller_type` | Echo: `1756-L85E` major 35 |
| 4 | `gui_import_l5x` | L5X → timestamped ACD working copy |
| 5 | `gui_verify` | Must be 0 errors before download |
| 6 | `echo_snapshot` | Always before overwrite download |
| 7 | `echo_download` | Then CIP prove-out |

## No-EDS devices (JOYO / Damin known set)

Modules that historically lacked usable EDS on the engineering PC:

- `weight_plc`, `Festo`, `ArrangleIO`, `PressureF`, `Host`, `SecondCount`

Typical dependent cleanup when choosing temporary strip:

- Drop / NOP `EM13_SpaceFeeder01`, `EM30_IO_Status`
- Convert consumed tags such as `ReceiveFrom170` to Base
- Watch PF525 AOI / related type fallout after import

Work only on timestamped copies; do not destroy the user’s golden ACD.

## PackML online prove-out order

```text
Selected = EM00 + EUT
Mode auto (HMI_I_MachineMode=1 → Unit01_Auto)
Clear hold/stop events
Clear → Stop → Reset → Sts_Idle
  (watch Wrk_SEQ_Resetting; SEQ leftover 1000 ≠ real Resetting)
Start → Sts_Starting → Sts_Execute
Master CommandVelocity / position wrapping
Then gates (e.g. G_SealAllowRun) + sample EUT
```

Do not claim EM logic correct from forced CIP motion alone.

## New EM pitfalls (learned on EM16_SealingDown)

### StartExecutionDone deadlock

```text
StartExecutionDone ← Sts_Started ← Wrk_EM_Executing ← StartExecutionDone
```

For non-cam EMs use pass-through:

```text
XIC(Inp_Commands.StartExecution)OTE(Out_EMConditions.StartExecutionDone);
```

### Motion trigger latch

Rising edge into window must **`OTL`** the trigger; clear on MAM `.PC`.  
`OTE` + `ONS` only enables MAM for one scan — unreliable for Seconds-based moves.

### Angle window + time/distance

- Window: `LIM(OnDeg±Win)` / `LIM(OffDeg±Win)` on `Ref_MasterAxis.CommandPosition`
- Gates: `Unit01_Auto` + allow-run + HMI active
- Time+distance without MATC: `MAM(..., Seconds, ...)`

### CIP sampling

One-scan bits are easy to miss over CIP. Also watch:

- `MI_MAM_On.IP` / `.PC`, `MI_MAM_Off.IP` / `.PC`
- Axis `ActualPosition` / `CommandPosition`

## GUI client pitfalls (MCP)

- Window title after import may omit `in <file>.ACD` — match ACD basename stem
- Errors RICHEDIT may be invisible — still readable; status bar confirms verify
- Allow ≥3s settle after import before re-verify

## Typical tags

| Area | Tags |
|------|------|
| State | `UN01_MachineStateModel.Sts_*`, `Program:OP01_Production.Wrk_SEQ_Resetting\|Starting` |
| Done | `UN01.Commands.*`, `UN01.EMConditions_ALL.*`, `EMxx_Conditions.*` |
| Master | `EM00_MachineMaster.ActualPosition`, `CommandVelocity` |
| Gate | `Unit01_Auto`, `G_SealAllowRun` |

## Counter smoke test (non-PackML)

```text
ADD(Cnt,1,Cnt);
GRT(Cnt,5000)MOV(0,Cnt);
```

Write `Cnt≈4990`, expect wrap into a low band — do not watch climb from 0 over CIP.
