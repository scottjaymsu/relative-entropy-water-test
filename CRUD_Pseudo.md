# relative-entropy-water-test


### calcforce.py -> .cpp CRUD ###


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
