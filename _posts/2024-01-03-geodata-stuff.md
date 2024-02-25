---
layout: post
title:  "geo-something"
date:   2024-01-03 14:19:17 -0700
# categories: work
---

<!-- catalog, add directable links

Random code utilizing geospatial python libraries
---> 

## Batch Spatial Join

Batch merges, joins, and cleans shapefiles. Joins multiple shapefiles in specified directory to one reference shapefile. Example of practical usage of geopandas python library. 

{% highlight python %}
import os
import glob
import pandas as pd 
import geopandas as gpd

class Batch_Spatial_Join():

    def __init__(self, directory, batch_file, file_type='*zip', 
                    join_type='left', pred='intersects', point=False):
        '''
        .merge combines shapefiles and returns geodataframe
        .join spatially joins shapefiles and compares lengths
        .clean returns an editable geodataframe of joined shapefiles
        '''

        self.dir = directory # folder of shapefiles to merge 
        self.batch = batch_file # batch number shapefile to spatially join to 
        self.file_type = file_type  # what kind of shapefiles to read: default '*zip'
        # specify pred as within and point as True when joining points to polygons 
        self.pred = pred # binary predicate: default 'intersects'
        self.point = point # default False
        self.join = join_type

        self.merge = self.merge_shapefiles() 
        self.join = self.spatial_join() 
        self.clean = self.col_clean() 

    def merge_shapefiles(self):
        '''
        Merges shapefiles from specified directory and outputs geodataframe 
        Converts polygons to points if specified
        '''

        shp_list = [gpd.read_file(shp) for shp in glob.iglob(os.path.join(self.dir, self.file_type)] 
        gdf = gpd.pd.concat(shp_list)
        self.gdf = gdf

        # converts polygons to points when specified
        if self.point == True: 
            self.gdf['geometry'] = self.gdf['geometry'].centroid

        return self.gdf

    def spatial_join(self):
        '''
        Spatially joins merged geodataframe to specified file
        '''

        batch_no = gpd.read_file(self.batch)
        # change projection 
        # batch_no = batch_no.to_crs(4326) 
        joined = gpd.sjoin(
            left_df=batch_no, # retains geometry of this column
            right_df=self.gdf, # joined shapefiles 
            how=self.join, # by default preforms 'left' join
            predicate=self.pred
        )
        # joined = joined.to_crs(4326)
        self.joined = joined
        # compares the length of the merged shapefile to the spatially joined file
        return "Length of merged shapefile: {}. Length of spatial join: {}".format(len(self.gdf),len(joined))

    def col_clean(self, var=False): 
        '''
        Returns editable geodataframe
        Condition to clean variable if specified
        '''
        
        if var is not False: 
            self.joined.dissolve(by=var).reset_index()
        return self.joined

{% endhighlight %}

Saves geodataframe to Esri shapefile
{% highlight python %}
gdf.to_file(filename='Batch#', driver='ESRI Shapefile') to save result
{% endhighlight %}
