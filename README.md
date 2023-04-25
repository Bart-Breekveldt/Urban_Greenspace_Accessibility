### Models for categorizing the share of population that has access to urban greenspace in cities all over the world and measuring these cities' inequalities.

The model is attached to the study of "Modelling greenspace accessibility at multi-spatial contexts: a pilot study of comparing the E2SFCA- and Gravity models" presented as a conference paper on the GISRUK 2023 in Glasgow (UK, Scotland) at April 19, 2023. The conference paper can be found on Zendodo (https://zenodo.org/record/7823447). The study compares different approaches in measuring UGSA and their suitability, executing the model on 15 world cities and measuring inequalities of this access to urban greenspace. The model aims to create a reproducible and scalable workflow for comparing multiple urban centres all over the world in their access to urban greenspace (UGS) on more and more diverse cities than is done until now.  UGSA is measured at E2SFCA and Gravity models, and in the study itself comparing them. Both models don't require any manual input files, as their input is from OpenStreetMap and Google Earth Engine (GEE) which greenspace data is validated against local city data, with cities retain their relative differences while using OSM data. The code consists of installing the miniconda environment, followed by the modelling of UGSA, classification and measuring their inequalities. 

![afbeelding](https://user-images.githubusercontent.com/83957293/234334226-4f89e585-f28a-45d4-be4b-14d0d4c49f58.png)<br />
_Visual representation the cities included_

The model aims to create a reproducible and scalable algorithm, that with insertion of the cities and catchment areas can potentially calculate the UGSA on any (urban) place in the world. The aim included expanding on excisting model on more and larger cities than done until now. The aim of constructing two models is enabling to create a sensitivity analysis to compare both sources. The modular models aims to execute a most representative as possible set of cities around the world for the model being representative to that world, especially for the global south. This was restricted by data availability of local peer-review UGS data. Selected cities for research and validation were: Addis Ababa, Bologna, Cochabamba, Damascus, Detroit, Dhaka, Ghent, Indore, Kampala, Memphis, Philadelphia, Seattle, Shijiazhuang, Tel Aviv and Washington D.C. Classification is aimed to be comparable and easily interpretable between models. Inequality is measured by (Spatial) GINI coefficients (eleborated on later). 

**Setting up the environment**


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
- rasterio
- libpysal
- networkx -- for shortest path finding
- osmnx -- for extracting data from OpenStreetMap
- shapely (geometry, ops) -- for working with geometries

WorldPop data (has been used for validation, WorldPoP through GEE is more solid, used afterwards for the main model):
- socket -- for communicating with the WorldPoP repo
- wpgpDownload -- WorldPoP repository (https://github.com/wpgp/wpgpDownloadPy)

WorldPoP through Google Earth Engine
- ee (earthengine_api)
- geemap

(Spatial)-GINI packages;
- pysal
- inequality.gini

**Block 0: cities and assumptions:**
Cities can be stated in the file "cities.xlsx" and their included OSM_area can be defined, i.e. "Twin cities" can be defined by the OSM_area of "Minneapolis, St. Paul", with them under the hood inserted seperately in OSM and merged afterwards. The Y/N column can be used to include certain cities from the Excel, or the "isin" can be used to select some manually. Be aware of cities or regions with duplicate names. Multiple thresholds can be stated as assumption. The model takes the maximum threhold for calculting populations reach to UGS, afterwards summarizing and adjusting to the several thresholds. Other assumptions are used in later blocks and will be mentioned there. The image below gives the workflow of the algorithm (which feeds what)

![afbeelding](https://user-images.githubusercontent.com/83957293/175813393-94d76ea8-7b5c-40bf-9c04-4f9c4d9999ae.png)

**Block 1: city boundaries / downloading country grids:**
This block extracts city boundaries as defined in OSM_area. Boundaries for obtaining UGS are enlarged with a buffer equal to the largest set threshold to create equal oppertunities for each population center, while the buffer in obtaining road networks are enlarged with an additional 1000m to ensure smooth wayfinding. The model extracts from GEE (or with validation through WorldPoP itself) all ISO3 unique country codes, and clips and OSM_areas of all the given cities from it. The file is called "iso_countries.xlsx" and downloaded from the WorldPoP GitHub, to ensure GEE-stability over direct WorldPoP GitHub connection. The downloads get 100m resolution population grids for the stated countries.

**Block 2: city grid extraction:**
The grids are opened and reconstructed with rasterio, which has a speed advantage. The grids are then dissolved by the block-0 stated grid size assumption. Grids without population are removed, if a grids lacks residents, there are no residents who need UGS-access. Grids larger than 95% of max grid size are subsetted to only include full grids, while taking into account long-lat differences when moving in any direction. UGS is calculated from grid centroids, those are taken. When moving from raster-geoms to polygon-geoms, the smallest of buffers can occur, these are resolved by a new polygon between x and y coordinate extends.

**Block 3: road networks:**
Road networks are extracted from OpenStreetMap, with the edge effect and smoothness buffers. Road nodes and road edges are stored independently. The graph is a comprehensive network structure of all nodes and edges with street information. All roads; for walking, cycling and driving are taken for determining UGS-access. Walking is the main aim, but in most countries, the driving network is better documented, sidewalks or possibility to walk along the road are assumed. This mode can be changed, to i.e. only accomodate driving. At standard the roads are made undirected, because pedestrians can walk both ways, this can be unset. Highways are removed as well as trunk roads over 40 mph or 65km/h as they are assumed to be unsuitible for pedestrians.

**Block 4: urban greenspace extraction:**
UGSA for the non-validation model is extracted by OSM-tags. These OSM tags need to adhere to the following requirements. The tags can be found within the leisure, nature and landuse categories of OSM.
- Tag represents an area
- The area is outdoor
- The area is (semi-)publically available
- The area is likely to contain trees, grass and/or greenery
- The area can reasonably be used for walking or recreational activities

For the local model this requires a loop-import with uniformely named files from local sources. Added buffer of max threshold to ensure equal oppertunities from all grids to urban greenspace. Only polygons and multipolygons are taken, for lines and points UGS area can't be calculated. Greenspaces are assumed as one if in X-meters of each other (can be stated in assumptions). The fuzzy_contuigity libpysal fucntion combined with dissolving on its components performs this task. Parks smaller than assumed size are removed. WHO standard is 0.04 ha. Greenspaces are taken fully if they occur (partly) anywhere within the catchment area.

**Block 5 park-entry points:**
All combinations of road nodes and UGSs are compared. All nodes not in the parks (larger than 0 meters away) and until a UGS buffer distance (assumption) are considered UGS-entry points. For the Gravity-model, a 'walkable' UGS size is extracted from every park-entry point. UGS extent in radius from the UGS-entry point further away than this a radius assumption in meters are assumed not adding extra attractiveness to that UGS. This is to prevent large overreach from enormous UGSs (i.e. Central Park, New York). A medium sized UGS just around the corner is probably more attractive than a mega-UGS kilometers away, definitely for daily use. UGS attractiveness is determined by road network distance (distance decay) and walkable UGS size. The walkable UGS size is compared against its median, UGS sizes are 'inflated' or 'deflated' by amounts resulting from previous research. At E2SFCA a walkable area isn't considered, because the population demand is used in combination with the attractiveness of distance decay and UGS-size, thus the full UGSs capacity is taken into account.

An example from Chandigarh, India. Green are parks, red park-entry points, orange roads and pink walkable park size.

![afbeelding](https://user-images.githubusercontent.com/83957293/175826535-3d81e0e3-b104-476f-9f73-2ff012f0441e.png)

**Block 5.5 simplifying park-entry toplogy (optional)**
For computational efficiency road network topology can be simplified at UGS-entry points by merging nearby park-entry points. Its optional because merging can affect the UGS-access siginicantly scores when not used carefully. Road networks may be simplified more generally and more intelligent:(https://geoffboeing.com/2020/06/whats-new-with-osmnx/), but not used here.

**Block 6: determine suitable grid-parkentry pairs:**
This block checks all combinations of grid-centroids and UGSentry-points in chunks. All UGS-entry points within the Euclidean distance of the maximum threshold are determined to calculate routes over the road network later. Euclidean thresholds are inflated or deflated due to UGS-size; large UGSs deflate the Eudlidean threshold, smaller UGSs inflate the Euclidean threshold. After this the nearest road nodes from the grid centroid are determined. If, the literal one-in-a-million chance that two nodes are the same distance (distances in meters in OSM are rounded by three decimals), the first numerical-index node is taken. If any of these grid-UGS-entry combinations are within the static assumed threshold, the combination is saved for route calculation. Afterwards a Euclidean road-entry distance grid-road-entry cost is attached. The billions of potential combinations are divided into chunks to prevent memory-errors on PC's.

**Block 7: route calculation of suitable grid-parkentry pairs:**
Routes are calculated by the NetworkX shortest path algorithm, which uses the Bi-Directional Dijkstra algorithm. From both the origin and destintion nodes, the algotihm finds the length to every other connected node, attaches the values to the visited nodes and iterates over the node with the lowest attached scores which are unvisited. Paths are given with OSM-IDs, unique for every OSM-extraction. The preferred route-calculation is from gridroad-entry node to parkentry-node, but for some node combinations this isn't possible, because the road is disconnected from the road network or some crucial roads are one-way. If this isn't possible the reverse way is tried. If this also isn't possbible nearest nodes to the gridentry and parkentry are determined. The route calculation follows the following order at each nearest node in try-except clauses.

1.	gridnode to nearest to the original failed parknode
2.	The reverse of 1.
3.	nearest gridnode to the failed one and route to park
4.	The reverse of 3.

The number of nearest node-iterations can be set as an assumption. If a route is found in from the above try-except order this route is added to a list, which get a length of 1. The while loop stops when the length of the list reaches 1, thus when a route is found. For finding the nearest nodes the gridroad-entry osmid and parkentry osmid are referenced back to their node geometries, which returns a list with the distance to each other node. The second element in the list is taken as the first nearest node (the first element is always the node itself) and at each iteration the next node on this list is taken, the number of while-iterations is preset as assumption. Routes are calculated in chunks, geometries dissolved from the whole route.

**Block 8: Model scoring Gravity / E2SFCA:**
For grids there can be multiple UGSentry points that fit within the specified gravity variants' range. The best pairs per grid-UGS are extraced. The function locals() sets a string to a variable. From research the difference in attractiveness in UGS-size in comparison with distance decay is the 3rd square root of the UGS-size divided by the median (of all a city's UGS-sizes), as an UGS double in size doesn't mean someone is willing to travel twice the distance. This 3rd square root, is a variant and a 2nd sq-root and 5th sq-root variant are also given in the scoring as upper and lower boundaries. At last a raw spatial proximity variant, which doesn't take size into account is calculated as a comparison. For each variant the route-cost including grid-entry cost is compared against the UGS_size-weighted gravity variant threshold. For each gravity variant, scores decay linearly as threshold minus the cost. 
Then these are population-weighted and summarized by gravity variant and threshold. Resulting dataframes are unstacked to concatenate every additional city in a new column.

E2SFCA also departs from the best grid-UGSentry combinations, it gets the Gaussian function depending on the threshold in distance decay, the supply, demand and final E2SFCA-score which is also population-weighted to get a city's final score. Both a population-normalized and non-normalized score and mean distance, area and supply is stated.

Score results (when processed in QGIS) can look like this:

![afbeelding](https://user-images.githubusercontent.com/83957293/234409161-fb016d3e-6dcf-4411-8ba2-05ff12c90b2d.png)

**Classification**
Classification aims to get to a comparison between the Gravity and E2SFCA-models, one of the aims of this research. The results are classified in a way that is intuitively interpretable for both researchers and non-researchers. Scoring is divided in categories access to UGS: no, low, mediocre, sufficient, good and excellent. For the E2SFCA this is based on the WHO-guidelines of the amount of UGS per person 9m2 required and 30m2 preferred, where over 9m2 is mediocre and over 30m2 is good scoring. The Gravity scores are based on the amount of "perfect scores", which is an median sized UGS at 0m distance. This can be two median sized UGS at 500m distance when taling about a 1000m threshold. The amount of perfect scores results in its classification, also in UGS of no, low, mediocre, sufficient, good and excellent access.

**Measuring inequality.**
Inequality is often measured with the GINI-index, useful in economics, but ignores the spatial component. The spatial GINI adjusts for this and checks how much GINI is due to non-spatial heterogenity. It compares the near and far hetereogenity, the near being one-level orbit in terms of grid-cell scores of Queen's contiguity (all 8 surrounding cells seen as near) at each ego-cell before aggregating it to a spatial GINI inquality, effectively factoring out the spatial component in the GINI-calculation.

**Limtations and other considerations:**
Performance of the script is determined by city size (more grids), park area (more park-entry points), the catchment area depending on the largest threshold, but road network density as most influential. The total network size doesn't matter, only combinations within Euclidean distance are taken, computational cost increases exponentially outwards by the number of steps it takes to complete the route, which depends on road network density. 10 cities (Philadelphia, Denver, Ghent, Amsterdam, Dhaka, Dublin, Vancouver, Tel Aviv, Shanghai, Washington D.C.) took 5 hours per script run on average, about 30 mns per city. London, UK couldn't be calculated within reasonable time (average 14 hours), due to the sheer size of parks, city size and road network density, it checks all boxes. Other packages like Igraph or NetworKit, a more powerful PC (Windows 16 GB RAM i7 8750H used) or other languages like R or C++ may be used for these performance intensitve cities or regions.

Scores of different thresholds are additive (scores of lower threshold should be added to get the total threshold score). High categorization got a score of 1, medium of 0.5 and low of 0.25. Scores are taken on a 100-point scale per city. For the ten cities used in research scores can be seen in graph:

![afbeelding](https://user-images.githubusercontent.com/83957293/234412769-2bc1e8d6-5874-44d6-b826-9c9a08e39e48.png)

Other limitations include the assumption of park-entry points as intersections surrounding the park, some parks are fenced, and have specific entry points, but this data is much harder to come by. Factors other than park attractiveness can be of influence to the parks' attractiveness, like the presence of water, amount of vegetation or availability of specific amenities like benches in the park. Especially for larger thresholds public transport can be faster or more convinient to reach a park. When doing driving or public transport travel time may be the better option, then the possible Euclidean combinations should be travel-time based in addition to adding travel times or travel speeds from the OSMnx module. In the networkX shortest path algorithm, travel time can be set as mode. A limtation for the modular model includes that urban greenspace is extracted from OpenStreetMap which may not be completely accurate, or you depend on local sources. Categorizing parks on basis of satellite images like NDVI may be the solution, but this is probably difficult.

Creator: B.E. Breekveldt (Bart), Masters student Applied Data Science, Faculty of Science, Utrecht University (b.e.breekveldt@students.uu.nl)

Supervisor: S.M. Labib (Labib), Assistant Professor of Data Science and Health, Faculty of GeoSciences, Utrecht University (s.m.labib@uu.nl)
