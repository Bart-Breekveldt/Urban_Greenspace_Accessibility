#The study of urban greenspace accessibility aims to calculate the urban greenspace access for the population of cities from all over the world. 

The models take  population grids and calculate the shortest paths to parks to calculate the access to greenspace. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal. The Modular WorldPoP-OSM Gravity Model is fully modular; which means that for every city in the world the model can be executed. This has the disadvantage at the unvertainty of the OpenStreetMap data being incomplete. For that purpose the Local Multiple Gravity Model is developed,  in which you can control to have a peer-reviewed dataset, but it then obviously lacks the modularity.

The model calculates the shortest routes from combinations of grids and park-entry points preselected by the euclidean distance between them. The park-entry points are formed at default for nodes which are not in the park and not further than 25 metres away from the parks edge. A larger park is more attractive than a smaller one, the amount this increased attractiveness is stated as the amount of Gravity a park posesses to attract people from its surroundings. In this model the Gravity is solely determined by park size, not on other factors that can also influence park attractiveness in the real-world.

The models form a blockwise structure. This means that code is divided into large blocks all of which contain a chunk of the whole code, instead of a function-based model where functions are predefined and the code is placed in one block as a whole. For time-consuming blocks, time is measured every X-iterations depending on the block. The explanation structure follows these blocks for first the Modular WorldPoP-OSM Gravity Model.

**Block -1 packages (dependencies):**
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

**Block 0: cities and assumptions**
Cities given here are taken from OpenStreetMap in the code. Be aware of cities having the same name like San Jose, Costa Rica and San Jose, California, USA. Larger cities can mostly just be given as city or by adding their country, but be aware to check. Think of the several 'Springfields' in the US. Have in the back of your mind that cities can have different administrative levels and borders. Like London, which doesn't include the City of London if you don't state it as 'Greater London'. Provinces and cities can also have the same name, like in Dhaka or Donetsk.

Multiple thresholds can be states as assumptions. The computational cost of multiple thresholds is only as large as the largest threshold you put in the model. The model takes all the possible combinations and calculates the routes on the largest threshold. Afterwards routes are filtered with the lower thresholds to get their scores and routes within reach. The threshold of 300m is equal to the WHO standard. The other assumptions will be mentioned and referenced from a later block.

