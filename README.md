# VexRiscv_FPGA_debugging
In this tutorial, I will replicate the work in [this](https://tomverbeure.github.io/2021/07/18/VexRiscv-OpenOCD-and-Traps.html) tutorial but with using Cmod A7 FPGA and vscode gui for debugging. 
### 1. Generating bitstream
I used Xilinx Vivado to implemenet the design on FPGA 
* The RTL files you will need are [top.v](https://github.com/tomverbeure/vexriscv_ocd_blog/blob/main/rtl/top.v) and [VexRiscvWithDebug.v](https://github.com/tomverbeure/vexriscv_ocd_blog/blob/main/spinal/VexRiscvWithDebug.v).
* You will need the program files which you can find [here](https://github.com/NouranAbdelaziz/VexRiscv_FPGA_debugging/tree/main/mem_files) Those files will be generated when you clone [vexriscv_ocd_blog repo](https://github.com/tomverbeure/vexriscv_ocd_blog) and build the software using Makefile in sw directory. The program toggles the three LEDs in the FPGA in sequence. You will need to convert the hex files to mem files inorder to be recognized by Vivado.
* You will also need a constraints file for the Cmod A7 FPGA which you can find in this repository [here](https://github.com/NouranAbdelaziz/VexRiscv_FPGA_debugging/blob/main/vexriscv_cmodA7.xdc). If you want the ready bitstream to program your FPGA with, you can find it [here](https://github.com/NouranAbdelaziz/VexRiscv_FPGA_debugging/blob/main/vexriscv_toggle.bit)

### 2. Hardware Setup
The hardware tools I used are:
* Cmod A7 35T FPGA
* ARM-USB-TINY-H
* USB Type B
* jumper wires
The USB Type B cable will be used to connect the laptop with the JTAG FTDI cable (arm-usb-tiny-h) and the jumper wires will be used to connect the JTAG FTDI to the FPGA pins as follows:
* FPGA pin 3 will be connected to pin 9 in JTAG FTDI (tck)
* FPGA pin 4 will be connected to pin 7 in JTAG FTDI (tms)
* FPGA pin 5 will be connected to pin 5 in JTAG FTDI (tdi)
* FPGA pin 6 will be connected to pin 13 in JTAG FTDI (tdo)
* FPGA PMOD VCC will be connected to pin 1 in JTAG FTDI (vref)
* FPGA PMOD GND will be connected to pin 4 in JTAG FTDI (vref)
  
### 3. Program the FPGA
You can use Vivado as well to program the FPGA or Digilent Adept if not availble. After connecting the FPGA to the computer using USB cable, run this command in the location of the bitstream file
```
djtgcfg prog -d CmodA7 -i 0 -f vexriscv_toggle.bit
```
You will see the RGB LEDs in FPGA are toggling in sequence. 

### 4. OpenOCD configuration
You will need to use the configuration for the specific ftdi you are using. In my case I used the olimex-arm-usb-tiny-h.cfg configuration file. You need to change the ocd-only target command to: 
```
$(OPENOCD) -f interface/ftdi/olimex-arm-usb-tiny-h.cfg  \
		-c "adapter speed 10; transport select jtag" \
		-f "vexriscv_init.cfg"
```
and then in the ``vexriscv_ocd_blog/sw`` directory run:
```
make ocd-only
```
This will connect OpenOCD to the target and makes it ready to listen to the GDB commands on 3333. You will get this output:
```
Open On-Chip Debugger 0.11.0+dev-04033-g058dfa50d (2023-06-15-11:35)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
jtag
Info : set servers polling period to 50ms
Info : clock speed 10 kHz
Info : JTAG tap: vexrisc_ocd.bridge tap/device found: 0x10001fff (mfg: 0x7ff (<invalid>), part: 0x0001, ver: 0x1)
[vexrisc_ocd.cpu0] Target successfully examined.
Info : starting gdb server for vexrisc_ocd.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
Halting processor
requesting target halt and executing a soft reset
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```
### 5. Connect GDB
You can either connect GDB using CLI or GUI using vscode
For CLI run this command in ``vexriscv_ocd_blog/sw`` directory:
```
make gdb
```
You will get this and will notice that the LEDs stopped toggling 
```
Reading symbols from progmem.elf...
Remote debugging using localhost:3333
0x000000ba in wait_led_cycle (ms=0) at main.c:12
12	    if (REG_RD_FIELD(STATUS, SIMULATION) == 1){
(gdb)
```
You can then type the commands you want. For example you can use those commands to load the program, reset, then make a break point at main and then continue the program.
```
(gdb) load
Loading section .text, size 0x202 lma 0x0
Start address 0x00000000, load size 514
Transfer rate: 4112 bits in <1 sec, 514 bytes/write.
(gdb) br main
Breakpoint 1 at 0x102: file main.c, line 55.
(gdb) monitor soft_reset_halt
requesting target halt and executing a soft reset
(gdb) c
Continuing.

Program stopped.
main () at main.c:55
55	    global_cntr = 0;
(gdb) c
Continuing.
```

For the debugging using vscode, open the Debug panel (CTRL + SHIFT + D) and select “Add Configuration > GDB” through the top left dropdown arrow. Create a GDB configuration in launch.json and add the following:
```
    "version": "0.2.0",
    "configurations": [
        {
            "name": "GDB",
            "type": "gdb",
            "request": "launch",
            "cwd": "${workspaceRoot}",
            "target": "/home/nouran/vexriscv_ocd_blog/sw/progmem.elf",
            "gdbpath" : "/opt/riscv64-unknown-elf-toolchain-10.2.0-2020.12.8-x86_64-linux-ubuntu14/bin/riscv64-unknown-elf-gdb",
            "autorun": [
                "target extended-remote localhost:3333",
                "symbol-file /home/nouran/vexriscv_ocd_blog/sw/progmem.elf"
                ]
        }
        
    ]
}
```
Change the "target" and the "symbol-file" to be the path for the .elf file of the program. Then run the debugger using the green arrow. After that you can use the debug control buttons to restart, step over, step back, continue and you can insert break points as well. Use the restart and step over buttons to execute line by line and see the LEDs color change one at a time. 
