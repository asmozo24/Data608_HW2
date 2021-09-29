# Data608_HW2
Mapping Data


---
title: "Data608_HW2a"
author: "Alexis Mekueko"
date: "9/19/2021"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

```{python}
import pandas as pd

import datashader as ds
import datashader.transfer_functions as tf
import datashader.glyphs
from datashader import reductions
from datashader.core import bypixel
from datashader.utils import lnglat_to_meters as webm, export_image
from datashader.colors import colormap_select, Greys9, viridis, inferno
import copy


from pyproj import Proj, transform
import numpy as np
from IPython.display import display

#import pandas as pd
import urllib
import json
import datetime
import colorlover as cl

import plotly.offline as py
import plotly.graph_objs as go
from plotly import tools

# from shapely.geometry import Point, Polygon, shape
# In order to get shapley, you'll need to run [pip install shapely.geometry] from your terminal

from functools import partial

from IPython.display import GeoJSON

import webbrowser
import pandas as pd
from tempfile import NamedTemporaryFile
import warnings
import plotly.graph_objects as go
from ipywidgets import widgets
#py.init_notebook_mode()
py.init_notebook_mode(connected=True)

print('good')




```

[Github Link](https://github.com/asmozo24/Data608_HW2)
[Web Link](https://rpubs.com/amekueko/811772)

For module 2 we'll be looking at techniques for dealing with big data. In particular binning strategies and the datashader library (which possibly proves we'll never need to bin large data for visualization ever again.)

To demonstrate these concepts we'll be looking at the PLUTO dataset put out by New York City's department of city planning. PLUTO contains data about every tax lot in New York City.

PLUTO data can be downloaded from (here)[https://www1.nyc.gov/site/planning/data-maps/open-data/dwn-pluto-mappluto.page]. We Unzipped them to the same directory as this notebook, and you should be able to read them in using this (or very similar) code. 


<!-- ```{r } -->



<!-- masterFile <- read.csv("pluto_21v2.csv", header= TRUE) -->
<!-- View(masterFile) -->
<!-- View(masterFile) -->
<!-- str(masterFile) -->
<!-- dim(masterFile) -->
<!-- sum(is.na(masterFile$appbbl))# Returning the column names with missing values -->


<!-- BX <- masterFile %>% -->
<!--       filter (borough =="BX") -->


<!-- write.csv(BX,"~/R/Data608_HW2\\BX.csv", row.names = FALSE) -->


<!-- BK <- masterFile %>% -->
<!--       filter (borough =="BK") -->
<!-- write.csv(BK,"~/R/Data608_HW2\\BK.csv", row.names = FALSE) -->

<!-- MN <- masterFile %>% -->
<!--       filter (borough =="MN") -->
<!-- write.csv(MN,"~/R/Data608_HW2\\MN.csv", row.names = FALSE) -->

<!-- QN <- masterFile %>% -->
<!--       filter (borough =="QN") -->
<!-- write.csv(QN,"~/R/Data608_HW2\\QN.csv", row.names = FALSE) -->

<!-- SI <- masterFile %>% -->
<!--       filter (borough =="SI") -->
<!-- write.csv(SI,"~/R/Data608_HW2\\SI.csv", row.names = FALSE) -->




<!-- ``` -->


```{python}


# Code to read in v17, column names have been updated (without upper case letters) for v18...We can use low_memory for low/upper case
setwd("~/R/Data605_HW2")
 BK = pd.read_csv('BK.csv', low_memory = False)
 BX = pd.read_csv('BX.csv', low_memory = False)
 MN = pd.read_csv('MN.csv', low_memory = False)
 QN = pd.read_csv('QN.csv', low_memory = False)
 SI = pd.read_csv('SI.csv', low_memory = False)

ny = pd.concat([BK, BX, MN, QN, SI], ignore_index=True)
head(ny)
# or 
#ny = pd.read_csv('pluto_21v2.csv', low_memory = False)

# Not really sure why we have to gothrough the sub-datafraame and recombine after...this could have been one call of a master file including all other sub-dataframe

# Getting rid of some outliers
ny = ny[(ny['yearbuilt'] > 1850) & (ny['yearbuilt'] < 2020) & (ny['numfloors'] != 0)]



```

There are some prework for the geographic component of this data, which we'll be relying on for datashader.

The lattitude and longitude for this dataset uses a flat x-y projection (assuming for a small enough area that the world is flat for easier calculations), and this needs to be projected back to traditional lattitude and longitude.

```{python}
 wgs84 = Proj("+proj=longlat +ellps=GRS80 +datum=NAD83 +no_defs")
 nyli = Proj("+proj=lcc +lat_1=40.66666666666666 +lat_2=41.03333333333333 +lat_0=40.16666666666666 +lon_0=-74 +x_0=300000 +y_0=0 +ellps=GRS80 +datum=NAD83 +to_meter=0.3048006096012192 +no_defs")
 ny['xcoord'] = 0.3048*ny['xcoord']
 ny['ycoord'] = 0.3048*ny['ycoord']
 ny['lon'], ny['lat'] = transform(nyli, wgs84, ny['xcoord'].values, ny['ycoord'].values)

 ny = ny[(ny['lon'] < -60) & (ny['lon'] > -100) & (ny['lat'] < 60) & (ny['lat'] > 20)]

#Defining some helper functions for DataShader
background = "black"
export = partial(export_image, background = background, export_path="export")
cm = partial(colormap_select, reverse=(background!="black"))

```

## Part 1: Binning and Aggregation
Binning is a common strategy for visualizing large datasets. Binning is inherent to a few types of visualizations, such as histograms and 2D histograms (also check out their close relatives: 2D density plots and the more general form: heatmaps.

While these visualization types explicitly include binning, any type of visualization used with aggregated data can be looked at in the same way. For example, lets say we wanted to look at building construction over time. This would be best viewed as a line graph, but we can still think of our results as being binned by year:

```{python}

trace = go.Scatter(x = ny.groupby('yearbuilt').count()['bbl'].index, y = ny.groupby('yearbuilt').count()['bbl'])

layout = go.Layout(xaxis = dict(title = 'Year Built'),yaxis = dict(title = 'Number of Lots Built'))

fig = go.FigureWidget(data = [trace], layout = layout)

#py.iplot(fig)
fig
ny.head()
#ny.tail()

```

## Question
After a few building collapses, the City of New York is going to begin investigating older buildings for safety. The city is particularly worried about buildings that were unusually tall when they were built, since best-practices for safety hadn’t yet been determined. Create a graph that shows how many buildings of a certain number of floors were built in each year (note: you may want to use a log scale for the number of buildings). Find a strategy to bin buildings (It should be clear 20-29-story buildings, 30-39-story buildings, and 40-49-story buildings were first built in large numbers, but does it make sense to continue in this way as you get taller?)




```{python}



def df_window(df):
    with NamedTemporaryFile(delete=False, suffix='.html') as f:
        df.to_html(f)
    webbrowser.open(f.name)

#df = pd.DataFrame({'a': [10, 10, 10, 11], 'b': [8, 8 ,8, 9]})
df_window(ny)

column = ny["yearbuilt"]
column1 = ny['numfloors'].max() # 104
column2 = ny['numfloors'].min() # 0.1
max_value = column.max() #2019
min_value = column.min() # 1851

year_fl = ny[["yearbuilt", "numfloors"]]




```

## Part 2: Datashader
Datashader is a library from Anaconda that does away with the need for binning data. It takes in all of your datapoints, and based on the canvas and range returns a pixel-by-pixel calculations to come up with the best representation of the data. In short, this completely eliminates the need for binning your data.

As an example, lets continue with our question above and look at a 2D histogram of YearBuilt vs NumFloors:
```{python}



fig = go.FigureWidget(
    data = [
        go.Histogram2d(x=ny['yearbuilt'], y=ny['numfloors'], autobiny=False, ybins={'size': 1}, colorscale='Greens')
    ]
)

fig


```

This shows us the distribution, but it's subject to some biases discussed in the Anaconda notebook Plotting Perils.

Here is what the same plot would look like in datashader:

```{python}

#Defining some helper functions for DataShader
background = "black"
export = partial(export_image, background = background, export_path="export")
cm = partial(colormap_select, reverse=(background!="black"))

cvs = ds.Canvas(800, 500, x_range = (ny['yearbuilt'].min(), ny['yearbuilt'].max()), 
                                y_range = (ny['numfloors'].min(), ny['numfloors'].max()))
agg = cvs.points(ny, 'yearbuilt', 'numfloors')
view = tf.shade(agg, cmap = cm(Greys9), how='log')
export(tf.spread(view, px=2), 'yearvsnumfloors')

```

That's technically just a scatterplot, but the points are smartly placed and colored to mimic what one gets in a heatmap. Based on the pixel size, it will either display individual points, or will color the points of denser regions.

Datashader really shines when looking at geographic information. Here are the latitudes and longitudes of our dataset plotted out, giving us a map of the city colored by density of structures:

```{python}


NewYorkCity   = (( 913164.0,  1067279.0), (120966.0, 272275.0))
cvs = ds.Canvas(700, 700, *NewYorkCity)
agg = cvs.points(ny, 'xcoord', 'ycoord')
view = tf.shade(agg, cmap = cm(inferno), how='log')
export(tf.spread(view, px=2), 'firery')


```

Interestingly, since we're looking at structures, the large buildings of Manhattan show up as less dense on the map. The densest areas measured by number of lots would be single or multi family townhomes.

Unfortunately, Datashader doesn't have the best documentation. Browse through the examples from their github repo. I would focus on the visualization pipeline and the US Census Example for the question below. Feel free to use my samples as templates as well when you work on this problem.

Question
You work for a real estate developer and are researching underbuilt areas of the city. After looking in the Pluto data dictionary, you've discovered that all tax assessments consist of two parts: The assessment of the land and assessment of the structure. You reason that there should be a correlation between these two values: more valuable land will have more valuable structures on them (more valuable in this case refers not just to a mansion vs a bungalow, but an apartment tower vs a single family home). Deviations from the norm could represent underbuilt or overbuilt areas of the city. You also recently read a really cool blog post about bivariate choropleth maps, and think the technique could be used for this problem.

Datashader is really cool, but it's not that great at labeling your visualization. Don't worry about providing a legend, but provide a quick explanation as to which areas of the city are overbuilt, which areas are underbuilt, and which areas are built in a way that's properly correlated with their land value.




