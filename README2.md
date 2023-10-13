# 🚉 End Station
## Goal
Use static GTFS data and Python to find "true" end stations (at the end of a line **and** with no connections) for subway (*U-Bahn*) and suburban rail (*S-Bahn*) lines in Berlin. 

Even though it would be easier to just look at a map.
![S-Bahn Map](https://upload.wikimedia.org/wikipedia/commons/5/57/Karte_sbahn_berlin.png)
## The challenge:
My friend Lou has a project. They want to visit all the end stations on the Berlin S-Bahn and U-Bahn system. However, to count as a truly satisfying end station, the station must not only be at the end of its own line, but must also lack an interchange to another line. Lou wants only the end-iest of end stations.

I want to find the end stations and present them to Lou in a format they can use for their project.
## The data:
Berlin provides public transport data in the now-standard [GTFS](https://gtfs.org/) format, which stands for General (formerly Google) Transit Feed Specification. It is an [open standard](https://beyondtransparency.org/chapters/part-2/pioneering-open-data-standards-the-gtfs-story/) used by transit agencies to present their data in a uniform und useable way. It encompasses two types of data: realtime and static.
- The realtime data is what drives (pun intended) our on-the-go transit apps and notifies us of delays and changes.
- The static data (also known as *schedule data*) tells us and our apps about how all the points on the network are linked together. This is the GTFS data that will allow us identify the end stations.

### Static GTFS data format
GTFS is a data format in its own right, meant for "[GTFS-consuming applications](https://www.transitwiki.org/TransitWiki/index.php/Category:GTFS-consuming_applications)". However, conveniently, it is also easily readable in other software environements because it is merely a ZIP comprised of CSV (as TXT) files. The exact tables provided vary by agency. The following are the minimum required [table files for static GTFS](https://gtfs.org/schedule/reference/):
-  `stops.txt` showing stops' name and location
-  `stop_times.txt` showing every stop on every individual trip
-  `trips.txt` showing trips' route, direction, destination* (headsign), and more
-  `routes.txt` showing routes' transportation type, number, line colour, and more
-  `agency.txt`

My [GTFS download for Berlin](https://daten.berlin.de/datensaetze/vbb-fahrplandaten-gtfs) also contains these conditionally required and optional tables:
-  `frequencies.txt` (note: empty), `calendar.txt` and `calendar_dates.txt` showing service frequency, day and date availability
-  `pathways.txt` and `levels.txt` showing platform positions inside stations
-  `shapes.txt` showing a trip's path, expressed as a sequence of co-ordinates 
-  `transfers.txt` showing the connections between stations and details about these transfers

GTFS does not contain a simple list of `stops` on `route`. Instead, `stop_ID`s appear as sequences of `stop_times` in individual `trips`, which in turn belong to various `routes`. We need to join the IDs of stops, trips and routes to the (giant) `stop_times.txt`. 

I would therefore need to combine the following tables and fields to find the end stations if I were to make a database:

`stop_times.txt` for the sequence of stops
- "trip_id"         (primary key 1) (foreign key of trips.trip_id)
- "stop_sequence"   (primary key 2)
- "arrival_time"
- "departure_time"
- "stop_id" (foreign key of stops.stop_id)
- "pickup_type"
- "drop_off_type"

`transfers.txt` for the transfers between different routes (or in our case, the lack thereof)

- "from_stop_id"    (primary key 1) (foreign key of stops.stop_id)
- "to_stop_id"      (primary key 2) (foreign key 2 of stops.stop_id)
- "from_route_id"   (primary key 3) - might not be needed
- "to_route_id"     (primary key 4) - might not be needed
- "transfer_type"`  see [docs](https://gtfs.org/schedule/reference/#transferstxt)
- "min_transfer_time" 

`trips.txt` for matching the trip IDs to routes
- "trip_id"         (primary key)
- "route_id"        (foreign key of routes.route_id)
- "trip_headsign"   *
- "trip_short_name" (can be route number but appears not to be used for train numbers here)
- "direction_id"

`stops.txt` for matching the stop IDs to stops
- "stop_id"         (primary key)
- "stop_name"       
- "stop_lon"
- "stop_lat"

\* the "headsign" destination is not to be confused with the end of a line; some trips do not travel the whole route. In our data this is a text field rather than an ID.

## Tools for GTFS
There are many [packages designed to handle GTFS data](https://gtfs.org/resources/gtfs) in Python, created by various developers. These are mainly used for geographical and frequency data, but I found three that I may be able to use to find the end stations in **Python**, more easily than by manually creating a database:

|    | Package | Data Structure | Potential Functions |
| ---| ------- | -------------- | ------------------- |
| 1.    | [`gtfs_kit`](https://mrcagney.github.io/gtfs_kit_docs/)   | Pandas  | `gtfs_kit.stops.get_stops` and `gtfs_kit.validators.check_transfers`|
| 2. | [`transit_service_analyst`](https://github.com/psrc/transit_service_analyst/wiki/transit_service_analyst-documentation) | Pandas | `get_line_stops_gdf()` |
| 3. | [`gtfs_utils`](https://open-bus-gtfs-utils.readthedocs.io/en/latest/route_stats.html)    | Pandas |  `route_stats` and `trip_stats`| 

I also noted down a few tools for **SQL**, in case the above didn't do what I needed: 
1. [`node-gtfs`](https://github.com/BlinkTagInc/node-gtfs) SQLite
2. [`gtfs-via-postgres`](https://github.com/public-transport/gtfs-via-postgres) PostgreSQL
3. [`gtfs-schema`](https://github.com/tyleragreen/gtfs-schema) Schema builder
4. [`gtfsdb`](https://github.com/OpenTransitTools/gtfsdb) Works with various SQL

## Data manipulation with `gtfs_kit`
After testing the various packages, `gtfs_kit` was the one that met my data requirements the closest. This module uses Pandas and Shapely to manipulate the GTFS data. `read_feed` loads our GTFS data into an instance of the module's `Feed` class, ignoring non-GTFS files and stripping whitespaces from column names as it does so.
Knowing that the Berlin data is using Extended GTFS journey types, I extract the following `route_type`s directly by number, from the result of `get_routes`:
- **109**: Suburban Railway.	Examples:	S-Bahn (DE)
- **400**: Urban Railway Service   
    - Note: In Berlin, not "402: Underground Service", despite the Extended specification giving U-Bahn as an example of this type