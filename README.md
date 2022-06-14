#The study of urban greenspace accessibility aims to calculate the urban greenspace access for the population of cities from all over the world. 

The models take  population grids and calculate the shortest paths to parks to calculate the access to greenspace. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal. The Modular WorldPoP-OSM Gravity Model is fully modular; which means that for every city in the world the model can be executed. This has the disadvantage at the unvertainty of the OpenStreetMap data being incomplete. For that purpose the Local Multiple Gravity Model is developed,  in which you can control to have a peer-reviewed dataset, but it then obviously lacks the modularity.

The model calculates the shortest routes from combinations of grids and park-entry points preselected by the euclidean distance between them. The park-entry points are formed at default for nodes which are not in the park and not further than 25 metres away from the parks edge. A larger park is more attractive than a smaller one, the amount this increased attractiveness is stated as the amount of Gravity a park posesses to attract people from its surroundings. In this model the Gravity is solely determined by park size, not on other factors that can also influence park attractiveness in the real-world.

The models form a blockwise structure. This means that code is divided into large blocks all of which contain a chunk of the whole code, instead of a function-based model where functions are predefined and the code is placed in one block as a whole. For time-consuming blocks, time is measured every X-iterations depending on the block. For the inserted cities, all blocks on default exectute each city one by one, otherwise it is stated at that block. The explanation structure follows these blocks for first the Modular WorldPoP-OSM Gravity Model.

**Block -1 packages (dependencies):**
The dependencies are the same for both models. They can be divided in 'standard' packages, which are highly used by the Python community, geopackages for the Geo-component in the algorithm and the WorldPoP required packages. All can be intalled by pip install, except for the WorldPoP algorithm source, where the installation is stated in its GitHub.

standard:
- numpy
- pandas
- math
- time
- itertools
- os -- for creating file paths if non-existent
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

The size inflation factor is set as a full gravity function, parks twice the size are twice as attractive. This is overstated as mentioned earlier. According to the paper 'How do travel distance and park size influence urban park visits? from Xingyue Tu, Ganlin Huang, Jianguo Wu and Xuan Gao from June 2020 in Elsevier parksize attracitveness difference compared against distance decay follows about the distribution of the full gravity factor to the power of 1/3 for the city of Beijing. Two others to the power of 1/2 (bias towards larger parks) and to the power of 1/5 (bias towards smallar parks) are added as upper and lower gravity alternatives. These gravity functions have a degressive trend: the larger the park, the less value the added size is to park attractiveness. All these values are compared to the median walkable size of all parks and result in an inflation factor for the determined distance decay variant, plus a plain entrance variant with no gravity. At last all but the park-entry point geometry are exluded so the data can be exported as shapefile.

**Block 5.5: simplifying toplogy (optional):**
Park-entry points can be very close to each other due to the road network density, which are often multiple parts of an intersection that can be counted as one. A buffer around a point can be taken to merge certain park entry points that overlap after the merging. This park_entry_point_merge can be set in the assumptions. The same principle of fuzzy_contiguity as merging the parks is used in this blocks, but instead of dissolving on components, which would result in a node to be detached from the road network, instead the road node closest to the centroid of the nodes under consideration (nodes within each others buffer) is taken, the original dataframe is subsetted with the index returned for the closest node. This simplyfying topology is useful to improve the computational efficiency of the road network, but might alter the scores because the network distance increases a bit (or a lot in the case o a lot of closeby alleys seen as part-entry points). Using this simplification can be useful where adding a small distance doesn't alter the end scores that much and the number of park-entry points is very large. Road network efficiency can also be improved by simplifying the networks themselves, it suffers from the same trade-off as with the park-entry points.(https://geoffboeing.com/2020/06/whats-new-with-osmnx/)

**Block 6 determine grid-parkentry pairs within euclidean threshold distance:**
This block checks all the grid-centroid & park-entry pairs and checks the euclidean distance between them. Network distance can't be bigger than the euclidean distance, all within the largest threshold are selected. This is done in chunks to ease the computational load: block_combinations which can be set in the assumptions. For getting al listed combinations itertools product is used. The grid and park-entry node information is then joined. The euclidean distances are inflated by the gravity inflation factors as described in block 5. If a park is twice the size, the gravity ** (1/3) factor is 1.26. If then the euclidean distance of a grid-parkentry pair is 1000 meters, the relative distance then deflates due to higher than median attractiveness 1000 / 1.26 ~ 794m. This also means that larger parks in gravity variants have a larger extent in attracting grid citizens to that park, and vice versa, because the threshold is still 1000m. 

