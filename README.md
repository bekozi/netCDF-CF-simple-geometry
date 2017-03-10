[![Build Status](https://travis-ci.org/twhiteaker/netCDF-CF-simple-geometry.svg?branch=master)](https://travis-ci.org/bekozi/netCDF-CF-simple-geometry)

# netCDF-CF-simple-geometry

**DRAFT** CF Convention for Representing Simple Geometry Types

## Scope

* Extends existing CF capabilities in CF 1.7 to include _geometries_ of types multi-point, line, multi-line,  polygon, and multi-polygon.
* Limited to linear connections between nodes in the coordinate reference system the nodes are defined in. May support parametric or otherwise curved features in the future.
* Uses the `cell_bounds` concept to allow geometries to be attached to a Discrete Sampling Geometry _instance_ variable or can be used alone to store geometry.

## Use Cases

* Encode watershed model time series and polygons in a single file. Archiving the model output and geometry is the purpose.
* Encode a streamflow value for each river segment in the conterminous U.S. at a given point in time.

## Proposal

* Support line and polygon geometry in netCDF-3 for potential inclusion in CF 1.8 with a clear path to enhanced netCDF-4 for inclusion in a CF 2.0. 
* Define a standard that the CF timeSeries feature type could use to specify spatial 'coordinates' or 'bounds' of a timeSeries variable.
* Support well known text (WKT) style for encoding multipolygons with holes and multilines while following existing NetCDF-CF implementation patterns as much as possible.

## What does the existing specification do that's similar to this proposal? 
For any coordinate variable, `cell_bounds` can be defined to describe the boundaries of the region (typically in space-time) meant to be represented by the variable. This capability, described in [section 7.1](http://cfconventions.org/cf-conventions/cf-conventions.html#cell-boundaries), supports three types of `cell_bounds`: 1-d bounds such as a time period, 2-d bounds such as the bounding rectangle of a cell in a 2-d grid, and n-d bounds to describe a cell with more than four edges.  
For example, given 2-d grid of cells that do not lie on an orthoganal grid and consequently have curved cell edges. If these cells are to be represented acurately using lat/lon `cell_bounds` many nodes are required to describe their cuvature. In this case, a `lat_bnds` and `lon_bnds` variable will have `p` vertices, where p is the number needed to adequately represent the curvature of the cells.  
Extending Example 7.2 in the NetCDF-CF 1.7 specification, and assuming 10 vertices per edge plus corners are needed, this would look like:

```
dimensions:
  imax = 128;
  jmax = 64;
  nv = 44;
variables:
  float lat(jmax,imax);
    lat:long_name = "latitude";
    lat:units = "degrees_north";
    lat:bounds = "lat_bnds";
  float lon(jmax,imax);
    lon:long_name = "longitude";
    lon:units = "degrees_east";
    lon:bounds = "lon_bnds";
  float lat_bnds(jmax,imax,nv);
  float lon_bnds(jmax,imax,nv);
```

With regards to time series, Discrete Sampling Geometries (DSGs) handle data from one or (a collection of) `timeSeries` (point), `Trajectory`, `Profile`, `TrajectoryProfile` or `timeSeriesProfile` geometries. Measurements are from a **point** (timeSeries and Profile) or **points** along a trajectory. This proposal builds on the core DSG `point` and `timeSeries` type which provides support for basic time series use cases e.g., a timeSeries which is measured (or modeled).

## What doesn't the existing specification do that we need? 
From the perspective of time series associated with polygons, DSGs have no system to define a geometry (multi-point, polyline, polygon, etc) and an association with a time series that applies over that entire geometry.  
> e.g, The expected rainfall in this watershed polygon for some period of time is 10 mm.  

Current practice has been to assign a nominal point (centroid or some other location) or just use an ID and forgo spatial information within a NetCDF-CF file. In order to satisfy a number of environmental modeling use cases, a way to encode a geometry (multi-point, line, polygon, multiLine, or multiPolygon) that is the static spatial element for which one or more timeseries can be associated.

The current n-d `cell_bounds` mechanism is limited in that every cell must have the same number of verticies. In order to support geometries more generally an additional type of `cell_bounds` are needed. Typically, these geometries will each have drastically different numbers of nodes and may have multiple parts and/or holes. This requirement is not supported by the existing n-d `cell_bounds` but conceptually, a geometry can be associated with a cell the same as an n-d geometry, albeit with some additional metadata and data.

## How does what we are proposing work with / build on the current specification?
In this proposal, we intend to provide an encoding to define collections of geometries (point, line, polygon, multiLine, or multiPolygon). It is specified as a new type of `cell_bounds` that can be attached to any spatial coordinate variable allowing description of a geometry representation of a cell or entity. This will interface cleanly with the information encoded in Discrete Sampling Geometries, enabling DSGs and Geometries to be used in tandem to describe relevant datasets. Because this proposal builds on the concept of `cell_bounds`, it is also applicable to other use cases where normal n-d `cell_bounds` are not sufficient.

---

### Example: Representation of WKT-style polygons in a NetCDF-3 timeSeries featureType file.

The existing NetCDF-CF DSG specification allows encoding of `timeSeries` referenced to a point via the typical `coordinates` attribute. Starting with this, CF 1.6 - style example.

```
netcdf timeSeries_example {
dimensions:
 instance = 2 ;
 time = 5 ;
 strlen = 9 ;
variables:
 char instance_name(instance, strlen) ;
   instance_name:cf_role = "timeseries_id" ;
 int time(time) ;
   time:units = "days since 2000-01-01" ;
 double lat(instance) ;
   lat:units = "degrees_north" ;
   lat:standard_name = "latitude" ;
 double lon(instance) ;
   lon:units = "degrees_east" ;
   lon:standard_name = "longitude" ;
  int crs ;
    crs:grid_mapping_name = "latitude_longitude";
    crs:longitude_of_prime_meridian = 0.0 ;
    crs:semi_major_axis = 6378137.0 ;
    crs:inverse_flattening = 298.257223563 ;
 double someVariable(instance) ;
   someVariable:long_name = "a variable describing a single-valued attribute of a polygon" ;
   someVariable:coordinates = "lat lon" ;
   someVariable:grid_mapping = "crs" ;
 double someData(instance, time) ;
   someData:coordinates = "time lat lon" ;
   someVariable:grid_mapping = "crs" ;
// global attributes:
    :Conventions = "CF-1.6" ;
    :featureType = "timeSeries" ;

data:

 instance_name =
  "name_one",
  "name_two" ;

 time = 1, 2, 3, 4, 5 ;
 
 lat = 23, 23 ;
 
 lon = 16, 25 ;
 
 someVariable = 1, 2 ;

 someData =
  1, 2, 3, 4, 5,
  1, 2, 3, 4, 5 ;
}
```

Below is sample CDL demonstrating how polygons can be encoded in NetCDF-3 using a contiguous ragged array-like encoding. There are three details to note in the example below.  
- The newly introduced `cell_bounds` attributes on the lat and lon coordinates both pointing to the `sf` variable.  
- The `sf` variable that contains only attributes describing the system of variables that make up the geometry collection.  
- The cf_role `geometry_x_node` and `geometry_y_node`.  

Using the existing NetCDF-CF convention for auxiliary coordinate variables along with this new type of cell_bounds, we can attach geometries for each so-called `instance` in a DSG dataset. Note that this system can also be identified via the `cf_role`s `geometry_x_node` and `geometry_y_node`. Note that the example below also includes a mechanism to handle multipolygon features that also contain holes.

```
netcdf multipolygon_example {
dimensions:
 instance = 2 ;
 node = 15 ;
 time = 5 ;
 strlen = 9 ;
variables:
 char instance_name(instance, strlen) ;
   instance_name:cf_role = "timeseries_id" ;
 int time(time) ;
   time:units = "days since 2000-01-01" ;
 double lat(instance) ;
   lat:units = "degrees_north" ;
   lat:standard_name = "latitude" ;
   lat:cell_bounds = "sf" ;
 double lon(instance) ;
   lon:units = "degrees_east" ;
   lon:standard_name = "longitude" ;
   lon:cell_bounds = "sf" ;
  int crs ;
    crs:grid_mapping_name = "latitude_longitude" ;
    crs:longitude_of_prime_meridian = 0.0 ;
    crs:semi_major_axis = 6378137.0 ;
    crs:inverse_flattening = 298.257223563 ;
 int sf; // containing variable -- datatype irrelevant because no data
   sf:geom_type = "multipolygon" ;
   sf:node_count = "node_count"; // variable containing count of nodes per geometry -- may span multiple parts.
   sf:part_node_count = "part_node_count" ; // variable containing count of nodes per part -- not required for single part geometry types.
   sf:part_type = "part_type" ; // Variable indicating if parts are holes or not -- not required unless polygons with holes are present.
   sf:node_coordinates = "x y" ; // variables containing spatial node data.
 int node_count(instance); // count of coordinates in each instance geometry
 int part_count(instance) ; // count of parts belonging to each geometry
 double x(node) ;
   x:units = "degrees_east" ;
   x:standard_name = "longitude" ; // or projection_x_coordinate
   x:cf_role = "geometry_x_node" ;
 double y(node) ;
   y:units = "degrees_north" ;
   y:standard_name = "latitude" ; // or projection_y_coordinate
   y:cf_role = "geometry_y_node" ;
 double someVariable(instance) ;
   someVariable:long_name = "a variable describing a single-valued attribute of a polygon" ;
   someVariable:coordinates = "lat lon" ;
   someVariable:grid_mapping = "crs" ;
 double someData(instance, time) ;
   someData:coordinates = "time lat lon" ;
   someVariable:grid_mapping = "crs" ;
// global attributes:
    :Conventions = "CF-1.8" ;
    :featureType = "timeSeries" ;

data:

 instance_name =
  "name_one",
  "name_two" ;

 time = 1, 2, 3, 4, 5 ;
 
 lat = 23, 23 ;
 
 lon = 16, 25 ;
 
 someVariable = 1, 2 ;

 someData =
  1, 2, 3, 4, 5,
  1, 2, 3, 4, 5 ;
  
 node_count = 10, 5 ;

 part_count = 2, 1 ; // first geom has two parts
 
 x = 35, 30, 25, 26, 35, 22, 22, 15, 10, 22, 30, 30, 20, 10, 30 ; // note anti-clockwise order of rings indicates outside rings.
 
 y = 25, 30, 28, 23, 25, 22, 27, 25, 20, 22, 10, 20, 20, 15, 10 ;
}
```

### How to interpret:
 
Starting from the time series featureType:

1) See CF-1.8 conventions  
2) See the `timeSeries` featureType  
3) Find the `timeseries_id` `cf_role`  
4) Find the `coordinates` attribute of element variables for the instance dimension  
5) See the `cell_bounds` attribute of the auxiliary coordinate variable
6) See the `geom_type` in the variable referenced by `cell_bounds`
8) Iterate over geometries found in the `node_coordinates` variable. Geometries found using the the `node_count` variable Geometry parts found using the `part_count` variable.

Or, without reference to the timeseries:

1) See CF-1.8 conventions  
2) See the `geom_type` of `multipolygon`
3) Iterate over geometries found in the `node_coordinates` variable. Geometries found using the the `node_count` variable. Geometry parts found using the `part_count` variable.
