## Intel Building and Emulation

The building of codes for the physical FPGA chips is a time consuming process. Firstly it is useful to have a powerful system for undertaking the actual build process, which is the case for this testbed, and also vendors provide emulation support such that users can much more quickly build and test their codes on the CPU without having to do the full build for the FPGA.

### Intel toolchain

You should first load the _quartus_ module, `module load quartus`. This will make available Intel specific tooling, including _aoc_ and _aocl_ into your environment, as well as setting up the correct OpenCL environment for hardware emulation and Stratix-10 board environment. We have developed a simple number sum example [here](https://github.com/FPGAtestbed/intel_sum_example) to demonstrate using the Intel toolchain on the testbed, this can also be used as a skeleton structure for more complex FPGA codes if that is helpful.

```console
[username@nextgenio-login2 ~]$ git clone https://github.com/FPGAtestbed/intel_sum_example.git
[username@nextgenio-login2 ~]$ cd intel_sum_example
```

The example contains a makefile which provides configurations for building for emulation and physical hardware. 

#### Building and running for emulation

In Intel terminology _emulation_ is running in software, where the OpenCL device code is compiled against libraries and executed on the host. This is far quicker than building for hardware, however catches less errors and does not support all hardware modes (such as multi-kernel). 

```console
[username@nextgenio-login2 ~]$ make
[username@nextgenio-login2 ~]$ aoc -march=emulator src/device/device.cl
.....
[username@nextgenio-login2 ~]$ export CL_CONTEXT_EMULATOR_DEVICE_INTELFPGA=1
[username@nextgenio-login2 ~]$ ./host 10
Execution mode: Emulation
Using AOCX: device.aocx
Kernel initialization is complete
Memory initialization is complete
Kernel execution has been scheduled
Kernel execution has completed successfully
```

Note here how _make_ builds the host executable only, and the _aoc_ command is the Altera OpenCL compiler, with _march=emulator_ selecting to build for emulation. The _CL_CONTEXT_EMULATOR_DEVICE_INTELFPGA_ environment variable is required as this informs the emulator how many devices to emulate.

#### Building for the FPGA

Building for actual hardware will take around two hours for this example, a challenge is that this building process is then connected to the console, so closing that will terminate the building of the bitstream. There are a number of ways around this, for instance by using the graphical desktop then when you quit out of X2GO it will typically keep the session open and running. However, a more complete way is to use the _nohup_ command. We would generally suggest redirecting stderr and tailing the output file too.

```console
[username@nextgenio-login2 ~]$ nohup aoc src/device/device.cl &> output &
[username@nextgenio-login2 ~]$ tail -fn2000 output
```

Note that in the host program it is very similar to launch the emulator or physical bitstream apart from the selection of the OpenCL platform. This is currently undertaken in the host program based on the _CL_CONTEXT_EMULATOR_DEVICE_INTELFPGA_ environment variable.

Detailed information about the _aoc_ compiler and other Quartus Prime Pro commands can be found in the [programming guide](https://www.intel.com/content/www/us/en/docs/programmable/683846/22-1/overview.html) and [best practice guide](https://www.intel.com/content/www/us/en/docs/programmable/683176/18-1/introduction-to-standard-edition-best.html).

### Leveraging the HBM2

The Stratix-10 MX has 16GB of HBM2 sitting across 32-banks, therefore each bank has 512MB. By default all global memory will be allocated in HBM bank 0, which will cause issues with both performance (contention across input and output data) and also limits the size of global memory to that specific bank. You can specify which bank to allocate into via the `__attribute__((buffer_location("HBM0")))` attribute on the arguments of the OpenCL kernel).

The following code will allocate data across HBM banks 0 to 4 respectively. You do not need to add any specific HBM bank information into the host driver code, however the buffer should be created with the `CL_MEM_READ_WRITE | CL_MEM_HETEROGENEOUS_INTELFPGA` flags.

```c
__kernel 
void my_kernel(__global __attribute__((buffer_location("HBM0"))) double * const restrict input1, 
               __global __attribute__((buffer_location("HBM1"))) double * const restrict input2, 
               __global __attribute__((buffer_location("HBM2"))) double * restrict output1, 
               __global __attribute__((buffer_location("HBM3"))) double * restrict output2)
```