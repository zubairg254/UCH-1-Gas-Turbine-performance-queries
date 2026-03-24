# Gas Turbine Performance Analysis System
## Product Requirements Document (PRD)

---

## Executive Summary

This document provides comprehensive documentation for the UCH 1 2026 Heat Rate Analysis Power Query system. The system consists of 20 interconnected queries that process gas turbine performance data, apply correction factors, and generate performance reports for three GE MS9001E (Frame 9E) gas turbines (GT1A, GT1B, GT1C) operating in a combined-cycle power plant.

**System Purpose**: Calculate corrected performance metrics (output, heat rate, efficiency, exhaust parameters) by applying 33 individual correction factors to measured operational data, with detailed impact analysis per correction factor.

**Key Features**:
- Date-range filtering via Excel Named Ranges (StartDate, EndDate)
- FCBL (Full Combined Base Load) condition filtering
- Dual heat rate calculation methods (kJ/kWh and BTU/kWh)
- RH correction at compressor inlet after evaporative cooler
- MW and HR impact analysis for all 9 correction factor groups
- OFWW (Offline Water Wash) schedule integration
- Consolidated 3-GT summary with hourly totals

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Query Inventory](#query-inventory)
3. [Query Dependencies](#query-dependencies)
4. [Data Flow](#data-flow)
5. [Correction Factor Framework](#correction-factor-framework)
6. [Query Detailed Specifications](#query-detailed-specifications)
7. [Usage Patterns](#usage-patterns)
8. [Performance Calculations](#performance-calculations)

---

## 1. System Architecture

### 1.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA SOURCES LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│  • Base Data 2026 (Hourly operational data + OFWW join)         │
│  • OFWW (Offline Water Wash schedule)                           │
│  • Daily_fuel_gas_composition (LHV, H/C ratio & GCV)            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  UTILITY FUNCTIONS LAYER                        │
├─────────────────────────────────────────────────────────────────┤
│  • fnInterp1D            (1D linear interpolation)              │
│  • fnInterp2D            (2D bilinear interpolation)            │
│  • fnInterpFuelCurve     (Fuel composition interpolation)       │
│  • fnRH_at_Comp_Inlet   (RH at compressor inlet after cooler)  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│            CORRECTION FACTOR FUNCTIONS LAYER                    │
├─────────────────────────────────────────────────────────────────┤
│  • fnCF_CIT              (CFs #1-5:   CIT effects)              │
│  • fnCF_RH               (CFs #6-9:   Humidity effects)         │
│  • fnCF_ShaftSpeed       (CFs #10-13: Speed effects)            │
│  • fnCF_FuelTemp         (CFs #14-15: Fuel temp effects)        │
│  • fnCF_InletDP          (CFs #16-19: Inlet pressure loss)      │
│  • fnCF_ExhaustPressure  (CFs #20-22: Exhaust back-pressure)    │
│  • fnCF_ExhaustDP        (CFs #23-25: HRSG duct pressure)       │
│  • fnCF_BaroPressure     (CFs #26-29: Barometric pressure)      │
│  • fnCF_FuelComp         (CFs #30-33: Fuel composition)         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              PERFORMANCE CALCULATION LAYER                      │
├─────────────────────────────────────────────────────────────────┤
│  • GT1A_Performance  (GT1A corrected performance)               │
│  • GT1B_Performance  (GT1B corrected performance)               │
│  • GT1C_Performance  (GT1C corrected performance)               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   REPORTING LAYER                               │
├─────────────────────────────────────────────────────────────────┤
│  • GT_Combined_Summary (Consolidated 3-GT report + totals)      │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 System Components

| Layer | Component Count | Purpose |
|-------|----------------|---------|
| Data Sources | 3 | Raw operational data (Base Data 2026), fuel composition (Daily_fuel_gas_composition), and OFWW schedule |
| Utility Functions | 4 | Mathematical interpolation helpers (fnInterp1D, fnInterp2D, fnInterpFuelCurve) + RH at compressor inlet (fnRH_at_Comp_Inlet) |
| Correction Factors | 9 | Apply 33 individual CFs to measurements |
| Performance Queries | 3 | Calculate corrected metrics per GT + MW/HR impact analysis |
| Reporting | 1 | Consolidated multi-GT summary with totals + MW/HR impacts |

---


## 2. Query Inventory

### 2.1 Complete Query List

| # | Query Name | Type | Setting | Purpose |
|---|------------|------|---------|---------|
| 1 | **Base Data 2026** | Data Source | Connection Only | Loads and filters hourly operational data (FCBL only), joins OFWW, date-range filtered |
| 2 | **OFWW** | Data Source | Connection Only | Loads Offline Water Wash schedule per GT |
| 3 | **Daily_fuel_gas_composition** | Data Source | Excel Table | Daily fuel LHV (Btu/lbm), H/C ratio, and GCV (BTU/SCF) |
| 4 | **fnInterp1D** | Utility Function | Connection Only | 1D linear interpolation helper |
| 5 | **fnInterp2D** | Utility Function | Connection Only | 2D bilinear interpolation helper |
| 6 | **fnInterpFuelCurve** | Utility Function | Connection Only | Fuel composition curve interpolation |
| 7 | **fnRH_at_Comp_Inlet** | Utility Function | Connection Only | RH at compressor inlet after evap cooler (isenthalpic process) |
| 8 | **fnCF_CIT** | CF Function | Connection Only | Compressor Inlet Temperature CFs (#1-5) |
| 9 | **fnCF_RH** | CF Function | Connection Only | Relative Humidity CFs (#6-9) |
| 10 | **fnCF_ShaftSpeed** | CF Function | Connection Only | Shaft Speed Ratio CFs (#10-13) |
| 11 | **fnCF_FuelTemp** | CF Function | Connection Only | Fuel Temperature CFs (#14-15) |
| 12 | **fnCF_InletDP** | CF Function | Connection Only | Inlet Differential Pressure CFs (#16-19) |
| 13 | **fnCF_ExhaustPressure** | CF Function | Connection Only | Exhaust Back-Pressure CFs (#20-22) |
| 14 | **fnCF_ExhaustDP** | CF Function | Connection Only | HRSG Duct Pressure CFs (#23-25) |
| 15 | **fnCF_BaroPressure** | CF Function | Connection Only | Barometric Pressure CFs (#26-29) |
| 16 | **fnCF_FuelComp** | CF Function | Connection Only | Fuel Composition CFs (#30-33) |
| 17 | **GT1A_Performance** | Performance Query | Load to Worksheet | GT1A corrected performance metrics + MW/HR impacts |
| 18 | **GT1B_Performance** | Performance Query | Load to Worksheet | GT1B corrected performance metrics + MW/HR impacts |
| 19 | **GT1C_Performance** | Performance Query | Load to Worksheet | GT1C corrected performance metrics + MW/HR impacts |
| 20 | **GT_Combined_Summary** | Report Query | Load to Worksheet | Consolidated 3-GT summary with totals + MW/HR impacts |

---

## 3. Query Dependencies

### 3.1 Dependency Hierarchy

```
Level 0 (No Dependencies):
├── fnInterp1D
├── fnInterp2D
├── fnRH_at_Comp_Inlet
├── OFWW
└── Base Data 2026
    └── Depends on: OFWW (left-joined for water wash indicators)

Level 1 (Depends on Level 0):
└── fnInterpFuelCurve
    └── Depends on: fnInterp1D

Level 2 (Depends on Level 0-1):
├── fnCF_CIT
│   └── Depends on: fnInterp1D
├── fnCF_RH
│   └── Depends on: fnInterp2D
├── fnCF_ShaftSpeed
│   └── Depends on: fnInterp2D
├── fnCF_FuelTemp
│   └── Depends on: fnInterp1D
├── fnCF_InletDP
│   └── Depends on: fnInterp2D
├── fnCF_ExhaustPressure
│   └── Depends on: fnInterp2D
├── fnCF_ExhaustDP
│   └── Depends on: fnInterp1D
├── fnCF_BaroPressure
│   └── Depends on: fnInterp2D
└── fnCF_FuelComp
    └── Depends on: fnInterp1D, fnInterpFuelCurve

Level 3 (Depends on Level 0-2):
├── GT1A_Performance
│   └── Depends on: Base Data 2026, Daily_fuel_gas_composition,
│       fnRH_at_Comp_Inlet,
│       fnCF_CIT, fnCF_RH, fnCF_ShaftSpeed, fnCF_FuelTemp,
│       fnCF_InletDP, fnCF_ExhaustPressure, fnCF_ExhaustDP,
│       fnCF_BaroPressure, fnCF_FuelComp
├── GT1B_Performance
│   └── Depends on: [Same as GT1A]
└── GT1C_Performance
    └── Depends on: [Same as GT1A]

Level 4 (Depends on Level 0-3):
└── GT_Combined_Summary
    └── Depends on: GT1A_Performance, GT1B_Performance, GT1C_Performance
```

### 3.2 Critical Path Analysis

**Critical Path**: Base Data 2026 → fnInterp* → fnCF_* → GT*_Performance → GT_Combined_Summary

**Bottleneck Queries**:
- **Base Data 2026**: All performance queries depend on this
- **fnInterp1D/2D**: All CF functions depend on these
- **All 9 CF functions**: Each performance query calls all 9 CF functions

---


## 4. Data Flow

### 4.1 End-to-End Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│ STEP 1: DATA INGESTION                                          │
├──────────────────────────────────────────────────────────────────┤
│ Base Data 2026:                                                  │
│   • Loads hourly operational data from Excel (file path from    │
│     Named Range "filepth")                                       │
│   • Filters by date range (StartDate/EndDate Named Ranges)      │
│   • Filters for FCBL (Full Combined Base Load) conditions:      │
│     - All 3 GTs: IGV > 84° AND Gen_MW > 100                     │
│   • Left-joins OFWW schedule (GT1A_OFWW, GT1B_OFWW, GT1C_OFWW)  │
│   • Renames 130+ raw PI tags to friendly column names           │
│   • Adds Date Only, Hour columns (Hour 24 → previous day)       │
│                                                                  │
│ Daily_fuel_gas_composition (Excel table):                       │
│   • Loads daily fuel data: "Uch 1 LHV BTU/lbm", "H/C Ratio",    │
│     "UCH 1 GCV" (BTU/SCF)                                        │
│   • Inner-joined to GT performance queries by Date              │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ STEP 2: PARAMETER PREPARATION (Per GT)                          │
├──────────────────────────────────────────────────────────────────┤
│ For each GT (A/B/C):                                             │
│   • Extract GT-specific measurements from Base Data 2026        │
│   • Calculate Wet Bulb Temperature (ASHRAE Hyland-Wexler):      │
│     - Iterative psychrometric calculation (max 50 iterations)   │
│     - Convergence criterion: |h_wb - h_db| < 0.001              │
│   • Calculate derived parameters:                                │
│     - CIT_degC (Compressor Inlet Temp)                          │
│     - RH_pct (Ambient Relative Humidity)                        │
│     - RH_at_CIT (RH at compressor inlet after evap cooler)      │
│       → Uses fnRH_at_Comp_Inlet (isenthalpic process, W const)  │
│       → Falls back to ambient RH if null                         │
│     - BaroPressure_mbara (Ambient Pressure, already in mbar)    │
│     - FuelLHV_kJkg (LHV in kJ/kg: Btu/lbm × 2.326)              │
│     - HC_ratio (H/C ratio, default 4.0 if missing)              │
│     - ShaftSpeedRatio (measured RPM / 3000)                     │
│     - FuelTemp_degC (measured, default 15°C if null)            │
│     - InletDP_mmH2O (in H₂O × 25.4, clamped [76.2, 177.8])     │
│     - DeltaExhDP_mmH2O (default 0 - no sensor)                  │
│     - ExhaustDP_mmH2O (default 266.7 = 10.5 in H₂O)            │
│   • Calculate Measured Heat Rate (two methods):                  │
│     Method 1 (kJ/kWh): (FF × 2.20462 × LHV_Btu × 3.6) / Gen_MW │
│     Method 2 (BTU/kWh): ((FF × 3600 / 0.9987 × 35.31467) ×     │
│                          GCV / 1.109) / (Gen_MW × 1000)         │
│   • Estimate Exhaust Flow (GE MS9001E mass balance):             │
│     AirFlow = 410 × (IGV/84) × (P/1013.25) × √(288.15/T_K)     │
│     ExhaustFlow = AirFlow + FuelFlow                             │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ STEP 3: CORRECTION FACTOR CALCULATION                           │
├──────────────────────────────────────────────────────────────────┤
│ Call 9 CF functions sequentially:                                │
│                                                                  │
│ 1. fnCF_CIT(CIT_degC)                                           │
│    → CF1 (ExhaustPressure), CF2-5 (Output, HR, Flow, Temp)     │
│                                                                  │
│ 2. fnCF_RH(RH_pct, CIT_degC)                                    │
│    → CF6-9 (Output, HR, Flow, Temp)                            │
│                                                                  │
│ 3. fnCF_ShaftSpeed(ShaftSpeedRatio, CIT_degC)                  │
│    → CF10-13 (Output, HR, Flow, Temp)                          │
│                                                                  │
│ 4. fnCF_FuelTemp(FuelTemp_degC)                                 │
│    → CF14-15 (Output, HR)                                       │
│                                                                  │
│ 5. fnCF_InletDP(InletDP_mmH2O, CIT_degC)                       │
│    → CF16-19 (Output, HR, Flow, Temp)                          │
│                                                                  │
│ 6. fnCF_ExhaustPressure(DeltaExhDP_mmH2O, CIT_degC)            │
│    → CF20-22 (Output, HR, Temp)                                │
│                                                                  │
│ 7. fnCF_ExhaustDP(ExhaustDP_mmH2O)                              │
│    → CF23-25 (Output, HR, Temp)                                │
│                                                                  │
│ 8. fnCF_BaroPressure(BaroPressure_mbara, CIT_degC)             │
│    → CF26-29 (Output, HR, Flow, Temp)                          │
│                                                                  │
│ 9. fnCF_FuelComp(FuelLHV_kJkg, HC_ratio)                        │
│    → CF30-33 (Output, HR, Flow, Temp)                          │
│                                                                  │
│ Each CF function returns a record with 2-5 correction factors   │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ STEP 4: CORRECTED PERFORMANCE CALCULATION                       │
├──────────────────────────────────────────────────────────────────┤
│ Expand all CF records into individual columns (33 CFs total)    │
│                                                                  │
│ Calculate Combined Correction Factors:                           │
│   • Combined_OutputCF = CF2 × CF6 × CF10 × CF14 × CF16 ×       │
│                         CF20 × CF23 × CF26 × CF30               │
│   • Combined_HRCF = CF3 × CF7 × CF11 × CF15 × CF17 ×           │
│                     CF21 × CF24 × CF27 × CF31                   │
│   • Combined_ExhFlowCF = CF4 × CF8 × CF12 × CF18 ×             │
│                          CF28 × CF32                             │
│                                                                  │
│ Calculate Corrected Metrics (both methods):                      │
│   • CorrectedOutput_MW = MeasuredOutput_MW / Combined_OutputCF  │
│   • CorrectedHeatRate (kJ/kWh) = MeasuredHeatRate / Combined_HRCF│
│   • CorrectedHeatRate_BTU_kWh = GTHeatRate_BTU_kWh / Combined_HRCF│
│   • CorrectedEfficiency = 3412.1416 / CorrectedHeatRate_BTU_kWh │
│   • CorrectedExhaustTemp = MeasuredExhaustTemp -                │
│       (CF5 + CF9 + CF13 + CF19 + CF22 + CF25 + CF29 + CF33)    │
│   • CorrectedExhaustFlow = MeasuredExhaustFlow /                │
│       Combined_ExhFlowCF                                         │
│                                                                  │
│ Calculate Operational Parameters:                                │
│   • Evaporative Cooler Effectiveness (Calc WB & H2 Bldg WB)    │
│   • Cooler Status (OFF / ON / Subcooling / Review / No Data)   │
│   • MMBTU/hr = FuelFlow × 2.20462 × LHV × 3600 / 1,000,000     │
│   • Compressor Pressure Ratio = P_disch_bar / (P_ambient_mbar/1000)│
│   • Compressor Isentropic Efficiency: η_c = (PR^0.2857 - 1) /  │
│       (T2/T1 - 1), where 0.2857 = (γ-1)/γ, γ=1.4 (dry air)     │
│                                                                  │
│ Calculate MW Impact per Correction Factor:                       │
│   • MW_CIT = Gen_MW × (1/CF2_OutputRatio - 1)                  │
│   • MW_RH = Gen_MW × (1/CF6_OutputRatio - 1)                   │
│   • [Similar for all 9 output CFs]                              │
│   • CF > 1 → negative (correction reduces output)               │
│   • CF < 1 → positive (correction increases output)             │
│                                                                  │
│ Calculate HR Impact per Correction Factor (BTU/kWh):            │
│   • HR_CIT = GTHeatRate_BTU_kWh × (1/CF3_HeatRateRatio - 1)    │
│   • HR_RH = GTHeatRate_BTU_kWh × (1/CF7_HeatRateRatio - 1)     │
│   • [Similar for all 9 heat rate CFs]                           │
│   • CF > 1 → positive (correction increases HR = penalty)       │
│   • CF < 1 → negative (correction decreases HR = benefit)       │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ STEP 5: REPORTING & AGGREGATION                                 │
├──────────────────────────────────────────────────────────────────┤
│ GT_Combined_Summary:                                             │
│   • Stacks GT1A, GT1B, GT1C performance rows                    │
│   • Adds Month column (format: "MMM-yyyy")                       │
│   • Adds GT Efficiency = 3412.14 / GTHeatRate_BTU_kWh           │
│   • Adds Corrected GT Efficiency = 3412.14 / CorrectedHeatRate_BTU_kWh│
│   • Groups by (Date Only + Hour)                                 │
│   • Appends TOTAL row per hour:                                  │
│     - Sum: MMBTU/hr, Gen_MW, CorrectedOutput_MW, MW_* impacts   │
│     - Null: All other metrics (intensive properties)             │
│   • Interleaves: GT1A → GT1B → GT1C → TOTAL per hour           │
│   • Renames columns to friendly display names                    │
│   • Includes MW and HR impact columns for all 9 CF groups       │
└──────────────────────────────────────────────────────────────────┘
```

---


## 5. Correction Factor Framework

### 5.1 Correction Factor Categories

The system applies 33 individual correction factors organized into 9 functional groups:

| CF Group | CF Numbers | Parameters | Outputs | Purpose |
|----------|-----------|------------|---------|---------|
| **CIT** | CF1-CF5 | CIT_degC | ExhP, Out, HR, Flow, Temp | Compressor inlet temperature effects |
| **RH** | CF6-CF9 | RH_pct, CIT_degC | Out, HR, Flow, Temp | Relative humidity effects |
| **ShaftSpeed** | CF10-CF13 | SpeedRatio, CIT_degC | Out, HR, Flow, Temp | Shaft speed variation effects |
| **FuelTemp** | CF14-CF15 | FuelTemp_degC | Out, HR | Fuel gas temperature effects |
| **InletDP** | CF16-CF19 | InletDP_mmH2O, CIT_degC | Out, HR, Flow, Temp | Inlet pressure loss effects |
| **ExhaustPressure** | CF20-CF22 | DeltaExhDP_mmH2O, CIT_degC | Out, HR, Temp | Exhaust back-pressure effects |
| **ExhaustDP** | CF23-CF25 | ExhaustDP_mmH2O | Out, HR, Temp | HRSG duct pressure effects |
| **BaroPressure** | CF26-CF29 | BaroPressure_mbara, CIT_degC | Out, HR, Flow, Temp | Barometric pressure effects |
| **FuelComp** | CF30-CF33 | FuelLHV_kJkg, HC_ratio | Out, HR, Flow, Temp | Fuel composition effects |

### 5.2 Correction Factor Types

| CF Type | Count | Application Method | Purpose |
|---------|-------|-------------------|---------|
| **Output Ratio** | 9 | Multiplicative (divide measured by CF) | Corrects generator output (MW) |
| **Heat Rate Ratio** | 9 | Multiplicative (divide measured by CF) | Corrects heat rate (kJ/kWh) |
| **Exhaust Flow Ratio** | 6 | Multiplicative (divide measured by CF) | Corrects exhaust mass flow (kg/s) |
| **Exhaust Temp Diff** | 8 | Additive (subtract CF from measured) | Corrects exhaust temperature (°C) |
| **Exhaust Pressure** | 1 | Absolute value (not a ratio) | Reference exhaust pressure (mmH₂O) |

### 5.3 Combined Correction Factor Formulas

```
Combined_OutputCF = CF2 × CF6 × CF10 × CF14 × CF16 × CF20 × CF23 × CF26 × CF30

Combined_HRCF = CF3 × CF7 × CF11 × CF15 × CF17 × CF21 × CF24 × CF27 × CF31

Combined_ExhFlowCF = CF4 × CF8 × CF12 × CF18 × CF28 × CF32

Combined_ExhTempDiff = CF5 + CF9 + CF13 + CF19 + CF22 + CF25 + CF29 + CF33
```

### 5.4 Baseline Conditions

All correction factors are normalized to these design/baseline conditions:

| Parameter | Baseline Value | Units |
|-----------|---------------|-------|
| Compressor Inlet Temperature | 15.0 | °C |
| Relative Humidity | 48.5 | % |
| Shaft Speed Ratio | 1.0 | (3000 RPM) |
| Fuel Temperature | 26.7 | °C |
| Inlet Differential Pressure | 127.0 | mm H₂O |
| Exhaust Back-Pressure Deviation | 0.0 | mm H₂O |
| HRSG Duct Pressure | 337.82 | mm H₂O |
| Barometric Pressure | 1005.91 | mbar |
| Fuel LHV | 12552 | kJ/kg |
| H/C Ratio | 3.86 | - |

**At baseline conditions, all CFs = 1.0 (ratios) or 0.0 (additive)**

---


## 6. Query Detailed Specifications

### 6.1 Data Source Queries

#### 6.1.1 Base Data 2026
**Type**: Data Source Query  
**Setting**: Connection Only  
**Purpose**: Loads and filters hourly operational data for all three gas turbines

**Key Operations**:
1. Reads date range parameters from Named Ranges:
   - `StartDate` and `EndDate` (single-cell Named Ranges)
   - Converts to date type if needed
2. Loads Excel workbook from file path specified in `filepth` Named Range table
3. Promotes headers and transforms column types (130+ PI tags)
4. Adds calculated columns:
   - `Date Only`: Adjusted date (Hour 24 → previous day)
   - `Hour`: Hour of day (0-23, with 24 for midnight)
5. Filters by date range: `Date Only >= StartDate AND Date Only <= EndDate`
6. Adds `Load_Type` column:
   - "FCBL" if ALL three GTs meet: IGV > 84° AND Gen_MW > 100
   - "Non-FCBL" otherwise
7. Filters for FCBL rows only
8. Left-joins OFWW schedule:
   - Joins on `Date Only` = OFWW `Date`
   - Expands GT-A, GT-B, GT-C columns
   - Creates GT1A_OFWW, GT1B_OFWW, GT1C_OFWW (text or null)
9. Renames 130+ raw PI tags to friendly column names

**Output Columns** (selected):
- Timestamps: `Date Only`, `Hour`, `Time Stamp`
- Ambient: `Ambient_Temp_DB`, `Rel_Humidity`, `Ambient_Press`, `H2_Bldg_WB_Temp`
- GT1A: `GT1A_Gen_MW`, `GT1A_Fuel_Flow`, `GT1A_Comp_Inlet_Temp`, `GT1A_Turb_Exh_Temp`, `GT1A_Fuel_Temp`, `GT1A_Turbine_Speed`, `GT1A_Diff_Press`, `GT1A_IGV_Position`, `GT1A_Comp_Disch_Temp`, `GT1A_Comp_Disch_Press`, `GT1A_OFWW`
- GT1B: (same structure as GT1A)
- GT1C: (same structure as GT1A)

**Dependencies**: OFWW query, Named Ranges (StartDate, EndDate, filepth)

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

### 6.2 Utility Function Queries

#### 6.2.1 fnInterp1D
**Type**: Utility Function  
**Setting**: Connection Only  
**Purpose**: 1D linear interpolation helper (mirrors VBA Interp1D)

**Signature**:
```powerquery
(xList as list, yList as list, xVal as number) as any
```

**Parameters**:
- `xList`: X-axis breakpoints (must be sorted ascending)
- `yList`: Y-axis values corresponding to xList
- `xVal`: Input value to interpolate

**Returns**: Interpolated Y value at xVal

**Algorithm**:
1. Range check: xVal must be within [xMin, xMax]
2. Find bracketing indices: i where xList{i} ≤ xVal < xList{i+1}
3. Linear interpolation: `y = y0 + (y1 - y0) × (xVal - x0) / (x1 - x0)`

**Error Handling**: Throws "OutOfRange" error if xVal outside table bounds

**Dependencies**: None

**Used By**: fnCF_CIT, fnCF_FuelTemp, fnCF_ExhaustDP, fnInterpFuelCurve, fnCF_FuelComp

---

#### 6.2.2 fnInterp2D
**Type**: Utility Function  
**Setting**: Connection Only  
**Purpose**: 2D bilinear interpolation helper (mirrors VBA Interp2D)

**Signature**:
```powerquery
(xList as list, yList as list, zGrid as list, xVal as number, yVal as number) as any
```

**Parameters**:
- `xList`: X-axis breakpoints (e.g., CIT values)
- `yList`: Y-axis breakpoints (e.g., RH values)
- `zGrid`: 2D grid of Z values (list of lists: zGrid{xIndex}{yIndex})
- `xVal`: Input X value
- `yVal`: Input Y value

**Returns**: Bilinearly interpolated Z value at (xVal, yVal)

**Algorithm**:
1. Range check: (xVal, yVal) must be within table bounds
2. Find bracketing indices: ix, iy
3. Calculate interpolation weights: tx, ty
4. Bilinear formula: `z = z00×(1-tx)×(1-ty) + z10×tx×(1-ty) + z01×(1-tx)×ty + z11×tx×ty`

**Error Handling**: Throws "OutOfRange" error if (xVal, yVal) outside table bounds

**Dependencies**: None

**Used By**: fnCF_RH, fnCF_ShaftSpeed, fnCF_InletDP, fnCF_ExhaustPressure, fnCF_BaroPressure

---

#### 6.2.3 fnInterpFuelCurve
**Type**: Utility Function  
**Setting**: Connection Only  
**Purpose**: Interpolate within a single H/C sub-curve (3 LHV breakpoints)

**Signature**:
```powerquery
(FuelLHV as number, lhv0 as number, lhv1 as number, lhv2 as number,
 v0 as number, v1 as number, v2 as number) as any
```

**Parameters**:
- `FuelLHV`: Input fuel LHV (kJ/kg)
- `lhv0, lhv1, lhv2`: Three LHV breakpoints for this H/C curve
- `v0, v1, v2`: Corresponding CF values at each LHV breakpoint

**Returns**: Interpolated CF value at FuelLHV

**Algorithm**: Delegates to fnInterp1D with 3-point curve

**Dependencies**: fnInterp1D

**Used By**: fnCF_FuelComp

---

#### 6.2.4 fnRH_at_Comp_Inlet
**Type**: Utility Function  
**Setting**: Connection Only  
**Purpose**: Calculate Relative Humidity at Compressor Inlet after Evaporative Cooler

**Signature**:
```powerquery
(CIT_degC as number, AmbientTemp_degC as number, RH_Ambient_pct as number, Pressure_mbar as number) as any
```

**Parameters**:
- `CIT_degC`: Compressor Inlet Temperature (°C)
- `AmbientTemp_degC`: Ambient dry-bulb temperature before evap cooler (°C)
- `RH_Ambient_pct`: Ambient relative humidity before evap cooler (%)
- `Pressure_mbar`: Ambient barometric pressure (mbar)

**Returns**: RH at compressor inlet as PERCENTAGE (0-100), or null on invalid inputs

**Algorithm** (Isenthalpic Process - Humidity Ratio W Conserved):
1. Convert pressure: mbar → kPa (÷10)
2. Calculate saturation vapor pressure at ambient T1 (Tetens equation):
   - P_sat [kPa] = 0.6108 × exp(17.27 × T / (T + 237.3))
3. Calculate actual vapor pressure at ambient: P_v1 = (RH/100) × P_sat_T1
4. Calculate humidity ratio before evap cooler: W1 = 0.622 × P_v1 / (P_total - P_v1)
5. W2 = W1 (conserved through isenthalpic evap cooler)
6. Calculate saturation vapor pressure at CIT (T2)
7. Reverse calculate actual vapor pressure at CIT: P_v2 = W2 × P_total / (0.622 + W2)
8. Calculate RH at CIT: RH2 = (P_v2 / P_sat_T2) × 100
9. Clamp result to [0, 100]

**Baseline**: When evap cooler OFF (CIT = AmbientTemp), output = RH_Ambient_pct

**Error Handling**: Returns null for invalid inputs (T ≤ -273.15, RH < 0 or > 100, P ≤ 0)

**Dependencies**: None

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---


### 6.3 Correction Factor Function Queries

#### 6.3.1 fnCF_CIT
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #1-5 based on Compressor Inlet Temperature

**Signature**:
```powerquery
(CIT_degC as number) as record
```

**Input Parameters**:
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF1_ExhaustPressure_mmH2O`: Exhaust pressure (absolute, mm H₂O) - NOT a ratio
- `CF2_OutputRatio`: Output correction ratio
- `CF3_HeatRateRatio`: Heat rate correction ratio
- `CF4_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF5_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.78°C to 40.56°C (10 breakpoints)
- Method: 1D linear interpolation

**Baseline**: CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp1D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.2 fnCF_RH
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #6-9 based on Relative Humidity and CIT

**Signature**:
```powerquery
(RH_pct as number, CIT_degC as number) as record
```

**Input Parameters**:
- `RH_pct`: Relative Humidity (%)
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF6_OutputRatio`: Output correction ratio
- `CF7_HeatRateRatio`: Heat rate correction ratio
- `CF8_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF9_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.8°C to 40.6°C (6 breakpoints)
- Y-axis: RH from 0% to 100% (9 breakpoints)
- Method: 2D bilinear interpolation

**Baseline**: RH = 48.5%, CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp2D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.3 fnCF_ShaftSpeed
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #10-13 based on Shaft Speed Ratio and CIT

**Signature**:
```powerquery
(SpeedRatio as number, CIT_degC as number) as record
```

**Input Parameters**:
- `SpeedRatio`: Shaft speed ratio (measured RPM / 3000)
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF10_OutputRatio`: Output correction ratio
- `CF11_HeatRateRatio`: Heat rate correction ratio
- `CF12_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF13_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.8°C to 40.6°C (6 breakpoints)
- Y-axis: SpeedRatio from 0.98 to 1.02 (9 breakpoints)
- Method: 2D bilinear interpolation

**Baseline**: SpeedRatio = 1.0 (3000 RPM), CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp2D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.4 fnCF_FuelTemp
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #14-15 based on Fuel Gas Temperature

**Signature**:
```powerquery
(FuelTemp_degC as number) as record
```

**Input Parameters**:
- `FuelTemp_degC`: Fuel gas temperature (°C)

**Output Record Fields**:
- `CF14_OutputRatio`: Output correction ratio
- `CF15_HeatRateRatio`: Heat rate correction ratio

**Interpolation Table**:
- X-axis: FuelTemp from -1.1°C to 54.4°C (9 breakpoints)
- Method: 1D linear interpolation

**Baseline**: FuelTemp = 26.7°C → All ratios = 1.0

**Dependencies**: fnInterp1D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.5 fnCF_InletDP
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #16-19 based on Inlet Differential Pressure and CIT

**Signature**:
```powerquery
(InletDP_mmH2O as number, CIT_degC as number) as record
```

**Input Parameters**:
- `InletDP_mmH2O`: Inlet differential pressure (mm H₂O)
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF16_OutputRatio`: Output correction ratio
- `CF17_HeatRateRatio`: Heat rate correction ratio
- `CF18_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF19_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.8°C to 40.6°C (6 breakpoints)
- Y-axis: InletDP from 76.2 to 177.8 mm H₂O (9 breakpoints)
- Method: 2D bilinear interpolation

**Baseline**: InletDP = 127.0 mm H₂O, CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp2D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.6 fnCF_ExhaustPressure
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #20-22 based on Exhaust Back-Pressure Deviation and CIT

**Signature**:
```powerquery
(DeltaExhDP_mmH2O as number, CIT_degC as number) as record
```

**Input Parameters**:
- `DeltaExhDP_mmH2O`: SIGNED deviation from design back-pressure (mm H₂O)
  - Negative = lower than design (better performance)
  - Positive = higher than design (worse performance)
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF20_OutputRatio`: Output correction ratio
- `CF21_HeatRateRatio`: Heat rate correction ratio
- `CF22_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.8°C to 40.6°C (6 breakpoints)
- Y-axis: DeltaExhDP from -127.0 to +127.0 mm H₂O (9 breakpoints)
- Method: 2D bilinear interpolation

**Baseline**: DeltaExhDP = 0.0 mm H₂O (at design), CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp2D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.7 fnCF_ExhaustDP
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #23-25 based on HRSG Duct Pressure

**Signature**:
```powerquery
(ExhaustDP_mmH2O as number) as record
```

**Input Parameters**:
- `ExhaustDP_mmH2O`: Absolute HRSG duct differential pressure (mm H₂O)

**Output Record Fields**:
- `CF23_OutputRatio`: Output correction ratio
- `CF24_HeatRateRatio`: Heat rate correction ratio
- `CF25_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: ExhaustDP from 210.82 to 464.82 mm H₂O (3 breakpoints)
- Method: 1D linear interpolation

**Baseline**: ExhaustDP = 337.82 mm H₂O → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp1D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.8 fnCF_BaroPressure
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #26-29 based on Barometric Pressure and CIT

**Signature**:
```powerquery
(BaroPressure_mbara as number, CIT_degC as number) as record
```

**Input Parameters**:
- `BaroPressure_mbara`: Barometric (ambient) pressure (mbar absolute)
- `CIT_degC`: Compressor Inlet Temperature (°C)

**Output Record Fields**:
- `CF26_OutputRatio`: Output correction ratio
- `CF27_HeatRateRatio`: Heat rate correction ratio
- `CF28_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF29_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Table**:
- X-axis: CIT from -17.8°C to 40.6°C (6 breakpoints)
- Y-axis: BaroPressure from 955.62 to 1056.21 mbar (9 breakpoints)
- Method: 2D bilinear interpolation

**Baseline**: BaroPressure = 1005.91 mbar, CIT = 15°C → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp2D

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---

#### 6.3.9 fnCF_FuelComp
**Type**: Correction Factor Function  
**Setting**: Connection Only  
**Purpose**: Calculate CFs #30-33 based on Fuel Composition (LHV and H/C ratio)

**Signature**:
```powerquery
(FuelLHV_kJkg as number, HC_ratio as number) as record
```

**Input Parameters**:
- `FuelLHV_kJkg`: Fuel Lower Heating Value (kJ/kg)
- `HC_ratio`: Hydrogen-to-Carbon ratio (dimensionless)

**Output Record Fields**:
- `CF30_OutputRatio`: Output correction ratio
- `CF31_HeatRateRatio`: Heat rate correction ratio
- `CF32_ExhaustFlowRatio`: Exhaust flow correction ratio
- `CF33_ExhaustTempDiff_degC`: Exhaust temperature difference (°C, additive)

**Interpolation Method**:
1. Three H/C curves: 3.74, 3.86, 4.0
2. Each curve has 3 LHV breakpoints
3. First interpolate within each H/C curve (1D, LHV)
4. Then interpolate across H/C curves (1D, H/C ratio)

**Baseline**: LHV = 12552 kJ/kg, H/C = 3.86 → All ratios = 1.0, Temp diff = 0.0

**Dependencies**: fnInterp1D, fnInterpFuelCurve

**Used By**: GT1A_Performance, GT1B_Performance, GT1C_Performance

---


### 6.4 Performance Calculation Queries

#### 6.4.1 GT1A_Performance / GT1B_Performance / GT1C_Performance
**Type**: Performance Calculation Query  
**Setting**: Load to Worksheet  
**Purpose**: Calculate corrected performance metrics for each gas turbine

**Note**: All three queries (GT1A, GT1B, GT1C) follow identical logic, differing only in the GT-specific column names they reference.

**Data Sources**:
1. Base Data 2026 (filtered for FCBL conditions)
2. Daily_fuel_gas_composition (joined by Date)

**Processing Steps**:

**Step 1: Data Preparation**
- Select only ambient + GT-specific columns from Base Data 2026
- Filter rows from StartFrom date onward (configurable: StartYear, StartMonth, StartDay)
- Join fuel composition data (Inner Join by Date)
- Filter out rows with null/zero LHV

**Step 2: Wet Bulb Temperature Calculation**
- Calculate Wet Bulb Temperature using ASHRAE Hyland-Wexler method
- Iterative psychrometric calculation (max 50 iterations, convergence < 0.001)

**Step 3: Parameter Preparation**
- Convert units:
  - LHV: Btu/lbm → kJ/kg (× 2.326)
  - InletDP: in H₂O → mm H₂O (× 25.4)
  - ExhaustDP: in H₂O → mm H₂O (× 25.4)
- Calculate derived parameters:
  - ShaftSpeedRatio = measured RPM / 3000
  - FuelTemp_degC (use measured, default 15°C if null)
  - InletDP_mmH2O (clamped to [76.2, 177.8] mm H₂O)
  - DeltaExhDP_mmH2O (default 0 - no sensor)
  - ExhaustDP_mmH2O (default 266.7 mm H₂O = 10.5 in H₂O)
- Calculate Measured Heat Rate:
  ```
  MeasuredHeatRate = (FuelFlow_kg/s × 2.20462 × LHV_Btu/lbm × 3.6) / Gen_MW
  ```
- Estimate Exhaust Flow (GE MS9001E mass balance):
  ```
  AirFlow = 410 × (IGV/84) × (P_mbar/1013.25) × √(288.15/T_inlet_K)
  ExhaustFlow = AirFlow + FuelFlow
  ```

**Step 4: Correction Factor Calculation**
- Call all 9 CF functions:
  1. fnCF_CIT(CIT_degC)
  2. fnCF_RH(RH_pct, CIT_degC)
  3. fnCF_ShaftSpeed(ShaftSpeedRatio, CIT_degC)
  4. fnCF_FuelTemp(FuelTemp_degC)
  5. fnCF_InletDP(InletDP_mmH2O, CIT_degC)
  6. fnCF_ExhaustPressure(DeltaExhDP_mmH2O, CIT_degC)
  7. fnCF_ExhaustDP(ExhaustDP_mmH2O)
  8. fnCF_BaroPressure(BaroPressure_mbara, CIT_degC)
  9. fnCF_FuelComp(FuelLHV_kJkg, HC_ratio)
- Expand all CF records into 33 individual columns

**Step 5: Combined Correction Factors**
```powerquery
Combined_OutputCF = CF2 × CF6 × CF10 × CF14 × CF16 × CF20 × CF23 × CF26 × CF30

Combined_HRCF = CF3 × CF7 × CF11 × CF15 × CF17 × CF21 × CF24 × CF27 × CF31

Combined_ExhFlowCF = CF4 × CF8 × CF12 × CF18 × CF28 × CF32
```

**Step 6: Corrected Performance Metrics**
```powerquery
CorrectedOutput_MW = MeasuredOutput_MW / Combined_OutputCF

CorrectedHeatRate (kJ/kWh) = MeasuredHeatRate / Combined_HRCF

CorrectedHeatRate_BTU_kWh = GTHeatRate_BTU_kWh / Combined_HRCF

CorrectedEfficiency = 3412.1416 / CorrectedHeatRate_BTU_kWh

CorrectedExhaustTemp = MeasuredExhaustTemp - 
    (CF5 + CF9 + CF13 + CF19 + CF22 + CF25 + CF29 + CF33)

CorrectedExhaustFlow = MeasuredExhaustFlow / Combined_ExhFlowCF
```

**Step 7: Operational Parameters**
- Evaporative Cooler Effectiveness (Calculated WB):
  ```
  EvapCooler_Eff_Calc_WB = (T_DB - T_CIT) / (T_DB - T_WB_Calc)
  ```
- Evaporative Cooler Effectiveness (H2 Building WB):
  ```
  EvapCooler_Eff_H2_Bldg_WB = (T_DB - T_CIT) / (T_DB - T_WB_H2)
  ```
- Cooler Status:
  - eff = null: "No Data"
  - eff < 0: "Review CIT > DBT"
  - eff > 1: "Subcooling"
  - eff < 0.05: "OFF"
  - else: "ON"
- MMBTU/hr:
  ```
  MMBTU_hr = FuelFlow_kg/s × 2.20462 × LHV_Btu/lbm × 3600 / 1,000,000
  ```
- Compressor Pressure Ratio:
  ```
  Compressor_PR = P_disch_bar / (P_ambient_mbar / 1000)
  ```
- Compressor Isentropic Efficiency:
  ```
  η_c = (PR^0.2857 - 1) / (T2/T1 - 1)
  where 0.2857 = (γ-1)/γ, γ=1.4 (dry air)
  ```

**Step 8: MW Impact per Correction Factor**
- Formula: Gen_MW × (1/CF_OutputRatio - 1)
- Calculated for all 9 output CFs:
  - MW_CIT = Gen_MW × (1/CF2_OutputRatio - 1)
  - MW_RH = Gen_MW × (1/CF6_OutputRatio - 1)
  - MW_SS = Gen_MW × (1/CF10_OutputRatio - 1)
  - MW_FuelTemp = Gen_MW × (1/CF14_OutputRatio - 1)
  - MW_InletDP = Gen_MW × (1/CF16_OutputRatio - 1)
  - MW_ExhPress = Gen_MW × (1/CF20_OutputRatio - 1)
  - MW_ExhDP = Gen_MW × (1/CF23_OutputRatio - 1)
  - MW_BaroPress = Gen_MW × (1/CF26_OutputRatio - 1)
  - MW_FuelComp = Gen_MW × (1/CF30_OutputRatio - 1)
- Interpretation:
  - CF > 1 → negative impact (correction reduces output)
  - CF < 1 → positive impact (correction increases output)

**Step 9: HR Impact per Correction Factor (BTU/kWh)**
- Formula: GTHeatRate_BTU_kWh × (1/CF_HeatRateRatio - 1)
- Calculated for all 9 heat rate CFs:
  - HR_CIT = GTHeatRate_BTU_kWh × (1/CF3_HeatRateRatio - 1)
  - HR_RH = GTHeatRate_BTU_kWh × (1/CF7_HeatRateRatio - 1)
  - HR_SS = GTHeatRate_BTU_kWh × (1/CF11_HeatRateRatio - 1)
  - HR_FuelTemp = GTHeatRate_BTU_kWh × (1/CF15_HeatRateRatio - 1)
  - HR_InletDP = GTHeatRate_BTU_kWh × (1/CF17_HeatRateRatio - 1)
  - HR_ExhPress = GTHeatRate_BTU_kWh × (1/CF21_HeatRateRatio - 1)
  - HR_ExhDP = GTHeatRate_BTU_kWh × (1/CF24_HeatRateRatio - 1)
  - HR_BaroPress = GTHeatRate_BTU_kWh × (1/CF27_HeatRateRatio - 1)
  - HR_FuelComp = GTHeatRate_BTU_kWh × (1/CF31_HeatRateRatio - 1)
- Interpretation:
  - CF > 1 → positive impact (correction increases HR = penalty)
  - CF < 1 → negative impact (correction decreases HR = benefit)

**Step 10: Calculation Methodology Strings**
- Add dynamic formula strings showing the CF calculation breakdown
- Example: "150.5 MW ÷ CF_CIT(1.0234) ÷ CF_RH(0.9987) ÷ ... = 148.2 MW"

**Output Columns** (key columns, 80+ total):
- Timestamps: Date Only, Hour, Time Stamp, GT_Unit
- Measured: GT1X_Gen_MW, FuelFlow_kg_s, MMBTU_hr, MeasuredOutput_MW, MeasuredHeatRate
- Corrected: CorrectedOutput_MW, CorrectedHeatRate, CorrectedEfficiency, CorrectedExhaustTemp, CorrectedExhaustFlow
- Combined CFs: Combined_OutputCF, Combined_HRCF, Combined_ExhFlowCF
- Individual CFs: CF1-CF33 (all 33 correction factors)
- Operational: Calculated_WB_Temp, EvapCooler_Eff_Calc_WB, Cooler_Status, Compressor_PR, Compressor_Efficiency
- Ambient: Ambient_Temp_DB, Rel_Humidity, Ambient_Press
- GT Measurements: GT1X_Comp_Inlet_Temp, GT1X_Turb_Exh_Temp, GT1X_Comp_Disch_Temp, GT1X_Comp_Disch_Press, GT1X_IGV_Position

**Dependencies**:
- Base Data 2026
- Daily_fuel_gas_composition (Excel table)
- fnCF_CIT, fnCF_RH, fnCF_ShaftSpeed, fnCF_FuelTemp, fnCF_InletDP, fnCF_ExhaustPressure, fnCF_ExhaustDP, fnCF_BaroPressure, fnCF_FuelComp

**Used By**: GT_Combined_Summary

---


### 6.5 Reporting Query

#### 6.5.1 GT_Combined_Summary
**Type**: Reporting Query  
**Setting**: Load to Worksheet  
**Purpose**: Consolidate GT1A, GT1B, GT1C performance into a single report with hourly totals

**Data Sources**:
- GT1A_Performance
- GT1B_Performance
- GT1C_Performance

**Processing Steps**:

**Step 1: Column Selection & Standardization**
- Select key columns from each GT performance query (40+ columns)
- Include MW and HR impact columns for all 9 CF groups
- Rename GT-specific columns to generic names:
  - GT1A_Comp_Inlet_Temp → Comp_Inlet_Temp
  - GT1A_Turb_Exh_Temp → Turb_Exh_Temp
  - GT1A_Gen_MW → Gen_MW
  - GT1A_OFWW → OFWW
  - (Same for GT1B, GT1C)

**Step 2: Stack All GT Rows**
- Combine GT1A, GT1B, GT1C rows using Table.Combine
- Result: All three GTs in a single table with unified column names

**Step 3: Add Month Column**
- Format: "MMM-yyyy" (e.g., "Jan-2026")

**Step 4: Add Efficiency Columns**
- GT_Efficiency = 3412.14 / GTHeatRate_BTU_kWh
- Corrected_GT_Efficiency = 3412.14 / CorrectedHeatRate_BTU_kWh
- 3412.14 BTU/kWh = 1 kWh (100% efficiency baseline)

**Step 5: Generate TOTAL Rows**
- Group by (Date Only + Hour)
- For each hour, create one TOTAL row:
  - **Sum**: MMBTU_hr, Gen_MW, CorrectedOutput_MW, MW_* impacts (9 columns)
  - **Null**: All other numeric columns (intensive properties: HR, efficiency, temperatures, pressures, etc.)
  - **GT_Unit**: "TOTAL"
  - **Cooler_Status**: null

**Step 6: Interleave Detail & Total Rows**
- Add sort key:
  - GT1A = 1
  - GT1B = 2
  - GT1C = 3
  - TOTAL = 4
- Sort by: Date Only (asc), Hour (asc), _SortKey (asc)
- Result: GT1A → GT1B → GT1C → TOTAL for each hour

**Step 7: Rename to Display Names**
- Ambient_Temp_DB → "Ambient Temp DB (°C)"
- Rel_Humidity → "Rel Humidity (%)"
- GT_Unit → "Gas Turbine"
- CorrectedOutput_MW → "Corrected Output (MW)"
- MW_CIT → "MW Impact CIT"
- HR_CIT → "HR Impact CIT (BTU/kWh)"
- etc.

**Output Columns** (48 columns):
1. Month
2. Time Stamp
3. Date
4. Hour
5. Ambient Temp DB (°C)
6. Rel Humidity (%)
7. Ambient Press (mbar)
8. Calculated WB Temp (°C)
9. Gas Turbine (GT1A / GT1B / GT1C / TOTAL)
10. Comp Inlet Temp (°C)
11. RH at Comp Inlet (%)
12. Evap Cooler Eff Calc WB
13. Cooler Status
14. Turb Exh Temp (°C)
15. MMBTU/hr
16. Gen MW
17. Heat Rate (BTU/kWh)
18. GT Efficiency
19. Corrected Output (MW)
20. Corrected Heat Rate (BTU/kWh)
21. Corrected GT Efficiency
22. Compressor PR
23. Compressor Efficiency
24. Corrected Exhaust Temp (°C)
25. Corrected Exhaust Flow (kg/s)
26. OFWW
27-35. MW Impact CIT, RH, SS, FuelTemp, InletDP, ExhPress, ExhDP, BaroPress, FuelComp
36-44. HR Impact CIT, RH, SS, FuelTemp, InletDP, ExhPress, ExhDP, BaroPress, FuelComp (BTU/kWh)

**Row Structure Example**:
```
2026-01-15 08:00 | GT1A  | 150.5 MW | 10,250 kJ/kWh | ...
2026-01-15 08:00 | GT1B  | 148.2 MW | 10,310 kJ/kWh | ...
2026-01-15 08:00 | GT1C  | 149.8 MW | 10,280 kJ/kWh | ...
2026-01-15 08:00 | TOTAL | 448.5 MW | null          | ...
```

**Dependencies**:
- GT1A_Performance
- GT1B_Performance
- GT1C_Performance

**Used By**: End users (Excel worksheet)

---

## 7. Usage Patterns

### 7.1 Query Execution Order

When refreshing the entire system, queries execute in this order:

```
1. Base Data 2026 (loads raw data)
   ↓
2. fnInterp1D, fnInterp2D (utility functions, no data)
   ↓
3. fnInterpFuelCurve (depends on fnInterp1D)
   ↓
4. All 9 CF functions (depend on fnInterp1D/2D)
   ↓
5. GT1A_Performance, GT1B_Performance, GT1C_Performance (parallel execution)
   ↓
6. GT_Combined_Summary (depends on all 3 GT performance queries)
```

### 7.2 Common Use Cases

#### Use Case 1: Analyze Single GT Performance
**Objective**: Review corrected performance for GT1A only

**Queries to Refresh**:
1. Base Data 2026
2. GT1A_Performance

**Output**: GT1A_Performance worksheet with 80+ columns including all 33 CFs

---

#### Use Case 2: Compare All Three GTs
**Objective**: Side-by-side comparison of GT1A, GT1B, GT1C

**Queries to Refresh**:
1. Base Data 2026
2. GT1A_Performance
3. GT1B_Performance
4. GT1C_Performance
5. GT_Combined_Summary

**Output**: GT_Combined_Summary worksheet with interleaved GT rows + TOTAL rows

---

#### Use Case 3: Audit Correction Factor Calculations
**Objective**: Verify CF calculations for a specific hour

**Queries to Use**:
- GT1A_Performance (or GT1B/GT1C)

**Columns to Review**:
- Individual CFs: CF1-CF33
- Combined CFs: Combined_OutputCF, Combined_HRCF, Combined_ExhFlowCF
- Calculation formulas: Calc_CorrectedOutput_MW, Calc_CorrectedHeatRate

**Example Audit**:
```
Hour: 2026-01-15 08:00
MeasuredOutput_MW: 150.5
CF2_OutputRatio: 1.0234
CF6_OutputRatio: 0.9987
...
Combined_OutputCF: 1.0156
CorrectedOutput_MW: 150.5 / 1.0156 = 148.2 MW
```

---

#### Use Case 4: Analyze Fuel Composition Impact
**Objective**: Understand how fuel LHV and H/C ratio affect performance

**Queries to Use**:
- GT1A_Performance (or GT1B/GT1C)

**Columns to Review**:
- LHV_Btu_lbm, FuelLHV_kJkg, HC_ratio
- CF30_OutputRatio, CF31_HeatRateRatio, CF32_ExhaustFlowRatio, CF33_ExhaustTempDiff_degC

**Analysis**:
- Compare days with different fuel compositions
- Isolate fuel composition effects from other variables

---

#### Use Case 5: Evaporative Cooler Performance
**Objective**: Evaluate evaporative cooler effectiveness

**Queries to Use**:
- GT1A_Performance (or GT1B/GT1C)

**Columns to Review**:
- Ambient_Temp_DB, Calculated_WB_Temp, H2_Bldg_WB_Temp
- GT1A_Comp_Inlet_Temp
- EvapCooler_Eff_Calc_WB, EvapCooler_Eff_H2_Bldg_WB
- Cooler_Status

**Analysis**:
- Effectiveness typically 0.7-0.9 when ON
- Compare Calculated WB vs H2 Building WB effectiveness
- Identify subcooling conditions (eff > 1.0)

---


### 7.3 Query Modification Patterns

#### Pattern 1: Change Date Filter
**Objective**: Analyze different time periods

**Query to Modify**: GT1A_Performance, GT1B_Performance, GT1C_Performance

**Modification**:
```powerquery
// In each GT*_Performance query, modify these lines:
StartYear  = 2026,    // ◄ Change year
StartMonth = 1,       // ◄ Change month (1-12)
StartDay   = 1,       // ◄ Change day (1-31)
StartFrom  = #date(StartYear, StartMonth, StartDay),
```

**Impact**: Filters rows to only process data from StartFrom date onward

---

#### Pattern 2: Add New Correction Factor
**Objective**: Incorporate a new CF (e.g., CF34 for ambient temperature variation)

**Steps**:
1. Create new CF function query (e.g., fnCF_AmbientTemp)
2. Modify GT*_Performance queries:
   - Add CF function call
   - Expand new CF record
   - Update Combined_OutputCF / Combined_HRCF formulas
   - Add new CF columns to output

**Example**:
```powerquery
// Add CF function call
AddAT = Table.AddColumn(AddFC, "_at",
    each fnCF_AmbientTemp([Ambient_Temp_DB])),

// Expand record
Exp10 = Table.ExpandRecordColumn(Exp9, "_at",
    {"CF34_OutputRatio", "CF35_HeatRateRatio"}),

// Update combined CF
AddCombinedOutputCF = Table.AddColumn(Exp10, "Combined_OutputCF",
    each [CF2_OutputRatio] * ... * [CF30_OutputRatio] * [CF34_OutputRatio]),
```

---

#### Pattern 3: Change Baseline Conditions
**Objective**: Recalibrate CFs to new baseline (e.g., different site elevation)

**Queries to Modify**: All 9 fnCF_* queries

**Modification**:
- Update interpolation tables (xList, yList, zGrid)
- Ensure baseline condition returns CF = 1.0 (ratios) or 0.0 (additive)

**Example** (fnCF_BaroPressure):
```powerquery
// Change baseline from 1005.91 mbar to 1013.25 mbar (sea level)
yBaro = {955.62, 965.67, 975.73, 985.79, 1013.25, 1026.03, ...},
// Update z-tables accordingly so CF = 1.0 at 1013.25 mbar
```

---

#### Pattern 4: Add Operational Parameter
**Objective**: Calculate new derived metric (e.g., turbine efficiency)

**Query to Modify**: GT1A_Performance, GT1B_Performance, GT1C_Performance

**Modification**:
```powerquery
// Add new column after existing operational parameters
AddTurbineEff = Table.AddColumn(AddCompEff, "Turbine_Efficiency",
    each
        let
            T3_K = if [GT1A_Turb_Exh_Temp] = null then null 
                   else [GT1A_Turb_Exh_Temp] + 273.15,
            T2_K = if [GT1A_Comp_Disch_Temp] = null then null 
                   else [GT1A_Comp_Disch_Temp] + 273.15,
            // Turbine efficiency calculation
            η_t = ...
        in
            η_t,
    type number),
```

---

### 7.4 Performance Optimization Tips

#### Tip 1: Minimize Data Volume
- Use date filters early in the query (StartFrom parameter)
- Select only required columns from Base Data 2026
- Avoid loading unnecessary historical data

#### Tip 2: Leverage Query Folding
- Base Data 2026 loads from Excel - limited folding
- Consider migrating to SQL database for better folding
- Use native Excel table filtering where possible

#### Tip 3: Parallel Execution
- GT1A_Performance, GT1B_Performance, GT1C_Performance can execute in parallel
- Power Query automatically parallelizes independent queries
- Ensure sufficient system resources (RAM, CPU cores)

#### Tip 4: Connection-Only Queries
- All utility and CF functions are Connection Only
- They don't load data to worksheets
- Minimal memory footprint

#### Tip 5: Incremental Refresh
- For large datasets, consider incremental refresh
- Load only new/changed data since last refresh
- Requires Power Query incremental refresh setup

---

## 8. Performance Calculations

### 8.1 Key Formulas

#### Measured Heat Rate
```
Method 1 - Mass-based (kJ/kWh):
MeasuredHeatRate (kJ/kWh) = (FuelFlow_kg/s × 2.20462 × LHV_Btu/lbm × 3.6) / Gen_MW

Where:
  FuelFlow_kg/s: Fuel flow rate (kg/s)
  2.20462: Conversion factor (kg → lbm)
  LHV_Btu/lbm: Lower Heating Value (Btu/lbm)
  3.6: Conversion factor (kW → kJ/kWh)
  Gen_MW: Generator output (MW)

Method 2 - Volumetric (BTU/kWh):
GTHeatRate_BTU_kWh = ((FuelFlow_kg/s × 3600 / 0.9987 × 35.31467) × HHV_BTU_SCF / 1.109) / (Gen_MW × 1000)

Where:
  FuelFlow_kg/s: Fuel flow rate (kg/s)
  3600: Seconds per hour
  0.9987: Density correction (kg/s → Nm³/hr)
  35.31467: Conversion factor (Nm³/hr → SCF/hr)
  HHV_BTU_SCF: Gross Calorific Value (BTU/SCF)
  1.109: HHV to LHV conversion factor
  Gen_MW × 1000: Generator output (kW)
```

#### Corrected Output
```
CorrectedOutput_MW = MeasuredOutput_MW / Combined_OutputCF

Where:
  Combined_OutputCF = CF2 × CF6 × CF10 × CF14 × CF16 × CF20 × CF23 × CF26 × CF30
```

#### Corrected Heat Rate
```
CorrectedHeatRate (kJ/kWh) = MeasuredHeatRate / Combined_HRCF

Where:
  Combined_HRCF = CF3 × CF7 × CF11 × CF15 × CF17 × CF21 × CF24 × CF27 × CF31
```

#### Corrected Efficiency
```
CorrectedEfficiency = 3412.1416 / CorrectedHeatRate_BTU_kWh

Where:
  3412.1416: BTU/kWh at 100% efficiency (1 kWh = 3412.1416 BTU)
  CorrectedHeatRate_BTU_kWh: Corrected heat rate in BTU/kWh
  Result is a decimal (e.g., 0.35 = 35% efficiency)
```

#### MW Impact per Correction Factor
```
MW_Impact = Gen_MW × (1/CF_OutputRatio - 1)

Where:
  Gen_MW: Measured generator output (MW)
  CF_OutputRatio: Output correction factor (CF2, CF6, CF10, CF14, CF16, CF20, CF23, CF26, CF30)
  
Interpretation:
  CF > 1 → negative impact (correction reduces output)
  CF < 1 → positive impact (correction increases output)
  
Example: If Gen_MW = 150 MW and CF2_OutputRatio = 1.05
  MW_CIT = 150 × (1/1.05 - 1) = 150 × (-0.0476) = -7.14 MW
  (CIT correction reduces output by 7.14 MW)
```

#### HR Impact per Correction Factor (BTU/kWh)
```
HR_Impact = GTHeatRate_BTU_kWh × (1/CF_HeatRateRatio - 1)

Where:
  GTHeatRate_BTU_kWh: Measured heat rate (BTU/kWh)
  CF_HeatRateRatio: Heat rate correction factor (CF3, CF7, CF11, CF15, CF17, CF21, CF24, CF27, CF31)
  
Interpretation:
  CF > 1 → positive impact (correction increases HR = penalty)
  CF < 1 → negative impact (correction decreases HR = benefit)
  
Example: If GTHeatRate_BTU_kWh = 10,000 and CF3_HeatRateRatio = 0.98
  HR_CIT = 10,000 × (1/0.98 - 1) = 10,000 × 0.0204 = 204 BTU/kWh
  (CIT correction increases HR by 204 BTU/kWh = penalty)
```

#### Corrected Exhaust Temperature
```
CorrectedExhaustTemp (°C) = MeasuredExhaustTemp - Combined_ExhTempDiff

Where:
  Combined_ExhTempDiff = CF5 + CF9 + CF13 + CF19 + CF22 + CF25 + CF29 + CF33
```

#### Corrected Exhaust Flow
```
CorrectedExhaustFlow (kg/s) = MeasuredExhaustFlow / Combined_ExhFlowCF

Where:
  Combined_ExhFlowCF = CF4 × CF8 × CF12 × CF18 × CF28 × CF32
```

#### Estimated Exhaust Flow (GE MS9001E)
```
AirFlow (kg/s) = 410 × (IGV/84) × (P_mbar/1013.25) × √(288.15/T_inlet_K)

ExhaustFlow (kg/s) = AirFlow + FuelFlow

Where:
  410: MS9001E ISO base-load air flow (kg/s) at IGV=84°, 15°C, 1013.25 mbar
  IGV: Inlet Guide Vane position (degrees)
  84: Design IGV angle (degrees)
  P_mbar: Ambient pressure (mbar)
  1013.25: Standard atmospheric pressure (mbar)
  288.15: Standard temperature (K) = 15°C
  T_inlet_K: Compressor inlet temperature (K)
  FuelFlow: Fuel flow rate (kg/s)
```

#### MMBTU/hr
```
MMBTU_hr = FuelFlow_kg/s × 2.20462 × LHV_Btu/lbm × 3600 / 1,000,000

Where:
  FuelFlow_kg/s: Fuel flow rate (kg/s)
  2.20462: Conversion factor (kg → lbm)
  LHV_Btu/lbm: Lower Heating Value (Btu/lbm)
  3600: Seconds per hour
  1,000,000: Million BTU
```

#### Compressor Pressure Ratio
```
Compressor_PR = P_disch_bar / P_ambient_bar

Where:
  P_disch_bar: Compressor discharge pressure (bar)
  P_ambient_bar: Ambient pressure (mbar) / 1000
```

#### Compressor Isentropic Efficiency
```
η_c = (PR^0.2857 - 1) / (T2/T1 - 1)

Where:
  PR: Compressor pressure ratio
  0.2857: (γ-1)/γ, where γ=1.4 for dry air
  T1: Compressor inlet temperature (K)
  T2: Compressor discharge temperature (K)
```

#### Evaporative Cooler Effectiveness
```
EvapCooler_Eff = (T_DB - T_CIT) / (T_DB - T_WB)

Where:
  T_DB: Dry bulb temperature (°C)
  T_CIT: Compressor inlet temperature (°C)
  T_WB: Wet bulb temperature (°C) - either Calculated or H2 Building
```

---

### 8.2 Unit Conversions

| From | To | Factor | Formula |
|------|-----|--------|---------|
| Btu/lbm | kJ/kg | 2.326 | kJ/kg = Btu/lbm × 2.326 |
| kg | lbm | 2.20462 | lbm = kg × 2.20462 |
| in H₂O | mm H₂O | 25.4 | mm H₂O = in H₂O × 25.4 |
| mbar | bar | 0.001 | bar = mbar / 1000 |
| °C | K | +273.15 | K = °C + 273.15 |
| kW | kJ/kWh | 3.6 | kJ/kWh = kW × 3.6 |

---

### 8.3 Typical Value Ranges

| Parameter | Typical Range | Units | Notes |
|-----------|--------------|-------|-------|
| CIT | -10 to 35 | °C | Varies with ambient + evap cooler |
| RH | 10 to 90 | % | Site-dependent |
| Shaft Speed | 2940 to 3060 | RPM | ±2% of rated 3000 RPM |
| Fuel Temp | 10 to 40 | °C | Ambient to slightly heated |
| Inlet DP | 3 to 7 | in H₂O | Filter condition dependent |
| Exhaust DP | 8 to 13 | in H₂O | HRSG condition dependent |
| Baro Pressure | 980 to 1030 | mbar | Site elevation dependent |
| Fuel LHV | 11000 to 14000 | kJ/kg | Natural gas composition |
| H/C Ratio | 3.7 to 4.1 | - | Natural gas composition |
| Gen Output | 100 to 160 | MW | FCBL range per GT |
| Heat Rate | 9500 to 11000 | kJ/kWh | Lower is better |
| Efficiency | 33 to 38 | % | Higher is better |
| Exhaust Temp | 520 to 600 | °C | Load and condition dependent |
| Exhaust Flow | 400 to 450 | kg/s | Per GT |

---

## 9. Appendices

### 9.1 Glossary

| Term | Definition |
|------|------------|
| **CF** | Correction Factor - multiplier or additive adjustment to normalize performance |
| **CIT** | Compressor Inlet Temperature |
| **FCBL** | Full Combined Base Load - operating condition where all 3 GTs run at high load |
| **GT** | Gas Turbine |
| **H/C Ratio** | Hydrogen-to-Carbon ratio in fuel composition |
| **HR** | Heat Rate - energy input per unit electrical output (kJ/kWh) |
| **HRSG** | Heat Recovery Steam Generator |
| **IGV** | Inlet Guide Vane - variable geometry at compressor inlet |
| **LHV** | Lower Heating Value - energy content of fuel (excludes water vapor condensation) |
| **RH** | Relative Humidity |
| **WB** | Wet Bulb Temperature |

### 9.2 Acronyms

| Acronym | Full Form |
|---------|-----------|
| CF | Correction Factor |
| CIT | Compressor Inlet Temperature |
| DP | Differential Pressure |
| FCBL | Full Combined Base Load |
| GT | Gas Turbine |
| HR | Heat Rate |
| HRSG | Heat Recovery Steam Generator |
| IGV | Inlet Guide Vane |
| LHV | Lower Heating Value |
| MW | Megawatt |
| PR | Pressure Ratio |
| RH | Relative Humidity |
| RPM | Revolutions Per Minute |
| WB | Wet Bulb |

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-14 | System | Initial PRD creation |

---

**End of Document**