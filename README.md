# BigSpicy - Merging SPEF, Verilog and Spice information into Circuit protobuf and generating the spice file <br/>

Bigspicy is a tool for merging circuit descriptions (netlists), generating Spice decks modeling those circuits, generating Spice tests to measure those models, and analyzing the results of running Spice on those tests. Bigspicy allows you to combine structural Verilog from a PDK, Spice models of standard cells, a structural Verilog model of some circuit implemented in that PDK, and parasitics extracted into SPEF format. You can then reason about the electrical structure of the design however you want. Bigspicy generates Spice decks in Xyce format, though this can (and should) be extended to other Spice dialects.

This repo shows the steps for merging the SPEF, verilog and spice netlist into a circuit protobuf and generating the spice file of the design which can further be used to perform various tests and analysis.<br/>
In this repo, I have used the design of SISO register implemented using SKY130 PDKS. The RTL to GDS2 flow of the given design can be referred from the following github repo.<br/>
https://github.com/vamsidhar32/iiitb_SISO

## FLOWCHART : <br/>
 <p align="center">
  <img  src="/images/2.png">
</p>

## PREREQUISITES : <br/>
   
To install the python dependencies, follow the below steps: <br/>
```
git clone https://github.com/ujjawal0503/Bigspicy
cd BigSpicy/
sudo apt-get update
pip install -e ".[dev]"
pip install -r requirements.txt
sudo apt install -y protobuf-compiler iverilog
```
<br/>

Another prerequisite for this step is to compile protobufs into python file.(_pb2.py).<br/>
To compile the protobufs, type the below command in terminal in the BigSpicy(cloned_repo) directory:<br/>
```
git submodule update --init  
protoc --proto_path vlsir vlsir/*.proto vlsir/*/*.proto --python_out=.
protoc proto/*.proto --python_out=.
```
We also need tools like Xyce and XDM installed.<br/>
To install the mentioned tools use the following links:<br/>
XYCE: <br/>
```
Install the 'Serial' or 'Parallel' versions of Xyce. Follow the https://xyce.sandia.gov/documentation/BuildingGuide.html
```

https://xyce.sandia.gov/documentation/BuildingGuide.html <br/>
XDM: <br/>
XDM is needed to prepare Spice netlists generated for common proprietary Spice engines for use with Xyce. It is only needed to prepare the input libraries used by bigspicy, and only once for each corner in the PDK. 

Follow the XDM installation instructions on their GitHub clone. If existing XDM-translated libraries are available, you can skip this step. 

```
sudo apt install libboost-dev libboost-python-dev
sudo pip3 install pyinstaller  # 'sudo' needed to be install in site packages dir
wget https://github.com/Xyce/XDM/archive/refs/tags/Release-2.5.0.tar.gz
tar xf XDM-Release-2.5.0.tar.gz
cd XDM-Release-2.5.0/
mkdir build && cd build
cmake -DBOOST_ROOT=/usr/include/boost ../
make -j $(nproc)
sudo make install
```
https://github.com/Xyce/XDM <br/>
## Converting the PDKs: <br/>
First step is to convert the PDKs into Xyce format.<br/>
To convert the PDK's go to the directory where XDM is installed and type the following command:<br/>
```
xdm_bdl -s hspice "path to the pdk"/"file to be converted" -d lib
```
## Merging <br/>
We merge the files into circuit protobuf(final.pb) which is used to generate the whole module spice models and to conduct the various tests using Xyce.<br/>
To merge the files, follow the below steps in the BigSpicy directory: <br/>
```
./bigspicy.py \
   --import \
   --spef example_inputs/iiitb_bc/iiitb_bc.spef \
   --spice lib/sky130_fd_sc_hd.spice \
   --verilog example_inputs/iiitb_bc/iiitb_bc.v \
   --spice_header lib/sky130_fd_pr__pfet_01v8.pm3.spice \
   --spice_header lib/sky130_fd_pr__nfet_01v8.pm3.spice \
   --spice_header lib/sky130_ef_sc_hd__decap_12.spice \
   --spice_header lib/sky130_fd_pr__pfet_01v8_hvt.pm3.spice \
   --top iiitb_bc \
   --save final.pb \
```
This will generate final.pb file.<br/>
To specify the location of the final.pb file, go to bigspicy.py file and search for "def withoptions()" function. Change the "output_dir" variable to your desired path.<br/>

## Generating whole-module spice file <br/>
After generating the "final.pb" file, we now generate the spice file("spice.sp" in this case) for our design which can be further used to run tests.<br/>
This step takes the pdks, and the design as input and gives the spice file as output.<br/>
To generate the spice file, follow the below steps in BigSpicy directory: <br/>
```
./bigspicy.py --import \
    --verilog example_inputs/iiitb_bc/iiitb_bc.v \
    --spice lib/sky130_fd_sc_hd.spice \
    --spice_header lib/sky130_fd_pr__pfet_01v8.pm3.spice \
    --spice_header lib/sky130_fd_pr__nfet_01v8.pm3.spice \
    --spice_header lib/sky130_ef_sc_hd__decap_12.spice \
    --spice_header lib/sky130_fd_pr__pfet_01v8_hvt.pm3.spice \
    --save final.pb \
    --top iiitb_bc \
    --flatten_spice --dump_spice spice.sp
```
The above steps will generate "spice.sp" file in the mentioned directory.<br/>


## Generating test to measure input capacitance
```
./bigspicy.py \
    --load /tmp/bigspicy/final.pb \
    --spice_header lib/7nm_TT.pm \
    --spice_header lib/asap7sc7p5t_27_R.sp \
    --top fp_multiplier_asap7 \
    --working_dir /tmp/bigspicy \
    --generate_input_capacitance_tests
```
We take the protobuf file, PDK primitives file and the spice file of our module to generate the test manifest and circuit analysis protobuf files. We then run Xyce to perform tests.  

