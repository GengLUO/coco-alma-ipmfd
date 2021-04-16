# Alma: Execution-aware Masking Verification

Alma is an _execution-aware_ tool for formal verification of masked implementations.
In principle, Alma can verify any data-independent masked computation that can be 
implemented as a Verilog hardware circuit, with properly labeled secret shares and 
masks. However, as of now, the focus of Alma is to verify software executions on a
hardware level. This is also the focus of the related paper [COCO](https://eprint.iacr.org/2020/1294.pdf).

## Setup

1. Create a new virtual environment
``` bash
python3 -m venv dev
```

2. Activate the virtual environment
``` bash
source dev/bin/activate
```

3. Install the required python packages
``` bash
pip3 install -r requirements.txt
```

4. Install Yosys >= 0.9 and Verilator >= 4.106:
* **Easy**: install it using your favourite package manager
``` bash
apt install yosys verilator
```
* **Flexible**: Install then according to the official guides 
[here](https://github.com/YosysHQ/yosys/blob/yosys-0.9/README.md) and [here](https://www.veripool.org/projects/verilator/wiki/Installing).

After following these steps, the only additional thing you need is the hardware 
circuit you want to verify in Verilog or System Verilog. For System Verilog support,
either try recompilation into vanilla Verilog with [sv2v](https://github.com/zachjs/sv2v) 
or get a commercial license for [Symbiotic EDA](https://www.symbioticeda.com/seda-suite).

## Usage

Alma provides several Python programs that represent different stages of the verification flow.
In the following, we briefly show how each of them is used and what its purpose is. 

### 1. Parse the circuit
```
python3 parse.py --source VERILOG_FILES --top-module TOP_MODULE [optional arguments]
```
The arguments for the standard mode of operation are:
  * `--source`: file path(s) to the source file(s)
  * `--top-module`: the top module name
  
Special arguments include:
  * `--synthesis-file`: `parse.py` will automatically generate a Yosys synthesis script. In case one does not want to use this script, this option can be used to specify a custom Yosys synthesis script. Then, the `--source` option must not be used because the script already includes the paths of the sources.
  * `--label`: Custom output file path of label file. Default: `alma/tmp/labels.txt`
  * `--json`: Custom output file path of JSON file. Default: `alma/tmp/circuit.json`
  * `--netlist`: Custom output file path of netlist file. Default: `alma/tmp/circuit.v`
  * `--yosys`: `parse.py` will search for the Yosys binary using `which yosys`. In case one wants to use a specific Yosys version, the path can be specified with this option.
  * `--log-yosys`: Yosys synthesis output is required and will be written to `alma/tmp/yosys_synth_log.txt` 
  
The outputs of this step are a label file, the circuit in JSON format and the netlist file.

### 2. Label the registers and memory in the label file

`labels.txt` lists all registers and memory locations, which can be labeled as `share`, `mask` or `unimportant`.

For example, a valid labeling can look like this:

```
my_super_cpu.r1:0:17840: unimportant
my_super_cpu.r1:1:17840: unimportant
my_super_cpu.r2:0:17841: share 1
my_super_cpu.r3:0:17563: mask
my_super_cpu.r4:0:17420: share 1
```

The register name follows the format: `display_name:bit:index`, where `display_name` is a mangled 
name from the original design, `bit` indicates a specific bit in a register, and `index` is the
global index of the given bit and _not relevant for users_.

You need to label registers and ports that are parts of a secret with `share X` where _X_ is
the index of the given secret. Random bits are labeled with `mask`, whereas other public values 
such as control signals and the like are labeled with `unimportant`.

### 3. Generate an execution trace
```
python3 trace.py --testbench TB_FILE_PATH --netlist NETLIST_FILE_PATH [optional arguments]
```
The arguments for the standard mode of operation are:
  * `--testbench`: Path to the Verilator testbench.
  * `--netlist`: Path of Verilog netlist generated by Yosys in the parsing step.

Special arguments include:
  * `--skip-compile-netlist`: Use cached object files from a previous verilator run.
  * `--c-compiler`: Compiler used by Verilator (either clang or gcc), the default and recommended option is clang.
  * `--output-bin`: Path to verilated binary. Default: `alma/tmp/<netlist_name>`

### 4. Verify the masking implementation
```
python3 verify.py --json JSON_FILE_PATH --label LABEL_PATH --vcd TRACE_PATH [optional arguments]
```
The arguments for the standard mode of operation are:
  * `--json`: File path of JSON file
  * `--label`: File path of label file
  * `--vcd`: File path of VCD file

Special arguments include:
  * `--cycles`: The verification process will run until the end of the VCD trace per default (-1). In case it should abort earlier, this option can be used.
  * `--order`: Verification order, i.e., the number of probes. Default: 1
  * `--mode`: Verification can be done for the stable or transient case. Default: stable
  * `--probe-duration`: Specifies how a long probe records values. Default: once
  * `--trace-stable`: If specified, trace signals are assumed to be stable
  * `--rst-name`: Verification will start after the circuit reset is over. This is the name of the reset signal. Default: `rst_i`
  * `--rst-cycles`: Duration of the system reset in cycles. Default: 2
  * `--rst-phase`: Value of the reset signal which triggers the reset. Default: 1
  * `--num-leaks`: Number of leakage locations to be reported if the circuit is insecure. Default: 1
  * `--dbg-output-dir`: Directory in which debug traces (dbg-label-trace-?.txt, dbg-circuit-?.dot) are written. Default: `alma/tmp/`

## Resources

The `examples` directory contains instructions on how to perform software masking verification
on an improved version of the [IBEX RISC-V core](https://github.com/IAIK/coco-ibex) presented in the paper.
