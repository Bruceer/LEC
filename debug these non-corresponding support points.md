# Debug Non-corresponding Support Points

[TOC]
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
If there some extra FSM state register bits or the non-corresponding support key points are different bits of the same FSM state machine, then the netlist may have a different state encoding than the RTL. Confirm this with the person performing synthesis by asking him to report the FSM encoding for the FSM in the design. If this is indeed the case, then we will need to create a state encoding mapping file and tell LEC to perform similar encoding change.

Here is an example of what the FSM encoding file looks like for LEC. Note that in this example, RTL uses binary encoding and the netlist uses one-hot encoding:
```tcl
// fsm.txt
.fromstates state_reg[1] state_reg[0]
.tostates state_reg[3] state_reg[2] state_reg[1] state_reg[0]
.begin
00 0001
01 0010
10 0100
11 1000
.end
```
Once this file is created, you can read the file in LEC during setup mode:`read fsm encoding fsm.txt -golden`

## 4、Floating signals Z(f) in RTL design
Synthesis tools typically tie floating signals RTL designs to zero and optimize them. LEC, as a verification tool, does not do that by default. We can use these methods to <u>resolve floating signals in RTL designs</u>.
### 4-1 Resolve floating signals in RTL designs
Are there floating signals with value Z(f) on RTL side of the Non-corresponding Support?

Many synthesis tools automatically tie any floating signals to zeros in the RTL and optimize the logic. On the other hand, LEC doesn't by default perform this optimization. To tell LEC to perform this optimization (i.e. globally all the floating signals to zero), use the following command (before read design): `SETUP> set undriven signal 0 -golden`

Alternatively, you can tie the individual floating signals to specific values with the following command (after read design):`SETUP> add tied signal [0 | 1] name_of_the_floating_signal`

The above method is more tedious, but it is safer. Actually the best method is to remove these floating signals from RTL.

## 5、New primary inputs that end in _outputZ
This could be caused by inout modeling. We can try this to <u>resolve outputZ modeling</u>.
### 5-1 Resolve outputZ modeling
If there is a new primary input key point with the name "_outputZ" after the name of a primary output (or inout) pin in the non-corresponding support, this could be a modeling issue. By default, Conformal will create a model that will trigger a mismatch if there is a buffered tristate vs. a non-buffered tristate driving a black box or primary output.

This default "-output_z" modeling will insert an extra Z gate before a PO or the input of a black box to distinguish between a directly connected 'Z' output versus a Boolean 'Z' buffered by a Boolean gate. Each time Conformal adds this extra Z gate, you will see a Z gate created with a "outputZ" extension at the end. Also, it will show up in the modeling messages as: 

**F39: Added output Z gate(s) (Occurrence: x)**
To view details of each Z gate added, use the following: `LEC> report message -rule F39 -verbose`
To turn off this default modeling, use: `SETUP> set flatten model -nooutput_z`

## 6、Cut points
Whenever combinational feedback loops exist in a design, LEC has to place cut points in the loops in order to do the comparison. There are some <u>techniques for resolving cut points issues</u>.
### 6-1 techniques for resolving cut points issues
Cut points are automatically created by LEC when it detects combinational loops in a design. Combinational loops are undesirable and should be removed from the design whenever possible.
#### 6-1-1 Locating Cut Points/Loops
You can locate these combinational loops using one or more of the following methods and then remove these loops from the design if possible:

1、The following commands will print out the loops in your designs along with the location of the cut points:
```tcl
report path -feedback -net -source -golden > golden_cut.txt
report path -feedback -net -source -revised> revised_cut.txt
```
2、Alternatively, we can go to the Mapping Manger, left-click to select the CUT point and then right-click to open their schematics. The schematics will show the loop and double-clicking on individual cell will open up the Source window with the appropriate line high-lighted.

