# Implementing-A-User-Project-SPM

## Project Description
In this project we will be building on the [Caravel User Project](https://github.com/efabless/caravel_user_project) and show how we can use the template to harden our design a Serial-Parallel-Multiplier (SPM) and integrate it within the [caravel](https://github.com/efabless/caravel) chip. 

## Prerequisites 
You must go throw the [Quick start for caravel_user_project](https://github.com/efabless/caravel_user_project/blob/main/docs/source/quickstart.rst) tutorial to familiarize yourself on how to setup your environment and what are the dependencies you need to install before you start working on the project. 

## SPM Design 
SPM stands for serial parallel multiplier, where it takes two 32-bit numbers and performs multiplication to produce a 64-bit product.</br>

The design consists of 3 different modules:
* [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v) <br/>
* [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v) <br/>
* [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v) <br/>

As shown below, the first module [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v) has a 32-bit input standing for the multiplicand (x), then with each clock cycle, one bit is read from the multiplier (y), and one bit from the product (p) is calculated.
<p align="center">
  <img src="https://user-images.githubusercontent.com/56173018/179765856-c6b1b4d3-c4d7-420c-992c-e8e400bed66b.png" alt="spm"/>
</p>

The second module [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v) is responsible for shifting the multiplier to the right each clock cycle and feeding its least significant bit to the spm. It also takes the spm output (p), concatenates it to the final output and shifts it to the right until the multiplication process is done. The module receives a start signal when the two numbers are ready to be multiplied and send a done signal when the process is finished. Its block diagram:    
<p align="center">
  <img src="https://user-images.githubusercontent.com/56173018/179776347-cca2f741-d51a-46ee-9b8a-2fc6f7c775d2.png" alt="mul32"/>
</p>

The third module and its the top module of our project is [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v). This module is responsible for communicating with the caravel's management SoC through the wishbone bus, it recieves the multiplier and the multiplicand and then returns the final product. You can read more on the caravel chip architecture [here](https://github.com/efabless/caravel#caravel-architecture). Below are the module's block diagram along with the [user_project_wrapper.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_project_wrapper.v) block diagram as well: </br> </br>
<p align="center">
<img src="https://user-images.githubusercontent.com/56173018/179789831-b20a9770-8bcd-4e9f-8173-016cdcf46d69.png" alt="user_proj_mul32" height="400"/>
<img src="https://user-images.githubusercontent.com/56173018/179789822-7607e04d-72ba-4d5c-90bf-6d689fbbefc5.png" alt="project_wrapper" height="400"/>
</p>

## Steps to Implement Our Design
1. You must clone the cravel_user_project and setup your environment as mentioned in [Quick start for caravel_user_project](https://github.com/efabless/caravel_user_project/blob/main/docs/source/quickstart.rst) <br/>
2. Copy the design's source files [spm.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/spm.v), [mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/mul32.v), [user_proj_mul32.v](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/rtl/user_proj_mul32.v) into ``caravel_user_project/verilog/rtl`` <br/>
3. Navigate  ``caravel_user_project/openlane`` folder then make a copy of the forder ``user_proj_example`` into ``user_proj_mul32``. <br/>
4. Replace the ``config.tcl`` file in the ``user_proj_mul32`` folder that you just created with this [config.tcl](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/openlane/user_proj_mul32/config.tcl) file. <br/>
5. Now you can harden the design using [openlane](https://github.com/The-OpenROAD-Project/OpenLane); inside the folder ``caravel_user_project``, run the following command 
```bash
make user_proj_mul32
``` 
6. To get the project wrapper ready; go to the folder ``caravel_user_project/verilog/rtl`` an edit the file ``user_project_wrapper.v`` to change user_proj_example to user_proj_mul32. <br/>
7. Replace the ``config.tcl`` file under ``caravel_user_project/openlane/user_project_wrapper`` with this [config.tcl](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/openlane/user_project_wrapper/config.tcl) file. <br/>
8. Harden the project wrapper; inside the folder ``caravel_user_project``, run the following command 
```bash
make user_project_wrapper
``` 

## Testing Our design
As mentioned above, the purpose of our design is to recieve two numbers and return their product. And it was also shown that the communication that happens between the chip's management SoC and the user area was through the wishbone bus. That is why we will use the testbench [wb_port.c](https://github.com/ZeyadZaki/Implementing-A-User-Project-SPM/blob/main/verilog/dv/wb_port/wb_port.c) to test that our design works as expected. <br/> <br/>
The test basically assigns two values to the registers of the multiplicand and the multiplier and checks if the values of the product's register is correctly equal to their product or not. <br/>
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
