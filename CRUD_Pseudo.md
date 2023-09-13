# relative-entropy-water-test
### Notes for later 
- Test if program runs correctly on one processor 
- Then, potentially use multi-threading to run on multiple processors for increased speed 


### Relative Entropy- Applying forces from 3-dimensional density ###
- In density-guided simulations, additional forces are applied to atoms that depend on the gradient of similarity between a simulated density and a reference density. (GROMACS)
- Force calculations are based on computing a simulated density and its derivative with respect to the atoms positions, as well as a density-density derivative between the simulated and the reference density (GROMACS)

### Relative Entropy- Mathematics Overview ###
- Forces are dependent on:
	- foward model (atom positions translated into simulated density, p_sim(r))
	- similarity measure that describes how close the simulated density is to the reference density
	- forces scaled by force constant, k
- Effective potential energy is result
	- U = U_forcefield(r) - kS[p_ref, p_sim(r)]
	- S = similarity measure
- Corresponding density based forces that are added to simulation are:
	- F_density = k(∇r)S[p_ref, p_sim(r)]
	- This derivative decomposes into a similarity measure derivative and a simulated density model derivative, summed over all density voxels

### Understanding Simulated Density's Force Contribution ###

--------------------------------------------------------------------------------------------------------------------------------------------------
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
- CollectionMgr is created to calculate velocities, forces, and positions of each atom based on dest output rank
- Instance is this used to sendDataStream to be received by CollectionMaster object
	- CollectionMgr.C
	- 1. records the dest output rank of each atom 
	- 2. Distribute the atoms to the corresponding output 			process	
	- 1 and 2 are neeeded for positions and velocities
	- submitVelocities, submitForces, submitPositions
	 	- velocities, forces, and positions are calculated
	- sendDataStream & receiveDataStream
		- instances received using: CProxy_CollectionMaster cm(master); 
		- CollectionMaster.h


### disposePositions / UPDATE ###
- 
	- CollectionMaster.h / CollectionMaster.C
		- class CollectVectorInstance
		- class CollectVectorSequence

### Output.C 
- Object outputs the data collected on the master node (CollectionMaster)




<!-- 2. **CREATE** atom selection 
3. **CREATE** molecule
4. calcForce function returns idxs and foces 
5. store forces 
6. write external forces to data file
 -->