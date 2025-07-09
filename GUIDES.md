# UBID Guides

This document provides step-by-step &ldquo;how to&rdquo; guides for using UBID effectively. Whether you&apos;re a beginner looking to get started or an experienced user seeking advanced tips, you&apos;ll find practical instructions, examples, and best practices to help you make the most of UBID.

## UBID Assignment 101

In general, the workflow for assigning UBIDs to the features in a file is as follows:
1. Read the file.
2. Project coordinates to WSG 84.
3. Assign UBIDs using the projected coordinates.
4. Project coordinates back to their original projection.
5. Write the file with assigned UBIDs.

## UBID Assignment for Specific File Formats

### Comma-separated values (CSV)

A **CSV** file is a text-based, tabular file format.

CSV files are stored on disk with the extension `.csv`.

To assign a UBID to each row in a CSV file, use the [`buildingid`](https://github.com/pnnl/buildingid-py?tab=readme-ov-file#tutorials) command.

### Geodatabase

A **Geodatabase** is a proprietary GIS file format developed by [Esri](https://www.esri.com/) (a GIS software vendor).

Geodatabases are stored on disk as a directory with the extension `.gdb`.

To assign a UBID to each feature in a layer in a Geodatabase, use the [geopandas](https://geopandas.org/) package for the Python programming language:

The `requirements.txt` file for the project is as follows:
```text
git+https://github.com/pnnl/buildingid-py.git
geopandas
setuptools
```

An example Python script is as follows:
```python
#!/usr/bin/python3

import buildingid.code
import geopandas as gpd

import os

def UBID_series(gdf, codeLength=10):
    """
    This function assigns a UBID series to a data frame.

    Args:
        gdf (geopandas.GeoDataFrame): The data frame.
        codeLength (int): The UBID code length (default: 10).

    Returns:
        geopandas.GeoSeries: The series.
    """
    bounds = gdf.to_crs(epsg=4326).bounds
    centroid = gdf.centroid.to_crs(epsg=4326).get_coordinates()
    bounds['centerx'] = centroid['x']
    bounds['centery'] = centroid['y']
    return bounds.apply(lambda row: buildingid.code.encode(row['miny'], row['minx'], row['maxy'], row['maxx'], row['centery'], row['centerx'], codeLength=codeLength), axis=1)

if __name__ == '__main__':
    src = os.path.join('path', 'to', 'input.gdb')
    dst = os.path.join('path', 'to', 'output.gdb')

    layer = 'input_layer_name'

    if os.path.exists(src) and not os.path.exists(dst):
        gdf = gpd.read_file(src, layer=layer)

        # Assign UBIDs.
        gdf['UBID'] = UBID_series(gdf, codeLength=11)

        os.makedirs(os.path.dirname(dst), exist_ok=True)

        gdf.to_file(dst, driver='OpenFileGDB')
```

The Python script can be adapted to other GIS file formats. See [Reading and writing files](https://geopandas.org/en/stable/docs/user_guide/io.html) for more information.

Use the `ogrinfo -summary path/to/input.gdb` command to list the available layers for the Geodatabase.

### Shapefile

A **Shapefile** is a proprietary GIS file format developed by [Esri](https://www.esri.com/) (a GIS software vendor).

Shapefiles are stored on disk as a directory with the extension `.shp`. Typically, the contents of the directory is shared as a ZIP archive with the extension `.zip`.

To assign a UBID to each feature in a Shapefile, use a combination of [GNU Make](https://www.gnu.org/software/make/), [GDAL](https://gdal.org/), and the [`buildingid`](https://github.com/pnnl/buildingid-py?tab=readme-ov-file#tutorials) command.

The following `Makefile` extracts the contents of the ZIP archive, projects the coordinates to WGS 84, assigns UBIDs using the projected coordinates, projects the coordinates back to their original projection, and then creates a new ZIP archive with the assigned UBIDs.

```bash
.PHONY: all clean

# 1. Extract contents of ZIP archive.
$(BASENAME).cpg $(BASENAME).dbf $(BASENAME).prj $(BASENAME).sbn $(BASENAME).sbx $(BASENAME).shp $(BASENAME).shp.xml $(BASENAME).shx: $(BASENAME).zip
	unzip -o $(BASENAME).zip ;

# 2. Project coordinates to WGS 84.
$(BASENAME).csv: $(BASENAME).prj $(BASENAME).shp
	ogr2ogr -s_srs "$$(cat $(BASENAME).prj)" -t_srs "EPSG:4326" -f CSV $(BASENAME).csv $(BASENAME).shp -lco GEOMETRY=AS_WKT ;

# 3. Assign UBIDs.
#
# Note: The "--code-length" command-line argument, which specifies the Open Location Code grid resolution level, is set to 11.
#
# Note: The "--fieldname-code" command-line argument, which specifies the name of the new field for the assigned UBIDs, is set to "UBID".
#
# Note: If the `buildingid` command fails, an error message will be written to the "$(BASENAME).err.csv" file.
$(BASENAME).err.csv $(BASENAME).out.csv: $(BASENAME).csv
	buildingid append2csv wkt --code-length=11 --fieldname-code="UBID" --fieldname-wktstr="WKT" < $(BASENAME).csv > $(BASENAME).out.csv 2> $(BASENAME).err.csv ;

# 4. Project coordinates to their original projection.
$(BASENAME).out.dbf $(BASENAME).out.prj $(BASENAME).out.shp $(BASENAME).out.shx: $(BASENAME).prj $(BASENAME).out.csv
	ogr2ogr -s_srs "EPSG:4326" -t_srs "$$(cat $(BASENAME).prj)" -oo GEOM_POSSIBLE_NAMES="WKT" -oo KEEP_GEOM_COLUMNS=NO -f "ESRI Shapefile" $(BASENAME).out.shp $(BASENAME).out.csv ;

# 5. Create new ZIP archive with the assigned UBIDs.
$(BASENAME).out.zip: $(BASENAME).out.dbf $(BASENAME).out.prj $(BASENAME).out.shp $(BASENAME).out.shx
	zip $(BASENAME).out.zip $(BASENAME).out.dbf $(BASENAME).out.prj $(BASENAME).out.shp $(BASENAME).out.shx ;

all: $(BASENAME).out.zip

clean:
	rm -f $(BASENAME).cpg $(BASENAME).dbf $(BASENAME).prj $(BASENAME).sbn $(BASENAME).sbx $(BASENAME).shp $(BASENAME).shp.xml $(BASENAME).shx ;
	rm -f $(BASENAME).csv ;
	rm -f $(BASENAME).err.csv $(BASENAME).out.csv ;
	rm -f $(BASENAME).out.dbf $(BASENAME).out.prj $(BASENAME).out.shp $(BASENAME).out.shx ;
	rm -f $(BASENAME).out.zip ;
```

Copy the ZIP archive into the same directory as the `Makefile` and then run this command:
```bash
make all BASENAME=example
```

If the filename of the input ZIP archive is `example.zip`, then the value of the `BASENAME` argument should be `example`, and the filename of the output ZIP archive will be `example.out.zip`.
