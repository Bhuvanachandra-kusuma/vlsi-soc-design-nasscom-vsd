# Digital VLSI SoC Design and Planning — RTL to GDSII

> A 5-day hands-on workshop on the complete RTL-to-GDSII flow for digital VLSI SoC design,
> organised by **VSD (VLSI System Design)** in collaboration with **NASSCOM**.
> This repository documents my learning, lab outputs, and key takeaways from each day.

---

## Table of Contents

- [Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK](#day-1--inception-of-open-source-eda-openlane--sky130-pdk)
- [Day 2 — Floorplanning and Introduction to Library Cells](#day-2--floorplanning-and-introduction-to-library-cells)
- [Day 3 — Design and Characterisation of Library Cells using Magic & ngspice](#day-3--design-and-characterisation-of-library-cells-using-magic--ngspice)
- [Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis](#day-4--pre-layout-timing-analysis-and-clock-tree-synthesis)
- [Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA](#day-5--final-rtl-to-gdsii-using-tritonroute--opensta)

---

## Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK

### Theory

#### Understanding the Chip Package

When we look at any embedded board and point to what we call the "chip," we are actually looking at the **package** — a protective casing around the actual silicon die. The real chip sits in the centre of this package and communicates with the outside world via **wire bonding** — tiny wires that connect the chip's pads to the package pins.

#### Inside the Chip: Core, Pads, and Die

Zooming into the chip itself, all signals between the chip and the external world pass through **pads** placed around the periphery. The region enclosed by the pads is the **core** — this is where all the actual digital logic lives. Together, the core and pads form the **die**, which is the fundamental unit of chip manufacturing.

- **Foundry** — the place where chips are physically manufactured
- **Foundry IPs** — IP blocks that require specialised process knowledge (e.g., PLLs, SRAMs)
- **Macros** — reusable, purely digital logic blocks

![Chip Architecture](images/day1/02_chip_architecture_die_core_pads.png)
*Chip showing Die, Core, Pads, Foundry IPs and Macros*

#### From Software to Silicon — The ISA Bridge

A C program running on a chip goes through a multi-layer transformation:

1. The C code is compiled into **RISC-V assembly** (or another ISA)
2. The assembler converts it to **binary machine code (0s and 1s)**
3. This binary pattern needs an **RTL implementation** of the ISA
4. The RTL gets synthesised and goes through the full **PnR (Place and Route)** flow to become a physical layout

The system software stack (OS → Compiler → Assembler) acts as the bridge between what the programmer writes and what the hardware executes.

![RTL to GDSII Flow](images/day1/03_rtl_to_gdsii_flow_overview.png)
*Complete RTL to GDSII flow — software to silicon*

#### Why Open-Source EDA Matters

For a fully open-source ASIC design flow, three things are needed:

1. **RTL Designs** (e.g., from opencores.org)
2. **EDA Tools** (synthesis, P&R, verification)
3. **PDK Data** (process-specific design rules, standard cell libraries)

Historically, PDKs were proprietary and distributed only under NDAs. This changed in **June 2020**, when Google collaborated with SkyWater Technology to release the **Sky130 PDK** — the world's first open-source process design kit.

![Open Source Digital ASIC Design](images/day1/01_open_source_digital_asic_design.png)
*The open-source ASIC design ecosystem: RTL + EDA Tools + PDK*

#### OpenLANE and the RTL to GDSII Flow

**OpenLANE** is an open-source automated flow that takes an RTL netlist all the way to a GDSII layout file. It orchestrates multiple EDA tools:

| Stage | Tool(s) Used |
|---|---|
| Synthesis | Yosys, ABC |
| Floorplan & PDN | OpenROAD |
| Placement | OpenROAD |
| CTS | TritonCTS |
| Routing | FastRoute, TritonRoute |
| SPEF Extraction | OpenRCX |
| GDS Streaming | Magic, KLayout |
| Timing Analysis | OpenSTA |
| DRC & LVS | Magic, Netgen |

![OpenLANE ASIC Flow](images/day1/04_openlane_asic_flow_diagram.png)
*OpenLANE ASIC flow diagram showing all stages from RTL to GDSII*

---

### Lab — Running OpenLANE for `picorv32a`

#### Setting Up and Invoking OpenLANE

Navigate to the OpenLANE working directory and launch the flow in **interactive mode**:

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
./flow.tcl -interactive
package require openlane 0.9
```

#### Preparing the Design

Before synthesis, prepare the design to merge cell LEF and technology LEF files:

```bash
prep -design picorv32a
```

![Prep Design Complete](images/day1/05_prep_design_complete.png)
*`prep -design picorv32a` completing — merged LEF and run directory created*

#### Running Synthesis

```bash
run_synthesis
```

After synthesis completes, we calculate the **Flop Ratio** — the ratio of D flip-flops to the total number of cells:

```
Flop Ratio  =  Number of D Flip-Flops / Total Number of Cells
            =  1613 / 14876
            =  0.1084  →  10.84%
```

![Flop Ratio Decimal](images/day1/09_flop_ratio_decimal_0108.png)
*Calculator showing flop ratio: 0.1084296853993009*

![Flop Ratio Percentage](images/day1/10_flop_ratio_percentage_1084.png)
*Flop ratio as a percentage: 10.84%*

---

## Day 2 — Floorplanning and Introduction to Library Cells

### Theory

#### Chip Floorplanning — Utilisation Factor and Aspect Ratio

Floorplanning defines the physical organisation of the chip. Two fundamental parameters govern it:

- **Utilisation Factor** = Area occupied by Netlist / Total Core Area
  - A value of 0.5–0.6 is typical — leaving room for buffers, clock tree, and routing
- **Aspect Ratio** = Height / Width of the core
  - A ratio of 1 gives a square die; anything else gives a rectangle

#### Pre-Placed Cells and Decoupling Capacitors

**Pre-placed cells** — such as memories, PLLs, and complex IP blocks — are fixed in position before automated placement runs. Their location is decided manually based on connectivity.

**Decoupling capacitors** are placed around pre-placed cells to act as local charge reservoirs, compensating for voltage drops due to switching activity and ensuring clean power delivery.

#### Power Planning — Mesh and Rings

A robust power grid uses **power rings** around the core and a **power mesh** across the chip. Multiple VDD and VSS stripes on both metal layers ensure every standard cell has a nearby power tap — minimising IR drop and electromigration risk.

#### Pin Placement

Input and output pins are placed along the chip boundary. The placement is guided by connectivity — a pin that drives logic deep in the core should be physically closer to that logic. The area between the core and die boundary is blocked from automated cell placement to reserve space for pin buffers and ESD protection cells.

---

### Lab — Floorplan

#### Running Floorplan

```bash
run_floorplan
```

![Run Floorplan Complete](images/day1/06_run_floorplan_complete.png)
*Floorplan completing — DEF file generated in results/floorplan/*

After completion, inspect the generated DEF file:

```bash
cd results/floorplan/
less picorv32a.floorplan.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.floorplan.def &
```

![Magic IO Pins Equidistant](images/day1/07_magic_io_pins_equidistant.png)
*Magic — IO pins placed equidistantly along the die boundary*

---

### Lab — Placement

#### Running Placement

```bash
run_placement
```

Placement happens in two steps:
1. **Global Placement** — minimises wire length using the HPWL (Half Perimeter Wire Length) metric
2. **Detailed Placement** — legalises cell positions so that no cells overlap and all are on row sites

![Run Placement Complete](images/day1/08_run_placement_complete.png)
*Placement completing — cells legally placed with 0 violations*

#### Viewing Placement in Magic

```bash
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.placement.def &
```

![Magic Placement Full View](images/day1/11_magic_placement_full_view.png)
*Magic — full chip view after placement*

![Magic Placement Zoomed](images/day1/12_magic_placement_zoomed.png)
*Magic — zoomed into standard cell placement area*

![Magic Placement Full Chip](images/day1/15_magic_placement_full_chip.png)
*Magic — full chip with all standard cells placed*

![Placement Statistics Table](images/day1/16_placement_statistics_table.png)
*Placement statistics table showing cell counts and wire length*

![Placement Report Terminal](images/day1/13_placement_report_terminal.png)
*Placement report in terminal*

![Placement DEF Listing](images/day1/14_placement_def_listing.png)
*DEF file listing after placement*

![Placement Statistics Table 2](images/day1/18_placement_statistics_table2.png)
*Detailed placement statistics*

---

## Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

### Theory

#### CMOS Inverter — SPICE Simulation

To characterise a standard cell, we write a SPICE netlist describing the PMOS and NMOS transistors with their W/L ratios, supply voltage, input stimulus, and load capacitance. We then simulate and extract these timing parameters:

- **Rise Time** — time for output to go from 20% to 80% of VDD
- **Fall Time** — time for output to go from 80% to 20% of VDD
- **Propagation Delay** — time from 50% of input to 50% of output (both rising and falling)

```
Rise transition time  =  Time(output 80%) − Time(output 20%)
Fall transition time  =  Time(output 20%) − Time(output 80%)
Propagation delay     =  Time(output 50%) − Time(input 50%)
```

#### 16-Mask CMOS Fabrication Process (Brief Overview)

The chip fabrication follows a sequence of approximately 16 mask steps:

1. Substrate selection (p-type, high resistivity silicon)
2. Active region creation — field oxidation + Si₃N₄ mask to isolate regions
3. N-well and P-well formation via ion implantation
4. Gate oxide growth — thin SiO₂ layer for gate dielectric
5. Polysilicon gate deposition and patterning
6. LDD (Lightly Doped Drain) implantation
7. Source/Drain implantation and annealing
8. Contacts formation — tungsten plugs into silicon
9. Metal interconnect layers (aluminium or copper)
10. Final passivation and pad opening

#### Sky130 PDK — Layer Stack

The Sky130 PDK uses a 5-metal layer stack (li1, met1–met4 plus local interconnect). Each layer has specific minimum width and spacing rules defined in the tech file — violations of these rules are caught by DRC in Magic.

---

### Lab — Cloning and Characterising a Custom Inverter Cell

#### Cloning the Standard Cell Repository

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
cd vsdstdcelldesign
magic -T sky130A.tech sky130_inv.mag &
```

#### Identifying Components in Magic

In the Magic layout, use the `what` command in the tkcon window after selecting a region:

```tcl
# Select a region by pressing S over it, then:
what
```

![Magic Inverter PMOS](images/day2/01_magic_inverter_pmos_identified.png)
*PMOS transistor identified — sky130_fd_pr__pfet_01v8*

![Magic Inverter NMOS](images/day2/02_magic_inverter_nmos_identified.png)
*NMOS transistor identified — sky130_fd_pr__nfet_01v8*

![Magic Inverter Output Pin Y](images/day2/03_magic_inverter_output_pin_y.png)
*Output pin Y highlighted on the local interconnect layer*

![Magic Inverter Full Layout](images/day2/04_magic_inverter_full_layout.png)
*Complete inverter cell layout — PMOS on top, NMOS on bottom*

The CMOS inverter schematic for reference:

![CMOS Inverter Schematic](images/day3/06_cmos_inverter_schematic_handdrawn.png)
*CMOS inverter — PMOS pull-up network, NMOS pull-down network, VDD = 3.3V*

#### Extracting the SPICE Netlist from Magic

In the tkcon window:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

![SPICE Extract Commands](images/day3/08_spice_netlist_edited_for_sim.png)
*Edited SPICE netlist — model library paths corrected and stimulus added*

#### Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```

In the ngspice prompt:

```bash
plot y vs time a
```

![ngspice Simulation Launched](images/day2/05_ngspice_simulation_launched.png)
*ngspice launched with the extracted inverter SPICE netlist*

![ngspice Waveform Plot](images/day2/06_ngspice_waveform_plot.png)
*ngspice transient simulation — output Y (blue) vs input A (red) vs time*

#### Timing Characterisation Measurements

From the waveform, measure each timing parameter by clicking on the plot:

**Rise transition time** = Time(output at 80% of 3.3V = 2.64V) − Time(output at 20% of 3.3V = 0.66V)

![Rise Time 80pct Measurement](images/day2/12_rise_time_80pct_measurement.png)
*Rise time measurement at 80% point (2.64V)*

**Fall transition time** = Time(output at 20% = 0.66V) − Time(output at 80% = 2.64V)

![Fall Time Measurement](images/day2/13_fall_time_measurement.png)
*Fall time measurement*

**Propagation delay** = Time(output at 50% = 1.65V) − Time(input at 50% = 1.65V)

![Propagation Delay Measurement](images/day2/14_propagation_delay_measurement.png)
*Propagation delay measurement at 50% crossing*

![Timing Characterization Values](images/day2/15_timing_characterization_values.png)
*All timing characterisation values extracted from the waveform*

---

### Lab — DRC Fix: poly.9 Rule in sky130A.tech

The Sky130 tech file has an incorrectly implemented poly.9 rule — a poly-to-poly spacing violation that Magic does not flag. We fix it by editing the tech file.

Reference: [Sky130 Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html)

![Magic met3 DRC Violation](images/day3/12_magic_met3_drc_violation.png)
*met3 DRC violation shown in Magic — "Not implemented" spacing rule*

![Magic DRC Violation Highlighted](images/day3/13_magic_drc_violation_highlighted.png)
*DRC violation highlighted in the Magic layout view*

Open the tech file and add the missing spacing rules:

```bash
vi sky130A.tech
```

![sky130A Tech Poly Rules](images/day3/14_sky130A_tech_poly_rules.png)
*sky130A.tech — poly spacing rules section to be corrected*

After editing, reload the tech file and re-run DRC:

```tcl
tech load sky130A.tech
drc check
drc why
```

![Magic DRC Cleared After Fix](images/day3/15_magic_drc_cleared_after_fix.png)
*DRC cleared — 0 violations after applying tech file fix*

![sky130A Tech CIF Nwell Rules](images/day3/16_sky130A_tech_cifout_nwell_rules.png)
*sky130A.tech — cifout and nwell section also corrected*

---

### Lab — Synthesis with Custom Cell

After characterising the inverter, integrate it into the picorv32a design flow:

```bash
run_synthesis
```

![Synthesis Run Post Config](images/day3/09_synthesis_run_post_config.png)
*Synthesis run after updating config.tcl with custom cell*

![Synthesis Slack Report](images/day3/10_synthesis_slack_report.png)
*Timing slack report after synthesis*

![Synthesis TNS WNS Values](images/day3/11_synthesis_tns_wns_values.png)
*TNS (Total Negative Slack) and WNS (Worst Negative Slack) values post-synthesis*

![Synthesis With Custom Cell Run](images/day2/24_synthesis_with_custom_cell_run.png)
*Re-running synthesis with custom inverter cell included*

![Synthesis Custom Cell Log](images/day2/25_synthesis_custom_cell_log.png)
*Synthesis log confirming sky130_vsdinv cell is included*

![Synthesis Custom Cell Complete](images/day2/26_synthesis_custom_cell_complete.png)
*Synthesis completing with custom cell successfully integrated*

---

## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

### Theory

#### LEF Files and Standard Cell Port Guidelines

Before a custom cell can be used in OpenLANE, it needs a proper **LEF (Library Exchange Format)** file describing its physical boundary, pin locations, and metal layer assignments. Two rules must be satisfied:

1. All input and output ports must lie on the **intersection of horizontal and vertical routing tracks**
2. The cell **width must be an odd multiple of the horizontal track pitch**, and height an odd multiple of the vertical track pitch

These rules ensure the router can connect to cell pins without DRC violations.

#### Static Timing Analysis — Key Concepts

**Setup slack** = Data Required Time − Data Arrival Time *(must be ≥ 0)*

**Hold slack** = Data Arrival Time − Data Required Time *(must be ≥ 0)*

Key sources of timing uncertainty in STA:

- **OCV (On-Chip Variation)** — process/voltage/temperature variation modelled using derate factors on cell delays
- **Clock Uncertainty** — jitter and skew budgets added as margins on timing paths
- **CRPR (Clock Reconvergence Pessimism Removal)** — removes artificially pessimistic slack when launch and capture paths share the same clock buffers

#### Clock Tree Synthesis (CTS)

CTS builds a balanced tree of clock buffers to distribute the clock with minimal **skew** across all flip-flops. After CTS:

- **Hold timing must be re-checked** — CTS inserts buffers that add real delay; this can create hold violations
- **Setup timing should be re-verified** — the clock paths have changed and real propagation delays now apply

---

### Lab — Custom Cell LEF Generation and Integration

#### Setting Port Classes for the Inverter

In the Magic tkcon window, define port class and use for each pin:

```tcl
port class input
port use signal
```

![Inverter Layout Port Class Set](images/day4/01_inverter_layout_port_class_set.png)
*config.tcl updated — EXTRA_LEFS and custom lib paths added*

![Inverter Layout Magic Open](images/day3/01_inverter_layout_magic_open.png)
*Inverter layout open in Magic for port definition*

![Inverter Port A Defined](images/day3/04_inverter_port_a_defined.png)
*Port A (input) — class input, use signal*

![Inverter Port Y Defined](images/day3/05_inverter_port_y_defined.png)
*Port Y (output) — class output, use signal*

#### Checking Pins on Track Intersections

Load the tracks file and set the grid:

```bash
less $PDK_ROOT/sky130A/libs.tech/openlane/sky130_fd_sc_hd/tracks.info
```

```tcl
grid 0.46um 0.34um 0.23um 0.17um
```

![Inverter Pins On Track Intersection](images/day4/02_inverter_pins_on_track_intersection.png)
*Inverter A and Y pins confirmed sitting on li1 track intersections*

#### Writing the LEF File

```tcl
lef write
```

![vsdstdcelldesign Directory LEF](images/day4/03_vsdstdcelldesign_directory_lef.png)
*vsdstdcelldesign directory — sky130_vsdinv.lef generated*

#### Copying Files to picorv32a/src

```bash
cp sky130_vsdinv.lef ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
cp libs/sky130_fd_sc_hd__* ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
```

#### Editing config.tcl to Include the Custom Cell

```tcl
set ::env(LIB_SYNTH)   "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)  [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

![Synthesis Custom Cell Included](images/day3/07_synthesis_custom_cell_included.png)
*sky130_vsdinv custom cell successfully included in synthesis*

![Synthesis Error Fix](images/day4/14_synth_error_fix.png)
*Synthesis error resolved after correcting config.tcl paths*

![Synthesis Custom Cell OK](images/day4/15_synth_custom_ok.png)
*Synthesis completing successfully with custom cell*

---

### Lab — Static Timing Analysis with OpenSTA

Create `pre_sta.conf` in the OpenLANE directory and `my_base.sdc` in `designs/picorv32a/src/`, then run:

```bash
sta pre_sta.conf
```

![STA Hold Slack Report](images/day4/04_sta_hold_slack_report.png)
*STA hold slack report — min path analysis*

![STA Setup Slack](images/day4/16_sta_setup_slack.png)
*STA setup slack — worst negative slack (WNS)*

![STA Timing Report](images/day4/18_sta_timing_report.png)
*Full STA timing report — critical path details*

![STA TNS WNS](images/day4/19_sta_tns_wns.png)
*TNS and WNS values from STA*

---

### Lab — Clock Tree Synthesis

```bash
run_cts
```

![run_cts With OpenROAD Magic](images/day4/05_run_cts_with_openroad_magic.png)
*run_cts executing — TritonCTS building the clock tree*

![Magic Routed Chip Post CTS](images/day4/06_magic_routed_chip_post_cts.png)
*Magic — chip layout after CTS showing clock buffer insertion*

![Cell List Post CTS Terminal](images/day4/07_cell_list_post_cts_terminal.png)
*Terminal showing cell list post-CTS — clock buffers (clkbuf) added*

#### Post-CTS Timing Analysis in OpenROAD

```bash
openroad
```

```tcl
read_lef /openLANE_flow/designs/picorv32a/runs/.../tmp/merged.lef
read_def /openLANE_flow/designs/picorv32a/runs/.../results/cts/picorv32a.cts.def
write_db pico_cts.db
read_db pico_cts.db
read_verilog /openLANE_flow/designs/picorv32a/runs/.../results/synthesis/picorv32a.v
read_liberty $::env(LIB_SYNTH_COMPLETE)
link_design picorv32a
read_sdc /openLANE_flow/designs/picorv32a/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4
```

![OpenROAD Post CTS Timing](images/day4/09_openroad_post_cts_timing.png)
*OpenROAD post-CTS timing report — min and max path analysis*

![STA Post CTS Slack Analysis](images/day4/10_sta_post_cts_slack_analysis.png)
*Post-CTS STA — setup and hold slack both met*

---

## Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA

### Theory

#### Routing — Global vs Detailed

Routing happens in two stages:

1. **Global Routing (FastRoute)** — divides the chip into routing regions (GCells) and finds approximate paths for each net, respecting layer and congestion constraints. Produces routing *guides* rather than exact wire segments.

2. **Detailed Routing (TritonRoute)** — takes the global routing guides and assigns exact wire segments, vias, and metal tracks while adhering strictly to DRC rules (spacing, width, via enclosure).

#### Design Rule Check (DRC) — Common Violations

- **Min spacing** — two wires too close on the same layer
- **Min width** — a wire narrower than the process minimum
- **Antenna violation** — a long metal segment accumulating charge during plasma etching, potentially destroying gate oxide. Fix: insert antenna diodes at input pins or jump to a higher metal layer using a via.

#### SPEF and Post-Route STA

After routing, parasitic resistance and capacitance of the actual wires are extracted into a **SPEF (Standard Parasitic Exchange Format)** file. These parasitics are back-annotated into the netlist and STA is re-run — this is the **final sign-off timing check** before tape-out.

---

### Lab — Power Distribution Network

```bash
gen_pdn
```

The PDN creates VDD and VSS **power rings** around the core, **power stripes** across the chip on met4/met5, and **power rails** along every standard cell row on met1. Standard cells tap into these rails through their VPwr and VGnd pins.

---

### Lab — Routing

```bash
run_routing
```

![Run Routing Complete Log](images/day5/02_run_routing_complete_log.png)
*run_routing completing — 0 DRC violations, all nets routed*

#### Viewing the Final Routed Layout in Magic

```bash
magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.def &
```

![Magic Routed Chip Full View](images/day5/03_magic_routed_chip_full_view.png)
*Magic — complete routed picorv32a chip (full view)*

![Magic Routed Placement Zoomed](images/day5/04_magic_routed_placement_zoomed.png)
*Magic — zoomed into routed placement showing metal routing layers*

![Magic Routed Detail Filler Cells](images/day5/05_magic_routed_detail_filler_cells.png)
*Magic — filler cells and decap cells visible in routed layout*

![Magic Routed Custom Cell Placed](images/day5/06_magic_routed_custom_cell_placed.png)
*Magic — custom sky130_vsdinv inverter placed and routed in the design*

![Magic Routed Final Layout](images/day5/07_magic_routed_final_layout.png)
*Magic — final routed layout ready for GDSII generation*

#### Post-Route OpenROAD Timing

![OpenROAD Post CTS Timing](images/day5/01_openroad_post_cts_timing.png)
*OpenROAD post-route timing — setup and hold paths both met after routing*

---

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, Placement, CTS, detailed routing |
| **Magic VLSI** | Layout editor, DRC, LEF/SPICE extraction |
| **OpenSTA** | Static Timing Analysis |
| **ngspice** | SPICE simulation for cell characterisation |
| **TritonRoute** | Detailed routing |
| **Netgen** | LVS (Layout vs Schematic) |
| **Sky130 PDK** | SkyWater 130nm open-source PDK |

---

## Key Learnings

- Understood how a chip moves from RTL to GDSII using a fully open-source toolchain — from Yosys synthesis through OpenROAD P&R to Magic DRC
- Got hands-on with floorplanning, placement, CTS, and routing for the `picorv32a` RISC-V processor core on Sky130
- Learned to characterise custom standard cells using Magic and ngspice and integrate them into an existing flow via LEF
- Gained practical experience with STA — setup/hold slack, OCV, CRPR, and timing closure using OpenSTA and OpenROAD
- Understood how parasitics from post-route SPEF extraction affect timing sign-off numbers

---

## Acknowledgements

- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design) — for designing and delivering this workshop
- **Nickson Jose** — for the `vsdstdcelldesign` repository used throughout Day 3 and Day 4 labs
- **NASSCOM FutureSkills Prime** — for organising and facilitating the workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
- [Sky130 Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html)
