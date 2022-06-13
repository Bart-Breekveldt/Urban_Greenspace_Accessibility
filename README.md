The study of urban greenspace accessibility aims to calculate the urban greenspace access for the population of cities from all over the world. The models take the population grids and calculate the shortest paths to parks to calculate the access to greenspace. The scores are aggregated mainly by grid, but also by park and weighted to the population of the origin grids into a high, medium, low and no access category score. Cities can be compared by this score. Two models fulfill this goal. One of the models is fully modular; which means that for every city in the world the

The used package-dependencies of this repo are:
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


