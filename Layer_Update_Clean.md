# **LAYER UPDATE AUTOMATION**

### IMPORT PACKAGES AND SET UP DIRECTORIES


```python
#IMPORT PACKAGES

#for AGOL 
from arcgis.gis import GIS
from arcgis.features import GeoAccessor, GeoSeriesAccessor, FeatureLayerCollection
from arcgis.map import Map
from IPython.display import display

#for geoprocessing 
from arcgis.geoprocessing import import_toolbox
import arcpy
from arcpy.lr import *

#for file/folder manipulation
from pathlib import Path
import zipfile
from zipfile import ZipFile
import os
import glob
import shutil 
from datetime import datetime
import time

#for downloading google sheets
import requests
import webbrowser
```

### AGOL CREDENTIALS


```python
#IMPORTANT 
#Fill in with your your username and password, url endpoint always the same within organization

url  = "YOUR ORGANIZATIONAL URL"
username = "YOUR ORGANIZATIONAL USERNAME"
```

### password


```python
password = "YOUR ORGANIZATIONAL PASSWORD"
```

### SET UP FILE PATHING/DIRECTORIES


```python
#CREATE DIRECTORY 

#set file path
path_str = r'YOUR FILE PATH HERE'
data_path = Path(path_str)

#make directory
if not data_path.exists():
    data_path.mkdir()
```


```python
#SET UP DATE & TIME FOR VERSIONING

#Get current date
current_date = datetime.now()
# Format the date as a string
date = current_date.strftime("_%Y-%m-%d.%H.%M") 
```


```python
#DELETE OLD FILES FROM DIRECTORY

#If you'd like to keep each iteration comment out this code

for filename in os.listdir(data_path):
    filepath = os.path.join(data_path, filename)
    try:
        shutil.rmtree(filepath)
    except OSError:
        os.remove(filepath)
```


```python
#CREATE NEW SUB-FOLDER FOR THIS ITERATION

#format paths for sub folders
data_folder = path_str + '\\' + 'Data' + date + '\\'
data_folder_path = Path(data_folder)

#create sub folders
if not data_folder_path.exists():
    data_folder_path.mkdir()
```

### ACCESS AGOL AND DOWNLOAD FILES


```python
#CONNECT TO AGOL
gis = GIS(url, username, password)
```


```python
#PULL ROUTE LAYER BY ID

routes_id = 'ROUTE/REFERENCE LAYER ID'
routes_dl = gis.content.get(routes_id)
```


```python
#Download as ZIP 

result = routes_dl.export(routes_dl.title, 'Shapefile')
if not data_folder_path.exists():
    data_folder_path.mkdir()
zip_path = data_folder_path.joinpath('Routes.zip')
extract_path = data_folder_path.joinpath('Routes')
routes_zip = result.download(save_path=data_folder_path)

#delete duplicated routes shp in AGOL to save space
result.delete()
```


```python
#Extract ZIP as folder 

routes_data = data_folder + 'Routes' + '\\'
routes_data_path = Path(routes_data)

if not routes_data_path.exists():
    routes_data_path.mkdir()
zip_file = ZipFile(zip_path)
zip_file.extractall(path=routes_data_path)
```


```python
#DOWNLOAD CSV WITH WEBBROWSER

## Create download link for new spreadsheet
sheet_url = r"FULL GOOGLE SHEET URL"
url_1 = sheet_url.replace("/edit?gid=0#gid=0", "/export?format=csv")

## Open link
get_url= webbrowser.open(url_1)
```


```python
#Wait 5 seconds to ensure csv is downloaded 

time.sleep(5)
```


```python
chrome_folder = os.path.join(os.path.expanduser("~"), "Downloads\Chrome", "*.csv")

csv_dl = max(glob.glob(chrome_folder), key=os.path.getctime)
shutil.move(csv_dl, data_folder)
```


```python
osow_csv = shutil.move(csv_dl_path, data_folder_path)
```

### RUN GEOPROCESSING--MAKE ROUTE EVENT LAYER AND SAVE AS SHAPEFILE


```python
#CREATE OBJECTS FOR SHAPEFILE AND CSV

routes_obj_path = routes_data + 'Routes.shp'
routes = routes_obj_path
```


```python
# Set local variables
rt = routes
rid = "ROUTE" 
tbl = osow_csv
props_line = "ROUTE LINE BeginRefPoint EndRefPoint"
props_point = "ROUTE POINT BeginRefPoint EndRefPoint"
lyr = "LAYER NAME" #layer name must match current online layer to update
```


```python
# Run MakeRouteEventLayer 
arcpy.lr.MakeRouteEventLayer(rt, rid, tbl, props_line, lyr)
```

### SAVE LAYER AS ZIP FILE


```python
#Create folder to save .shp 
outFC_path = data_folder + 'LAYER NAME' #layer name must match current online layer to update
outFC_folder = outFC_path + '\\'

outFC_layer = Path(outFC_path)
if not outFC_layer.exists():
   outFC_layer.mkdir()
```


```python
#EXPORT FEATURE LAYER

# Set local variables
inFeatures = lyr
outFeatureClass = outFC_folder + lyr

#Run ExportFeatures
co_osow_dl = arcpy.conversion.ExportFeatures(inFeatures, outFeatureClass)
```


```python
#ZIP NEWLY CREATED SHAPEFILE 

#the following code creates a function to zip the file and exclude .lock files, which normally prevent zipping in an 'active' layer

zipSuf = ".zip"

def zip_folder(folder_path, output_path, exclude_types):
    with zipfile.ZipFile(output_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
        for root, dirs, files in os.walk(folder_path):
            for file in files:
                if not any(file.endswith(ext) for ext in exclude_types):
                    file_path = os.path.join(root, file)
                    zipf.write(file_path, os.path.relpath(file_path, folder_path))

if __name__ == '__main__':
    folder_path = outFC_layer
    output_path = outFC_path + zipSuf
    exclude_types = ['.lock']  # File types to exclude

    zip_folder(folder_path, output_path, exclude_types)
```


```python
#CREATE OBJECT FOR ZIP FILE FOR FEATURE LAYER UPDATE
layer_zip = output_path
```

### UPDATE FEATURE LAYER IN AGOL


```python
#Access current layer in AGOL by ID 

update_id = 'ID OF LAYER TO UPDATE'
old_layer = gis.content.get(update_id)
```


```python
layer_collection = FeatureLayerCollection.fromitem(old_layer)
```


```python
layer_collection.properties.layers[0].name
```


```python
layer_collection.manager.overwrite(os.path.join('data', 'updating_gis_content',
                               'updated_osow', osow_zip))
```

### PUBLISH LAYER TO AGOL


```python
#newshp = gis.content.add({}, osow_zip)
#NAME = newshp.publish()
```
