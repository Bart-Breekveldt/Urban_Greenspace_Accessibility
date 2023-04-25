### Models for categorizing the share of population that has access to urban greenspace in cities all over the world and measuring these cities' inequalities.

The model is attached to the study of "Modelling greenspace accessibility at multi-spatial contexts: a pilot study of comparing the E2SFCA- and Gravity models" presented as a conference paper on the GISRUK 2023 in Glasgow (UK, Scotland) at April 19, 2023. The conference paper can be found on Zendodo (). The study compares different approaches in measuring UGSA and their suitability, executing the model on 15 world cities and measuring inequalities of this access to urban greenspace. The model aims to create a reproducible and scalable workflow for comparing multiple urban centres all over the world in their access to urban greenspace (UGS). UGSA is measured at E2SFCA and Gravity models, and in the study itself comparing them. Both models don't require any manual input files, as their input is from OpenStreetMap and Google Earth Engine which greenspace data is validated against local city data, with cities retain their relative differences while using OSM data. The code consists of installing the miniconda environment, followed by the modelling of UGSA, classification and measuring their inequalities. 

![afbeelding](https://user-images.githubusercontent.com/83957293/234331801-90cd69df-4404-43db-888b-1ab6a2cadd33.png)
Visual representation of the process (excluding inequality measuring)

The model aims to create a modular algorithm which can determine populations access to urban greensspace (UGS) or parks. Modularity means that the model is fully replicable and that all cities can be inserted into the model, all catchment areas or thresholds (increased computational cost at increased thresholds taken into account) and several assumptions can be stated. The aim included expanding on excisting model on more and larger cities than done until now. The aim of constructing two models is enabling to create a sensitivity analysis to compare both sources. The modular models aims to execute a most representative as possible set of cities around the world for the model being representative to that world, especially for the global south. This was restricted by data availability of local peer-review UGS data. Selected cities for research and validation were: Amsterdam, Denver, Dhaka, Dublin, Ghent, Philadelphia, Shanghai, Tel Aviv, Vancouver and Washington D.C.

The model aims to categorize city populations' access to UGS. The models take  population grids and calculate the shortest paths to parks to calculate the access to UGS. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal, modular and local models. The modular model can take in any world city and calculate its UGS access, whil the local model is restricted on local greenspace data. Access to UGS experiences distance decay within a Euclidean determined catchment area in their attractiveness together with park size. The script follows a blockwise structure with every block exectuing a part.

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

**Block 0: cities and assumptions:**
Cities are taken from OpenStreetMap, its definitions apply to the input of cities. Cities can have same-named counterparts or same-named regions. Multiple thresholds can be stated as assumption. The model takes the maximum threhold for calculting populations reach to UGS, afterwards summarizing and adjusting to the several thresholds. Other assumptions are used in later blocks and will be mentioned there. The image below gives the workflow of the algorithm (which feeds what)

![afbeelding](https://user-images.githubusercontent.com/83957293/175813393-94d76ea8-7b5c-40bf-9c04-4f9c4d9999ae.png)

**Block 1: city boundaries / downloading country grids:**
This block extracts city boundaries as defined in OpenStreetMap. Boundaries contain a buffer ot the maximum threshold for edge effects (create equal oppertunities for grids on the edge to visit UGS) and 1000m for smoothness in way-finding. For the modular model it extracts the 'Countries' module from WpgpDownload with all ISO3 country codes, which generates a command to be executed in the terminal. When already downloaded it will continue, otherwise it will give an error and you should enter the generated commans lines(s) in the terminal and re-run the code afterwards. The downloads get 100m resolution population grids for the stated countries. When a country file is too big to be loaded, preprocessing with a mask overlay operation in QGIS or equivalent line of code can be used to decrease the raster size. QGIS is open source. Mask overlay is clipping a raster layer with a overlayed vector layer. Like masking California from the USA. A very small buffer (like 50m) is adviced to overcome small inconsistencies between layers.

**Block 2: city grid extraction:**
For the modular model, WorldPoP is used as a source. Grids are extracted from countries with corresponding ISO3 code, The city is then clipped from the country with the city boundaries. For the local model these are imported via a loop that requires a structured file-naming system from the Global Human Settlement Layer. The grids are then dissolved by the block-0 stated grid size assumption. Grids without population are removed, if a grids lacks residents, there are no residents who need UGS-access. Grids larger than 95% of max grid size are subsetted to only include full grids, while taking into account long-lat differences when moving in any direction. UGS is calculated from grid centroids, those are taken. When moving from raster-geoms to polygon-geoms, the smallest of buffers can occur, these are resolved by a new polygon between x and y coordinate extends.

**Block 3: road networks:**
Road networks are extracted from OpenStreetMap, with the edge effect and smoothness buffers. Road nodes and road edges are stored independently. The graph is a comprehensive network structure of all nodes and edges with street information. All roads; for walking, cycling and driving are taken for determining UGS-access. Walking is the main aim, but in most countries, the driving network is better documented, sidewalks or possibility to walk along the road are assumed. This mode can be changed, to i.e. only accomodate driving.

**Block 4: urban greenspace extraction:**
For the modular model, UGS is extracted from OpenStreetMap: parks within the leisure category. For the local model this requires a loop-import with uniformely named files from local sources. Added buffer of max threshold to ensure equal oppertunities from all grids to urban greenspace. Only polygons and multipolygons are taken, for lines and points UGS area can't be calculated. Parks are assumed as one if in X-meters of each other (can be stated in assumptions). The fuzzy_contuigity libpysal fucntion combined with dissolving on its components performs this task. Parks smaller than assumed size are removed. WHO standard is 0.04 ha.

**Block 5 park-entry points:**
All combinations of road nodes and UGSs are compared. All nodes not in the parks (larger than 0 meters away) and until a park buffer distance (assumption) are considered park-entry points, with entry points attached with a UGS-ID. A 'walkable' park size is extracted from every park-entry point. Park extent in radius from the park-entry point further away than this a radius assumption in meters are assumed not adding extra attractiveness to that park. This is to prevent large overreach from enormous parks (i.e. Central Park, New York). A medium sized park just around the corner is probably more attractive than a mega-park kilometers away, definitely for daily use. This walkable is intersected with the park polygon to create a walkable park area. Attracitveness is determined by road network distance (distance decay) and walkable park size. The walkable park size is compared against its median, park sizes are 'inflated' or 'deflated' according to their size. The second and third square root of these factors mirror reality according to survey-studies in Beijing and Wuhan. These are gravity variants, where the variant defines the relative gravity (difference) of the park. A no-gravity variant and a fifth square root variant which is about equally spaced between the third suqare root and no-graviry variant is also added as a gravity variant. 

An example from Chandigarh, India. Green are parks, red park-entry points, orange roads and pink walkable park size.

![afbeelding](https://user-images.githubusercontent.com/83957293/175826535-3d81e0e3-b104-476f-9f73-2ff012f0441e.png)

**Block 5.5 simplifying park-entry toplogy (optional)**
For computational efficiency road network topology can be simplified at park-entry points by merging nearby park-entry points. Its optional because merging can affect the UGS-access siginicantly scores when not used carefully. Road networks may be simplified more generally and more intelligent:(https://geoffboeing.com/2020/06/whats-new-with-osmnx/), but not used here.

**Block 6: determine suitable grid-parkentry pairs:**
This block checks all combinations of grid-centroids and parkentry-points in chunks. All park-entry points within the Euclidean distance of the maximum threshold are determined to calculate routes over the road network later. Euclidean thresholds are inflated or deflated according to the gravity variants' park size factor: large parks deflate the Eudlidean threshold, smaller parks inflate the Euclidean threshold. If any of these are within the static assumed threshold, the combination is saved for route calculation. After this the nearest road nodes from the grid centroid are determined. If, the literal one-in-a-million chance that two nodes are the same distance (distances in meters in OSM are rounded by three decimals), the first numerical-index node is taken. Afterwards a Euclidean road-entry distance is attached grid-road-entry cost. The next appendage of block 6 summarizes the amount of combinations that have been found per city.

**Block 7: route calculation of suitable grid-parkentry pairs:**
Routes are calculated by the NetworkX shortest path algorithm, which uses the Bi-Directional Dijkstra algorithm. From both the origin and destintion nodes, the algotihm finds the length to every other connected node, attaches the values to the visited nodes and iterates over the node with the lowest attached scores which are unvisited. Paths are given with OSM-IDs, unique for every OSM-extraction. The preferred route-calculation is from gridroad-entry node to parkentry-node, but for some node combinations this isn't possible, because the road is disconnected from the road network or some crucial roads are one-way. If this isn't possible the reverse way is tried. If this also isn't possbible nearest nodes to the gridentry and parkentry are determined. The route calculation follows the following order at each nearest node in try-except clauses.

1.	gridnode to nearest to the original failed parknode
2.	The reverse of 1.
3.	nearest gridnode to the failed one and route to park
4.	The reverse of 3.

The number of nearest node-iterations can be set as an assumption. If a route is found in from the above try-except order this route is added to a list, which get a length of 1. The while loop stops when the length of the list reaches 1, thus when a route is found. For finding the nearest nodes the gridroad-entry osmid and parkentry osmid are referenced back to their node geometries, which returns a list with the distance to each other node. The second element in the list is taken as the first nearest node (the first element is always the node itself) and at each iteration the next node on this list is taken, the number of while-iterations is preset as assumption. Routes are calculated in chunks, geometries dissolved from the whole route.

**Block 8: get best grid-parkenty combination, get grid scores:**
For grids there can be multiple parkentry points that fit within the specified gravity variants' range. The best pairs per grid-park combinations are extraced. The function locals() sets a string, that includes the gravity variant, to a variable. For each gravity variant, scores within the specified threshold are subsetted. Scores decay linearly as threshold minus the cost. Population scores, hectares that can be reached and the m2 per person are calculated alongside. The adjusted score adjustes thresholds scores to the increased difference between the threshold and their area, to get a theoretical equal catchment area for all thresholds, which then can be compared. On basis of the raw score, grids are categorized in high (>= threshold score), medium (>= threshold score / 2), low (< threshold score / 2) and no (no access). This is also done for the adjusted scores. These apopulation-weighted and summarized by gravity variant and threshold. Resulting dataframes are unstacked to concatenate every additional city in a new column. Other metrics can be aggregated upwards by slightly adjusting the coce, but isn't done in here.

Score results (when processed in QGIS) can look like this:

![afbeelding](https://user-images.githubusercontent.com/83957293/175823555-b46035be-44b1-4f6c-aa67-4c2cb5432b42.png)

**Block 9: get UGS scores:**
The best pairs per grid-park combinations per gravity variant are taken from block 8, UGS scores are calculated in the same stucture as the grid scores. For the parks the potential extra metric aggregation potentials are the same, here it isn't done yet.

**Block 10: get preferred park for each grid:**
Here I assume the theory of the rationally thinking and calculating person on basis of this model when thinking of visiting greenspace. He/She/It will choose the park for leisure activities with the best score. If a grid-park pair is within the threshold (the earlier subsetted dataframes per distance decay variants), the best grid-park pair is subsetted to determine the preferred park of residents of a certain grid. This is a different angle in which the most popular parks from a preference perspective can be determined which can be used to aggregate to a parks' service capacity.

**Limtations and other considerations:**
Performance of the script is determined by city size (more grids), park area (more park-entry points), the catchment area depending on the largest threshold, but road network density as most influential. The total network size doesn't matter, only combinations within Euclidean distance are taken, computational cost increases exponentially outwards by the number of steps it takes to complete the route, which depends on road network density. 10 cities (Philadelphia, Denver, Ghent, Amsterdam, Dhaka, Dublin, Vancouver, Tel Aviv, Shanghai, Washington D.C.) took 5 hours per script run on average, about 30 mns per city. London, UK couldn't be calculated within reasonable time (average 14 hours), due to the sheer size of parks, city size and road network density, it checks all boxes. Other packages like Igraph or NetworKit, a more powerful PC (Windows 16 GB RAM i7 8750H used) or other languages like R or C++ may be used for these performance intensitve cities or regions.

Scores of different thresholds are additive (scores of lower threshold should be added to get the total threshold score). High categorization got a score of 1, medium of 0.5 and low of 0.25. Scores are taken on a 100-point scale per city. For the ten cities used in research scores can be seen in graph:

![afbeelding](https://user-images.githubusercontent.com/83957293/175823620-e85700b2-a5b3-401b-9611-7f808d612874.png)

Other limitations include the assumption of park-entry points as intersections surrounding the park, some parks are fenced, and have specific entry points, but this data is much harder to come by. Factors other than park attractiveness can be of influence to the parks' attractiveness, like the presence of water, amount of vegetation or availability of specific amenities like benches in the park. Especially for larger thresholds public transport can be faster or more convinient to reach a park. When doing driving or public transport travel time may be the better option, then the possible Euclidean combinations should be travel-time based in addition to adding travel times or travel speeds from the OSMnx module. In the networkX shortest path algorithm, travel time can be set as mode. A limtation for the modular model includes that urban greenspace is extracted from OpenStreetMap which may not be completely accurate, or you depend on local sources. Categorizing parks on basis of satellite images like NDVI may be the solution, but this is probably difficult.

Creator: B.E. Breekveldt (Bart), Masters student Applied Data Science, Faculty of Science, Utrecht University (b.e.breekveldt@students.uu.nl)

Supervisor: S.M. Labib (Labib), Assistant Professor of Data Science and Health, Faculty of GeoSciences, Utrecht University (s.m.labib@uu.nl)
