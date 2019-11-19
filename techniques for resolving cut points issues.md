#   Techniques for Resolving Cut Points issues  

Cut points are automatically created by LEC when it detects combinational loops in a design. Combinational loops are undesirable and should be removed from the design whenever possible.   

## Locating Cut Points/Loops  

You can locate these combinational loops using one or more of the following methods and then remove these loops from the design if possible:   

1. The following commands will print out the loops in your designs along with the location of the cut points: 

   ```tcl
   report path -feedback -net -source -golden > golden_cut.txt
   report path -feedback -net -source -revised > revised_cut.txt
   ```

2. Alternatively, we can go to the Mapping Manger, left-click to select the CUT point and then right-click to open their schematics. The schematics will show the loop and double-clicking on individual cell will open up the Source window with the appropriate line high-lighted.   

## Loops Functioning as DLATes 

Some combinational loops can be modeled as latches. However, Conformal does not do that by default. Instead it leaves the loops alone. As a result, you may see in your non-corresponding support points CUT on one design and DLAT on the other. In such cases, you may want to try the following option to see if Conformal can model the loops as DLATes, thereby allowing the mapping to occur correctly, and removing the false NEQ.   

`  SETUP> set flatten model -loop_as_dlat   `

## Removing Unnecessary Cut Points

IO pads can add combinational loops to a design that otherwise does not have any. These combinational loops are ok to leave in design. However, if IO pads are instantiated in both the golden and revised designs, it would simplify comparison if these IO pads blackboxed. Since the same IO pads are instantiated in the both designs, we are not missing anything by blackboxing these IO pads. By blackbox'ing these IO pads, we don't have deal with cut points. To blackbox these IO pads:   

`  add notranslate module pad_cell* -both // pad_cell* are module name of the IO pad cells   `

## Some Automatic Techniques for Resolving Cut Point Issues

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

## Manually Applying Cut Points

LEC cuts loops based on net names. If the same net names exist in both designs but they are really not the same net, this can cause LEC create incorrect cut points. It is also possible that the loops are complex or the loops may be shared. Regardless of what may have happened, if automatic loop cutting method does not work, we can try manually add these cut points.

