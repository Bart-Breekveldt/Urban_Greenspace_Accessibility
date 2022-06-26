### Models for categorizing the share of population that have access to urban greenspace in cities all over the world.

The model aims to categorize city populations' access to urban greenspace (UGS) or parks. The models take  population grids and calculate the shortest paths to parks to calculate the access to UGS. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal, modular and local models. The modular model can take in any world city and calculate its UGS access, whil the local model is restricted on local greenspace data. Access to UGS experiences distance decay within a Euclidean determined catchment area in their attractiveness together with park size. The script follows a blockwise structure with every block exectuing a part.

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

![afbeelding](https://user-images.githubusercontent.com/83957293/175813393-94d76ea8-7b5c-40bf-9c04-4f9c4d9999ae.png)