#### 6-1-2 Loops Functioning as DLATes
Some combinational loops can be modeled as latches. However, Conformal does not do that by default. Instead it leaves the loops alone. As a result, you may see in your non-corresponding support points CUT on one design and DLAT on the other. In such cases, you may want to try the following option to see if Conformal can model the loops as DLATes, thereby allowing the mapping to occur correctly, and removing the false NEQ.
```tcl
SETUP> set flatten model -loop_as_dlat
```
#### 6-1-3 Removing Unnecessary Cut Points
IO pads can add combinational loops to a design that otherwise does not have any. These combinational loops are ok to leave in design. However, if IO pads are instantiated in both the golden and revised designs, it would simplify comparison if these IO pads blackboxed. Since the same IO pads are instantiated in the both designs, we are not missing anything by blackboxing these IO pads. By blackbox'ing these IO pads, we don't have deal with cut points. To blackbox these IO pads:
```tcl
// pad_cell* are module name of the IO pad cells
add notranslate module pad_cell* -both 
```
#### 6-1-4 Some Automatic Techniques for Resolving Cut Point Issues
If we determine that the combinational loops are valid and we still would like LEC to keep them. But LEC fails the comparison because cut points show up in the non-corresponding support as unbalanced cut points. It is possible that LEC cuts the loops in different places between the Golden and Revised designs. If this is the case, try the following commands to turn on a special loop cutting algorithm:
```tcl
set flatten model -cut_remove_redundant
set flatten model -cut_place_algo 1 // undocumented option
```
or
```tcl
set flatten model -cut_remove_redundant
set flatten model -cut_algo 1 // undocumented option
```
or
```tcl
analyze setup -cut -verbose
```
#### 6-1-5 Manually Applying Cut Points
LEC cuts loops based on net names. If the same net names exist in both designs but they are really not the same net, this can cause LEC create incorrect cut points. It is also possible that the loops are complex or the loops may be shared. Regardless of what may have happened, if automatic loop cutting method does not work, we can try **<u>manually add these cut points</u>**.

### 6-2 manually add these cut points

Finding the right places to add the cut points can be tricky sometimes, and the process can be iterative. 

- First we need the reports that list the loops and their cut point locations. 

  ```tcl
  report path -feedback -net -source -golden > golden_cut.txt
  report path -feedback -net -source -revised > revised_cut.txt
  ```
- In these reports, examine the loops. Normally, only one cut point is needed to break a loop. One sign that indicates potential problems is multiple CUT points within a single loop. If this is the case, try to locate a net that exists in both designs. Also try to locate a net with a real instance path names, instead of numbered one generated by the LEC. Once we locate such nets, add cut points on them:
  ```tcl
  add cut point golden_net -golden
  add cut point revised_net -revised
  ```
- Usually at this point, we should run LEC again with this new manual cut points to see if the result improves (i.e. reduced number of cut points, balanced cut points, fewer NEQ, etc.) Sometimes, this is all we need to get LEC on the right path to get a clean run.
- If not (but the result improves), we can continue the process again by generating a set of new loop reports and follow the above steps to add new cut points and run LEC.
- If result seems worse, we have to contact the designer for help in locating places to put these cut points.

## 7、Blackbox outputs
It is possible that output pins for BBOX are changed so that there are uncorresponding support. Follow these steps to <u>resolve BBOX non-correponding supports</u>.
### 7-1 resolve BBOX non-correponding supports
Non-corresponding supports involving BBOX output pins can be caused by one or more of the following:

1、A BBOX output could be tied to a constant on one design. If this is so, we can use the following command to tied the other BBOX output pin to a constant.[^1]
[^1]Note: If a BBOX has been instantiated multiple times, uniquify command needs to be used before doing the add output stuck_at.
 ```tcl
add output stuck_at 0 opin_1 -module mBBOX -revised
 ```

2、BBOX output pin names could have been changed between golden and revised. If it involves only a few pins, we can manually them with add mapped point:
 ```tcl
add mapped point uBBOX uBBOX -output_pin z0 new_z0
 ```

If it involves many BBOX output pins and there are some kind of patterns to the change, then we can map these pins with renaming rule:
 ```tcl
add renaming rule r0 ... -pin -bbox BBOX_module
 ```
