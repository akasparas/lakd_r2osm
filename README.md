# lakd_r2osm

Lithuanian Road Agency provides "Open Data" information in json format about _temporary_ restrictions on state roads at http://restrictions.eismoinfo.lt . This software is to integrate that information into Open Street Maps--both public database and private snapshot.

## Available information
This information source covers only Lithuanian state operated roads--i.e. no roads in cities, no local roads, no private roads. Time scope is from single day speed limit due to roadside barrier fixing, to several months diversions due to road reconstruction, to year+ speed or weight limitation due to damaged road. There are 100+ restrictions at any time.

Information about _permanent_ restrictions, like low bridge, narrow tunnel or limited speed due to pedestrians crossing are in different database.

TODO: *Licence* of provided information?

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
`location/street` | `ref` | First word in LAKD source states road number. Changes in OSM are only posible if `ref` attribute of the way contains this number
`location/polyline` || geographic information where restriction is in force'
`location/[direction=BOTH_DIRECTIONS]` | | Changes are possible
`location/[direction=ONE_DIRECTION]` | | TODO: how to reliably distinguish which direction we should add restriction to? Most likely adding should be done manually, removal can be performed automatically.
 `starttime` | | Timestamp when restriction should start. Usually midnight local time. In 90%+ cases _before_ `creationtime`.
 `endtime` | | Timestamp when restriction should end. Usually 23:00 local time.
 `updatetime` | | Timestamp when information about restriction was updated.
