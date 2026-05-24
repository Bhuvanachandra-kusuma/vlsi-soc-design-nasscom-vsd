# vlsi-soc-design-nasscom-vsd
RTL2GDSII flow using OpenLANE &amp; SkyWater 130nm PDK | NASSCOM-VSD SoC Design Workshop



set ::env(LIB_SYNTH) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"

set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"

set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"

set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"

set ::env(EXTRA_LEFS) [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]

magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech lef read designs/picorv32a/runs/22-05_17-21/tmp/merged.lef def read designs/picorv32a/runs/22-05_17-21/results/placement/picorv32a.placement.def &


grep -i "sky130_vsdinv" designs/picorv32a/runs/22-05_17-21/results/synthesis/picorv32a.synthesis.v


magic -T $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech designs/picorv32a/runs/22-05_17-21/results/magic/picorv32a.mag &


set_cmd_units -time ns -capacitance pF -current mA -voltage V -resistance kOhm -distance um

read_liberty -min /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib

read_liberty -max /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib

read_verilog /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/22-05_17-21/results/synthesis/picorv32a.synthesis.v

link_design picorv32a

read_sdc /home/vsduser/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/my_base.sdc

report_checks -path_delay min_max -fields {slew trans net cap input_pin}

report_tns

report_wns


nano ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/my_base.sdc

set ::env(CLOCK_PORT) clk

set ::env(CLOCK_PERIOD) 5.000

create_clock [get_ports $::env(CLOCK_PORT)] -name $::env(CLOCK_PORT) -period $::env(CLOCK_PERIOD)

set IO_PCT 0.2

set input_delay_value [expr $::env(CLOCK_PERIOD) * $IO_PCT]

set output_delay_value [expr $::env(CLOCK_PERIOD) * $IO_PCT]

set_max_fanout $::env(SYNTH_MAX_FANOUT) [current_design]


set clk_indx [lsearch [all_inputs] [get_port $::env(CLOCK_PORT)]]

set all_inputs_wo_clk [lreplace [all_inputs] $clk_indx $clk_indx]

set all_inputs_wo_clk_rst $all_inputs_wo_clk

set_input_delay $input_delay_value -clock [get_clocks $::env(CLOCK_PORT)] $all_inputs_wo_clk_rst

set_output_delay $output_delay_value -clock [get_clocks $::env(CLOCK_PORT)] [all_outputs]

set_driving_cell -lib_cell sky130_fd_sc_hd__clkbuf_6 -pin X $all_inputs_wo_clk_rst

set_load 5 [all_outputs]










