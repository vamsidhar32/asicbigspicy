# BigSpicy - Merging SPEF, Verilog and Spice information into Circuit protobuf and generating the spice file
This repo shows the steps for merging the SPEF, verilog and spice netlist into a circuit protobuf and generating the spice file of the design which can further be used to perform various tests and analysis.
In this repo, I have used the design of SISO shift register implemented using SKY130 PDKS. The RTL to GDS2 flow of the given design can be referred from the following github repo. https://github.com/vamsidhar32/iiitb_SISO

### FLOW CHART

## PREREQUISITES :
To install the python dependencies, follow the below steps:
```
git clone https://github.com/ujjawal0503/Bigspicy
cd BigSpicy/
sudo apt-get update
pip install -e ".[dev]"
pip install -r requirements.txt
sudo apt install -y protobuf-compiler iverilog
```

Another prerequisite for this step is to compile protobufs into python file.(_pb2.py).
To compile the protobufs, type the below command in terminal in the BigSpicy(cloned_repo) directory:
```
git submodule update --init  
protoc --proto_path vlsir vlsir/*.proto vlsir/*/*.proto --python_out=.
protoc proto/*.proto --python_out=.
```


We also need tools like Xyce and XDM installed.     
To install the mentioned tools use the following links:  
XYCE:   
https://xyce.sandia.gov/documentation/BuildingGuide.html  
XDM:  
https://github.com/Xyce/XDM   

## Converting the PDKs:

First step is to convert the PDKs into Xyce format.  
To convert the PDK's go to the directory where XDM is installed and type the following command:

```
xdm_bdl -s hspice "path to the pdk"/"file to be converted" -d lib
```

## Merging

We merge the files into circuit protobuf(final.pb) which is used to generate the whole module spice models and to conduct the various tests using Xyce.  

To merge the files, follow the below steps in the BigSpicy directory:  
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
This will generate final.pb file.  
To specify the location of the final.pb file, go to bigspicy.py file and search for "def withoptions()" function. Change the "output_dir" variable to your desired path.  


## Generating whole-module spice file

After generating the "final.pb" file, we now generate the spice file("spice.sp" in this case) for our design which can be further used to run tests.
This step takes the pdks, and the design as input and gives the spice file as output.  

To generate the spice file, follow the below steps in BigSpicy directory:  
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
The above steps will generate "spice.sp" file in the mentioned directory.

## Future Works

Presently we are working on performing tests on the generated spice file "spice.sp". 
We are trying to find the path delay for few paths using Xyce and compare the same with other available tools. 
We expect this to be a lot faster method for timing analysis than the other tools available now. 


## ACKNOWLEDGMENTS

* Kunal Ghosh, Director, VSD Corp. Pvt. Ltd.
* Madhav Rao, Professor, IIIT-Bangalore
* Nanditha Rao, Professor, IIIT-Bangalore
