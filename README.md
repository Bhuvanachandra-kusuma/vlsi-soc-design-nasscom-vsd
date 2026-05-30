# VLSI SoC Design & Planning — NASSCOM VSD Workshop

> **A complete RTL-to-GDSII physical design flow** using open-source EDA tools, the SkyWater Sky130 PDK, and the `picorv32a` RISC-V processor as the target design.

---

## Table of Contents

- [Overview](#overview)
- [Tool & PDK Stack](#tool--pdk-stack)
- [Day 1 — Inception of Open-Source EDA, OpenLane & Sky130 PDK](#day-1--inception-of-open-source-eda-openlane--sky130-pdk)
- [Day 2 — Good Floorplan vs Bad Floorplan & Introduction to Library Cells](#day-2--good-floorplan-vs-bad-floorplan--introduction-to-library-cells)
- [Day 3 — Design Library Cell Using Magic Layout and ngspice Characterization](#day-3--design-library-cell-using-magic-layout-and-ngspice-characterization)
- [Day 4 — Pre-Layout Timing Analysis & Importance of Good Clock Tree](#day-4--pre-layout-timing-analysis--importance-of-good-clock-tree)
- [Day 5 — Final Steps for RTL2GDS Using tritonRoute & OpenSTA](#day-5--final-steps-for-rtl2gds-using-tritonroute--opensta)
- [Key Results Summary](#key-results-summary)
- [Repository Structure](#repository-structure)

---

## Overview

This repository documents the **NASSCOM VSD 5-Day Hands-On Workshop** on Digital VLSI SoC Design and Planning. The workshop covers the complete physical design flow from RTL to GDSII using entirely open-source EDA tooling:

- Synthesis → Floorplan → Placement → CTS → Routing → Sign-off
- Custom CMOS inverter cell (`sky130_vsdinv`) designed from scratch and integrated into `picorv32a`
- DRC violations identified and fixed using Magic VLSI
- Timing analysis performed with OpenROAD's built-in STA engine

---

## Tool & PDK Stack

| Tool | Role |
|------|------|
| **OpenLANE** | RTL-to-GDSII automated flow orchestrator |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, placement, CTS, routing, STA |
| **Magic VLSI** | Layout viewing, LEF extraction, DRC |
| **ngspice** | SPICE-level circuit simulation |
| **Netgen** | LVS (Layout vs. Schematic) |
| **KLayout** | GDS viewing & DRC |
| **SkyWater Sky130** | Open-source 130 nm PDK |

---

## Day 1 — Inception of Open-Source EDA, OpenLane & Sky130 PDK

### Theory: Open-Source Digital ASIC Design

The open-source EDA ecosystem enables a complete ASIC design flow without proprietary tools. The three pillars are:

1. **RTL designs** — open-source IP cores (e.g., `picorv32a`, `ibex`)
2. **EDA Tools** — Yosys, OpenROAD, Magic, ngspice, Netgen
3. **PDK** — SkyWater Sky130 (the first manufacturable open PDK, released in 2020 via Google)

![Open-Source Digital ASIC Design](images/day1/01_open_source_digital_asic_design.png)
*The three enabling pillars of open-source digital ASIC design: open RTL, open EDA tools, and open PDK.*

---

### Theory: Chip Architecture — Die, Core & Pads

A chip consists of the **die** (the silicon area), the **core** (where logic sits), and **I/O pads** arranged around the periphery. The relationship between these defines utilization factor and aspect ratio — two critical floorplan parameters.

- **Core Utilization Factor** = Area of Netlist / Area of Core
- **Aspect Ratio** = Height / Width of core

![Chip Architecture — Die, Core and Pads](images/day1/02_chip_architecture_die_core_pads.png)
*Die, core, and I/O pad arrangement showing how macros and standard cells populate the core area.*

---

### Theory: RTL-to-GDSII Flow Overview

The complete physical design flow transforms an RTL description into a manufacturable GDSII file through these stages:

| Stage | Description |
|-------|-------------|
| **Synthesis** | Maps RTL to gate-level netlist using standard cell library |
| **Floorplan** | Sets die/core area, places I/O pins, adds power rings |
| **Placement** | Places standard cells in legal positions |
| **CTS** | Clock Tree Synthesis — balances clock skew |
| **Routing** | Connects all nets using metal layers |
| **Sign-off** | DRC, LVS, STA verification; export GDSII |

![RTL to GDSII Flow Overview](images/day1/03_rtl_to_gdsii_flow_overview.png)
*End-to-end RTL-to-GDSII flow from HDL source to tape-out-ready GDSII.*

---

### Theory: OpenLANE ASIC Flow Diagram

OpenLANE wraps and orchestrates all open-source tools into a single automated flow. It supports both interactive and fully automated modes and includes built-in design exploration for tuning synthesis strategies, utilization, and timing.

![OpenLane ASIC Flow Diagram](images/day1/04_openlane_asic_flow_diagram.png)
*OpenLANE's internal tool chain: Yosys → OpenROAD → Magic → Netgen, all driven by a Tcl-based flow.*

---

### Lab 1.1 — Design Preparation

The first step is launching the OpenLANE interactive environment and preparing the `picorv32a` design. This merges the standard cell LEF files, reads `config.tcl`, and creates the `runs/` directory structure.

```bash
# Launch OpenLANE Docker container
cd ~/Desktop/work/tools/openlane_working_dir/openlane
docker run -it --rm -v $(pwd):/openLANE_flow -v $PDK_ROOT:$PDK_ROOT \
  -e PDK_ROOT=$PDK_ROOT -u $(id -u $USER):$(id -g $USER) efabless/openlane:v0.21

# Inside the container
./flow.tcl -interactive
package require openlane 0.9

# Prepare the picorv32a design
prep -design picorv32a
```

![Design Preparation Complete](images/day1/05_prep_design_complete.png)
*`prep -design picorv32a` completes successfully — merged LEF created, run directory initialized.*

---

### Lab 1.2 — Synthesis, Floorplan & I/O Pins

After synthesis (`run_synthesis`), the floorplan is generated. OpenLANE places I/O pins equidistantly around the core boundary by default.

```bash
run_synthesis
run_floorplan
```

![Run Floorplan Complete](images/day1/06_run_floorplan_complete.png)
*`run_floorplan` completes — DEF file generated with core area, power straps, and I/O pin placement.*

```bash
# View floorplan in Magic
cd runs/<run>/results/floorplan/
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.lef \
  def read picorv32a.floorplan.def &
```

![Magic — I/O Pins Equidistant](images/day1/07_magic_io_pins_equidistant.png)
*Magic VLSI showing the floorplan with I/O pins evenly distributed around the die boundary.*

---

### Lab 1.3 — Placement

Global placement (`run_placement`) places all standard cells legally inside the core area. The tool optimizes for wire length while respecting density constraints.

```bash
run_placement
```

![Run Placement Complete](images/day1/08_run_placement_complete.png)
*`run_placement` finishes — global placement done, overflow converged, legalization complete.*

---

### Lab 1.4 — Flop Ratio Calculation

An important synthesis metric is the **flop ratio** — the proportion of D flip-flops in the design.

```
Flop Ratio = Number of DFFs / Total Number of Cells
           = 1613 / 14876
           = 0.1084   (10.84%)
```

![Flop Ratio — Decimal 0.108](images/day1/09_flop_ratio_decimal_0108.png)
*Synthesis report showing raw DFF and cell counts used to compute the flop ratio.*

![Flop Ratio — 10.84 Percent](images/day1/10_flop_ratio_percentage_1084.png)
*Calculated flop ratio of 10.84% confirms the design is logic-dominated (not flip-flop dominated).*

---

### Lab 1.5 — Placement Views in Magic

After placement, the layout is inspected in Magic to verify standard cell rows and pin accessibility.

![Magic — Placement Full View](images/day1/11_magic_placement_full_view.png)
*Full chip view post-placement showing standard cell rows inside the core and I/O pins on the boundary.*

![Magic — Placement Zoomed](images/day1/12_magic_placement_zoomed.png)
*Zoomed view of the core area showing individual standard cells placed in rows.*

![Placement Report Terminal](images/day1/13_placement_report_terminal.png)
*OpenLANE terminal output showing placement metrics: HPWL, overflow, and cell density.*

![Placement DEF Listing](images/day1/14_placement_def_listing.png)
*DEF file listing confirming component placement coordinates for all standard cells.*

![Magic — Placement Full Chip](images/day1/15_magic_placement_full_chip.png)
*Complete chip floorplan with all standard cells placed; power rails visible as horizontal stripes.*

---

### Lab 1.6 — Placement Statistics

OpenLANE generates detailed placement statistics including pin counts, cell area, and utilization.

![Placement Statistics Table](images/day1/16_placement_statistics_table.png)
*Placement statistics: total cells, area, utilization factor, and HPWL summary.*

![Placement Magic Full View 2](images/day1/17_placement_magic_full_view2.png)
*Alternative Magic view of the completed placement showing the full die with all elements.*

![Placement Statistics Table 2](images/day1/18_placement_statistics_table2.png)
*Extended statistics table with per-layer wire length estimates and density breakdown.*

---

## Day 2 — Good Floorplan vs Bad Floorplan & Introduction to Library Cells

### Theory: Floorplanning Concepts

A **good floorplan** minimizes wire length, avoids congestion, and satisfies timing. Key parameters:

- **Utilization Factor**: 50–60% is typical for production designs (leaves room for routing)
- **Aspect Ratio**: 1.0 (square) is ideal; deviations can cause congestion hotspots
- **Pre-placed cells**: Memories, PLLs, and analog blocks placed before standard cells
- **Decoupling capacitors**: Added near switching blocks to stabilize VDD/VSS

### Theory: Library Cells & Timing Characterization

Standard cells in the Sky130 PDK are characterized for:
- **Propagation delay** (50% input → 50% output)
- **Rise/Fall time** (20%→80% transitions)
- **Setup/Hold time** for sequential elements
- **Leakage & dynamic power**

### Lab 2.1 — Custom Inverter Cell Exploration in Magic

The `sky130_vsdinv` custom inverter is opened in Magic to examine its layout structure. PMOS (on top, connected to VDD) and NMOS (on bottom, connected to VSS) are identified.

![Magic — Inverter PMOS Identified](images/day2/01_magic_inverter_pmos_identified.png)
*Magic layout view with the PMOS transistor highlighted — note connection to VDD (top metal rail).*

![Magic — Inverter NMOS Identified](images/day2/02_magic_inverter_nmos_identified.png)
*NMOS transistor of the inverter highlighted — connects to VSS (bottom metal rail).*

![Magic — Inverter Output Pin Y](images/day2/03_magic_inverter_output_pin_y.png)
*Output port `Y` of the inverter identified in Magic — metal1 connection to the drain of both transistors.*

![Magic — Inverter Full Layout](images/day2/04_magic_inverter_full_layout.png)
*Full inverter layout showing PMOS, NMOS, input port A, output port Y, VDD, and VSS connections.*

---

### Lab 2.2 — ngspice SPICE Simulation

The inverter's SPICE netlist is extracted from Magic and simulated with ngspice to characterize its timing behavior.

```bash
# In Magic tkcon
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

```bash
# Run ngspice simulation
ngspice sky130_vsdinv.spice
```

![ngspice Simulation Launched](images/day2/05_ngspice_simulation_launched.png)
*ngspice launched with the extracted SPICE netlist — models loaded from Sky130 PDK.*

![ngspice Waveform Plot](images/day2/06_ngspice_waveform_plot.png)
*Transient simulation waveforms: input (blue) and output (red) showing clean inverter switching behavior.*

---

### Lab 2.3 — Post-Synthesis Placement View

After synthesis, the initial placement can be viewed in Magic to get a sense of cell density.

![Magic — Placement View Post-Synthesis](images/day2/07_magic_placement_view_post_synth.png)
*Magic layout view immediately after synthesis — cells are unplaced (stacked at origin), showing pre-placement state.*

---

### Lab 2.4 — Floorplan Exploration in Magic

The generated floorplan DEF is loaded into Magic to inspect I/O placement and standard cell rows.

![Magic — Blank Floorplan](images/day2/08_magic_blank_floorplan.png)
*Empty core area after floorplan — I/O pins placed on boundary, core interior empty before placement.*

![Magic — Floorplan Standard Cell Rows](images/day2/09_magic_floorplan_stdcell_rows.png)
*Standard cell rows (site rows) visible inside the core — horizontal tracks where cells will be placed.*

![Magic — Floorplan I/O Detail](images/day2/10_magic_floorplan_io_detail.png)
*Zoomed view of the I/O ring showing pin metal connections at the die boundary.*

![Magic — Floorplan Cell Detail](images/day2/11_magic_floorplan_cell_detail.png)
*Close-up of the floorplan showing well-tap cells and decap cells pre-placed in the core.*

---

### Lab 2.5 — Timing Characterization Measurements

Critical timing parameters are measured from the ngspice transient waveforms.

**Rise Time** = time for output to rise from 20% to 80% of VDD

![Rise Time 80% Measurement](images/day2/12_rise_time_80pct_measurement.png)
*Ngspice waveform cursor measurement for rise time: 80% VDD crossing point identified.*

**Fall Time** = time for output to fall from 80% to 20% of VDD

![Fall Time Measurement](images/day2/13_fall_time_measurement.png)
*Ngspice waveform cursor measurement for fall time: 20% VDD crossing point identified.*

**Propagation Delay** = time from 50% input to 50% output transition

![Propagation Delay Measurement](images/day2/14_propagation_delay_measurement.png)
*Propagation delay measured at 50% VDD crossing for both input and output signals.*

---

### Lab 2.6 — Timing Values Summary

| Parameter | Value |
|-----------|-------|
| Rise Time (20%→80%) | ~0.063 ns |
| Fall Time (80%→20%) | ~0.042 ns |
| Propagation Delay (rise) | ~0.060 ns |
| Propagation Delay (fall) | ~0.027 ns |

![Timing Characterization Values](images/day2/15_timing_characterization_values.png)
*Summary of all four key timing characterization values extracted from the ngspice simulation.*

![Timing Values Zoomed](images/day2/16_timing_values_zoomed.png)
*Zoomed waveform view showing precise cursor placement for timing measurements.*

---

### Lab 2.7 — Memory Cell & Advanced Layout Views

![Magic — Memory Read Cell View](images/day2/17_magic_mem_rd_cell_view.png)
*Magic view of a memory read cell from the Sky130 standard cell library — shows complex multi-transistor layout.*

---

### Lab 2.8 — Placement Completion & Routing Views

After placement, OpenLANE reports completion statistics and the layout can be inspected.

![Placement Completion Log](images/day2/18_placement_completion_log.png)
*OpenLANE terminal log confirming placement completion with HPWL and overflow metrics.*

![Magic — Routing Wire View](images/day2/19_magic_routing_wire_view.png)
*Magic layout showing global routing wires — metal1 and metal2 connections between cells visible.*

![Magic — Routed Chip Full](images/day2/20_magic_routed_chip_full.png)
*Full chip view post-routing — metal layers form a dense interconnect mesh across the core.*

![Placement Stats Table](images/day2/21_placement_stats_table.png)
*Detailed placement statistics table: cell count, total area, utilization, and per-metal wire lengths.*

---

### Lab 2.9 — Standard Cell Close-Up Views

![Magic — Zoomed Standard Cells](images/day2/22_magic_zoomed_stdcells.png)
*Zoomed Magic view showing individual standard cells with poly, diffusion, and metal1 connections.*

![Magic — Standard Cell Close View](images/day2/23_magic_stdcell_close_view.png)
*Very close view of a cluster of standard cells showing gate-level detail and abutting cell boundaries.*

---

### Lab 2.10 — Custom Cell Integration into Synthesis

The `sky130_vsdinv` custom inverter is integrated into the `picorv32a` synthesis flow. This requires updating `config.tcl` with the extra LEF path and synthesis strategy.

```tcl
# In config.tcl
set ::env(EXTRA_LEFS) [glob $::env(DESIGN_DIR)/src/*.lef]
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
```

```bash
# In OpenLANE interactive
prep -design picorv32a -tag <run_name> -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
run_synthesis
```

![Synthesis with Custom Cell Run](images/day2/24_synthesis_with_custom_cell_run.png)
*OpenLANE synthesis run initiated with the custom `sky130_vsdinv` LEF included via `EXTRA_LEFS`.*

![Synthesis Custom Cell Log](images/day2/25_synthesis_custom_cell_log.png)
*Synthesis log showing the custom inverter cell being processed by Yosys alongside standard cells.*

![Synthesis Custom Cell Complete](images/day2/26_synthesis_custom_cell_complete.png)
*Synthesis completes successfully — `sky130_vsdinv` confirmed present in the synthesized netlist.*

---

## Day 3 — Design Library Cell Using Magic Layout and ngspice Characterization

### Theory: CMOS Inverter Design

The CMOS inverter is the foundational cell in digital design. It consists of a complementary pair:
- **PMOS** pull-up network (connected to VDD)
- **NMOS** pull-down network (connected to VSS)

When input is HIGH → NMOS ON, PMOS OFF → Output LOW  
When input is LOW → PMOS ON, NMOS OFF → Output HIGH

The `sky130_vsdinv` cell follows the Sky130 design rules: 130 nm minimum gate length, specific layer usage (li1, metal1, metal2), and standard cell height of 2.72 µm.

### Lab 3.1 — Inverter Layout in Magic

The custom inverter is cloned from the reference repository and opened in Magic for detailed inspection.

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
cd vsdstdcelldesign
magic -T sky130A.tech sky130_vsdinv.mag &
```

![Inverter Layout Magic Open](images/day3/01_inverter_layout_magic_open.png)
*Magic opens the `sky130_vsdinv.mag` file — full inverter layout with all layers visible.*

![Inverter Layout Port View](images/day3/02_inverter_layout_port_view.png)
*Port annotations visible on the layout — A (input), Y (output), VDD, and VSS labeled.*

![Inverter Layout with TKcon](images/day3/03_inverter_layout_with_tkcon.png)
*Magic layout window alongside the TKcon (Tcl console) used for entering Magic commands.*

---

### Lab 3.2 — Defining Ports on the Inverter

Ports must be explicitly defined on the layout before LEF extraction. This tells Magic which geometry represents each pin.

```tcl
# In TKcon — select the A input polygon, then:
port make
port class input
port use signal

# Select the Y output polygon, then:
port make
port class output
port use signal
```

![Inverter Port A Defined](images/day3/04_inverter_port_a_defined.png)
*Port `A` (input) defined on the poly gate connection — port class set to `input`.*

![Inverter Port Y Defined](images/day3/05_inverter_port_y_defined.png)
*Port `Y` (output) defined on the metal1 drain connection — port class set to `output`.*

---

### Theory: CMOS Inverter — Hand-Drawn Schematic

Understanding the transistor-level schematic reinforces the relationship between the layout geometry and electrical behavior.

![CMOS Inverter Schematic Hand-Drawn](images/day3/06_cmos_inverter_schematic_handdrawn.png)
*Hand-drawn CMOS inverter schematic showing PMOS (top) and NMOS (bottom) with input A and output Y.*

---

### Lab 3.3 — Custom Cell in Synthesis Netlist

After integration, the synthesized Verilog netlist is checked to confirm `sky130_vsdinv` appears.

```bash
grep sky130_vsdinv runs/<run>/results/synthesis/picorv32a.synthesis.v
```

![Synthesis Custom Cell Included](images/day3/07_synthesis_custom_cell_included.png)
*`grep` confirms `sky130_vsdinv` is instantiated in the post-synthesis netlist — integration successful.*

---

### Lab 3.4 — SPICE Netlist Editing for Simulation

The extracted SPICE netlist is manually edited to add stimulus (pulse source) and include the Sky130 SPICE models for accurate simulation.

```spice
* sky130_vsdinv.spice
.include ./libs/pshort.lib
.include ./libs/nshort.lib

M1 Y A VDD VDD pshort_model.0 w=3.5 l=0.25
M2 Y A VSS VSS nshort_model.0 w=1 l=0.25

C1 A VSS 0.075f
C2 Y VSS 2f

VDD VDD 0 3.3
VSS VSS 0 0
Va A VSS PULSE(0 3.3 0 0.1ns 0.1ns 2ns 4ns)

.tran 1n 20n
.control
run
.endc
.end
```

![SPICE Netlist Edited for Simulation](images/day3/08_spice_netlist_edited_for_sim.png)
*Edited SPICE file showing stimulus definition, model includes, and transistor sizing.*

---

### Lab 3.5 — Post-Config Synthesis Run

After updating `config.tcl` with the custom cell settings, synthesis is re-run to pick up the new configuration.

```tcl
set ::env(FP_CORE_UTIL) 35
set ::env(PL_TARGET_DENSITY) 0.4
```

![Synthesis Run Post Config](images/day3/09_synthesis_run_post_config.png)
*Synthesis run with updated `config.tcl` — reduced utilization (35%) to aid placement and routing.*

---

### Lab 3.6 — Synthesis Timing Reports

After synthesis with the custom cell, timing reports show slack values for the design.

![Synthesis Slack Report](images/day3/10_synthesis_slack_report.png)
*Post-synthesis timing report: worst negative slack (WNS) and total negative slack (TNS) shown.*

![Synthesis TNS WNS Values](images/day3/11_synthesis_tns_wns_values.png)
*Close-up of TNS and WNS values from the synthesis timing report — basis for timing closure decisions.*

---

### Lab 3.7 — DRC Violation Detection in Magic

Magic's DRC engine is used to check Sky130 design rules. Violations are highlighted interactively.

```tcl
# In Magic TKcon
drc check
drc why
```

![Magic — met3 DRC Violation](images/day3/12_magic_met3_drc_violation.png)
*Magic DRC flagging a metal3 spacing violation on a test layout — white dots indicate DRC error regions.*

![Magic — DRC Violation Highlighted](images/day3/13_magic_drc_violation_highlighted.png)
*DRC error region highlighted in Magic — the violation details are shown in the TKcon window.*

---

### Lab 3.8 — DRC Fix via Tech File Edit (poly.9 Rule)

The `poly.9` spacing rule violation requires editing the `sky130A.tech` file to add the missing rule check.

```bash
# Edit the tech file
vi sky130A.tech
# Add poly.9 spacing rule for poly-to-poly contact spacing
```

![Sky130A Tech Poly Rules](images/day3/14_sky130A_tech_poly_rules.png)
*`sky130A.tech` file showing the poly spacing rules section where `poly.9` is defined.*

![Magic — DRC Cleared After Fix](images/day3/15_magic_drc_cleared_after_fix.png)
*After tech file edit and `drc check` re-run — DRC violations cleared, layout is clean.*

---

### Lab 3.9 — Advanced DRC Rules (cifout / nwell)

Additional DRC rules for `cifout` (CIF output layers) and nwell are inspected in the tech file.

![Sky130A Tech cifout nwell Rules](images/day3/16_sky130A_tech_cifout_nwell_rules.png)
*Tech file section showing `cifout` and nwell-related DRC rules for the Sky130A process.*

---

### Lab 3.10 — Port Class Assignment for LEF Export

Before generating the LEF file, all ports must have correct `port class` and `port use` attributes set.

```tcl
# Set port classes for LEF extraction
port A class input
port Y class output  
port VDD class inout
port VSS class inout

port A use signal
port Y use signal
port VDD use power
port VSS use ground
```

![Inverter Layout Port Class Set](images/day3/17_inverter_layout_port_class_set.png)
*TKcon showing port class and use assignments for all four ports — required for correct LEF generation.*

---

## Day 4 — Pre-Layout Timing Analysis & Importance of Good Clock Tree

### Theory: Pre-Layout STA & Clock Tree Synthesis

**Static Timing Analysis (STA)** verifies that all flip-flop setup and hold time constraints are met:

- **Setup slack** = Data Required Time − Data Arrival Time (must be ≥ 0)
- **Hold slack** = Data Arrival Time − Data Required Time (must be ≥ 0)

**Clock Tree Synthesis (CTS)** builds a balanced clock network:
- Minimizes **clock skew** (difference in clock arrival times between flip-flops)
- Minimizes **clock latency** (total clock path delay)
- Uses buffer insertion and wire sizing

### Theory: LEF File & Track Grid Alignment

Before a custom cell can be placed by the tool, its pins must align to the **routing track grid**. For Sky130:
- `li1` horizontal tracks: pitch = 0.46 µm, offset = 0.23 µm
- `li1` vertical tracks: pitch = 0.34 µm, offset = 0.17 µm

### Lab 4.1 — Port Class Set for LEF Export (Final Check)

Final verification that all port classes and uses are correctly set before LEF extraction.

![Inverter Layout Port Class Set](images/day4/01_inverter_layout_port_class_set.png)
*Final port class check in Magic TKcon — all four ports (A, Y, VDD, VSS) correctly classified.*

---

### Lab 4.2 — Pins on Track Intersection Verification

The input and output pins of the inverter must lie on the intersection of horizontal and vertical routing tracks.

```tcl
# In Magic TKcon — load track info
grid 0.46um 0.34um 0.23um 0.17um
```

![Inverter Pins on Track Intersection](images/day4/02_inverter_pins_on_track_intersection.png)
*Magic grid overlay showing that ports A and Y align to li1 track intersections — placement rule satisfied.*

---

### Lab 4.3 — vsdstdcelldesign Directory & LEF File

After LEF extraction (`lef write`), the output LEF file is placed in the design's `src` directory.

```tcl
# Extract LEF
lef write sky130_vsdinv.lef
```

![vsdstdcelldesign Directory LEF](images/day4/03_vsdstdcelldesign_directory_lef.png)
*Directory listing showing `sky130_vsdinv.lef` generated — ready to be copied into the `picorv32a/src/` folder.*

---

### Lab 4.4 — Pre-CTS STA: Hold Slack Analysis

Before running CTS, OpenROAD's STA engine is used to check timing. Hold violations post-placement are analyzed.

```bash
# In OpenLANE interactive
run_placement
openroad
read_lef /openLANE_flow/designs/picorv32a/runs/<run>/tmp/merged.lef
read_def /openLANE_flow/designs/picorv32a/runs/<run>/results/placement/picorv32a.placement.def
read_liberty -max $::env(LIB_SLOWEST)
read_liberty -min $::env(LIB_FASTEST)
read_sdc /openLANE_flow/designs/picorv32a/src/my_base.sdc
check_setup -verbose
report_checks -path_delay min -fields {slew trans net cap input_pin} -format full_clock_expanded
```

![STA Hold Slack Report](images/day4/04_sta_hold_slack_report.png)
*OpenROAD STA output showing hold slack report — all hold paths must be positive for correct operation.*

---

### Lab 4.5 — Clock Tree Synthesis with OpenROAD

CTS is run using OpenROAD's `TritonCTS` engine, which builds a clock tree meeting the specified skew target.

```bash
run_cts
```

![Run CTS with OpenROAD Magic](images/day4/05_run_cts_with_openroad_magic.png)
*`run_cts` executing — TritonCTS building the clock tree; Magic opens automatically to show the result.*

![Magic — Routed Chip Post CTS](images/day4/06_magic_routed_chip_post_cts.png)
*Magic layout after CTS — clock buffers inserted throughout the core, visible as additional cells.*

---

### Lab 4.6 — Cell List Post CTS

After CTS, the cell list is queried to confirm that clock buffers have been added by TritonCTS.

```bash
# In OpenROAD after CTS
write_db pdn.db
read_db pdn.db
report_checks -path_delay min_max -fields {slew trans net cap input_pin} -format full_clock_expanded
```

![Cell List Post CTS Terminal](images/day4/07_cell_list_post_cts_terminal.png)
*Terminal listing of cells post-CTS — `clkbuf_*` cells added by TritonCTS are visible in the cell list.*

---

### Lab 4.7 — Magic Floorplan View Post CTS

The layout is inspected in Magic after CTS to visualize the clock buffer placement.

![Magic — Floorplan Post CTS](images/day4/08_magic_floorplan_post_cts.png)
*Magic view post-CTS showing the core with clock buffers placed throughout — clock network physically realized.*

---

### Lab 4.8 — Post-CTS Timing Analysis

OpenROAD STA is run again after CTS with the actual clock tree network for accurate timing analysis.

```bash
# In OpenROAD
read_lef /openLANE_flow/designs/picorv32a/runs/<run>/tmp/merged.lef
read_def /openLANE_flow/designs/picorv32a/runs/<run>/results/cts/picorv32a.cts.def
read_liberty -max $::env(LIB_SLOWEST)
read_liberty -min $::env(LIB_FASTEST)
read_sdc /openLANE_flow/designs/picorv32a/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pin} -format full_clock_expanded
```

![OpenROAD Post CTS Timing](images/day4/09_openroad_post_cts_timing.png)
*OpenROAD timing report post-CTS with propagated clocks — both setup and hold paths analyzed.*

![STA Post CTS Slack Analysis](images/day4/10_sta_post_cts_slack_analysis.png)
*Post-CTS slack analysis: setup slack (WNS) and hold slack values confirming timing closure status.*

---

## Day 5 — Final Steps for RTL2GDS Using tritonRoute & OpenSTA

### Theory: Routing

**TritonRoute** (OpenROAD's detailed router) performs:
1. **Global routing** — coarse assignment of nets to routing regions
2. **Detailed routing** — precise wire placement following DRC rules
3. **Via insertion** — inter-layer connections

Sky130 uses 6 metal layers (li1, met1–met5) with specific pitch and width rules.

### Theory: Power Distribution Network (PDN)

Before routing signal nets, the PDN (power grid) is built:
- Power rings around the core (VDD and VSS)
- Power straps across the core (horizontal and vertical)
- Standard cell rails connected to straps via vias

### Lab 5.1 — Post-CTS Timing Baseline (Pre-Routing)

Before routing, OpenROAD STA confirms the timing baseline carried forward from CTS.

![OpenROAD Post CTS Timing](images/day5/01_openroad_post_cts_timing.png)
*Pre-routing timing report — establishes the timing baseline before detailed routing adds wire parasitics.*

---

### Lab 5.2 — Run Routing

Routing is initiated with TritonRoute. This is the most compute-intensive step and generates the final routed DEF.

```bash
run_routing
```

![Run Routing Complete Log](images/day5/02_run_routing_complete_log.png)
*`run_routing` completion log — TritonRoute finishes with 0 DRC violations; total wire length and via count reported.*

---

### Lab 5.3 — Final Routed Layout in Magic

The routed DEF is loaded into Magic for visual inspection and sign-off verification.

```bash
cd runs/<run>/results/routing/
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.lef \
  def read picorv32a.def &
```

![Magic — Routed Chip Full View](images/day5/03_magic_routed_chip_full_view.png)
*Full chip view post-routing — all signal nets routed across metal layers 1–5; dense interconnect mesh visible.*

![Magic — Routed Placement Zoomed](images/day5/04_magic_routed_placement_zoomed.png)
*Zoomed view of the routed chip showing metal routing between standard cells with via stacks for layer changes.*

---

### Lab 5.4 — Filler Cells & Custom Cell Verification

After routing, filler cells are inserted to maintain well continuity. The custom inverter `sky130_vsdinv` is verified to be present in the final layout.

![Magic — Routed Detail Filler Cells](images/day5/05_magic_routed_detail_filler_cells.png)
*Detail view showing filler cells (`sky130_fd_sc_hd__fill_*`) inserted between standard cells for N-well continuity.*

![Magic — Routed Custom Cell Placed](images/day5/06_magic_routed_custom_cell_placed.png)
*The custom `sky130_vsdinv` cell located in the final routed layout — confirms full end-to-end integration.*

---

### Lab 5.5 — Final GDSII Layout

The complete `picorv32a` RISC-V core has been successfully taken from RTL all the way to a routed, DRC-clean GDSII layout.

![Magic — Routed Final Layout](images/day5/07_magic_routed_final_layout.png)
*Final routed layout of `picorv32a` viewed in Magic — the complete RTL-to-GDSII flow is demonstrated successfully.*

---

## Key Results Summary

| Metric | Value |
|--------|-------|
| **Design** | `picorv32a` (RISC-V RV32IMC) |
| **PDK** | SkyWater Sky130 (130 nm) |
| **Custom Cell** | `sky130_vsdinv` integrated ✅ |
| **Total Cells (post-synthesis)** | ~14,876 |
| **Flop Ratio** | 10.84% |
| **Core Utilization** | 35% |
| **Target Placement Density** | 0.40 |
| **Routing DRC Violations** | 0 ✅ |
| **CTS Clock Skew** | < 0.5 ns |
| **Final Output** | GDSII + routed DEF ✅ |

### Configuration (`config.tcl` key settings)

```tcl
set ::env(DESIGN_NAME) "picorv32a"
set ::env(EXTRA_LEFS) [glob $::env(DESIGN_DIR)/src/*.lef]
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
set ::env(FP_CORE_UTIL) 35
set ::env(PL_TARGET_DENSITY) 0.4
```

---

## Repository Structure

```
vlsi-soc-design-nasscom-vsd/
├── README.md
└── images/
    ├── day1/       # 18 images — EDA concepts, floorplan, placement
    ├── day2/       # 26 images — Library cells, ngspice, custom cell integration
    ├── day3/       # 17 images — Magic layout, DRC, SPICE simulation
    ├── day4/       # 10 images — STA, CTS, post-CTS timing
    └── day5/       #  7 images — Routing, final GDSII layout
```

> **Total: 78 lab screenshots** documenting the complete RTL-to-GDSII flow.

---

## Acknowledgements

- **NASSCOM & VSD** — for the workshop content and hands-on lab structure
- **efabless & Google** — for the OpenLANE flow and SkyWater Sky130 PDK
- **Nickson Jose** — for the `vsdstdcelldesign` reference repository
- **Timothy Edwards** — for Magic VLSI Layout Tool

---

*Workshop completed May 2026 | Tools: OpenLANE v0.21 · Magic 8.3 · ngspice-37 · Sky130A PDK*
