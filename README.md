# Digital VLSI SoC Design and Planning — RTL to GDSII

> A hands-on workshop covering the complete RTL-to-GDSII physical design flow for a digital SoC, built entirely on **open-source EDA tools** — **OpenLANE**, **Magic**, **ngspice** and **OpenSTA** — targeting the **SkyWater Sky130 PDK**.
> This repository documents the concepts, lab exercises and results captured across all five days of the workshop, using the `picorv32a` RISC-V core as the reference design.

---

## Table of Contents

- [Day 1 — Introduction to Open-Source ASIC Design, OpenLANE and Sky130 PDK](#day-1--introduction-to-open-source-asic-design-openlane-and-sky130-pdk)
- [Day 2 — Chip Floorplanning, Library Cells and Standard Cell Placement](#day-2--chip-floorplanning-library-cells-and-standard-cell-placement)
- [Day 3 — Design and Characterization of Standard Cells using Magic and ngspice](#day-3--design-and-characterization-of-standard-cells-using-magic-and-ngspice)
- [Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis](#day-4--pre-layout-timing-analysis-and-clock-tree-synthesis)
- [Day 5 — Final RTL to GDSII Flow: Power Distribution, Routing and Post-Route Timing Analysis](#day-5--final-rtl-to-gdsii-flow-power-distribution-routing-and-post-route-timing-analysis)
- [Tools & Environment](#tools--environment)
- [Key Learnings](#key-learnings)
- [Acknowledgements](#acknowledgements)
- [References](#references)

---

## Day 1 — Introduction to Open-Source ASIC Design, OpenLANE and Sky130 PDK

### Objective

Understand how a digital integrated circuit is designed — starting from a hardware description and ending with a manufacturable layout — and get introduced to the architecture of an IC, the role of Process Design Kits (PDKs), and the complete RTL-to-GDSII implementation flow.

### Understanding What Lies Inside a Chip

A microcontroller board such as an Arduino contains much more than the visible package mounted on the PCB. The package protects the silicon die and provides electrical connections to the outside world through bonding wires.

Inside the die, the digital logic is implemented within the **core**, while the surrounding **I/O pads** provide communication with external devices. The key building blocks inside a chip are:

| Block | Description |
|---|---|
| **Macros** | Pre-designed functional blocks reused across designs |
| **Standard Cells** | Basic logic elements used during synthesis |
| **Foundry IPs** | Process-dependent blocks such as SRAMs, PLLs and other analog components supplied by the foundry |

Understanding this hierarchy helps build a clear picture of how a modern System-on-Chip is physically organized before moving into the implementation flow.

### From Software to Hardware

A C program is first translated into **RISC-V assembly**, followed by machine code consisting of binary instructions. These instructions are executed by a processor whose functionality is described using **RTL (Register Transfer Level)**.

Since the workshop uses the **picorv32a** RISC-V processor as the reference design, this establishes the connection between the RISC-V Instruction Set Architecture and the RTL implementation that is later synthesized and physically implemented using OpenLANE.

### Importance of Open-Source PDKs

Traditionally, semiconductor Process Design Kits (PDKs) were proprietary and accessible only under commercial agreements, making ASIC design difficult for students and independent engineers.

This changed with the release of the **SkyWater SKY130 Process Design Kit** — the first widely available open-source production PDK, developed through a collaboration between Google and SkyWater Technology. SKY130 made it possible to perform a complete RTL-to-GDSII implementation using entirely open-source tools, forming the foundation on which the OpenLANE flow operates.

### RTL to GDSII ASIC Design Flow

The major implementation stages of the flow are:

- **Logic Synthesis** — Converts RTL into a technology-mapped gate-level netlist using the standard cell library
- **Floorplanning** — Defines the die dimensions, core utilization, macro placement and power distribution network
- **Placement** — Determines the physical locations of all standard cells while minimizing congestion and wire length
- **Clock Tree Synthesis (CTS)** — Builds the clock distribution network to reduce clock skew and maintain timing consistency
- **Routing** — Creates all metal interconnections between cells using the available routing layers
- **Parasitic Extraction** — Extracts resistance and capacitance information from the routed layout
- **Static Timing Analysis (STA)** — Verifies that timing requirements are satisfied under defined operating conditions
- **Physical Verification** — Performs Design Rule Checking (DRC) and Layout Versus Schematic (LVS) to ensure the layout is manufacturable and functionally equivalent to the synthesized netlist
- **GDSII Generation** — Produces the final layout database sent for fabrication

### OpenLANE RTL-to-GDSII Flow

The OpenLANE framework integrates multiple open-source EDA tools into a single automated ASIC implementation flow.

![OpenLANE RTL-to-GDSII flow overview](images/day1/image1.png)

| Design Stage | Tool |
|---|---|
| Logic Synthesis | Yosys, ABC |
| Floorplanning | OpenROAD |
| Power Distribution | OpenROAD |
| Placement | OpenROAD |
| Clock Tree Synthesis | TritonCTS |
| Global Routing | FastRoute |
| Detailed Routing | TritonRoute |
| RC Extraction | OpenRCX |
| Static Timing Analysis | OpenSTA |
| Layout Generation | Magic, KLayout |
| DRC | Magic |
| LVS | Netgen |

Understanding the responsibility of each tool makes it easier to visualize how OpenLANE automates the complete physical design flow while still allowing every stage to be executed and analyzed independently.

### Lab — Setting Up OpenLANE

The OpenLANE environment was launched in interactive mode and the **picorv32a** design was prepared for implementation.

```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
prep -design picorv32a
```

![OpenLANE interactive mode and design prep](images/day1/image2.png)

### Running Logic Synthesis

The synthesis stage was executed for the **picorv32a** RISC-V processor. During synthesis:

- RTL was converted into a gate-level netlist
- Technology mapping was performed using the SKY130 standard cell library
- Area and cell utilization statistics were generated
- Timing and synthesis reports were produced for further analysis

![Synthesis run output](images/day1/image3.png)
![Synthesis statistics report](images/day1/image4.png)

### Flop Ratio Calculation

From the synthesis statistics report, the number of sequential elements and total instantiated cells were obtained.

```
Flop Ratio = Number of D Flip-Flops / Total Number of Cells
           = 1613 / 15762
           ≈ 0.1023  ->  10.233 %
```

This metric provides a quick estimate of the proportion of sequential logic present in the synthesized design and serves as a useful validation parameter during synthesis.

---

## Day 2 — Chip Floorplanning, Library Cells and Standard Cell Placement

### Objective

Cover the physical planning stage of ASIC implementation. Before placing any standard cells, the overall chip dimensions, power distribution strategy, I/O organization and macro locations must be defined — this stage determines how efficiently the design can be routed and whether timing and power targets can be achieved later.

### Understanding Chip Floorplanning

Floorplanning is the first physical design stage where the logical design begins to take a physical shape. Instead of dealing only with RTL or gates, the design is now mapped onto a silicon area by defining the dimensions of the **die** and the **core**.

The **die** represents the complete silicon chip, while the **core** is the region reserved for placing the digital logic. During floorplanning, the width and height of both regions are determined based on the complexity of the design and the available silicon area. The aspect ratio of the core also influences routing efficiency and congestion in later stages.

**Core utilization** defines how much of the available core area is occupied by the synthesized netlist. A moderate utilization is generally preferred because it leaves enough free space for buffering, routing resources and future optimizations — an overly dense design often results in routing congestion and timing violations.

### Pre-Placed Cells and Decoupling Capacitors

Not every block inside a chip can be placed automatically. Large modules such as SRAMs, memories, PLLs and other hard macros are positioned manually before automated placement begins. These are known as **pre-placed cells**, and their locations are selected carefully to minimize wire length, improve connectivity and simplify routing.

**Decoupling capacitors** act as nearby charge reservoirs. During switching activity, large current demands can temporarily reduce the local supply voltage due to resistance in the power network; decoupling capacitors quickly supply current whenever these voltage drops occur, helping maintain stable power delivery and preserve the circuit's noise margins. Their placement around critical macros significantly improves power integrity.

### Power Planning and I/O Organization

A reliable power distribution network is essential for any integrated circuit. Floorplanning includes the creation of a structured power network using VDD and VSS rails distributed across multiple metal layers. Power rings and meshes ensure that every standard cell receives a stable supply while minimizing voltage drop and electromigration effects.

The placement of input and output pins is equally important, since these pins provide communication between the core and external circuitry — their positions are chosen based on the connectivity of the design so signals can reach their destinations with minimal routing complexity.

The region reserved for I/O structures and pad cells is intentionally kept free from standard cell placement. This blockage prevents placement tools from occupying areas required for routing, pad connections and other peripheral circuitry, making the overall Place-and-Route (PnR) process more efficient.

### Lab — Floorplanning in the OpenLANE Flow

The `run_floorplan` command generated the initial physical representation of the design by defining the die area, core dimensions, placement boundaries and power planning information, and produced the Design Exchange Format (DEF) file containing the physical description of the floorplan.

```tcl
run_floorplan
```

![Floorplan generation output](images/day2/image1.png)

### Viewing the Floorplan

The generated DEF file was loaded into **Magic Layout** for visualization.

```bash
cd results/floorplan/
less picorv32a.def
```

![Floorplan DEF file](images/day2/image2.png)

```bash
magic -T /home/vscode/.ciel/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.nom.lef \
  def read picorv32a.def &
```

Inspecting the floorplan made it possible to observe the core boundary, placement rows, I/O regions and reserved spaces created during floorplanning — providing the first graphical representation of the design and demonstrating how the logical netlist is translated into a physical layout before standard cell placement begins.

![Floorplan view in Magic — core boundary](images/day2/image3.png)
![Floorplan view in Magic — placement rows](images/day2/image4.png)
![Floorplan view in Magic — I/O regions](images/day2/image5.png)
![Floorplan view in Magic — reserved spaces](images/day2/image6.png)

### Standard Cell Placement

After successful floorplanning, the placement stage was executed using the `run_placement` command. OpenLANE positioned all synthesized standard cells within the predefined placement rows while respecting the locations of pre-placed macros and the placement blockages created during floorplanning. The placement engine optimized cell locations to reduce interconnect length while preparing the design for clock tree synthesis and routing.

```tcl
run_placement
```

The final placement result showed that all standard cells were legally placed without overlapping, producing an optimized physical arrangement for the subsequent implementation stages.

![Placement result — legally placed standard cells](images/day2/image7.png)
![Placement layout view](images/day2/image8.png)

---

## Day 3 — Design and Characterization of Standard Cells using Magic and ngspice

### Objective

Understand how a standard cell is designed, characterized and verified before becoming part of a standard cell library. This involved exploring a custom CMOS inverter layout, extracting its SPICE netlist, performing post-layout simulations and understanding Design Rule Checking (DRC) using Magic.

### Standard Cell Design Flow

A standard cell is developed through the following sequence of stages before it is added to a technology library:

1. Define the logic function
2. Design the transistor-level CMOS circuit
3. Create the physical layout following process design rules
4. Extract the SPICE netlist from the layout
5. Perform post-layout simulation
6. Characterize timing parameters
7. Generate the timing library used by synthesis and STA tools

The characterization process measures important timing parameters:

- Rise Transition Time
- Fall Transition Time
- Rise Propagation Delay
- Fall Propagation Delay

These parameters are obtained from the simulated output waveform and later become part of the standard cell timing library.

### 16-Mask CMOS Fabrication Process (Brief Overview)

The chip fabrication follows a sequence of about 16 mask steps:

1. Substrate selection (p-type, high resistivity)
2. Active region creation (field oxidation + Si3N4 mask)
3. N-well and P-well formation (ion implantation)
4. Gate oxide growth
5. Polysilicon gate deposition
6. Source/Drain implantation (LDD + halo)
7. Contacts and metal layers
8. Final passivation

### Lab 1 — Exploring the Custom Inverter Layout

The custom inverter standard cell repository was cloned and opened using Magic Layout.

```bash
cd ~/Desktop/OpenLane
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
cd vsdstdcelldesign
cp /home/vsduser/Desktop/OpenLane/pdks/sky130A/libs.tech/magic/sky130A.tech .
magic -T sky130A.tech sky130_inv.mag &
```

The inverter layout was explored to identify:

- PMOS and NMOS devices
- Input and output connections
- VDD and GND rails
- Metal layers and contacts
- Overall cell structure

![Custom inverter layout in Magic](images/day3/image1.png)

### Lab 2 — SPICE Extraction from Magic

After verifying the layout, the transistor-level SPICE netlist was extracted from Magic. Commands executed inside the **tkcon** window:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

The generated SPICE file was then modified by including the required model files, simulation parameters and input stimulus before running electrical simulations.

![Extracted SPICE netlist](images/day3/image2.png)
![SPICE netlist editing](images/day3/image3.png)
![Final SPICE deck ready for simulation](images/day3/image4.png)

### Lab 3 — Post-Layout ngspice Simulation

The extracted SPICE netlist was simulated using **ngspice**.

```bash
ngspice sky130_inv.spice
```

![ngspice simulation session](images/day3/image5.png)

To display the waveform:

```
plot y vs time a
```

The generated waveform was used to measure:

- Rise Transition Time
- Fall Transition Time
- Rise Cell Delay
- Fall Cell Delay

These measurements characterize the switching performance of the inverter after considering layout parasitics.

![Output waveform](images/day3/image6.png)
![Rise/fall transition measurement](images/day3/image7.png)
![Cell delay measurement](images/day3/image8.png)
![Waveform analysis](images/day3/image9.png)
![Waveform analysis continued](images/day3/image10.png)

### Lab 4 — Design Rule Checking (DRC)

The final exercise demonstrated how Magic verifies layouts against the manufacturing rules defined by the SKY130 technology.

![DRC check in Magic](images/day3/image11.png)

An older technology file containing incomplete DRC rules was examined and corrected by referring to the SKY130 process documentation. The following rules were analyzed:

- `poly.9`
- `difftap.2`
- `nwell.4`

After modifying the technology file, the updated rules were loaded and verified. The corrected technology file successfully detected the violations that were previously ignored, demonstrating how DRC ensures that layouts satisfy the fabrication constraints defined by the process.

> Reference: [Sky130 Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html)

![Technology file rule correction — before](images/day3/image12.png)
![Technology file rule correction — poly.9](images/day3/image13.png)
![Technology file rule correction — difftap.2](images/day3/image14.png)
![Technology file rule correction — nwell.4](images/day3/image15.png)
![DRC violations detected after correction](images/day3/image16.png)
![DRC violations detected — layer view](images/day3/image17.png)
![DRC rule verification](images/day3/image18.png)
![DRC rule verification continued](images/day3/image19.png)
![Final corrected DRC deck](images/day3/image20.png)

---

## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

### Objective

Integrate the custom inverter designed on Day 3 into the OpenLANE flow. The custom standard cell was converted into a LEF file, added to the picorv32a design, verified through placement, and analyzed using OpenSTA. This session also introduced the fundamentals of Clock Tree Synthesis (CTS) and pre-layout timing analysis.

### LEF Generation and Standard Cell Guidelines

For a custom standard cell to be recognized by the physical design flow, it must be represented using a **LEF (Library Exchange Format)** file, which contains the physical information required by placement and routing tools, including the cell boundary, pin locations and routing layers.

Before generating the LEF, the layout was verified to satisfy standard cell design guidelines:

- Input and output pins should lie on routing track intersections
- Cell width should match the routing track pitch requirements
- Cell height should align with the vertical routing tracks

These conditions ensure that the custom cell can be legally placed and routed alongside the existing SKY130 standard cell library.

### Timing Analysis and Clock Tree Synthesis — Concepts

A brief introduction to Static Timing Analysis (STA) covered setup timing, propagation delay and transition times, along with timing uncertainties such as process variation, clock skew and jitter. The purpose of Clock Tree Synthesis (CTS) was also discussed — dedicated clock buffers are inserted to distribute the clock signal uniformly across the design while minimizing clock skew.

After CTS, timing must be analyzed again since the clock network changes and new delays are introduced.

### Lab 1 — Preparing the Custom Cell

The custom inverter layout was first verified against the routing grid before generating the LEF file. Commands executed in **Magic**:

```tcl
help grid
grid 0.46um 0.34um 0.23um 0.17um
```

After confirming the alignment, the LEF file was generated:

```tcl
lef write
```

![Routing grid alignment](images/day4/image1.png)
![Routing grid setup continued](images/day4/image2.png)
![LEF generation](images/day4/image3.png)
![Generated LEF file](images/day4/image4.png)

### Lab 2 — Integrating the Custom Cell into OpenLANE

The generated LEF and Liberty files were copied into the **picorv32a/src** directory.

```bash
cp sky130_vsdinv.lef ~/OpenLane/designs/picorv32a/src/
cp libs/sky130_fd_sc_hd__* ~/OpenLane/designs/picorv32a/src/
```

The OpenLANE configuration was updated by modifying the **config.tcl** file to include the newly generated LEF and Liberty libraries:

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

This allowed OpenLANE to recognize the custom inverter as part of the standard cell library during synthesis and physical implementation.

![config.tcl updated with custom LEF and Liberty paths](images/day4/image5.png)

### Lab 3 — Running Synthesis with the Custom Cell

The design was prepared again after adding the custom library.

```tcl
prep -design picorv32a
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
run_synthesis
```

Different synthesis parameters such as synthesis strategy, sizing and fanout were also modified to improve timing quality and reduce violations before continuing with the physical design flow.

![Synthesis with custom cell — run output](images/day4/image6.png)
![Synthesis statistics with custom cell](images/day4/image7.png)
![Synthesis report](images/day4/image8.png)
![Synthesis parameter tuning](images/day4/image9.png)
![Sizing and fanout adjustments](images/day4/image10.png)
![Post-tuning synthesis result](images/day4/image11.png)
![Final synthesis report](images/day4/image12.png)

### Lab 4 — Floorplanning and Placement

Once synthesis successfully recognized the custom inverter, the design proceeded through floorplanning and placement.

```tcl
run_floorplan
run_placement
```

The generated placement DEF was opened in **Magic** to verify that the custom inverter had been successfully inserted into the design.

```bash
magic -T sky130A.tech \
  lef read ../../tmp/merged.lef \
  def read picorv32a.placement.def &
```

Using the **expand** command, the internal routing layers and proper abutment between neighboring standard cells were inspected:

```tcl
expand
```

The custom inverter was observed to be correctly placed together with the SKY130 standard cell library, confirming successful integration into the OpenLANE flow.

![Placement DEF with custom inverter](images/day4/image13.png)
![Custom inverter abutment with neighboring cells](images/day4/image14.png)
![Expanded internal routing layers view](images/day4/image15.png)

### Lab 5 — Pre-Layout Static Timing Analysis

A custom STA configuration file and SDC constraints were created before running OpenSTA.

```tcl
sta pre_sta.conf
```

The timing report highlighted setup violations and paths with large fanout. Various synthesis parameters were adjusted to reduce timing violations and improve the Worst Negative Slack (WNS). The session also demonstrated simple timing ECOs by replacing low-drive cells with higher drive-strength equivalents and observing the resulting timing improvements.

![OpenSTA pre-CTS timing report](images/day4/image16.png)
![Setup violation paths](images/day4/image17.png)
![Fanout analysis](images/day4/image18.png)
![WNS improvement after parameter tuning](images/day4/image19.png)
![Timing ECO — cell swap](images/day4/image20.png)
![Timing ECO result](images/day4/image21.png)
![Post-ECO timing report](images/day4/image22.png)
![Final pre-CTS timing summary](images/day4/image23.png)

### Lab 6 — Clock Tree Synthesis

After completing timing analysis, Clock Tree Synthesis was performed.

```tcl
run_cts
```

The CTS-generated DEF was loaded into OpenROAD for post-CTS timing analysis.

```tcl
write_verilog <RUN_DIR>/results/synthesis/picorv32a.v
run_cts

openroad
read_lef <RUN_DIR>/tmp/merged.nom.lef
read_def <RUN_DIR>/results/cts/picorv32a.def
write_db pico_cts.db
read_db pico_cts.db
read_verilog <RUN_DIR>/results/synthesis/picorv32a.v
read_liberty $::env(LIB_SYNTH_COMPLETE)
link_design picorv32a
read_sdc <DESIGN_DIR>/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} \
  -format full_clock_expanded -digits 4
```

Different CTS clock buffer configurations were also explored to observe their impact on timing and clock skew.

![CTS run output](images/day4/image24.png)
![Post-CTS OpenROAD session](images/day4/image25.png)
![Post-CTS timing report](images/day4/image26.png)
![Clock buffer configuration comparison](images/day4/image27.png)

---

## Day 5 — Final RTL to GDSII Flow: Power Distribution, Routing and Post-Route Timing Analysis

### Objective

Complete the physical implementation of the design by generating the power distribution network, performing routing and understanding post-route timing verification — converting the placed design into a fully connected layout ready for final verification and GDSII generation.

### Routing in ASIC Physical Design

After placement, all standard cells are physically located inside the core but remain electrically disconnected. The routing stage creates the metal interconnections required to implement the logical connectivity described by the netlist. Routing is performed in two stages:

- **Global Routing** — Determines an approximate routing path for every net while considering routing resources, congestion and preferred metal layers. The output serves as a routing guide for the next stage.
- **Detailed Routing** — Generates the exact wire segments, vias and metal connections by following the routing guides while satisfying all Design Rule Check (DRC) constraints defined by the process technology.

Together, these stages complete the physical interconnections required for the design.

### Parasitic Extraction and Post-Route Timing Analysis

Once routing is completed, the physical wires introduce additional resistance and capacitance that were not present during synthesis or placement. These parasitic effects are extracted into a **SPEF (Standard Parasitic Exchange Format)** file, which is then used during Static Timing Analysis (STA). By considering the actual interconnect delays, post-route timing analysis provides a more accurate representation of the final chip performance before sign-off.

### Lab 1 — Power Distribution Network (PDN) Generation

Before routing, the Power Distribution Network (PDN) was generated to ensure that all standard cells receive reliable VDD and GND connections throughout the design. The PDN consists of power rails and metal straps distributed across the layout, providing a stable power supply while minimizing voltage drop.

```tcl
gen_pdn
```

### Lab 2 — Global and Detailed Routing

After generating the power network, the routing stage was executed using the OpenLANE flow.

```tcl
run_routing
```

During routing, OpenLANE first performs global routing to determine routing paths and then executes detailed routing to establish the final physical connections while satisfying the SKY130 design rules. The completed layout contains all signal, clock and power interconnections required for fabrication.

![Post-routing layout view](images/day5/image28.png)
![Final routed design](images/day5/image29.png)

---

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys / ABC** | RTL logic synthesis |
| **OpenROAD** | Floorplanning, power planning, placement |
| **TritonCTS** | Clock Tree Synthesis |
| **FastRoute** | Global routing |
| **TritonRoute** | Detailed routing |
| **OpenRCX** | Parasitic (RC) extraction |
| **OpenSTA** | Static Timing Analysis |
| **Magic** | Layout editing, visualization, DRC |
| **KLayout** | GDSII viewing and streaming |
| **Netgen** | Layout Versus Schematic (LVS) |
| **ngspice** | SPICE-level circuit simulation |
| **Sky130 PDK** | SkyWater 130 nm open-source Process Design Kit |

---

## Key Learnings

- Traced the complete journey of a chip from RTL description to a fabrication-ready GDSII file, entirely through open-source EDA tools
- Practiced the physical design stages — floorplanning, placement, clock tree synthesis and routing — on the `picorv32a` RISC-V core
- Learned the end-to-end process of designing a custom standard cell and integrating it into an existing OpenLANE library flow
- Built a working understanding of core STA concepts, including setup and hold slack, on-chip variation (OCV) and clock reconvergence pessimism removal (CRPR), using OpenSTA
- Saw how post-route parasitics extracted into a SPEF file influence final timing sign-off accuracy

---

## Acknowledgements

A huge thank you to Kunal Ghosh (Co-founder, VSD Corp. Pvt. Ltd.) and Nickson P Jose (Physical Design Engineer, Intel) for putting together such a well-structured and genuinely practical workshop. Running a real CPU from RTL to GDSII using nothing but open-source tools is something I didn't expect to be possible — and yet here we are.

- Kunal Ghosh — Co-founder, VSD (VLSI System Design)
- Nickson Jose — for the `vsdstdcelldesign` repository used in Day 3 labs
- NASSCOM — for facilitating this workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
