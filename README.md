# Implementing-A-User-Project-SPM

## Table of Contents
* [Project Description](#project-description)
* [Caravel](#caravel)
* [Openlane](#openlane)
* [SPM Design](#spm-design)
* [Wishbone Communication](#wishbone-communication)
* [Design Implementation](#design-implementation)
* [Design Simulation](#design-simulation)

## Project Description
In this project we will be building on the [Caravel User Project](https://github.com/efabless/caravel_user_project) and show how we can use the template to harden our design a Serial-Parallel-Multiplier (SPM) and integrate it within the [caravel](https://github.com/efabless/caravel) chip. 

## Caravel
[Caravel](https://github.com/efabless/caravel) is a template SoC for Efabless Open MPW and chipIgnite shuttles based on the Sky130 node from SkyWater Technologies. It provides a wrapper for the user project area where the user can add their own designs. <br/>
<p align="center">
  <img width="451" alt="Screen Shot 2022-07-31 at 12 59 41 PM" src="https://user-images.githubusercontent.com/56173018/182022965-96c2875e-91cf-40fc-8c59-cde0ddeaa16e.png">
  </p>

As shown in the above figure, caravel is composed of the harness frame plus two wrappers for drop-in modules for the management area and user project area. Our project will reside in the user project wrapper and it has access to the following utilities provided by the management SoC:
* 38 IO Ports
* 128 Logic analyzer probes
* Wishbone port connection to the management SoC wishbone bus.

We will be using the wishbone bus as the communication channel between the management SoC and the SPM, more on that [here](#wishbone-communication).

## Openlane
[Openlane](https://github.com/The-OpenROAD-Project/OpenLane) is an automated RTL to GDSII flow based on several components including OpenROAD, Yosys, Magic, Netgen, CVC, SPEF-Extractor, CU-GR, Klayout and a number of custom scripts for design exploration and optimization. The flow performs full ASIC implementation steps from RTL all the way down to GDSII. The architecture of Openlane:
<p align="center">
  <img width="686" alt="Screen Shot 2022-07-31 at 1 20 26 PM" src="https://user-images.githubusercontent.com/56173018/182023837-3ec8d154-8c5a-4d47-a80f-3fa4a9959bbd.png">
  </p>
  
When hardening your design using OpenLane, there are 3 available options to use: 

* The first option is hardening your design, then inserting the hardened design in the user project wrapper, then hardening the user project wrapper. This is the option that is chosen by default.

* The second option is flattening the user macro(s) with the user_project_wrapper then hardening everything altogether in one step. 

* The third option is a mix of both using some hardened macros and other flattened ones with the same warpper. Check the example of [caravel_ibex](https://github.com/efabless/caravel_ibex) as a reference for this technique. 


## SPM Design 
SPM stands for serial parallel multiplier, where it takes two 32-bit numbers and performs multiplication to produce a 64-bit product. </br>

The design consists of 3 different modules:
* [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v) <br/>
* [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v) <br/>
* [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v) <br/>

As shown below, the first module [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v) receives a 32-bit input standing for the multiplicand (x), then with each clock cycle, one bit is being read from the multiplier (y), and one bit from the product (p) is calculated and sent as an output. This module is instantiated from [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v).
<p align="center">
  <img src="https://user-images.githubusercontent.com/56173018/179765856-c6b1b4d3-c4d7-420c-992c-e8e400bed66b.png" alt="spm"/>
</p>

The second module [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v) is instantiated from [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v). It receives the multiplier (mp) and the multiplicand (mc) and is responsible for shifting the multiplier to the right each clock cycle and feeding its least significant bit to the spm module. It also takes the spm output (p), concatenates it to the final output and shifts it to the right until the multiplication process is done (64 clock cycles). The module receives a start signal when the two numbers are ready to be multiplied and sends the final product along with a done signal when the process is finished. Its block diagram:    
<p align="center">
  <img src="https://user-images.githubusercontent.com/56173018/179776347-cca2f741-d51a-46ee-9b8a-2fc6f7c775d2.png" alt="mul32"/>
</p>

The third module and its the top module of our project is [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v). This module is the one instantiated inside the [User’s Project Wrapper](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_project_wrapper.v). It is important to note that any User’s Project should have a fixed set of ports as you can find the verilog code. In summary, this module is responsible for communicating with the caravel's management core through the wishbone bus (more on that [here](#wishbone-communication)). Below are the module's block diagram along with the [user_project_wrapper.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_project_wrapper.v) block diagram: </br> </br>
<p align="center">
<img src="https://user-images.githubusercontent.com/56173018/179789831-b20a9770-8bcd-4e9f-8173-016cdcf46d69.png" alt="user_proj_mul32" height="400"/>
<img src="https://user-images.githubusercontent.com/56173018/179789822-7607e04d-72ba-4d5c-90bf-6d689fbbefc5.png" alt="project_wrapper" height="400"/>
</p>

## Wishbone Communication
We implement the SPM in the user project area as a peripheral that can be accessed by firmware on the management SoC. The management core communicates with the SPM using the wishbone bus, there are other ways of communication such as the Logic Analyzer (LA) signals, however, in this project, only the wishbone communication is going to be discussed. <br/>

As shown in the [caravel architecture](#caravel), the management core exposes the wishbone bus to the user’s project area. This wishbone bus is used for communication between the management core and the peripheral implemented in the user’s project area. In this case, the management core is called a “master” and the peripheral in the user’s project area is called a “slave”. The reason for that is that the management core is the one initiating any read or write operation. The wishbone bus is a shared bus among all the peripherals in the user’s project area, which means that at any point in time, the management core can communicate with only one slave in the user’s project area. <br/>

<p align="center">
<img width="662" alt="Screen Shot 2022-07-31 at 5 01 39 PM" src="https://user-images.githubusercontent.com/56173018/182033252-d2defdf4-4a59-4cb2-b597-56fd7c3e5fe9.png">
</p>

As shown in the above figure, the user’s project area in our case has 4 registers, where each one is considered a slave. These registers  correspond to the multiplicand, the multiplier and 2 registers for the product since the wishbone bus has a data bus of width 32 bits only. The registers are memory-mapped, which means that they can be accessed using addresses in the memory. The addresses used for these 4 variables are shown in the below table: <br/>

Operand | Variable in Verilog | Address in memory | Name of register 
--- | --- | --- | --- 
Multiplicand | MC | 0x30000000 | reg_mprj_slave_X 
Multiplier | MP | 0x30000004 | reg_mprj_slave_Y
Product (least significant 32 bits) | P[31:0] | 0x30000008 | reg_mprj_slave_P0 
Product (most significant 32 bits) | P[63:32] | 0x3000000C | reg_mprj_slave_P1 

There are 10 signals used to write/read to a wishbone slave port that are summarized in the below table:
Signal Name in verilog |	# bits |	Signal meaning |	How is the signal used?
--- | --- | --- | --- 
wb_clk_i |	1 |	Clock |	A square wave that can be at either high or low. Data is read or written at either the posedge (positive edge) or negedge (negative edge)
wb_rst_i | 1 | Reset | A reset restores the status of all registers to the initial value (which is normally zero)
wbs_stb_i/wbs_cyc_i |	1 |	Strobe/Bus Cycle |	If both of these signals are high this means that there is a valid data transfer taking place. The ANDing of those two signals is the enable signal for any slave port register. 
wbs_we_i | 1 |	Write enable for input | If this signal is high, then a write operation is taking place, if it is low, then a read operation is taking place.
wbs_sel_i |	4	| Select for input | If a write operation is taking place, the slave port has a 32-bit register. That register is divided into 4 8-bit units. Each bit on this bus tells if the corresponding 8-bits coming from the wbs_dat_i should be written or not. For example, if wbs_sel_i[0] is 1 and this is a write operation, then the slave register bits 0-7 are going to be updated with the values read from wbs_dat_i[7:0]
wbs_dat_i |	32 | Input data |	This is the data that is written from a master to a slave in a write operation
wbs_adr_i |	32 | Input Address | Each slave port is mapped to an address in memory. wbs_adr_i specifies that address. 
wbs_ack_o |	1	| Acknowledgement |	This signal is set to high following a successful read or write operation. In the case of the SPM, a read operation for the P register is not going to take place until the multiplication is done, which means that wbs_ack_o will stay low until the multiplication is finished.
wbs_dat_o	| 32 | Output Data | At the end of a write operation, the data sent on wbs_dat_o is the same as the input data received from wbs_dat_i which indicates a successful read operation. At the end of a read operation, the data sent on wbs_dat_o is the data stored in slave port with address wbs_adr_i

The above table explains what each signal means in a wishbone slave and how each signal can be used. In the figure below, it shows a slave port register for MP and MC in the SPM. This figure is not valid for the Product registers (register storing the product) since the ack signal when reading from P is dependent on the completeness of the multiplication operation as well. 

<p align="center">
  <img src="https://user-images.githubusercontent.com/56173018/183260884-6f0ecae7-02dd-42d5-93e6-57fc70c2a198.png" alt="slave port"/>
</p>
  
## Design Implementation
In this section we will discuss the steps needed to start your project, setup your environment, then harden the design along with the wrapper. We will be using the first technique while hardening; which is hardening the design first then inserting the hardened design in the user project wrapper, then hardening the user project wrapper.

1. To start the project you first need to create a new repository based on the [caravel_user_project](https://github.com/efabless/caravel_user_project/) template and make sure your repo is public and includes a README.

   *   Follow https://github.com/efabless/caravel_user_project/generate to create a new repository.
   *   Clone the reposity using the following command:
  
```bash    
git clone <your github repo URL>
```

2. To setup your local environment run:
```bash
cd <project_name> # project_name is the name of your repo

mkdir dependencies

export OPENLANE_ROOT=$(pwd)/dependencies/openlane_src # you need to export this whenever you start a new shell

export PDK_ROOT=$(pwd)/dependencies/pdks # you need to export this whenever you start a new shell

# export the PDK variant depending on your shuttle, if you don't know leave it to the default
export PDK=sky130B

make setup
```

- This command will setup your environment by installing the following:
  * caravel_lite (a lite version of caravel)
  * management core for simulation
  * openlane to harden your design
  * pdk


3. The environment is now ready we can start adding our SPM Design files; copy the design's source files [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v), [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v), [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v) into ``<project_name>/verilog/rtl`` <br/>

4. To Harden the design using [Openlane](https://github.com/The-OpenROAD-Project/OpenLane) you need to:
  * Navigate  ``<project_name>/openlane`` folder to make a copy of the forder ``user_proj_example`` into ``user_proj_mul32``. <br/>
  * Edit to the configurations file ``config.tcl`` in the ``user_proj_mul32`` folder that you just created as follows:
  ```script
##### Change the design name
set ::env(DESIGN_NAME) user_proj_mul32

##### Specify the design files
set ::env(VERILOG_FILES) "\
	$::env(CARAVEL_ROOT)/verilog/rtl/defines.v \
	$script_dir/../../verilog/rtl/user_proj_mul32.v \
	$script_dir/../../verilog/rtl/mul32.v \
	$script_dir/../../verilog/rtl/spm.v"
  
##### Disable basic placement
# set ::env(PL_BASIC_PLACEMENT) 1

##### Increase the target density:
set ::env(PL_TARGET_DENSITY) 0.35
```
To learn more about openlane configurations check out this [documentation](https://github.com/The-OpenROAD-Project/OpenLane/blob/master/configuration/README.md)
  * Now our design is ready to be hardened; inside the folder ``caravel_user_project``, run the following command 
```bash
make user_proj_mul32
  ``` 
5. Now that our design is hardened, we need to harden the project wrapper 
  * Go to the folder ``caravel_user_project/verilog/rtl`` an edit the file ``user_project_wrapper.v`` to change the module being instantiated from user_proj_example to user_proj_mul32. 
  * Edit the wrapper's configurations file ``config.tcl`` under ``<project_name>/openlane/user_project_wrapper`` as follows:
```script
##### Black-box verilog and views 
set ::env(VERILOG_FILES_BLACKBOX) "\
	$::env(CARAVEL_ROOT)/verilog/rtl/defines.v \
	$script_dir/../../verilog/rtl/user_proj_mul32.v"

set ::env(EXTRA_LEFS) "\
	$script_dir/../../lef/user_proj_mul32.lef"

set ::env(EXTRA_GDS_FILES) "\
	$script_dir/../../gds/user_proj_mul32.gds"

##### Add the following line 
set ::env(GLB_RT_ADJUSTMENT) 0.1
```
  * Now you can harden the project wrapper; inside the folder ``caravel_user_project``, run the following command 
```bash
make user_project_wrapper
``` 

### Hardening Outputs and Reports
* The ``make <design_name>`` command starts the ASIC flow to produce the layout in GDS2 format starting from the design HDL files. When the flow finishes, you should see the message:
```bash
[SUCCESS]: Flow complete.
```
* As the flow runs, it produces lots of files, such as reports and log files, under the ``run_path``:
``<project_name>/openlane/<design_name>/runs/<design_name>``
  
* Openlane's ASIC flow ends with physical verification. This begins by streaming out the GDS followed by running DRC, LVS, and Antenna checks on the design. 
* A final summary report is produced by default as ``<run-path>/reports/final_summary_report.csv``, for more details about the contents of the report check this [documentation](https://github.com/The-OpenROAD-Project/OpenLane/blob/master/regression_results/datapoint_definitions.md).  
* A final manufacturability report is produced by default as ``<run-path>/reports/manufacturability_report.csv``, this report contains the magic DRC, the LVS, and the antenna violations summaries.
* The final GDS-II file can be found under ``<run-path>/results/final/gds`` and can be displayed using Klayout
* You can also examine the layout after each step (for planning, placement, cts, ...), we can load the step DEF file produced by any step into Klayout. For example, to examine the layout after floor-planning, do the following:
  * Copy the LEF file ``<run_path>/tmp/merged.lef`` into the the folder ``<run_path>/results/floorplan``:
  * Invoke Klayout
  * Inside klayout: File ⇒ Import ⇒ DEF/LEF, then choose the def file inside the floorplan folder and make sure that the ``merged.lef`` file is used as well.

## Design Simulation
Generally, a testbench is used to make sure that a design is working before actually fabricating it since fabrication is an expensive process. When we write a testbench we want to make sure that the design is performing the correct functionality. A testbench in caravel user’s project is written as a c file and a verilog file. The c file is the one that has code to be executed on the management core, thus this file is the one controlling the signals sent to any peripheral device in the user’s project area. The verilog file (or test bench)  is used to simulate the management core and its interaction with the user project area. 

Using the same test bench, there are 3 types of verification that can be performed summarized in the below table:
Verification Type | What is done in this type of verification | Terminal Commands  
--- | --- | --- 
RTL (Register Transfer Level) | The RTL is the verilog code that was written by the programmer. The RTL verification does not take into account any delay information. | make verify-testbench_name-rtl
GL (Gate Level) | In OpenLane flow, a gate level netlist is generated after the synthesis (the first step in the flow) and then updated in multiple other steps. This means that it is necessary to harden the design before performing GL or GL-SDF simulations. | make verify-testbench_name-gl
SDF (Standard Delay Format) | SDF is a file format which is generated by the Static timing analysis (STA) tool. Simply explained, the SDF has more accurate information about the delays caused by each wire, which might cause timing violations. | make verify-testbench_name-gl-sdf

Returning back to the SPM example, here are the two files for the test bench, [wb_port.c](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/dv/wb_port/wb_port.c) and [wb_port.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/dv/wb_port/wb_port.v). We defined the names of the 4 registers that were used by the SPM; reg_mprj_slave_X, reg_mprj_slave_Y, reg_mprj_slave_P0 and reg_mprj_slave_P1. Those names are going to be used as variable names in the c file. For example, if we write in c:

```script
reg_mprj_slave_X = 3
reg_mprj_slave_Y = 10
if(reg_mprj_slave_P0 == 30 && reg_mprj_slave_P1==0)
```
This means: write 3 to the multiplicand register and write 10 to the multiplier register . Then read the product registers, if the product is equal to 30, do the statements following the if condition. Also, in the c file, there is a variable called ``reg_wb_enable``, which must be set to 1 preceding any read or write operation. We also manipulate a variable called ``reg_mprj_datal``, because this corresponds to the ``mprj_io[15:0]`` bits in the verilog test bench. We check the changes in the value stored on those bits to know if a successful multiplication operation has taken place. 


Before running the test you need to do the following steps:
1. Replace the file ``wb_port.c`` under ``caravel_user_project/verilog/dv/wb_port`` with [wb_port.c](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/dv/wb_port/wb_port.c) <br/>
2. Edit the file ``includes.rtl.caravel_user_project`` under ``caravel_user_project/verilog/includes`` and add the following lines:
```script
-v $(USER_PROJECT_VERILOG)/rtl/user_proj_mul32.v
-v $(USER_PROJECT_VERILOG)/rtl/mul32.v
-v $(USER_PROJECT_VERILOG)/rtl/spm.v
``` 
3. Edit the file ``includes.gl.caravel_user_project`` under ``caravel_user_project/verilog/includes`` to replace ``user_proj_example.v`` to ``user_proj_mul32.v`` <br/>

Now you can run your tests on both rtl and gl using the following commands:
```bash
make verify-wb_port-rtl

make verify-wb_port-gl
``` 
NOTE: The rtl test can be simulated before you harden the design, the gl can only be simulated after hardening
