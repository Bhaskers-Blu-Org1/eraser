**ERASER** - **E**arly-stage **R**eliability and **S**ecurity **E**valuation for **R**ISC-V
===========================================================================================

**Author**: Karthik Swaminathan <kvswamin@us.ibm.com>

[![Build Status](https://travis-ci.org/IBM/eraser.svg?branch=master)](https://travis-ci.org/IBM/eraser)

![GitHub](https://img.shields.io/github/license/IBM/eraser.svg)
![GitHub forks](https://img.shields.io/github/forks/IBM/eraser.svg?style=social)
![GitHub stars](https://img.shields.io/github/stars/IBM/eraser.svg?style=social)
![GitHub watchers](https://img.shields.io/github/watchers/IBM/eraser.svg?style=social)

**Introduction**
----------------

**ERASER** is a framework for end-to-end estimation of processor vulnerability,
particularly to radiation-induced soft errors in RISC-V based processors.

The ERASER toolflow comprises of the following components:

1. **RTL core model**: This tool flow uses the Rocket core from Berkeley
   (github.com/chipsalliance/rocket-chip), though it can be extended to use any
   RISC-V based core.

2. **Microbenchmark generator (Microprobe)**: This is a microbenchmark
   generation tool capable of generate testcases of instruction sequences for
   several ISAs (including RISC-V). It is used to generate single instruction
   testcases for an initial characterization of te RISC-V core, as well as
   stressmarks based on the instruction sequences generated by SERMiner.

3. **Early-stage latch-level vulnerability estimation and stressmark generation
   (SERMiner)**: The SERMiner tool parses the switching (VCD) files output from
   RTL simulation of the core model, and generates latch-level data switching
   statistics. The macro, or RTL module-level residency is obtained from these
   statistics in terms of the average time for which each bit retains its state
   across the entire macro. These residency statistics are then thresholded based
   on a user defined parameter (RESIDENCY\_THRESHOLD) and evaluated across all
   macros and all instructions. These aggregate thresholded statistics are then
   used to determine the sequence of instructions that maximize residency as well
   as macro coverage, based on an algorithm described in [2], which in turn is
   used to generate the stressmark. The latches with the highest residency
   obtained from simulation of the stressmark on the core model are then earmarked
   for targeted fault injection for further validation.

4. **Latch-level fault injection (Chiffre)**: Chiffre is a latch-level fault
   injection tool, used to prune the list of vulnerable latches by eliminating
   those that are derated, that is, they do not affect the overall output even
   when a fault is injected in them. Focusing any and all protection strategies on
   this final list of latches would maximize RAS coverage across the entire core.

![ERASER flow](./ERASER.png?raw=true "ERASER flow")

The above figure shows a representative flow for the ERASER RAS estimation
methodology.

**First time setup**
--------------------

Execute the following command once:

```
./bootstrap_environment.sh
```

It will checkout all the dependencies as submodules and build them as needed. In
case something fais, we refer you to the detailed installation instructions
of each dependency.

Check the following section for more details. If the installation suceeds move
to the example documentation provided.

**Dependencies**
----------------

1. Rocket tools: git clone https://github.com/freechipsproject/rocket-tools

   To avoid issues with the example scripts provided, check out this dependency in
   `<ERASER_BASE_DIR>/rocket-tools` and install it in `<ERASER_BASE_DIR>/rocket-tools-install`
   directory.

2. Rocket core: git clone https://github.com/chipsalliance/rocket-chip.git

   To avoid issues with the example scripts provided, check out this dependency in
   `<ERASER_BASE_DIR>/rocket-chip`.

3. Microprobe: git clone https://github.com/IBM/microprobe.git

   To avoid issues with the example scripts provided, check out this dependency in
   `<ERASER_BASE_DIR>/microprobe`.

4. Chiffre: git clone https://github.com/IBM/chiffre.git

   To avoid issues with the example scripts provided, check out this dependency in
   `<ERASER_BASE_DIR>/chiffre`.

5. Perl modules:

   - List::MoreUtils

If you already have these dependencies installed in your system, you can
create the appropiate symbolic links that point to them to avoid long first
time check-out and compilation times. Execute:

```
  cd <ERASER_BASE_DIR>
  ln -s <path_to_your_microprobe> microprobe
  ln -s <path_to_your_chiffre> chiffre
  ln -s <path_to_your_rocket-chip> rocket-chip
  ln -s <path_to_your_rocket-tools> rocket-tools
  ln -s <path_to_your_rocket-tools-install> rocket-tools-install
```

**Environment setup**
---------------------

* Execute::

```
source eraser_setenv
```

You should see the ERASER symbol if all environment variables have been set correctly.
This script also sets the microprobe enviroment as checks the correctness of the
enviroments (i.e. the necessary files generated during the first time setup are
present). You'll need to setup the environment every time you start a new ERASER
session in a new shell.

**Example methodology**
-----------------------

* Generate single instruction testcase binaries in microprobe for all instructions
  in the RISC-V ISA. Detailed instructions on running microprobe for RISC-V are
  provided at https://github.com/IBM/microprobe.git

  1. Default parameter values:

     - DD=0 (Dependency distance),
     - LS=10000 (Loop size to equal instructions to be simulated, to prevent
       additional branching instructions from being included in the evaluation).

     You can modify the script default parameters by editing the example scripts
     provided in the following steps.

  2. Generate single instruction testcases:

     ```
     cd $ERASER_HOME/testcases && ./run_all_inst_testcases.sh
     ```

     The previous command will generate the per instruction test cases into
     `$ERASER_HOME/testcases/src` directory.

  3. Generate binaries for testcases in bin folder:

     ```
     cd $ERASER_HOME/testcases/bin && make src_dir=$ERASER_HOME/testcases/src	
     ```

     If compilation fails, ensure that you have installed the rocket-tools
     depedency correctly and the the environment variables used point to the
     right places.

* Generate output VCD files by running the generated testcases in Rocket chip
  emulator:

  1. Compile Emulator with debug to enable VCD file generation, if not done during
     the setup script.

     ```
     cd $ROCKETCHIP_HOME/emulator && make debug
     ```

  2. Generate VCD files for all instruction binaries:

     ```
     mkdir -p /tmp/VCD_FILES
     for INST in $(cat $SERMINER_CONFIG_HOME/inst_list.txt); do
        DD=0; $ROCKETCHIP_HOME/emulator/emulator-freechips.rocketchip.system-DefaultConfig-debug --dump-start=30000 --max-cycles=40000 -c -v /tmp/VCD_FILES/${INST}_${DD}.vcd $ERASER_HOME/testcases/bin/riscv_ipc-p-${INST}_${DD}
     done;
     ```

     Simulations should be run for the order of 1000s of instruction (~10K in our
     experiment, preceded by 30K cycles warmup) to generate VCD files
     with sufficient activity.

* Parse VCD files using the VCD parser to generate latch activity statistics.
  Todo do so, execute:

  ```
  mkdir -p /tmp/VCD_STATS
  for inst in $(cat $SERMINER_CONFIG_HOME/inst_list.txt); do
    DD=0; $ERASER_HOME/utils/vcdstats --activities /tmp/VCD_FILES/${inst}_${DD}.vcd > /tmp/VCD_STATS/${inst}.stats
  done;
  ```

* Run the latch activity modeling tool in SERMiner to obtain aggregate latch and macro activity statistics.
  The Residency Threshold is a user-defined parameter between 0 and 1. In the example, we use a value of
  0.99, but that can vary depending on the relative distributions of switching activities
  across instructions. The command to execute should be as the following:

  ```
  $SERMINER_HOME/src/gen_latch_macro_ranking_riscv.pl <DIR CONTAINING VCD STATS> <OUTPUT DIR> <RESIDENCY THRESHOLD>
  ```

  for example:

  ```
  $SERMINER_HOME/src/gen_latch_macro_ranking_riscv.pl /tmp/VCD_STATS /tmp/VCD_OUT 0.99
  ```

* Generating the stressmarks. Once the estatistics for each instruction have
  been generated, we can use them to define a sequence of instructions that
  maximize the metric of insterest, generating a stressmark. For instance:

  1. Run `gen_mpseq_cmd_riscv.sh` to generate the Microprobe stressmark generation command:

     ```
     $SERMINER_HOME/src/gen_mpseq_cmd_riscv.sh <RANKED LATCH OUTPUT DIR> <STRESSMARK OUTPUT DIR> <RESIDENCY THRESHOLD> <NUM PERMUTATIONS> <DEPENDENCY DISTANCE>
     ```

     For example:

     ```
     $SERMINER_HOME/src/gen_mpseq_cmd_riscv.sh /tmp/VCD_OUT/res_th_0.99 /tmp/VCD_OUT/stressmarks 0.99 10 0
     ```

  2. Run the generated command in the Microprobe environment, which should be already
     set, to generate stressmarks.

* Generate latch statistics for stressmark:

  1. Repeat previous steps on the generated stressmarks: compile, emulate to generate VCD files
     and then parse the results.

  2. Initial latch ranking list is determined from the latches in the ranking list
     output obtained from running the generated stressmarks

* Validation on Chiffre (optional):

  1. Detailed instructions on how to run fault injection experiments
     are provided at https://github.com/IBM/chiffre

  2. While the standalone version provides a more general solution across multiple
     cores and architectures, LeChiffre provides a fault injection methodology
     specific to the Rocket core using a RoCC-attached accelerator.

  3. List of latched targeted for injection can be set with the isFaulty method

