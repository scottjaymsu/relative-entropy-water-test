# relative-entropy-water-test


### calcforce.py -> .cpp CRUD ###


1. **CREATE** atom coords array / **READ atom coordinates**
- readCoords
	- #include <fstream>
	- #include <map>
	- create fstream object 										*ofstream MyFile("");* 
	- create pbc_info variable from fstream object lines [-3:] 		*publication info*
	- create atoms variable from fstream object lines [:-3] 		*atom coords (x,y,z)*
	- store atom_coords in std::map<int, std::pair<int, int>> 		*map[x: [y,z]]*

2. **UPDATE* 
