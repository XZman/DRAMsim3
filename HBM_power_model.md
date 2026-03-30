# HBM Power Model in DRAMsim3

This document explains how HBM (High Bandwidth Memory) power is modeled in the DRAMsim3 simulator. The model separates power into three distinct components: **DRAM array power**, **controller (logic layer) power**, and **I/O power**.

---

## Table of Contents

1. [Overview](#overview)
2. [DRAM Power](#dram-power)
   - [Power Parameters (IDD Currents)](#power-parameters-idd-currents)
   - [Energy Formulas](#energy-formulas)
   - [Background Power](#background-power)
   - [Total Energy and Average Power](#total-energy-and-average-power)
3. [Controller (Logic Layer) Power](#controller-logic-layer-power)
4. [I/O Power](#io-power)
5. [Configuration Files](#configuration-files)
6. [Statistics Output](#statistics-output)
7. [Building with Thermal Support](#building-with-thermal-support)

---

## Overview

The HBM power model in DRAMsim3 follows the JEDEC standard for DRAM power estimation. An HBM stack consists of multiple DRAM dies and a logic (base) die. The three power components are:

| Component | Description | Source in simulator |
|-----------|-------------|---------------------|
| **DRAM Power** | Energy consumed by ACT/READ/WRITE/REFRESH commands and bank/rank standby states | `configuration.cc`, `simple_stats.cc` |
| **Controller Power** | Logic layer (base die) power including PHY, memory controller circuits | `thermal.h/cc` (thermal build only) |
| **I/O Power** | Differential signaling on the data bus between the logic layer and the host | Embedded in IDD4R/IDD4W parameters |

The overall power breakdown is illustrated below:

```
┌────────────────────────────────────────────────────────┐
│                   HBM Package                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  DRAM Die N  (DRAM Array Power)                  │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  DRAM Die N-1                                    │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  ...                                             │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  DRAM Die 1                                      │  │
│  ├──────────────────────────────────────────────────┤  │
│  │  Logic/Base Die  (Controller Power + I/O Power)  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
           ▲ TSV interconnects between dies
```

Each HBM channel operates independently; power is tracked per channel and aggregated across all channels.

---

## DRAM Power

DRAM power is the dominant component and is computed using the standard JEDEC IDD current model. It is handled entirely in `src/configuration.cc` (parameter initialization) and `src/simple_stats.cc` (energy accumulation).

### Power Parameters (IDD Currents)

The following parameters are read from the `[power]` section of the configuration file:

| Parameter | Description | Typical HBM2 Value |
|-----------|-------------|---------------------|
| `VDD` | Supply voltage (V) | 1.2 V |
| `IDD0` | Active precharge current — one ACT+PRE cycle | 65 mA |
| `IDD2P` | Precharge power-down current | 28 mA |
| `IDD2N` | Precharge standby current (all banks precharged, CKE high) | 40 mA |
| `IDD3N` | Active standby current (at least one bank active, no command) | 55 mA |
| `IDD4R` | Operating burst read current | 390 mA |
| `IDD4W` | Operating burst write current | 500 mA |
| `IDD5AB` | All-bank auto-refresh current | 250 mA |
| `IDD5PB` | Per-bank refresh current | 5 mA |
| `IDD6x` | Self-refresh current | 31 mA |

All IDD values are per device. For HBM, `devices_per_rank = 1` (one 128-bit stack).

### Energy Formulas

Energy is pre-computed once at startup in `Config::InitPowerParams()` (`src/configuration.cc`). Units are **pico-Joules (pJ)** per command or per clock cycle.

#### Activation Energy (per ACT command)
```
act_energy_inc = VDD × (IDD0 × tRC − (IDD3N × tRAS + IDD2N × tRP)) × devices_per_rank
```
The formula subtracts the baseline active and precharge standby energy from the total measured during an activate-precharge cycle, isolating the incremental cost of the ACT command.

*HBM2 (8 Gb) example (tRC = tRAS + tRP = 34 + 14 = 48 cycles):*
```
act_energy_inc = 1.2 × (65 × 48 − (55 × 34 + 40 × 14)) × 1
               = 1.2 × (3120 − 2430)
               = 828 pJ
```

#### Read Energy (per READ command)
```
read_energy_inc = VDD × (IDD4R − IDD3N) × burst_cycle × devices_per_rank
```
`burst_cycle = BL / 2` for HBM (DDR-style double pumping; BL=4 → burst_cycle=2).

*HBM2 example:*
```
read_energy_inc = 1.2 × (390 − 55) × 2 × 1 = 804 pJ
```

#### Write Energy (per WRITE command)
```
write_energy_inc = VDD × (IDD4W − IDD3N) × burst_cycle × devices_per_rank
```
*HBM2 example:*
```
write_energy_inc = 1.2 × (500 − 55) × 2 × 1 = 1,068 pJ
```

#### All-Bank Refresh Energy (per REF command)
```
ref_energy_inc = VDD × (IDD5AB − IDD3N) × tRFC × devices_per_rank
```
*HBM2 example (tRFC = 260 cycles):*
```
ref_energy_inc = 1.2 × (250 − 55) × 260 × 1 = 60,840 pJ
```

#### Per-Bank Refresh Energy (per REFb command)
```
refb_energy_inc = VDD × (IDD5PB − IDD3N) × tRFCb × devices_per_rank
```

### Background Power

Background (static) energy is charged every clock cycle based on the current rank state. Tracking happens in `Controller::ClockTick()` (`src/controller.cc`):

| Rank State | Energy per Cycle | Formula |
|------------|-----------------|---------|
| Active (≥1 bank open) | `act_stb_energy_inc` | `VDD × IDD3N × devices_per_rank` |
| Idle (all banks precharged) | `pre_stb_energy_inc` | `VDD × IDD2N × devices_per_rank` |
| Power-down | `pre_pd_energy_inc` | `VDD × IDD2P × devices_per_rank` |
| Self-refresh | `sref_energy_inc` | `VDD × IDD6x × devices_per_rank` |

*HBM2 examples (per cycle):*
```
act_stb_energy_inc = 1.2 × 55 × 1 = 66.0 pJ/cycle
pre_stb_energy_inc = 1.2 × 40 × 1 = 48.0 pJ/cycle
sref_energy_inc    = 1.2 × 31 × 1 = 37.2 pJ/cycle
```

Each rank's state is tracked with three counters (`src/simple_stats.cc`):
- `rank_active_cycles` — cycles when at least one bank is open
- `all_bank_idle_cycles` — cycles when all banks are precharged
- `sref_cycles` — cycles spent in self-refresh

Background energy is accumulated per rank and summed across ranks each epoch.

### Total Energy and Average Power

Energy is accumulated every epoch (`epoch_period` cycles) in `SimpleStats::UpdateEpochStats()`:

```
total_energy (pJ) =
    num_act_cmds   × act_energy_inc
  + num_read_cmds  × read_energy_inc
  + num_write_cmds × write_energy_inc
  + num_ref_cmds   × ref_energy_inc
  + num_refb_cmds  × refb_energy_inc
  + Σ_rank [
        rank_active_cycles    × act_stb_energy_inc
      + all_bank_idle_cycles  × pre_stb_energy_inc
      + sref_cycles           × sref_energy_inc
    ]

average_power (mW) = total_energy (pJ) / num_cycles
```

Because `tCK` converts cycles to nanoseconds (and pJ/ns = mW), dividing pJ by the number of cycles gives milliwatts directly when `tCK = 1 ns`. For other clock frequencies the result is naturally scaled by the cycle count representing real time.

---

## Controller (Logic Layer) Power

In an HBM stack the logic (base) die houses the PHY, memory controllers, and crossbar routing. DRAMsim3 models this power through the **thermal module** (enabled at compile time with `-DTHERMAL=1`).

### Parameters

Add a `[thermal]` section to the config file:

```ini
[thermal]
logic_bg_power    = 5     ; Background (idle) logic layer power in mW
logic_max_power   = 25    ; Maximum (peak) logic layer power in mW
power_epoch_period = 10000 ; Thermal update interval in cycles
chip_dim_x        = 0.008  ; Chip X dimension (m)
chip_dim_y        = 0.008  ; Chip Y dimension (m)
amb_temp          = 40     ; Ambient temperature (°C)
mat_dim_x         = 1024
mat_dim_y         = 1024
bank_order        = 1
```

### Implementation

- `numP = num_dies + 1`: the thermal grid includes one extra layer for the logic die (`src/thermal.cc`).
- `avg_logic_power_` stores the current logic layer power in mW and is set via `ThermalCalculator::SetLogicPower()` or `UpdateLogicPower()`.
- Each thermal epoch, the logic power is distributed uniformly across all grid cells of the logic layer:

```cpp
// thermal.cc — UpdatePowerMaps
for (int i = dimX * dimY * (numP - 1); i < dimX * dimY * numP; i++) {
    p_map[j][i] += avg_logic_power_ / dimX / dimY * period;
}
```

This allows an external workload manager (or a co-simulated CPU model) to set a time-varying logic power value, which then feeds into the transient thermal solver.

### Modelling Controller Power Without the Thermal Module

If the thermal module is not compiled in, controller power is **not tracked** by the simulator. To account for it in post-processing:

```
controller_power (mW) ≈ logic_bg_power + f(utilization) × (logic_max_power − logic_bg_power)
```

where `f(utilization)` is a normalized request rate (e.g., `num_reads_done + num_writes_done` per cycle relative to peak bandwidth).

---

## I/O Power

HBM uses short, wide (128-bit per pseudo-channel) through-silicon via (TSV) interconnects instead of a conventional PCB bus. Because the TSV interconnects are extremely short (typically 5–50 μm), the per-bit switching energy is negligible compared to conventional DRAM I/O.

### How I/O Power Is Represented

DRAMsim3 does **not** model a dedicated I/O power term separate from the DRAM array power. I/O switching energy is **implicitly included** in the IDD4R and IDD4W parameters:

```
I/O contribution to read_energy_inc  = f(VDD, IDD4R, IDD3N, burst_cycle)
I/O contribution to write_energy_inc = f(VDD, IDD4W, IDD3N, burst_cycle)
```

`IDD4R` and `IDD4W` are measured at the device pins during a full burst read or write, so they already capture both array activation current and data bus switching current.

### Estimating I/O Power Separately

If you need to isolate I/O power from array power for analysis, you can use the following approximation based on HBM PHY specifications:

```
io_power_per_channel (mW) ≈ V_IO × C_IO × f_IO × BW_utilization
```

Where:
- `V_IO` ≈ 1.2 V for HBM2
- `C_IO` ≈ load capacitance per bit (typically 50–100 fF for TSV interconnect)
- `f_IO` = effective toggle rate = data rate / 2 (each bit toggles at half the data rate on average for random data)
- `BW_utilization` = fraction of peak bandwidth used (reported as `average_bandwidth` in stats)

For a typical HBM2 channel running at 2 Gbps per pin × 128 pins:

```
Peak data rate  = 2 Gbps × 128 = 256 Gbps = 32 GB/s per channel
Peak bandwidth  ≈ 32 GB/s (matches DRAMsim3 config channel_size/tREFI)
```

The simulator reports `average_bandwidth` (GB/s) per channel, which can be used directly to scale an empirical I/O power estimate.

---

## Configuration Files

DRAMsim3 ships with four HBM configuration files:

| File | Generation | Capacity | tCK |
|------|-----------|----------|-----|
| `configs/HBM_4Gb_x128.ini` | HBM1 (legacy, includes thermal section) | 4 Gb | 2 ns |
| `configs/HBM1_4Gb_x128.ini` | HBM1 | 4 Gb | 2 ns |
| `configs/HBM2_4Gb_x128.ini` | HBM2 | 4 Gb | 1 ns |
| `configs/HBM2_8Gb_x128.ini` | HBM2 | 8 Gb | 1 ns |

All files share the same `[power]` section format. The `[thermal]` section is present only in `HBM_4Gb_x128.ini` and must be added manually to other files when using the thermal build.

### Key HBM-Specific Structural Parameters

```ini
[dram_structure]
protocol     = HBM
bankgroups   = 4      ; 4 bank groups
banks_per_group = 4   ; 16 banks total per pseudo-channel
device_width = 128    ; 128-bit wide data bus
BL           = 4      ; Burst length 4 (DDR → 2 beats per cycle)
num_dies     = 4      ; Number of DRAM dies in the stack
hbm_dual_cmd = true   ; Two independent commands per clock cycle

[system]
bus_width    = 128    ; Total bus width per channel
channels     = 8      ; 8 independent channels per HBM stack
```

The `hbm_dual_cmd` option allows the scheduler to issue two independent commands in a single clock cycle (one per pseudo-channel), which is a unique feature of HBM. The stat `hbm_dual_cmds` counts how many cycles this occurred.

---

## Statistics Output

After simulation, DRAMsim3 writes power statistics to three files (prefix controlled by `output_prefix` in the config):

| File | Format | Content |
|------|--------|---------|
| `dramsim3.json` | JSON | Final aggregate stats for all channels |
| `dramsim3epoch.json` | JSON array | Per-epoch stats (time series) |
| `dramsim3.txt` | Text | Human-readable final stats |

When compiled with `-DTHERMAL=1`, additional thermal output files are produced:
- `dramsim3final_temp.csv` — final temperature map per die layer
- `dramsim3epoch_max_temp.csv` — maximum temperature per layer per epoch
- `dramsim3epoch_temp.csv` — full temperature grid per epoch (only when `output_level >= 2`)

### Power-Related Statistics Fields

| Stat | Unit | Description |
|------|------|-------------|
| `act_energy` | pJ | Total activation energy |
| `read_energy` | pJ | Total read burst energy |
| `write_energy` | pJ | Total write burst energy |
| `ref_energy` | pJ | Total all-bank refresh energy |
| `refb_energy` | pJ | Total per-bank refresh energy |
| `act_stb_energy.N` | pJ | Active standby energy of rank N |
| `pre_stb_energy.N` | pJ | Precharge standby energy of rank N |
| `sref_energy.N` | pJ | Self-refresh energy of rank N |
| `total_energy` | pJ | Sum of all energy components |
| `average_power` | mW | `total_energy / num_cycles` |
| `hbm_dual_cmds` | cycles | Cycles where two commands were issued |

### Example JSON Output (one channel)

```json
{
  "channel": 0,
  "num_cycles": 1000000,
  "num_reads_done": 50000,
  "num_writes_done": 25000,
  "num_act_cmds": 75000,
  "num_read_cmds": 50000,
  "num_write_cmds": 25000,
  "num_ref_cmds": 1024,
  "act_energy": 62100000.0,
  "read_energy": 40200000.0,
  "write_energy": 26700000.0,
  "ref_energy": 62361600.0,
  "act_stb_energy": {"0": 66000000.0},
  "pre_stb_energy": {"0": 48000000.0},
  "sref_energy":    {"0": 0.0},
  "total_energy": 305361600.0,
  "average_power": 305.36,
  "average_bandwidth": 12.8
}
```

---

## Building with Thermal Support

To enable the controller power and I/O thermal analysis:

```bash
# Using CMake
mkdir build && cd build
cmake .. -DTHERMAL=1
make -j4

# Using Make directly
make THERMAL=1
```

The thermal build activates `#ifdef THERMAL` code paths in `dram_system.cc` and instantiates the `ThermalCalculator` class. The logic layer power value can be injected at runtime via:

```cpp
// In your simulation driver
dram_system->thermal_calc_.SetLogicPower(logic_power_mw);
```

For a typical HBM2 stack, representative power budgets across the three components are:

| Component | Typical Power |
|-----------|--------------|
| DRAM array (all dies, all channels) | 15–30 W |
| Logic / controller layer | 2–8 W |
| I/O (embedded in IDD4R/IDD4W) | ~1–3 W |
| **Total HBM stack** | **~20–40 W** |

These values depend heavily on memory access patterns and operating frequency. Use the per-epoch stats to monitor power over time and identify hotspots.