**Block 1: City boundaries and downloading country-grids.**
The geocoder from OSMnx gives the city boundaries from the given cities. Then all the unique ISO3-codes are extracted for the purpose of extracting countries with multiple input cities only once. 'Countries' is a function from WpgpDownload which returns all ISO3 codes and country names. Then for all unique countries, the required packages will be displayed. The lines can be copied to the Jupyter Terminal where the country is downloaded to the system folder (in which your notebook runs). Code to extract directly from the system folder is in text. 
The six largest countries: Russia, Canada, the USA, China, Brazil and Australia, and especially Russia, can give an memory error when loading them into Python (downloading by the given download line works perfectly fine, it just takes longer with larger countries. With i.e. mask overlay in QGIS leaving out sparse or remote regions or specifying a certain state or province you can reduce the raster to a managable size.

**Block 2: Grid extraction:**
Grids are extracted, dissolved and formatted for use in path-finding.
Like in block 1 the ISO-country database is called in reverse to compare the ISO code with the specified country of the named city. rsplit splits a string on a certain character. Countries given must be present in the ISO3 countries list for the model to work. Then with a small for loop the index of the ISO-code is given that corresponds to the city input. The city is then clipped from the country and set to geopandas for data to be more managable. A sidenote here, the crs 4326 are standard coordinates (WGS84), crs 3043 transforms crs 4326 in meters. This comes in handy for taking buffers in meters for example.

Raw grid cells from WorldPoP are at 100m resolution. Their size can be preset in the assumptions block (grid_block_size), which is enlarged to the preferred size. Larger cells means less accuracy but improved speed. Dissolvement is done by the dissolve_key created by dividing the col and row numbers by a multiple of 100 and rounding them down. Incomplete cells are removed by only inluding cells which are at least 95% of max cell size. By the study's aim, determinine citizens access to urban greenspace, cells with a population of zero are useless and are thus removed. The centroids of the polygons are taken for pathfinding later in the script. Some 100m resolution grid cells have the smallest of margins between them, which results in a cell being a MultiPolygon. This is solved by creating a Polygon out of the max bounds of the input MultiPolygon.

**Block 3: extracting road networks**
The function of graph_from_place returns roads from the preset cities. It takes all roads networks; driving, walking and cycling to find the best park access for any given road user to that park. This can be changed in 'network_type' to only check park access when including walking, cycling or driving. Be aware that sidewalks or cycling infrastructure may not be that well documented in cities, that's why in this code 'all' is used, mainly to model pedestrian access to the parks.

The buffer distance follows the principle of equal oppertunities. If a grid is at the edge of the city and only parks and roads strict to that end are taken, grids at the edge may end up with lower scores and more difficult way-finding, because cities often don't have perfectly fitting boundaries. If the largest threshold stands at 1000m which uses euclidean distance, a perfect road will give you 1000m network cost, the max possible. The 1000m added on top of this is to ensure that weird gaps or weirdly shaped city boundaries can be somewhat mitigated to prevent road networks to be distorted largely. Then the nodes and edges are extracted from the graph and slightly formatted. Road_edges is reduced to road_conn is reduced in size for performance increase in the path-finding algorithm later. The same is true for setting a road_edge key as index of the edge origin an destination for road_conn.

**Block 4: extracting city greenspace (parks):**
For each city the park-geometries from the leisure category in OSM are taken with geometries_from_place. The buffer size of max thresholds follows the equal oppertunites principle as explained. Only Polygons and MultiPolygons are taken (parks that actually span some area, instead of a line or point). The assumption of 'one_park_buffer' can be preset in the assumptions block, this sets park geometries within, for example, 25 meters of each other as the same park. A road can run through the park to parking spots, but the park retains the same. For this assumption the buffer is taken in meters, parks that overlaps are grouped together with fuzzy_contiguity in the background, then are given a component ID and are dissolved according to this ID. Parks smaller than 0.04 hectares are too small regarding the WHO, and are thus excluded in the analysis.

**Block 5 park-entry points:**
In this block for all parks all road nodes of the city are compared in distance to that park. We often don't know the real entry points of a park, thus a secondary approach, taking all nodes that intersect with a parks buffer as entry points is the best alternative. If these nodes are not in the park (larger than 0 meters away) and are within the given assumption of park_entry_point_buffer, then the road nodes are stated as park entry_points. The park ID is replicated in the mat list variable to attach a park (theoretically a park-entry point can be the same for two parks). This list is reformatted to be added with the park-entry points in a dataframe. Park geometries are then attached.

The walkable park size is part of the assumption that people on average only walk a to certain extent and park-area larger than this measured from the park-entry point is useless to the average park user who wants quick park access. This assumption 'walkable_park_dist' can be set in the assumptions block. This assumption also prevents the Gravity model to get distorted as the gravity model for parks like Central Park (NY) or Hyde Park (London) having insane attractiveness up to kilometers away, more than access to a any nice small park just around the corner more suited for the average user. The walk area here is calculated according to the 'walkalbe_park_dist' assumption which is used in the gravity inflation factor calculation. The share of the park reachable is also calculated, but not processed any further.

The size inflation factor is set as a full gravity function, parks twice the size are twice as attractive. This is overstated as mentioned earlier. According to the paper 'How do travel distance and park size influence urban park visits? from Xingyue Tu, Ganlin Huang, Jianguo Wu and Xuan Gao from June 2020 in Elsevier parksize attracitveness difference compared against distance decay follows about the distribution of the full gravity factor to the power of 1/3 for the city of Beijing. Two others to the power of 1/2 (bias towards larger parks) and to the power of 1/5 (bias towards smallar parks) are added as upper and lower gravity alternatives. These gravity functions have a degressive trend: the larger the park, the less value the added size is to park attractiveness. At last all but the park-entry point geometry are exluded so the data can be exported as shapefile.

**Block 5.5: simplifying toplogy (optional):**
Park-entry points can be very close to each other due to the road network density, which are often multiple parts of an intersection that can be counted as one. A buffer around a point can be taken to merge certain park entry points that overlap after the merging. This park_entry_point_merge can be set in the assumptions. The same principle of fuzzy_contiguity as merging the parks is used in this blocks, but instead of dissolving on components, which would result in a node to be detached from the road network, instead the road node closest to the centroid of the nodes under consideration (nodes within each others buffer) is taken, the original dataframe is subsetted with the index returned for the closest node. This simplyfying topology is useful to improve the computational efficiency of the road network, but might alter the scores because the network distance increases a bit (or a lot in the case o a lot of closeby alleys seen as part-entry points). Using this simplification can be useful where adding a small distance doesn't alter the end scores that much and the number of park-entry points is very large. Road network efficiency can also be improved by simplifying the networks themselves, it suffers from the same trade-off as with the park-entry points.(https://geoffboeing.com/2020/06/whats-new-with-osmnx/)

Block 6 grid-parkentry combinations within euclidean threshold distance.





