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
### Collection / Creation ###

- The grid is being resized between each iteration using VMD's grid calculation code 
	- GridForceGrid.C
	- "scaled" radius, which is the "real" radius * 0.2 * resolution
	- Overall grid scaling would come from gridforcescale.

### Submit Positions
- " "
	- Sequencer.C
	- The idea is to "turn off" the integration for doing performance profiling in order to get an upper bound on the speedup available by moving the integration parts to the GPU.



1. **CREATE** atom coords data structure/ **READ** atom coordinates
- readCoords
	- #include <fstream>
	- #include <map>
	- create atom_coords in std::map<int, std::pair<int, int>> 		*map[x: [y,z]]*
	- create fstream object 										*ofstream MyFile("");* 
	- read pbc_info variable from fstream object lines [-3:] 		*publication info*
	- read atoms variable from fstream object lines [:-3] 			*atom coords (y,z)*
	- update coordinates in atom_coords map							*map[x: [y,z]]*
		- for i < atoms.size() {
			atom_coords.insert(i, std::pair())
		}

2. **CREATE** atom selection 
3. **CREATE** molecule
4. calcForce function returns idxs and foces 
5. store forces 
6. write external forces to data file
