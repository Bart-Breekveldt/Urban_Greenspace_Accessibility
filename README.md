The study of urban greenspace accessibility aims to calculate the urban greenspace access for the population of cities from all over the world. The models take  population grids and calculate the shortest paths to parks to calculate the access to greenspace. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal. The Modular WorldPoP-OSM Gravity Model is fully modular; which means that for every city in the world the model can be executed. This has the disadvantage at the unvertainty of the OpenStreetMap data being incomplete. For that purpose the Local Multiple Gravity Model is developed,  in which you can control to have a peer-reviewed dataset, but it then obviously lacks the modularity.

The model calculates the shortest routes from combinations of grids and park-entry points preselected by the euclidean distance between them. The park-entry points are formed at default for nodes which are not in the park and not further than 25 metres away from the parks edge. A larger park is more attractive than a smaller one, the amount this increased attractiveness is stated as the amount of Gravity a park posesses to attract people from its surroundings. In this model the Gravity is solely determined by park size, not on other factors that can also influence park attractiveness in the real-world.

The models form a blockwise structure. This means that code is divided into large blocks all of which contain a chunk of the whole code, instead of a function-based model where functions are predefined and the code is placed in one block as a whole. The explanation structure follows these blocks for first the Modular WorldPoP-OSM Gravity Model.

Block -1 used packages (dependencies):
The dependencies are the same for both models. They can be divided in 'standard' packages, which are highly used by the Python community, geopackages for the Geo-component in the algorithm and the WorldPoP required packages. All can be intalled by pip install, except for the WorldPoP algorithm source, where the installation is stated in its GitHub.

standard:
- numpy
- pandas
- math
- time
- itertools
- warnings -- for ignoring depriciation errors of used packages

geo-packages:
- geopandas
- georaster
- libpysal
- networkx -- for shortest path finding
- osmnx -- for extracting data from OpenStreetMap
- shapely (geometry, ops) -- for working with geometries

WorldPoP data:
- socket -- for communicating with the WorldPoP repo
- wpgpDownload -- WorldPoP repository (https://github.com/wpgp/wpgpDownloadPy)

Block 0: cities and assumptions
Cities given here are taken from OpenStreetMap in the code. Be aware of cities having the same name like San Jose, Costa Rica and San Jose, California, USA. Larger cities can mostly just be given as city and country, but be aware to check. Think of the several 'Springfields' in the US. Have in the back of your mind that cities can have different administrative levels and borders. Like London, which doesn't include the City of London if you don't state it as 'Greater London'. Provinces and cities can also have the same name, like in Dhaka or Donetsk.

Multiple thresholds can be states as assumptions. The computational cost of multiple thresholds is only as large as the largest threshold you put in the model. The model takes all the possible combinations and calculates the routes on the largest threshold. Afterwards routes are filtered with the lower thresholds to get their scores and routes within reach. The other assumptions will be mentioned and referenced from a later block.

Block 1: City boundaries and downloading country-grids.
The geocoder from OSMnx gives the city boundaries from the given cities. Then all the unique ISO3-codes are extracted for the purpose of extracting countries with multiple input cities only once. 'Countries' is a function from WpgpDownload which returns all ISO3 codes and country names. Then for all unique countries, the required packages will be displayed. The lines can be copied to the Jupyter Terminal where the country is downloaded to the system folder (in which your notebook runs). Code to extract directly from the system folder is in text.









