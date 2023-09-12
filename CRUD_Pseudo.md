# relative-entropy-water-test

### Original implementation notes ###
Gridforce notes:
We are building the relative entropy implementation from GROMACS.
https://manual.gromacs.org/current/reference-manual/special/density-guided-simulation.html

To use entropy, we set the gridforceEntropy boolean

When this is true, the gridforcecol (k) value becomes the "A", which is the weight between different atoms.
The gridforcecharge (q) value becomes the "radius" that spreads the atoms out onto the grid.
The default in VMD is to use a "scaled" radius, which is the "real" radius * 0.2 * resolution
Overall grid scaling would come from gridforcescale.


The first step is for NAMD to calculate a grid.
To do this, we need to get all the atoms together.

Then borrow VMD's grid calculation code. once the grid is together, everything is just translating what I have from python into CUDA.

#This is the call path to get coordinates.
collection->submitPositions(Sequencer.C)
cm.receivePositions(CollectionMgr.C)
disposePositions(c)(CollectionMaster.C)
Node::Object()->output->coordinate(seq,size,data,fdata,c->lattice);(Output.C)

--------------------------------------------------------------------------------------------------------------------------------------------------
### collection / CREATE ###
- The grid is being resized between each iteration using VMD's grid calculation code 
	- GridForceGrid.C
	- "scaled" radius, which is the "real" radius * 0.2 * resolution
	- Overall grid scaling would come from gridforcescale.

### submitPositions / READ ###
- Atom positions are submitted and sent to collection to be received 
	- Sequencer.C
	- The UPPER_BOUND macro is used to eliminate all of the per atom computation done for the numerical integration in Sequencer::integrate() other than the actual force computation and atom migration.
		- Purpose of Sequencer::integrate()? 
			- Determining net force on each atom?
		- The idea is to "turn off" the integration for doing performance profiling in order to get an upper bound on the speedup available by moving the integration parts to the GPU.

### receivePositions / READ ###
- ' '
	- CollectionMgr.C

### disposePositions / UPDATE ###

### Output.C 





<!-- 2. **CREATE** atom selection 
3. **CREATE** molecule
4. calcForce function returns idxs and foces 
5. store forces 
6. write external forces to data file
 -->