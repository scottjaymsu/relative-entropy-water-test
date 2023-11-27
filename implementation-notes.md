# Gromacs Annotations #
https://manual.gromacs.org/current/reference-manual/special/density-guided-simulation.html

## Relative Entropy- Applying Forces from 3-Dimensional Density ##
- In density-guided simulations, additional forces are applied to atoms that depend on the gradient of similarity between a simulated density and a reference density. (GROMACS)
- Force calculations are based on computing a simulated density and its derivative with respect to the atoms positions, as well as a density-density derivative between the simulated and the reference density (GROMACS)

## Relative Entropy- Mathematics Overview ##
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

## Simulated Density's Force Contribution ##
- Atoms are spared onto 3-dimensional lattice
	- Discrete Gauss transformation is used to spread atoms on grid between iterations
		- Spreading width (sigma) should be larger than the grid spacing of the reference density

## Density Similarity Measure's Force Contribution ##
- Multiple valid similarity measures between the reference density and the simulated density 
	- Specific similarity equation needs to be determined from legacy code
		1. Inner product of the simulated density 
		2. Negative relative entropy between two densities
		3. Cross correlation between two densities

## Declaring Regions to Fit ##
- A subset of atoms are chosen when pre-processing the simulation to which density--guided simulation forces are applied 
		- Only these atoms generate the simulated density that is compared to the reference density 

## Performance Factors ##
- Number of atoms in density-guided simulation group
	- N_atoms 
- Spreading range in multiples of Gaussian width, N_sigma
- Ratio of spreading width to input density grid spacing, r_sigma
- Number of voxels of input density, N_voxel
- Frequency of force calculations, N_voxel
- Communication cost when using multiple ranks c_comm (constant)

## Applying force every N-th step (Future Implementation) ##
- Cost of applying forces every integration step is reduced when applying the density-guided simulation forces only every N steps 
- The energy output frequency would need to be a mulitple of N 
- Limitations: 
	- Applying force every N-th steps does not work with the current implementation of infrequent evaluation of pressure coupling and the constraint virial

## Output ##
- Energy output file will contain an additional "density-fitting" term 
	- Energy that is added to the system from the density-guided simulations 
	- The lower the energy, the higher the similarity between simulated and reference density
------------------------------------------------------------------------------------------------------------------------------------------
# Parallel Programming #

- **Threads model** is a single process (a single program) in which can spawn multiple, concurrent "threads" (sub-programs)
	- Each thread runs independently of the others while being able to access the same memory space
- Threads can be spawned and killed by main program 
- **Process synchronization** is when multiple processess handshake at a certain point in order to commit a certain sequence of action 
	- Handshake is a signal between two devices or programs used to authenticate/coordinate
		- Example: hypervisor and an application in a guest virtual machine
	- Synchronization is found in multi-processor systems or any kind of concurrent processors
- **Thread synchronization** ensures that two or more concurrent processes or threads do not simultatneously execute some particular program segment known as critical section 
	- **Critical section/region** is a protected section that cannot be entered by more than one process or thread at a time
		- Essential for codes or processes that consist of the same variable or other resources that needed to be read or written but whose results depend on the order in which the actions occur
		- Therefore, other threads are suspended until the first leaves the critical section
		- Critical Region accesses a shared resources, such as a data structure, a peripheral device, or a network connection that would not operate in the context of multiple concurrent accessses
- **Potential synchronization methods:**
	- fork() function
	- OpenMP
	- MPI (Message Passing Interface)
		- Allows user to carry out a computation in parallel on an arbitrary number of coopoerating computers 
			- User writes a single program which runs on all the computers 
		- Data on each computer is stored serparately from that stored on the other computers 
			- Therefore, for data sharing, the computer must explicitly call the appropriate library routine requesting a data transfer 
--------------------------------------------------------------------------------------------------------------------------------------------------
# Original implementation notes #

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
# CRUD #

## collection / CREATE ##
- The grid is being resized between each iteration using VMD's grid calculation code 
	- GridForceGrid.C
	- "scaled" radius, which is the "real" radius * 0.2 * resolution
	- Overall grid scaling would come from gridforcescale.

## submitPositions / READ ##
- Atom positions are submitted and sent to collection to be received 
	- Sequencer.C
	- The UPPER_BOUND macro is used to eliminate all of the per atom computation done for the numerical integration in Sequencer::integrate() other than the actual force computation and atom migration.
		- Purpose of Sequencer::integrate()? 
			- Determining net force on each atom?
		- The idea is to "turn off" the integration for doing performance profiling in order to get an upper bound on the speedup available by moving the integration parts to the GPU.

## receivePositions / READ ##
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

## disposePositions / UPDATE ##
- 
	- CollectionMaster.h / CollectionMaster.C
		- class CollectVectorInstance
		- class CollectVectorSequence

## Output.C ##
- Object outputs the data collected on the master node (CollectionMaster)

## Potential Time Complexity Factors: ##
- Pre-determine specific size for subset of atoms when pre-processing simulation
- Apply force every N-th step 
	- Cost of applying forces every integration step is reduced when applying the density-guided simulation forces every N steps 
		- Frequency must be multiple of N 

--------------------------------------------------------------------------------------------------------------------------------------------------
# CHARM++ #
- Parallel programming framework
	- Represents the style of writing parallel programs
- Programmer: what to do in parallel
- System: where, when 
- Design principles
	- Overdecomposiiton
		- Break down work units & data units into more pieces than execution units
			- Cores/Nodes/...
	- Migratability
		- Programmer/Runtime can move work and data units
			- Runtime must keep track of where each unit is (naming and location management)
			-Runtime is adaptive
				- CollectionMaster.C???
	- Asynchrony
		- What sequence should the work units execute in?
			- Sequencer.C???
		- Message-driven execution:
			- Work-unit that has data("message") for next node should be executed next
			- Runtime System selects among ready work units
			Programmer should not specify what executes next, but can influence it via priorities
- Charm++ Model
	- Overdecomposed entiies are chares (C++ objects)
	- Chares contain methods designated as 'entry' methods 
		- Methods can be invoked asynchronously by
		  remote chares 
	- Chares are organized into indexed collections
		- organization method is unique to program 
		- CollectionMaster.C???
	- Chares communicate via asynchonous method invocations
