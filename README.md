meshBalancing
=============

This contains a single library, dynamicRefineBalancedFvMesh which is 
based on dynamicRefineFvMesh but adds mesh balancing for parallel cases
to the update() function.

Note: The redistributeParPlus function does NOT work. For some reason it does not
redistribute the cellLevel and pointLevel fields (hexRef8.distribute crashes) so
it is only redistributing the refinement history. The cellLevel volScalarField is
mapped, but pointLevel is a bit more tricky.

A workaround may be to reconstruct the cellLevel and pointLevel manually from the
distributed refinement history.


## Usage

To use the balancing library with an existing 'DyM' solver you have to make the following
edits to the case (in addition to the changes to the source code in the section below)

  1. Add a linking statement to the new library in controlDict
  
        libs
        (
            "libdynamicFvMesh-dev.so"
        );

  2. Add the balanceParDict dictionary to the system folder (example in the library folder)
  3. Change the mesh type in dynamicMeshDict to `dynamicRefineBalancedFvMesh`
  4. Add the following two entries to the dynamicRefineFvMeshCoeffs dictionary (see the example
     in the library folder)
  
        enableBalancing true;
        allowableImbalance 0.15;


## OpenFOAM Source Changes (for 2.2.x)

  1. [ __CRASH__ ] Add guard in src/dynamicMesh/polyTopoChange/polyTopoChange/refinementHistory.C in
     refinementHistory::distribute at line 927 to catch
     when fully un-refined cells are transferred
     
        if( newVisibleCells[i] >= 0 )
        {
            visibleCells_[constructMap[i]] = newVisibleCells[i] + offset;
        }
        
  2. [ __ANNOYANCE__ ] Add `if (debug)` guard around print statements in
     `refinementHistory::countProc` in the same file as edit #1 to prevent excessive
     printing to stdout during mesh balancing operations
     
  3. [ __CRASH__ ] Add the following to src/finiteVolume/fvMesh/wallDist/nearWallDist.C
     at `nearWallDist::correct()` inside the
     `mesh_.changing()` condition but before the patch sizes are set. If the
     total number of patches on a processor increases, there is a crash unless
     we increase the size of nearWallDist first.
          
        // If the number of patches on this processor increased, we need to
        // increase the number of patches in nearWallDist to match
        if( mesh_.boundary().size() != size() )
        {
            //Resize nearWallDist to match mesh patches
            resize(mesh_.boundary().size());

            //Set any newly created (unset) patches
            forAll(*this, patchI)
            {
                if( !set(patchI) )
                {
                    set
                    (
                        patchI, 
                        fvPatchField<scalar>::New
                        (
                            calculatedFvPatchScalarField::typeName,
                            mesh_.boundary()[patchI],
                            mesh_.V()
                        )
                    );
                }
            }
        }
        
  4. [ __CRASH__ ] Mapping of patches had an error for mixed patches. In
     src/dynamicMesh/fvMeshAdder/fvMeshAdderTemplates.C in function
     `fvMeshAdder::MapVolField` around line 263 there is a section where imported patch values
     are mapped to existing patches. This correctly maps the patch value, but
     omits mapping of the refValue, refGradient, and other stored fields in 
     mixed patches. The refValue is then left
     set to a section of allocated but unset memory if all the original faces 
     of that patch get moved to another processor. A similar edit may be required in
     the same file near line 578 for the MapSurfaceField function, although errors from
     this have not shown up in my tests so far.

     Remove the manual loop
     near the end of the function(s) and use the rmap function with a reversed map
     by replacing
     
        forAll(newFld, i)
        {
            label oldFaceI = newToAdded[i];
        
            if (oldFaceI >= 0 && oldFaceI < addedFld.size())
            {
                newFld[i] = addedFld[oldFaceI];
            } 
        }
        
    with
    
        labelList addedToNew(addedFld.size(),-1);
        forAll(newFld, i)
        {
            label oldFaceI = newToAdded[i];

            if (oldFaceI >= 0 && oldFaceI < addedFld.size())
            {
                addedToNew[oldFaceI] = i;
            } 
        }
        
        newFld.rmap(addedFld, addedToNew);
     
  5.  [ __CRASH__ ] When using a chemistry solver, the DimensionedField `deltaTChem` in 
      basicChemistryModel.H is not properly mapped/distributed. This is because both
      `fvMeshDistributor::distribute` and `fvMeshAdder::add` only consider GeometricFields,
      ignoring DimensionedFields. However, `fvMesh::mapFields` looks at both Geometric
      and Dimensioned fields.

      An easy workaround is to change `deltaTChem` to a GeometricField by making the following
      edits, then recompiling the entire src directory since a lot of libraries link to this.
      
      1. In basicChemistryModel.H change `DimensionedField<scalar, volMesh> deltaTChem_;`
         to `GeometricField<scalar, fvPatchField, volMesh> deltaTChem_;` and add
         `#include GeometricField.H` and `#include fvPatchField.H` to the list of included
         headers

      2. In basicChemistryModelI.H change both instances of `return deltaTChem_;` to
         `return deltaTChem_.dimensionedInternalField();`
      

  6.  [ __METHOD ERROR__ ] In the current implementation of dynamicRefineFvMesh, on which
      this is based, a refined cell can be coarsened if the minimum refinementField value
      in any of its child cells is less than the threshhold in the dictionary. This is
      nonsense and leads to oscillatory refinement. Consider the following cell, showing the
      value of `refinementField` in each cell
      
         +------+------+
         |      |      |
         |  0   | 9.9  |
         |      |      |
         +------+------+
         |      |      |
         | 9.9  | 9.9  |
         |      |      |
         +------+------+
         
     where refinement is triggered at a value of 10 and unrefinement at a value of, say, 0.5. 
     This cell, rather than being left
     alone, will be coarsened. It should use the maximum value rather than the minimum
     value. To change this, in `dynamicRefineFvMesh.H` on line 112 change `minCellField`
     to `maxCellField`. In `dynamicRefineFvMesh.C`, make the following 4 changes. Line 
     numbers here may not match your line numbers exactly, so use some common sense.
     Changes 1-3 will all be in the existing `minCellField` function and change 4 is the only
     location in the file that referenced the `minCellField` function.
     
     1. Line 646: change `minCellField` to `maxCellField`
     2. Line 648: change `GREAT` to `-GREAT`
     3. Line 656: change `min` to `max`
     4. Line 1236: change `minCellField` to `maxCellField`
