[![Build Status](https://travis-ci.org/bekozi/netCDF-CF-simple-geometry.svg?branch=master)](https://travis-ci.org/bekozi/netCDF-CF-simple-geometry)
[![Join the chat at https://gitter.im/bekozi/netCDF-CF-simple-geometry](https://badges.gitter.im/bekozi/netCDF-CF-simple-geometry.svg)](https://gitter.im/bekozi/netCDF-CF-simple-geometry?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# netCDF-CF-simple-geometry

**DRAFT** CF Convention for Representing Simple Geometry Types

## Scope

* Extends existing CF geometries in CF 1.7 to include lines and polygons
* Linear between nodes, not curves?
* Parametric shapes, e.g., circles, ellipses?

## Use Cases

* Encode watershed model time series and polygons in single file. Archiving the model output and geometry is the purpose.
* Encode a streamflow value for each river segment in the conterminous U.S. at a given point in time.

## Proposal

* Support line and polygon geometry in netCDF-3 with a clear path to enhanced netCDF-4. 
* Define a standard that the CF timeSeries feature type could use to specify spatial 'coordinates' of a timeSeries variable.
* Mimic well known text (WKT) style for encoding multipolygons with holes and multilines.

## What does the existing DSG specification do that's similar to this proposal? 
Discrete Sampling Geometries (DSGs) handle data from one or (a collection of) `timeSeries` (point), `Trajectory`, `Profile`, `TrajectoryProfile` or `timeSeriesProfile` geometries. Measurements are from a **point** (timeSeries and Profile) or **points** along a trajectory. In this proposal we reuse the core DSG `timeSeries` type which provides support for basic time series use cases e.g., a timeSeries which is measured (or modeled) at a given point.

## What doesn't DSG do that we need? 
DSGs have no system to define a geometry (point, polyline, polygon, etc) and an association with a time series that applies over that entire geometry e.g, The expected rainfall in this watershed polygon for some period of time is 10 mm. Current practice is to assign a nominal point (centroid?) or just use an ID and forgo spatial information within a NetCDF-CF file. In order to satisfy a number of environmental modeling use cases, we need a way to encode a geometry (point, line, polygon, multiLine, or multiPolygon) that is the static spatial element for which one or more timeseries can be associated to.

## How does what we are proposing work with / build on DSGs?
In this proposal, we intend to provide an encoding to define collections of geometries (point, line, polygon, multiLine, or multiPolygon). This will interface cleanly with the information encoded in Discrete Sampling Geometries, enabling DSGs and Geometries to be used in tandem to describe relevant datasets.

---

## Proposed extension of existing DSG concepts.

In NetCDF-CF 1.7, Discrete Sampling Geometries seperate dimensions and variables into two types--[instance and element](http://cfconventions.org/cf-conventions/cf-conventions.html#_collections_instances_and_elements). Instance refers to individual points, trajectories, profiles, etc. These would sometimes be referred to as features given that they are identified entities that can have associated attributes and be related to other entities. Element dimensions describe temporal or other dimensions to describe data on a per-instance basis. Here, we are extending the DSG `timeSeries` [featuretype](http://cfconventions.org/cf-conventions/cf-conventions.html#_features_and_feature_types) such that the geospatial coordinates of the instances can be multipolygon or multiline feature geometries.

### Example: Representation of WKT-style polygons in a NetCDF-3 timeSeries featureType.

Below is sample CDL demonstrating how polygons can be encoded in NetCDF-3 using a continuous ragged array-like encoding. There are three details to note in the example below.  
- The attribute `contiguous_ragged_dimension` with value of a dimension in the file.  
- The `geom_coordinates` attribute with a value containing a space seperated string of variable names.
- The cf_role `geometry_x_node` and `geometry_y_node`.  
These three attributes form a system to fully describe collections of multipolygon feature geometries. Any variable that has the `continuous_ragged_dimension` attribute contains integers that indicate the 0-indexed starting position of each geometry along the instance dimension. Any variable that uses the dimension referenced in the `continuous_ragged_dimension` attribute can be interpreted using the values in the variable containing the `contiguous_ragged_dimension` attribute. The variables referenced in the `geom_coordinates` attribute describe spatial coordinates of geometries. These variables can also be identified by the `cf_role`s `geometry_x_node` and `geometry_y_node`. Note that the example below also includes a mechanism to handle multipolygon features that also contain holes.

```
netcdf multipolygon_example {
dimensions:
  node = 47 ;
  indices = 55 ;
  instance = 3 ;
  time = 5 ;
  strlen = 5 ;
variables:
  char instance_name(instance, strlen) ;
    instance_name:cf_role = "timeseries_id" ;
  int coordinate_index(indices) ;
    coordinate_index:geom_type = "multipolygon" ;
    coordinate_index:geom_coordinates = "x y" ;
    coordinate_index:multipart_break_value = -1 ;
    coordinate_index:hole_break_value = -2 ;
    coordinate_index:outer_ring_order = "anticlockwise" ;
    coordinate_index:closure_convention = "last_node_equals_first" ;
  int coordinate_index_start(instance) ;
    coordinate_index_start:long_name = "index of first coordinate in each instance geometry" ;
    coordinate_index_start:contiguous_ragged_dimension = "indices" ;
  double x(node) ;
    x:units = "degrees_east" ;
    x:standard_name = "longitude" ; // or projection_x_coordinate
    X:cf_role = "geometry_x_node" ;
  double y(node) ;
    y:units = "degrees_north" ;
    y:standard_name = “latitude” ; // or projection_y_coordinate
    y:cf_role = "geometry_y_node"
  double someVariable(instance) ;
    someVariable:long_name = "a variable describing a single-valued attribute of a polygon" ;
  int time(time) ;
    time:units = "days since 2000-01-01" ;
  double someData(instance, time) ;
    someData:coordinates = "time x y" ;
    someData:featureType = "timeSeries" ;
// global attributes:
    :Conventions = "CF-1.8" ;
    
data:

 instance_name =
  "flash",
  "bang",
  "pow" ;

 coordinate_index = 0, 1, 2, 3, 4, -2, 5, 6, 7, 8, -2, 9, 10, 11, 12, -2, 13, 14, 15, 16,
    -1, 17, 18, 19, 20, -1, 21, 22, 23, 24, 25, 26, 27, 28, -1, 29, 30, 31, 32, 33,
    34, -2, 35, 36, 37, 38, 39, 40, 41, 42, -1, 43, 44, 45, 46 ;

 coordinate_index_start = 0, 30, 46 ;

 x = 0, 20, 20, 0, 0, 1, 10, 19, 1, 5, 7, 9, 5, 11, 13, 15, 11, 5, 9, 7,
    5, 11, 15, 13, 11, -40, -20, -45, -40, -20, -10, -10, -30, -45, -20, -30, -20, -20, -30, 30, 
    45, 10, 30, 25, 50, 30, 25 ;

 y = 0, 0, 20, 20, 0, 1, 5, 1, 1, 15, 19, 15, 15, 15, 19, 15, 15, 25, 25, 29, 
    25, 25, 25, 29, 25, -40, -45, -30, -40, -35, -30, -10, -5, -20, -35, -20, -15, -25, -20, 20,
    40, 40, 20, 5, 10, 15, 5 ;

 someVariable = 1, 2, 3 ;

 time = 1, 2, 3, 4, 5 ;

 someData =
  1, 2, 3, 4, 5,
  1, 2, 3, 4, 5,
  1, 2, 3, 4, 5 ;
}
```

### How to interpret:
 
Starting from the time series variables:

1) See CF-1.8 conventions  
2) See the `timeSeries` featureType  
3) Find the `timeseries_id` `cf_role`  
4) Find the `coordinates` attribute of data variables  
5) See that the variables indicated by the `coordinates` attribute have a `cf_role` `geometry_x_node` and `geometry_y_node` to determine that these are geometries according to this new specification  
6) Find the coordinate index variable with `geom_coordinates` that point to the nodes  
7) Find the variable with `contiguous_ragged_dimension` pointing to the dimension of the coordinate index variable to determine how to index into the coordinate index  
8) Iterate over polygons, parsing out geometries using the contiguous ragged start variable and coordinate index variable to interpret the coordinate data variables

Or, without reference to the timeseries:

1) See CF-1.8 conventions  
2) See the `geom_type` of `multipolygon`  
3) Find the variable with a `contiguous_ragged_dimension` matching the coordinate index variable's dimension  
4) See the `geom_coordinates` of `x y`  
5) Using the contiguous ragged start variable found in 3 and the coordinate index variable found in 2, geometries can be parsed out of the coordinate index variable and parsed using the hole and break values in it