3、Function of a BBOX output pin could have been moved and thus that pin is no longer needed. We can do the following to ignore those pins:

 ```tcl
SETUP> assign pin direction in BBOX_module_name inoutX
[-golden | -revised]
// the above is needed if the pin is an inout pin
SETUP> add ignored input inoutX -module BBOX_module_name
[-golden | -revised]
 ```

## 8、Incorrect key point types
It is possible that the non-correspondings have similar names, but they of different types. LEC cannot map key points that are of different types (i.e. DFF cannot be mapped to DLAT). Try these steps to <u>resolve key point with different types</u>.

### 8-1  resolve key point with different types
If one design has DFF and the other has latch. Does the latch seem to be the slave latch for the same DFF? If so, tell LEC to perform latch folding:
 ```tcl
SETUP> set flatten model -latch_fold
 ```
If the above doesn't work, check to see if the reset signal goes to both the master and slave latches. If the reset signal only goes to the master latch, an additional command is needed:
 ```tcl
SETUP> set flatten model -latch_fold
SETUP> set flatten model -latch_fold_master
 ```

## 9、Not-mapped DLAT in golden
Check to see if full_case and/or parallel_case have been turned off in the dofile. Latches need to be inferred if the cases are not completely specified. One way to ensure latches are not inferred would be to use "synopsys paralell_case full_case" in the RTL code. But if set directive off is used, then latches could be inferred.

## 10、Unexpected fanins
This could be a real NEQ introduced by the tool or person who created the revised design. We will have to <u>determine if this is a real difference</u>.

### 10-1 Determine if this is a real difference
To determine if this NEQ is real can be difficult. This is especially true if the cones involved are large. So the first thing to do here is to try to isolate the NEQ to a smaller sub-cone. Depending on the types of designs, there are several methods.

1、Are the NEQ points finite state machine (FSM) registers and are the states defined using enumerated type? If so, try using the option **-enumconstraint for read design**.

2、While we should have already checked for correct mapping, it would not hurt to do this again. If the incorrect mapping involves DFF/DLAT and if the number of NEQ is a small percentage of the total number of key points, the simplest thing to try is probably this:

 ```tcl
 delete mapped point -noneq -quick
 map key point
 add compare point -all
 compare
 ```
 Note that the above can be repeated if the number of NEQ decreases, but does not go to zero after the first try. Note that if the number of NEQ points is large, it may take a long time to map. Also, if there are real NEQ, then the above technique will not work.

3、If it is easy to do, try to use the latest version of and/or different synthesis tool to generate another netlist to see if the NEQ still exist. Sometimes **it is beneficial to isolate a parsing issue from an optimization issue**. For parsing issue, try reading in, elaborating, and writing out a elaborated netlist. And compare RTL with the elaborated netlist to see if there is a problem. If there is still NEQ, then the issue is with reading/elaborating of either Conformal and synthesis tool, and not with optimization.

4、If both golden and revised designs are hierarchical and they share common modules, then <u>hierarchical comparison</u> can be used to see if the NEQ can be isolated inside a smaller module, thus making the real NEQ determination simpler.

5、If either design is a flat or hierarchical comparison does not seem to be able to isolate the NEQ inside a smaller module, we can try to <u>introduce partition points within the NEQ cone</u> of influence to reduce the cone size.

6、If there is no way to reduce the cone size, we may have to use the error pattern provided by LEC and create a simulation testbench. **Simulation can provide a second opinion** if this NEQ is real.

### 10-2 hierarchical comparison
Hierarchical comparison can be used when both golden and revised designs share some common module boundaries. Its effectiveness in helping to isolate the NEQ in smaller modules depends on:

1、number of modules that are written out for hierarchical comparison: the more modules written out, the smaller cone of logics that we have to diagnose. If many modules are skipped for hierarchical comparison, we can try out these <u>techniques to increase the number of modules written out for hierarchical comparison</u>.

