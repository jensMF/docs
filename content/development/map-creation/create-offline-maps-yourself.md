---
title: Create Offline Maps Yourself
versions: '*'
---

[*OsmAndMapCreator*](https://wiki.openstreetmap.org/wiki/OsmAndMapCreator) can be used to create maps yourself if you want maps that are more up-to-date then the ones you can download from OsmAnd.
The following explains how, using a Un\*x/Linux/OS X like shell script and the nightly generated maps from [Geofabrik](http://download.geofabrik.de/), a German company that sells OpenstreetMap based maps and appliances.

## Shell script:

```
#!/bin/sh
WORK_FOLDER="/opt/OpenStreetMap"
# First download all the data
cd "$WORK_FOLDER/osm_files"
echo "Now in pwd\n"
rm *
wget -v -O Netherlands_europe.osm.pbf "http://download.geofabrik.de​/europe/netherlands-latest.osm.pbf"
#wget -v -O Luxembourg_europe.osm.pbf "http://download.geofabrik.de​/europe/luxembourg-latest.osm.pbf"
#wget -v -O Belgium_europe.osm.pbf "http://download.geofabrik.de​/europe/belgium-latest.osm.pbf"

cd $WORK_FOLDER
echo date > starttime.txt
echo "Now converting from osm.pbf to osmand obf\n"
cd "$WORK_FOLDER/OsmAndMapCreator"

java -Djava.util.logging.config.file=​logging.properties -Xms256M -Xmx2560M -cp "./OsmAndMapCreator.jar:​./lib/OsmAnd-core.jar:./lib/*.jar" net.osmand.data.​index.IndexBatchCreator ./batch.xml

cd $WORK_FOLDER
echo date > endtime.txt

echo "And finally moving the obf files from the index folder to the osmandmaps folder\n"
mv index_files/*.obf osmandmaps/
```

### Script Explanation
Line `WORK_FOLDER="/opt/OpenStreetMap"` is a variable to set the working folder. Inside this folder we have the maps in `osm_files`, `OsmAndMapCreator`, `index_files`and `gen_files`.

We go to the download folder `_osm_files` and use the command `wget` to download our map(s). `wget` is used with parameter `-O <name\>` to download the latest nightly map from Geofabrik, we save it in the name format OsmAnd prefers.

We go to folder `OsmAndMapCreator` where we installed/copied the OsmAndMapCreator program. It is best to use the program from this folder, or else you need to set all kind of environment variables. The line:

```
java -Djava.util.logging.config.file=​logging.properties -Xms256M -Xmx2560M -cp "./OsmAndMapCreator.jar:​./lib/OsmAnd-core.jar:./lib/*.jar" net.osmand.data.​index.IndexBatchCreator ./batch.xml
```

runs OsmAndMapcreator with our downloaded maps, processing all maps it will find in the download folder (including all older ones still present there).

We log the process to file (`-Djava.util.logging.config.file=logging.properties`), give OsmAndMapCreator a minimum amount of 256MB and a maximum amount of 2560MB (preferably more then 1024MB), and use the setup as specified in `batch.xml`.  

**Note:** A 32bit Operating system can address up to approximately **1.5GB**, meaning `-Xmx` can be no greater than `-Xmx1720M`. Greater values are accepted without errors, but not used.

The `batch.xml` file is found in the `OsmAndMapCreator` folder, together with the program, and contains settings for running the program. The line:

```
<process directory_for_osm_files=​"/opt/OpenStreetMap/osm_files" directory_for_index_files=​"/opt/OpenStreetMap/index_files" directory_for_generation=​"/opt/OpenStreetMap/gen_files"/>
```
specifies the working folders.

The next line:

```
skipExistingIndexesAt="/..." indexPOI="true" indexRouting="true" indexMap="true" indexTransport="true" indexAddress="true"
```

contains options to modify parts of your map. If you don't need routing and/or addresses, you can skip these by setting the parameters to "false".

You may also use multiple `batch.xml` files for different purposes.

The last two lines in the script move the created maps to the ``osmandmaps_ folder` where we store our maps (in this case).

Lines

```
echo date > starttime.txt
echo date > endtime.txt
```

are not really necessary but simply display how long the process takes.

## Scheduling
The shell script may be scheduled. Account e.g. for the Geofabrik map creation schedule (adjust for your time zone).

If you want to create a new map every night, you can add a crontab line like:

```
01 03 * * 7 /opt/OpenStreetMap/​osm.pbf_to_obf_convert.sh > /dev/null 2>&1
```
This will start the map creation at 03:01am which is currently after the Geofabrik Netherlands osm.pbf map has been generated (local time zone).

## Performance and Tuning
Creating maps is memory hungry and I/O intensive. In other words: It takes long to very long!
What can you do to improve performance:
- Use SSD disks.
- Use multiple disks.
- Use "in memory" processing.

### SSD disks
The modern "solid state" disks are 2-6 times as fast as conventional hard-disks and can improve your map creation performance dramatically.

### Multiple disks
Modern operating systems can access multiple disks simultaneously. Note that this really means **multiple disks** and **not** multiple partitions on one disk.
If you have your `process_directory_for_osm_files` on one disk and your `directory_for_generation` on another disk, you will see an nice and noticeable performance gain.

### In memory processing
You can process a great deal of the map creation in memory instead of on disk. In your `batch.xml`, one of the top lines contains:

```
<process_attributes mapZooms="" renderingTypesFile="" zoomWaySmoothness="" osmDbDialect="sqlite" mapDbDialect="sqlite"/>
```

* `osmDbDialect="sqlite" mapDbDialect="sqlite"` means your map generation process will take place on disk.
* Change to `osmDbDialect="sqlite_in_memory" mapDbDialect="sqlite_in_memory"` to run the process in memory.
This "in memory" processing will speed up the map generation by 10-50%, but requires a lot of memory. 10% to 50% depends on the map size. Smaller maps benefit less from in memory processing than larger maps, as disk access for initial reading and final map writing plays a bigger role, while larger maps require more "calculation".

In normal "on disk" processing a *nodes.tmp.odb* file is created from your *.osm* or *.osm.pbf* file. This *nodes.tmp.odb* file is a sqlite database file and it is about 15 to 25 times as big as the original *.osm.pbf* file which you downloaded from [geofabrik.de](http://download.geofabrik.de/). So if your original *.osm.pbf* file is 300MB, your *nodes.tmp.odb* file will be 5GB to 6GB! Note that smaller maps will be around the 15x factor whereas big maps (\>350MB) will end up in the 20x to 25X space increase.

With "in memory" processing this *nodes.tmp.odb* file will be created in your working memory. It means that the `-Xmx` parameter, which we discussed in the [Explanation](http://code.google.com/p/osmand/wiki/CreateOfflineMapsForYourself#Explanation) sections, needs to be big enough for both the *nodes.tmp.odb* and the normal processing that takes place in memory. You will need "the size of the nodes.tmp.odb" + 20-25%.

This means that for a 250MB *.osm.pbf*, which will generate a \~4.5GB *nodes.tmp.odb* file, you need about 5GB heapspace which requires an `-Xmx` value of `-Xmx5120M`.

* **Note**: a 32bit Operating system can address up to approximately 1.5GB. This means that your `-Xmx` value can no larger be then `-Xmx1720M`. A larger specification is accepted without errors, but not used.

So in effect you really need a 64bit Operating system to really benefit from "in memory" processing.
Note also that your `-Xmx` value should not be that big that your operating system starts swapping to disk. This will even decrease your performance below that of normal "on disk" processing.
Finally: your source *.osm.pbf* file can be no larger then 600MB as this would require up to 20GB working memory. If your source file exceeds 600MB, OsmAndMapCreator will switch back to normal "on disk" processing. You will be notified early in the process with a warning "Switching SQLITE in memory dialect to SQLITE"

## Common Issues
### OsmAndMapCreator fails with message: OutOfMemoryError

The file you try to process with OsmAndMapCreator is too large. Either
try to process a smaller file, or increase the memory for
OsmAndMapCreator in the .sh or .bat file. The `-Xmx` parameter specifies
how much memory the program can consume. Settings can be different for
64bit (more than 1.5GB) and 32bit (max around 1.5GB) machines.

### After converting an .osm to .obf with only a POI index, the .obf is empty, although original .osm file did contain POIs. What is wrong?

It could be that a crucial tag was missing for OsmAndMapCreator to recognize a POI when you converted the osm from another source, like Garmin. If a point in the OSM file looks like this:
```
  <node id='-24' visible='true' lat='1.3094000' lon='103.7784000'>
    <tag k='created_by' v='GPSBabel-1.4.2'/>
    <tag k='name' v='Street-Soccer Court'/>
  </node>
```
change it to contain an additional 'amenity' tag, like:
```
  <node id='-24' visible='true' lat='1.3094000' lon='103.7784000'>
    <tag k='created_by' v='GPSBabel-1.4.2'/>
    <tag k='name' v='Street-Soccer Court'/>
    <tag k='amenity' v='point' />
  </node>
```
 
Then convert the file using OsmAndMapCreator. You can check on the OSM site what tags are good ones to use, or you can just use this amenity.

## Using an Internet access proxy in OsmAndMapCreator

See
[http://stackoverflow.com/questions/120797/how-do-i-set-the-proxy-to-be-used-by-the-jvm](http://stackoverflow.com/questions/120797/how-do-i-set-the-proxy-to-be-used-by-the-jvm)
Use `OsmAndMapCreator.bat` or `OsmAndMapCreator.sh` (depending on your operating system) to start OsmAndMapCreator. This invokes java, so you just need to add the corresponding `-D... -D..` parameters to the invocation:
```
javaw.exe <b>-Dhttp.proxyHost=10.0.0.100 -Dhttp.proxyPort=8080</b> -Xms64M -Xmx720M -cp "./OsmAndMapCreator.jar;./lib/\*.jar" net.osmand.swing.OsmExtractionUI
```

## How to Produce Custom Vector Data for a Map

It is possible to create a customized OBF file with specific (own) vector data (hiking paths, speed cams, transport routes debug way info), and adjust the renderer to display it.

OsmAndMapCreator can process only OSM files (osm-xml, bz2, pbf). However the set of tags can be custom. To specify what tags/values need to be indexed by Creator please download and change [this](https://github.com/osmandapp/OsmAnd-resources/blob/master/obf_creation/rendering_types.xml) file. OsmAndMapCreator has an option to use custom rendering\_types.xml in the settings. Once file is created you can double check that data is present by utility binaryInspector with `-vmap` argument. This utility is packaged with OsmAndMapCreator.

Once the .obf file is ready you can create custom rendering file to display missing attributes. There is a [default rendering style](https://github.com/osmandapp/OsmAnd-resources/blob/master/rendering_styles/default.render.xml) which contains all information about rendering. It is good to have a look at it but it is very hard to open/edit it and understand. More convenient way to create your own rendering style is to create style that depends (inherits) default style. A good example of custom rendering style you can find [here](https://github.com/osmandapp/OsmAnd-resources/blob/master/rendering_styles/Winter-and-ski.render.xml.).

Currently OsmAndMapCreator doesn't support relation tagging. So you need to manually copy all tags from relations (like route color) to way tags by script.

