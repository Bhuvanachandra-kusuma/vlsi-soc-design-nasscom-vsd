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
