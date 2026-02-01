---
layout: single
classes: wide
title:  "CycloRank"
date:   2021-11-07
categories: projects
comments: true
description: "Ranking European cities by cycling infrastructure using crowdsourced OpenStreetMap data."
---

I measure and rank the cycling infrastructure of European cities by using crowdsourced data from OpenStreetMap. The
cities included have either a population above 400K people or are EU capitals. 

[Source code](https://github.com/skandium/cyclorank)

![cyclists](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/cyclists.jpg)
_Rush hour in Stockholm, [Gemma Evans, Unsplash](https://unsplash.com/@stayandroam)_

## Motivation

I've been adhering this year to a personal challenge to cycle as my main mode of inner-city transport. This coincided
with a period of massive resurgence in cycling popularity across Europe, as cities
like [Paris](https://www.euronews.com/green/2021/10/25/paris-is-investing-250-million-to-become-a-100-cycling-city)
or [Berlin](https://momentummag.com/berlin-unveils-3000-kilometer-cycling-network/)
have made significant investments to boost the modal share of cycling after the lockdowns. Even in my home city of
Tallinn, where supposedly only [1%](https://transpordiamet.ee/media/526/download) of trips are by bike, it became a
dominant topic in this year's local election. As in any politicised debate there is a tendency to revert from facts to
suitable narratives, I started wondering if there's an objective view based on data that would help to evaluate and rank
the efforts of different cities in making their environments more bike-friendly. After all, every city probably looks up
to Copenhagen or Amsterdam in this respect, but how well are the others doing?
[OpenStreetMap](https://en.wikipedia.org/wiki/OpenStreetMap), the world's largest open-source geographic database might
provide some of these answers.

### Similar projects

There already are a few similar rankings that I could find. For example,
the [Copenhagenize Index](https://copenhagenizeindex.eu/) gives subjective scores to a variety of areas such as
streetscape, culture and ambition. The city list comprises 600 cities with 600 000 inhabitants worldwide. Another one
is the [Bicycle Cities Index](https://www.coya.com/bike/index-2019) by digital insurance company Coya. The city list
selection is arbitrary, but the measurement is done on a number of objective indicators sourced across the
internet such as road infrastructure, bicycle usage, number of fatalities etc.

The reasons for developing yet another one:

* Copenhagenize Index and Bicycle Cities Index were last updated in 2019
* We want to treat fairly both the city selection (based on population size) and the metric measurement, refraining from
  subjective expert-assigned scores or manual lookup of data from different sources
* Using a standardised methodology based on OpenStreetMap will allow to repeat the experiment at a different time and
  scale it to an arbitrary number of cities, with little manual additional effort

## Methodology

To measure the cyclability of cities, I calculate the share of road infrastructure that has been marked explicitly as
cycling-friendly (either a bike lane or bike road) on [OpenStreetMap](https://wiki.openstreetmap.org/wiki/Bicycle) (OSM). The
entire navigable length is considered, thus a one-way road has half the length of a two-way. OSM has been deemed a fairly reliable
data source both [overall](https://wiki.openstreetmap.org/wiki/Research) and for [cycling](https://www.researchgate.net/publication/331293632_Using_OpenStreetMap_to_inventory_bicycle_infrastructure_A_comparison_with_open_data_from_cities).

The motivation behind measuring cycling path length is twofold: firstly, it has been shown to be [correlated](https://www.sciencedirect.com/science/article/pii/S2214140519301033) to the
popularity of cycling and secondly, the presence of dedicated bike lanes is correlated with increased safety, which should have a further reinforcement effect
on cycling popularity. While building many cycling lanes alone is not enough to facilitate a transition to widespread
cycling, it is probably a necessary condition.

To qualify as a cycling road, an OpenStreetMap [way](https://wiki.openstreetmap.org/wiki/Way) has to either:

* have lanes dedicated to cyclists or shared with buses (counted as lanes)
* have a sidewalk that explicitly allows cycling (counted as lanes)
* be a separate track for cyclists, a cycle street (mostly Belgium/Netherlands) or a bicycle road (mostly Germany)
* be a path or footway that has been designated to cyclists
  by [signs](https://wiki.openstreetmap.org/wiki/Tag:bicycle%3Ddesignated#:~:text=bicycle%20%3D%20designated%20is%20applied%20where,as%20cycleway%3Aboth%20%3D%20lane%20.)

It is worth noting that this definition of cycling road is far more restrictive than what a router such
as [OSRM](https://map.project-osrm.org/?z=14&center=59.439599%2C24.770308&loc=58.669798%2C24.079285&hl=en&alt=0&srv=1)
or Google Maps would use. That's because for high connectivity and actually calculating sensible routes between any
points A and B, routers often guide through ordinary, potentially unsafe streets. As a matter of fact, the part of the
road network considered in this study is only roughly 10% of all considered cyclable in OSRM.

### Caveats

As splitting between cycle lanes and cycle tracks (including segregated ones) is the only distinction we make from
OpenStreetMap data, it's not possible to quantify all the nuances of these roads. For example, a bike lane could be very
well designed: having narrower nearby car lanes, using speed limits or other speed reduction methods, penalising
parking on the lane etc. But it could also just be some red paint on a sidewalk that runs into lightning poles and bus
stops. Even if some of these fine-grained details could be captured from OSM, it's doubtful that they would be logged
with similar quality across Europe. In addition, cultural factors which are not included in the OSM data model, but
affect both the road network and cycling convenience probably exist.

### Spatial weighting

A possible caveat of the above approach is the topology of the city. Theoretically, there could be many recreational
bike roads near the outskirts of the city where space is abundant, but nothing in the city centre. This would not
facilitate cycling as a viable mean of transportation. For this reason, I apply exponential decay weights to the road
lengths, where the weights are a function of the road's distance from the city centre. More precisely, the formula for
weighting is:

`exp(decay_coef * min(max(distance_from_centre - t_min), t_max))`

where the t_min = 10th percentile of road distances from the centre and t_max = max(90th percentile of distances from
the centre, 15km). I calibrate the decay coefficient so that the above formula equals to 0.1 at t_max and 1.0 at t_min.
This guarantees two things:

* A road in the city centre has 10 times higher weight than a road on the 90th percentile of distance from city centre.
* Censoring t_max at 15km makes sure we do not put weight on faraway roads, in case the city polygon is very large.

Some sample cycling paths with their weights are visualised below on the example of Tallinn, where the circle around the
centroid with a radius of 8km (the 90th percentile) denotes the distance at which a road has 10 times less weight than a road
in the city centre.

![circle](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/road_samples.png)

### Additional metrics

While the main ranking is based on the above, I also calculate a few auxiliary metrics:

* The share of segregated cycling tracks (a subset of all tracks). This should be the gold standard of a cycling road.
* The share of cycling lanes (as opposed to tracks).
* The number of parking spots per km2.

While these are presented in the [full results](https://mlumiste.com/cyclorank), they're excluded from the ranking
calculation due to two reasons:

* There are some regional idiosyncrasies between how bike infra is built and logged to OpenStreetMap. While the overall
  navigable bike road share seems a fairly universal metric, the others yielded unintuitive rankings of the cities.
* I'm not qualified to weigh different types of roads - what's the value of a bike lane vs a segregated track?

### Software used

* [OpenStreetMap](https://www.openstreetmap.org) to download the country maps
  from [Geofabrik](https://www.geofabrik.de/)
* [Osmium tool](https://osmcode.org/osmium-tool/) to extract the city maps from country files and parse map elements
* [polygons.openstreetmap.fr](http://polygons.openstreetmap.fr/) to fetch GeoJSON boundaries for cities
* [Nominatim](https://nominatim.openstreetmap.org/ui/search.html) to look up city boundary IDs
* [Pyrosm](https://pyrosm.readthedocs.io/en/latest/) and [mplleaflet](https://github.com/jwass/mplleaflet) to visualise
  OSM elements in a notebook
* [CyclOSM](https://www.cyclosm.org/) was used as a sanity check visualisation tool

The base map files were downloaded between September 19 and October 5, 2021.

## Results

Copenhagen's neighbour Malmö takes the top spot this time, 17.3% of the entire navigable road network (16.2% weighted)
being explicitly for bikes. Most of the other cities on the list - from The Netherlands, Belgium and Scandinavia - are
not surprising either. I was not familiar with cycling in Germany,
but [this](https://second.wiki/wiki/radfahren_in_hannover) source for Hanover claims that Hanover, Bremen and Munich as
having the highest bike modal split, all make the top. Out of larger cities, Paris was already mentioned above. Thus it
seems the ranking is capturing what we'd expect it to.

### Top 30

| City   |   OSM id |   Area (km2) |   Navigable road length (km) |   Cycle road share (weighted) |   Rank |
|:------------|---------:|-----------:|--------------------:|---------------------------:|---------------:|
| Malmo       | [10663667](https://www.openstreetmap.org/relation/10663667) |     86.457 |             4427.65 |                      0.162 |              1 |
| Copenhagen  |  [2192363](https://www.openstreetmap.org/relation/2192363) |    108.951 |             4616.33 |                      0.136 |              2 |
| Valencia    |   [344953](https://www.openstreetmap.org/relation/344953) |    139.082 |             4143.81 |                      0.134 |              3 |
| Helsinki    |    [34914](https://www.openstreetmap.org/relation/34914) |    717.645 |            14339.5  |                      0.134 |              4 |
| Antwerp     |    [59518](https://www.openstreetmap.org/relation/59518) |    203.696 |             5693.08 |                      0.133 |              5 |
| Hanover     |    [59418](https://www.openstreetmap.org/relation/59418) |    204.013 |             8361.35 |                      0.13  |              6 |
| Rotterdam   |  [1411101](https://www.openstreetmap.org/relation/1411101) |    128.844 |             5962.23 |                      0.125 |              7 |
| Utrecht     |  [1433619](https://www.openstreetmap.org/relation/1433619) |     75.032 |             3609.5  |                      0.125 |              8 |
| Stockholm   |   [398021](https://www.openstreetmap.org/relation/398021) |    215.754 |            10363.5  |                      0.124 |              9 |
| Gothenburg  |   [935611](https://www.openstreetmap.org/relation/935611) |   1093.63  |            12253.4  |                      0.124 |             10 |
| Nantes      |    [59874](https://www.openstreetmap.org/relation/59874) |     65.795 |             2780.18 |                      0.122 |             11 |
| Munster     |    [62591](https://www.openstreetmap.org/relation/62591) |    303.304 |             7347.75 |                      0.122 |             12 |
| Bremen      |    [62559](https://www.openstreetmap.org/relation/62559) |    326.285 |            10271.5  |                      0.121 |             13 |
| Amsterdam   |   [271110](https://www.openstreetmap.org/relation/271110) |    219.504 |             8661.92 |                      0.12  |             14 |
| Aarhus      |  [1784663](https://www.openstreetmap.org/relation/1784663) |    471.397 |            10146.2  |                      0.112 |             15 |
| Reykjavik   |  [2580605](https://www.openstreetmap.org/relation/2580605) |    244.465 |             4330.58 |                      0.108 |             16 |
| Bologna     |    [43172](https://www.openstreetmap.org/relation/43172) |    140.667 |             3739.61 |                      0.108 |             17 |
| Toulouse    |    [35738](https://www.openstreetmap.org/relation/35738) |    118.029 |             5297.21 |                      0.108 |             18 |
| Lyon        |   [120965](https://www.openstreetmap.org/relation/120965) |     47.981 |             2494.39 |                      0.108 |             19 |
| Mannheim    |    [62691](https://www.openstreetmap.org/relation/62691) |    144.978 |             6018.61 |                      0.101 |             20 |
| Cologne     |    [62578](https://www.openstreetmap.org/relation/62578) |    405.011 |            14756.9  |                      0.101 |             21 |
| Hague       |   [192736](https://www.openstreetmap.org/relation/192736) |     98.144 |             4693.85 |                      0.1   |             22 |
| Bonn        |    [62508](https://www.openstreetmap.org/relation/62508) |    141.067 |             5416.14 |                      0.1   |             23 |
| Seville     |   [342563](https://www.openstreetmap.org/relation/342563) |    141.287 |             4250.41 |                      0.096 |             24 |
| Dusseldorf  |    [62539](https://www.openstreetmap.org/relation/62539) |    217.488 |             8323.37 |                      0.096 |             25 |
| Munich      |    [62428](https://www.openstreetmap.org/relation/62428) |    310.712 |            16984.8  |                      0.095 |             26 |
| Nuremberg   |    [62780](https://www.openstreetmap.org/relation/62780) |    187.351 |             7610.35 |                      0.091 |             27 |
| Vienna      |   [109166](https://www.openstreetmap.org/relation/109166) |    414.863 |            16373.9  |                      0.09  |             28 |
| Leicester   |   [162353](https://www.openstreetmap.org/relation/162353) |     73.389 |             3073.72 |                      0.089 |             29 |
| Paris       |     [7444](https://www.openstreetmap.org/relation/7444) |    105.391 |             6174.52 |                      0.089 |             30 |

Full results can be found [here](https://mlumiste.com/cyclorank), including the auxiliary metrics.

### Discussion

![map](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/cyclorank_map.png)

Here point size is proportional to the cycling road share. The map shows that at least bad weather seems to have little
(or even inverse) correlation to cycling infrastructure :)

A more convincing case could be made for the relationship with population size. Indeed, most of the top cycling cities
seem to be smaller cities. Using the populations of European cities
from [Wikipedia](https://en.wikipedia.org/wiki/List_of_cities_in_the_European_Union_by_population_within_city_limits)
and plotting against our metric shows that the largest city above 12% (weighted) share has a population of just
~exp(13.8)=1 million. However, over the whole dataset there seems to be no relationship between population and cycle
road share
(dashed line is the best linear fit). Even if there are some natural scaling factors that reduce the proportion of bike
roads in the few very largest cities, there are clearly many smaller cities with few bike roads that could still better
realise their potential.

![pop_scatter](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/pop_scatter.png)

To see whether the ranking is capturing the intended concepts, it is useful to visualise the cycling infrastructure on a
map and compare it to our quantitative measurements. I
use [CyclOSM](https://www.cyclosm.org/#map=12/47.1842/-1.5288/cyclosm), which seems to be the best such tool.

### Copenhagen vs Sofia

Firstly, comparing a top-ranking city with a bottom one (in CyclOSM maps, the more blue the better) shows that indeed,
the overall level of infrastructure is captured:

![copenhagen](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/copenhagen.png)

![sofia](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/sofia.png)

### Nantes and Lyon vs Barcelona

Nantes is an interesting outlier - it makes the top overall, but almost half of that comes from bike lanes. Indeed, it
visibly has predominantly cycling lanes (marked by the dashed line), rather than separate cycleways also in CyclOSM.

![nantes](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/nantes.png)

The same city planning model seems to be used in Lyon.

![lyon](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/lyon.png)

Contrasting Nantes is Barcelona - anyone who's cycled there knows the convenience of their designated cycleways that
have little overlap with car traffic. While Barcelona does not do too well in the overall ranking, it is at the top when
measured by segregated cycling tracks.

![barcelona](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/barcelona.png)

### Milan vs Zaragoza

Finally, to visualise the benefit of the spatial weighting of roads, we can look at examples of cities that would gain
or lose the most in the ranking if we did not do any weighting. For example, Milan would gain 29 places (from 86 to 57)
and visibly this is because central Milan has almost no bike roads.

![milan](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/milan.png)

Meanwhile, Zaragoza _only_ has bike roads in the very centre, and would lose 35 places (from 57 to 92) if there was no
weighting.

![zaragoza](https://images-for-web-s3.s3.eu-central-1.amazonaws.com/zaragoza.png)