2、whether logic is optimized or moved across module boundaries: if logic has been moved across module boundaries, then this diagnosis technique of trying to reduce cone size may not help because hierarchical comparison of the module would produce NEQ that may be false. In this case, that module cannot be compared hierarchically and has to compared at one level of hierarchy higher, making the cones bigger to diagnose.

Here are the steps to generate the hierarchical dofile:
 ```tcl
// read designs and apply appropriate constraints and 
// flattening options
write hier_compare dofile -constraint -noexact hier.do \
-replace dofile hier.do 
// or use the following for dynamic hierarchical comparison. 
// Please see documentation for its features. 
// run hier_compare hier.do 
 ```
If there are any NEQ submodules, go to the submodule to see if the cause of the NEQ can be determined.

#### 10-2-1 techniques to increase the number of modules written out for hierarchical comparison
- The first thing is to make sure golden design is "uniquified" before doing hierarchical dofile generation.
 ```tcl
 uniquify -all -nolib -golden
 ```
Otherwise, we may get a message likes this when doing hierarchical dofile generation:

> //Warning: Child module 'my_fifo' used in Golden module 'top' 2 times, but in Revised module 'top' 1 times (non-balanced). Skip 'my_fifo'

- Another thing is to use the proper option for write hier dofile. Below is an example of the command, if we plan on using dynamic hierarchical comparison. If not, remove -RUN_HIER_compare.
 ```tcl
 write hier dofile hier.do -replace -constraint -noexact 
 -INPUT_OUTPUT_Pin_equivalence -RUN_HIER_compare
 ```

- If a module has fewer than 50 instances, LEC by default will skip it for hierarchical comparison. 

> //Warning: Golden or Revised have modules with number of instances less than 50. Skip it for hierarchical comparison

To override the default, we can use the following to tell LEC write out all modules for hierarchical comparison.

 ```tcl
write hier dofile hier.do -replace -constraint -noexact -INPUT_OUTPUT_Pin_equivalence -RUN_HIER_compare -threshold 0
 ```

- If a module is a black box, then LEC cannot compare it hierarchically. LEC can only compare the logic outside of the black box.

> // Warning: Module 'my_fsm' is a black box in Golden. Skip it for hierarchical comparison

If this module is black boxed by mistake, check to see if the module is properly read in and that it has not been blacked via add notranslate module and add black box.

- If a module exists only in Golden, check to see if it is true by looking for it in the revised.
> // Warning: Module 'my_spare' exists in Golden only. Skip it for hierarchical comparison

Maybe this module has been flattened in the revised. If so, LEC cannot perform hierarchical comparison on this module. If not, we have to determine why the module does not exist in the revised design.

- If the module between golden and revised designs has different port names, then LEC cannot write out that module for hierarchical comparison.
> // Warning: Module 'my_ctrl' and 'my_ctrl' have mis-match ports

If this is the case, determine the name changes. Then use **add renaming rule -pin** to match up those ports.
- Sometimes module names will get changed in a such a way that LEC will not be able to see that they are supposed to be matched together.

> // Warning: Module 'my_pipe' exists in Golden only. Skip it for hierarchical comparison
> // Warning: Module 'my_pipe_test_edition_1' exists in Revised only. Skip it for hierarchical comparison* 

If this is the case, use **add renaming rule -module** to match up those modules names.

### 10-3 introduce partition points within the NEQ cone
Again the idea is to try to reduce the cone size so that we can examine the schematics to see if we can determine the cause of the NEQ. LEC has a command called "**add partition points**" that allows us to break up the cones of logic by introducing cut points in the designs.
 ```tcl
set system mode lec
add compare point -all
compare 
// NEQ exists
add partition point -name -all -effort low
delete part point -bad_cut -effort high
add compare point -all
compare
 ```
If the NEQ compare point becomes EQ, but the NEQ is now transferred to another CUT point, then hopefully it is now easier to debug the small NEQ cut points. Or it is also possible that all the CUT points came out EQ, but the original compare point is still NEQ. This is good because the compare point cone of logic may be smaller.