If a grid-parkentry pair is within the largest threshold of the distance decay variants, then its added to the dataframe as pair which path needs to be calculated later. Then additional grid and parkentry data is added again. After all chunks are done, the columns are renamed, filtered and its geographic elements restated. Then for all the grid centroids the nearest network-node is determined with the .sindex.nearest function and the euclidean distance to this node is saved. There is a small chance (one in a million quite literally) that two nodes are the exact same distance away. the .sindex.nearest returns two nodes, which gives an error. A try-except clause is added to pick the numerically lowest index from the two if this is the case. Road node geometries are then attached for each grid plus adding the entry cost in euclidean terms to access the nearest road node.

The next appendage of block 6 summarizes the amount of combinations that have been found per city.

**Block 7: calculate paths of all grid-parkentry pairs within euclidean threshold distance.**
This block calculates the routes for park-entry points that are potentially within reach for grid centroids. For this purpose the Dijkstra algorithm is used. This is done with the networkx shortest path function which needs the graph created in block 3 together with the osmid's for origin and destination nodes which are the grid-roadentry node and the parkentry-node. All nodes in OpenStreetMap have a certain id: the OSM-id. The calculation weight is set as travel distance, which takes the edge lenghts of the graph as weights. Other methods like travel time can also be inserted, but this requires adding add_edge_speeds to the network graph road network. The computational load is divided into blocks, like 250.000 to ease the post-processing, this can be preset in the assumptions.

The preferred route-calculation is from gridroad-entry node to parkentry-node, but for some node combinations this isn't possible, because the road is disconnected from the road network or some crucial roads are one-way. If this isn't possible the reverse way is tried. If this also isn't possbible nearest nodes to the gridentry and parkentry are determined. The route calculation follows the following order at each nearest node in try-except clauses.

 1. gridnode to nearest to the original failed parknode
 2. The reverse of 1.
 3. nearest gridnode to the failed one and route to park
 4. The reverse of 3.

The number of nearest node-iterations can be set as an assumption. If a route is found in from the above try-except order this route is added to a list, which get a length of 1. The while loop stops when the length of the list reaches 1, thus when a route is found. For finding the nearest nodes the gridroad-entry osmid and parkentry osmid are referenced back to their node geometries, which returns a list with the distance to each other node. The second element in the list is taken as the first nearest node (the first element is always the node itself) and at each iteration the next node on this list is taken, the number of while-iterations is preset as assumption. If after the specificed number of iterations no combination is found (like an unconnected island off the coast in city boundaries with no connection to the mainland), the message is printed that no route could be found. -1 stands for missing values due to errors when summarizing with NaN.

