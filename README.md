# lakd_r2osm

Lithuanian Road Agency (LAKD) provides "Open Data" information in JSON format about _temporary_ restrictions on state roads at http://restrictions.eismoinfo.lt . This software is to integrate that information into Open Street Maps--both public database and private snapshot. Intention is to make this process automatic as much as possible to do in safe way. When in doubt--prepare draft which should be checked and uploaded manually.

Software state: design.

## Available information
This information source covers only Lithuanian state operated roads--i.e. no roads in cities, no local roads, no private roads. Time scope is from single day speed limit due to roadside barrier fixing, to several months diversions due to road reconstruction, to year+ speed or weight limitation due to damaged road. There are 100+ restrictions at any time.

Information about _permanent_ restrictions, like low bridge, narrow tunnel or limited speed due to pedestrians crossing are in different database.

TODO: *License* of provided information?

## Attribute mapping

LAKD | OSM | Comments
--- | --- | ---
`_id` | `lakd:restriction:id` | used to remove information when restriction ends
`restrictions/[restrictionType=speedLimit]` | `maxspeed` | Maximum speed allowed. TODO: What to do if several speed are provided?
`restrictions/[restrictionType=weightLimit]` | `maxweight` | Maximum weight allowed.
`restrictions/[restrictionType=roadClosed]` | `access:no` | 
`restrictions/[restrictionType=*]` |  | No changes in OSM. Email maintainer about unknown information
`restrictions/[transportType=All]` | | default case
`restrictions/[transportType=lorry]` | `+:hgv` | 
`restrictions/[transportType=*]` | | No changes to OSM. Email maintainer about unhandled restriction
`location/street` | `ref` | First word in LAKD source states road number. Changes in OSM are only possible if `ref` attribute of the way contains this number
`location/polyline` || geographic information where restriction is in force'
`location/[direction=BOTH_DIRECTIONS]` | | Changes are possible
`location/[direction=ONE_DIRECTION]` | | TODO: how to reliably distinguish which direction we should add restriction to? Most likely adding should be done manually, removal can be performed automatically.
 `starttime` | | Timestamp when restriction should start. Usually midnight local time. In 90%+ cases _before_ `creationtime`.
 `endtime` | | Timestamp when restriction should end. Usually 23:00 local time.
 `updatetime` | | Timestamp when information about restriction was updated.

### encoding in OSM
There are two possible ways to encode these restrictions. Which one (and when) to use is still to be decided.

1. using [:conditional modifier](http://wiki.openstreetmap.org/wiki/Conditional_restrictions) to indicate dates when restriction will be in force
  - + _supported_ users will take into account routing date, so ended (not yet started) restrictions are not a problem;
  - - formal specification redirects to [`opening_hours`](http://wiki.openstreetmap.org/wiki/Key:opening_hours) format description, which lacks support for years; TODO: should we skip year? what to do with restrictions longer than year?
  - support in popular software:
    - **OsmAnd** -- TODO

2. use regular tags to indicate active restrictions _now_.
  - + universally supported
  - - if application's/ service's database is updated less frequently in comparison with restriction's lifetime, stale information about restriction can unfairly penalize some route. Update times of popular applications and services:
    - **OsmAnd** - monthly; with OSM Live paid service even hourly updates are possible
    
## Mode of operation

1. script should be run daily at the end of the day (23:xx). It should remove all restrictions which ends that day (or earlier) and insert/update restrictions which start/changes next day. Ideally this should happen before other tools updates their DB.
2. script should get restriction information from LAKD, check if number of records changed more than usual (>5--10%). In that case it should mail maintainer differences. Maintainer can run script manually with overridden limit if he sees only legitimate changes. This step is to prevent most problems at LAKD affecting OSM database.
3. _remove expired_ restrictions: for every restriction _r_, which is listed in our database as present in OSM, but does not appear in downloaded list or appear in the list but have `endtime` today or earlier:
  1. find all ways in OSM with `lakd:restriction:id` matching _`r`_`._id`
  2. if we have old value for this way, restore it
  3. if we do not have old value, just remove restriction
  4. remove `lakd:restriction:id` 
  
  - :bangbang: if LAKD's information is late and some users have seen and added restriction manually, we have restored stale information; could happen at the begining of use of this system, later should be very rare.
  - :bangbang: if some way had speed limit and that way was split during restriction with lowered speed limit period, new ways will not get old speed limit restored. TODO: should we store old value in OSM for it to be copied into new ways?
  
4. _update changed_ restrictions: for every restiction _r_, which is listed in our database as present in OSM, appear in downloaded list but have different `updatetime`:
  1. collect commands to remove _r_ as per (3);
  2. collect commands to add _r_ as per (5);
  3. remove commands which cancel one another
  
5. _insert new_ restrictions: for every restriction _r_, which is present in downloaded list but according to our database is not present in OSM and have duration above set limit and have `starttime` tomorrow or earlier and have `endtime` tomorrow or later:
  1. collect all the ways _w_, what are at least partially present in the _r_'s bounding box and have road number present in their `ref` attribute
  2. if `direction=BOTH_DIRECTIONS`
    1. for every _w1_ in _w_, where all nodes of _w1_ are _n_ meters (TODO: define _n_) around _r_'s `polyline`
      - if present store old restriction value
      - add restriction
      - add `lakd:restriction:id`
      - mark for automatic upload
    2. if sum of _w1_ lengths / length of _r_'s polyline is within \[0.95; 1.05\] consider success and continue with next _r_; TODO: am I too optimistic?
    3. otherwise mark that this _r_ needs manual adjustment and for every _w2_ in _w_, where **some** but not all nodes of _w2_ are _n_ meters around _r_'s `polyline`
      - if present store old restriction value
      - add restriction
      - add `lakd:restriction:id`
      - mark for manual adjustment
  3. if `direction=ONE_DIRECTION` follow (c) above. TODO: can we go with (a) with reduced _n_?

This way ensures that changes in the OSM database are made only when that restriction changes in LAKD.

If someone have a good reason to override information coming from LAKD he could remove `lakd:restriction:id` attribute and this system will stop changing these ways. At least until LAKD will give another restriction `_id`. Should we have more powerful kill switch?