Capacitances is an important parameter for any chip that we design. The propagation delays of a circuit depend on the capacitances of each and every standard cell.propagation delay is the amount of time it takes for signals to pass between two Flip-Flops. 
In order to make a chip work faster, we want the worst case delay to be as minimum as possible without violating the setup and hold time. The setup and hold time are maintained so that the clock is not running faster than the logic allows. 

## Setup and Hold Time 
In digital designs, each and every flip-flop has some restrictions related to the data with respect to the clock in the form of windows in which data can change or not. There is always a region around the clock edge in which input data should not change at the input of the flip-flop. This is because, if the data changes within this window, we cannot guarantee the output. 

Setup Time: Setup time is the amount of time required for the input to a Flip-Flop to be stable before a clock edge. 

Hold TIme: Hold time is the minimum amount of time required for the input to a Flip-Flop to be stable after a clock edge. 

<p align="center">
  <img  src="/images/2.png">
</p>

Setup time and hold time are said to be the backbone of timing analysis. Rightly so, for the chip to function properly, setup and hold timing constraints need to be met properly for each and every flip-flop in the design. If even a single flop exists that does not meet setup and hold requirements for timing paths starting from/ending at it, the design will fail and meta-stability will occur.


## Measuring Input Capacitance in BigSpicy
Input Files to these steps 
"Final.pb", Spice file for our design 
Output File 
All necessary test files, "test_manifest.pb", "circuit_analysis.pb" 


## Running Xyce to perform tests
```
cd /tmp/bigspicy
for test in *.linearZ.sp *.transient_*.sp; do
  ~/XyceInstall/Serial/bin/Xyce "${test}" &
done
```
## Linear and transient analysis

we then perform the linear and transient analysis using Xyce with the help of test manifest and circuit analysis protobuf files.

## Generating wire and whole module tests
```
./bigspicy.py \
    --load /tmp/bigspicy/final.pb \
    --spice lib/7nm_TT.pm \
    --spice lib/asap7sc7p5t_27_R.sp \
    --top fp_multiplier_asap7 \
    --working_dir /tmp/bigspicy \
    --generate_module_tests \
    --test_manifest /tmp/bigspicy/test_manifest.pb \
    --test_analysis /tmp/bigspicy/analysis.pb
```
We take the protobuf file, primitives, spice, test manifest and circuit analysis file to generate test for whole module.

### Include results of external module measurements
```
./bigspicy.py \
    --load /tmp/bigspicy/final.pb \
    --spice lib/7nm_TT.pm \
    --spice lib/asap7sc7p5t_27_R.sp \
    --top fp_multiplier_asap7 \
    --working_dir /tmp/bigspicy \
    --generate_module_tests \
    --test_manifest /tmp/bigspicy/test_manifest.pb \
    --test_analysis /tmp/bigspicy/analysis.pb \
    --analyze_input_capacitance_tests
```
### Run Xyce to perform tests
```
cd /tmp/bigspicy
for test in *.linearY.sp *.transient.sp; do
  ~/XyceInstall/Serial/bin/Xyce "${test}" &
done
```
## Perform analysis on wire and whole module tests

```
./bigspicy.py \
    --load /tmp/bigspicy/final.pb \
    --spice lib/7nm_TT.pm \
    --spice lib/asap7sc7p5t_27_R.sp \
    --top fp_multiplier_asap7 \
    --working_dir /tmp/bigspicy \
    --test_manifest /tmp/bigspicy/test_manifest.pb \
    --test_analysis /tmp/bigspicy/analysis.pb \
    --analyze_module_tests \
    --input_caps_csv=input_caps.csv \
    --delays_csv=delays.csv
```

We take Final.pb, PDK primitives, test_manifest, test_analysis, PDK spice decks. Input capacitance and delays will be analysed.


### Import all Skywater 130 sky130_fd_sc_hd standard cells into circuit proto

```
./bigspicy.py \
    --import \
    --spice ${PDK_ROOT}/share/pdk/sky130A/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice
    --save sky130hd.pb
    --working_dir /tmp/bigspicy
```
### Import all Skywater 130 primitives too
Caution! Globbing every spice file as in this example is not a good idea. You will likely end up with multiple definitions for the same circuit. But you can do it if you want.
```
./bigspicy.py \
    --import \
    --spice ${PDK_ROOT}/share/pdk/sky130A/libs.ref/sky130_fd_pr/spice/sky130_fd_pr__\* \
    --spice ${PDK_ROOT}/share/pdk/sky130A/libs.ref/sky130_fd_sc_hd/spice/sky130_fd_sc_hd.spice \
    --save sky130hd.pb \
    --working_dir /tmp/bigspicy
```
Likely :
```
warning: multiple definitions for subckt sky130_fd_pr__rf_nfet_01v8_bM04W3p00, overwriting previous
```

## Future Works <br/>
Presently we are working on performing tests on the generated spice file "spice.sp".<br/>
We are trying to find the path delay for few paths using Xyce and compare the same with other available tools. <br/>
We expect this to be a lot faster method for timing analysis than the other tools available now.<br/>


## ACKNOWLEDGMENTS <br/>
- Kunal Ghosh, Director, VSD Corp. Pvt. Ltd.<br/>
- Madhav Rao, Professor, IIIT-Bangalore<br/>
- Nanditha Rao, Professor, IIIT-Bangalore<br/>
- Arya Reais Parsi 