If a route is found (99% is found between the gridnode-entry and parknode-entry nodes), the route as a whole (a list osmid's) is added to the index for computational efficiency per iteration. In addition the osmid 'to', the route number and the step is added, each to a separate matrix, but with the same lengths. And the way calculated is added: 'normal way', 'reverse way, 'altered way' or 'no way'.

After a chunk is finished, the lists are unpacked wuth small for loops and put in a dataframe. The created key of combined from and to osmid's is for joining with the road_edges, which were extracted from the graph in block 3. If preferred, the route details per chunk can be exported from 'routes'. After adding geometries the routes are dissolved. The route cost, way-calculated and number of steps are added and merged with grid and park information. After all the processing, the route costs for every route and every distance decay variant are calculated as a separate variable.

**Block 8: determine best parkentry point for each grid to park, aggregate and categorize grid scores**
For grids there can be multiple parkentry points that fit within the specified distance decay variants' range. The best pairs per grid-park combinations are extraced. The function locals() sets a string to a variable. Each of the distance decay variants are assigned as variable for these extracted combinations which are subsetted from the original dataframe. For summarizing well, NaN are filled with -1 and set to integers. 

For each threshold if the total cost of the distance decay variant is smaller than the threshold it is selected. Its score is the inverse of the cost, linearly. Scores are multiplied by the origin grids population to get population scores. The adjusted score has to do with the difference that occurs when increasing the threshold, the catchment area increases much faster. The adjusted score controls for this fact, in which different threshold-scores can be compared with each other. In addition the total hectares reached by the grid and the hectares per person are calculated. These scores are added to the dataframe per distance decay variant. Then all the scores are summarized and the number of parks the can be reached per grid is added as a metric. Then dissolved line geoms, to visualize its route, and a combined list of park and parkentry numbers is added, for each grid reference all the parks/parkentry nodes which routes add to the scores. 

At the next stage the scores are categorized between high, medium, low and no access. High access is a score >= threshold, medium access is score >= threshold/2, low score is below this and no score is no park access for a grid within the threshold. This is also done for the adjusted scores. The population is added, the geom removed after storing it as variable. The scores, adjusted and unadjusted, are grouped by its categorization, where the population is summed per category. These are then normalized against the sum of all scores to determine which share of the population has high, medium, low and no access. These shares can be compared between thresholds, between distance decay variants and between cities. Each city is added to an unstacked table for the thresholds and distance decay variants. To be sure all grids are categorized, an outer join is performed with the grid dataferame. Other measures like hectares per person can also potentially be aggregated per city, but isn't done in here. The variables are automatically stored in a newly created map if it doesn't exist yet. The exported files are both the dissolved routes as shapefile with the scores as the grid geoms with scores as the csv file without any geom. Plus the popgrid and popgrid adjusted scores get exported.

**Block 9: aggregate park scores**
In this block the best grid-park pairs among all grid-parkentry combinations dataframes for each distance decay variant are taken from block 8. They don't have to be recalculated. Summarizing the park scores follows the same principles as summarizing the grid scores, but grouping with the Park_No instead of the Grid_No. The scores, population scores, population reached, adjusted scores and walkha per person are summarized per person. Routes, grid numbers and park numbers are also added hereafter. All parks are also added with an outer join. The park scores are categorized in the same fashion as the grids. A larger share of parks often has a high score, because there are generally more grids than parks. Aggregation via parks or grids to a total city score will result in the same score, thus it isn't done twice. For the parks the potential extra metric aggregation potentials are the same, also here it isn't done yet. The variables are automatically stored in a newly created map if it doesn't exist yet here also. The exported files are the same as the grids, except for the summarized population's access as this would be duplicative as explained.

**Block 10: get preferred parks for each grid:**
Here I assume the theory of the rationally thinking and calculating person on basis of this model. He/She/It will choose the park for leisure activities with the best score. If a grid-park pair is within the threshold (the earlier subsetted dataframes per distance decay variants), the best grid-park pair is subsetted to determine the preferred park of residents of a certain grid. This is a different angle in which the most popular parks from a preference perspective can be determined which can be used to aggregate to a parks' service capacity. File paths are also created i non existent. Difference in files is that in here no aggregeation is done, thus all files are detailed per threshold, which result in a bit more files. 

**Limitations and other considerations**
The performance of the script is determined by the city size, because this means more grids, the park area, which means more chance on park entry points, but most of all to the road network density. The size of the road network doesn't matter, the Dijkstra algorithm works incrementally and only considers the city pairs within a determined euclidean distance, as the pathfinding block (7) is the most computationally expensive. The Dijkstra algorithm, and even the BiDirectionalDijkstra become exponentially slower in number of route steps due to their nature, if the network is denser, more steps are needed, thus the process becomes slower. Cities that check all the boxes and mainly the box of network density like London, UK. 

London couldn't be calculated with this algortihm. Western European cities tend to check the boxes with many parks and a dense road network. Luckily London is by far the largest city (within city limits) in Western Europe, but when calculating for agglomerations or metro areas, this can become a harsh limitation. The PC I used has 16 GB RAM with Intel i7 8750H for the benchmark. 10 cities (Philadelphia, Denver, Ghent, Amsterdam, Dhaka, Dublin, Vancouver, Tel Aviv, Shanghai, Washington D.C.) for 300m grids took about 5 hours, about 30mns per city.

Other packages like Igraph or NetworKit, a more powerful PC or other languages like R or C++ may be used for these performance intensitve cities or regions.

Other limitations include:
- Due to grid cells, you don't get the actual population to population

