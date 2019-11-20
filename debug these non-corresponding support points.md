# debug these non-corresponding support points

## 1、Extra fanin related to scan
For example, an extra scan signals on the revised side. For RTL to gate comparison, and the gate design may have scan logic inserted. In this case, RTL and gate are supposed to be NEQ, until the scan logic is disabled. We can disable the scan logic with the one or more commands like:
```tcl
add pin constraint 0 scan_en -revised
add instance constraint 0 config_reg -revised
```
## 2、Extra fanin related to reset signals
- For redunancy, some library AND the asynchronous reset signal and the Q output of DFF's. This kind of scenario requires LEC to perform an extra modeling step before comparison can be made:
`set flatten model -seq_redundant` 
or
`analyze setup -verbose`
- If the non-corresponding supports relate to SET and RESET diagnosis points (for example one side there is reset and the other side set), asynchronous set and reset may have swapped between designs. We can try to see if this can be fixed by <u>resolving phase mapping issu</u>.
### 2-1 Resolving Phase Mapping issue
In general, phase mapping is very difficult to perform because there are so many ways to map the phase of state elements, even if all the key points can be mapped normally by name. There are three methods to perform phase mapping.
#### Method 1: Turn on phase mapping
The simplest thing is to try is the default LEC phase mapping.
```tcl
set mapping method -phase
set system mode lec
```
#### Method 2: phase mapping is done within a library cell
Identify the library cell that uses phase mapping. Then use the following commands to tell LEC that any DFF/DLAT from this library cell needs to be mapped with inverted phase.
```tcl
add mapping model D_F_LPH0002 -golden // if golden design
// instantiates this cell
add mapping model D_F_LPH0002 -revised
set mapping method -phasemapmodel
```
Note: IBM CU 11 library may require the above commands
#### Method 3: general phase mapping
This is a trial and error method. First we do a normal mapping (no phase). If there are any NEQ, then we delete the NEQ mapped points. And we tell LEC to phase map the key point that were NEQ before. After that we compare again. We do this iteratively until LEC says all compare point are EQ or we determine this method does not work. Here is a Tcl procedure that automates this iteration:
```tcl
proc incremental_map {MAX_LOOP} {
set noneq_count [get_compare_points -diff -count]
set i 0
while {$i < $MAX_LOOP && $noneq_count > 0} {
puts "\/\/ Note: iteration $i out of possible $MAX_LOOP"
vpx delete mapped point -noneq -quick
vpx set mapping method -phase
vpx map key point
# remodel or analyze setup may be needed here
vpx add compare point -all
vpx compare
vpx usage
set noneq_count [get_compare_points -diff -count]
set i [expr $i + 1]
}
}
```
Here is how we can use the Tcl script in a dofile:
```tcl
...
set mapping method -nophase
set system mode lec
add compare points -all
compare
tclmode
source incremental_map.tcl
incremental_map 20
vpxmode
```
If the above methods do not work, we may need to ask the person who created the revised design to provide a mapping file that has the phase information. Also note that this technique does not work if there are real NEQ's.

## 3、Extra fanin related to FSM state registers
The encoding for FSM's may have changed by synthesis tool. There is a method to <u>map the different FSM encodings in LEC</u>.
### 3-1 Map the different FSM Encodings in LEC

## 4、Floating signals Z(f) in RTL design
Synthesis tools typically tie floating signals RTL designs to zero and optimize them. LEC, as a verification tool, does not do that by default. We can use these methods to <u>resolve floating signals in RTL designs</u>.
### 4-1 Resolve floating signals in RTL designs

## 5、New primary inputs that end in _outputZ
This could be caused by inout modeling. We can try this to <u>resolve outputZ modeling</u>.
## 5-1 Resolve outputZ modeling

## 6、Cut points
Whenever combinational feedback loops exist in a design, LEC has to place cut points in the loops in order to do the comparison. There are some <u>techniques for resolving cut points issues</u>.
### 6-1 techniques for resolving cut points issues

## 7、Blackbox outputs
It is possible that output pins for BBOX are changed so that there are uncorresponding support. Follow these steps to <u>resolve BBOX non-correponding supports</u>.
### 7-1 resolve BBOX non-correponding supports

## 8、Incorrect key point types
It is possible that the non-correspondings have similar names, but they of different types. LEC cannot map key points that are of different types (i.e. DFF cannot be mapped to DLAT). Try these steps to <u>resolve key point with different types</u>.
### 8-1  resolve key point with different types

## 9、Not-mapped DLAT in golden
Check to see if full_case and/or parallel_case have been turned off in the dofile. Latches need to be inferred if the cases are not completely specified. One way to ensure latches are not inferred would be to use "synopsys paralell_case full_case" in the RTL code. But if set directive off is used, then latches could be inferred.

## 10、Unexpected fanins
This could be a real NEQ introduced by the tool or person who created the revised design. We will have to <u>determine if this is a real difference</u>.

### 10-1 Determine if this is a real difference
