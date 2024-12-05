import ee
import folium
import geopandas as gpd
from datetime import datetime, timedelta

ee.Authenticate()
ee.Initialize(project="ee-zzlly123456")

start_date_cst = '2024-07-17 00:00:00'
end_date_cst = '2024-07-18 18:00:00'

start_date_utc = (datetime.strptime(start_date_cst, '%Y-%m-%d %H:%M:%S') - timedelta(hours=8)).strftime('%Y-%m-%dT%H:%M:%S')
end_date_utc = (datetime.strptime(end_date_cst, '%Y-%m-%d %H:%M:%S') - timedelta(hours=8)).strftime('%Y-%m-%dT%H:%M:%S')
shp_path = "projects/ee-zzlly123456/assets/ht"
LP = ee.FeatureCollection(shp_path)

dataset = ee.ImageCollection('ECMWF/ERA5_LAND/HOURLY') \
    .filterDate(start_date_utc, end_date_utc) \
    .filterBounds(LP)

def ERA5P(image):
    img_I = image.select("total_precipitation")
    img_Ek = ee.Image(0.29).multiply(ee.Image(1).subtract(ee.Image(0.72).multiply(img_I.multiply(-0.05).exp())))
    img_Eh = img_Ek.multiply(img_I).multiply(1.668)
    img_R = img_Eh.multiply(img_I)
    return img_R.rename('Rfactor').copyProperties(image, ['system:time_start'])

erosivity = dataset.map(ERA5P)

erosivity_visualization = {
    'bands': ['Rfactor'],
    'min': 0,
    'max': 0.0005,
    'palette': [
        '000080', '0000d9', '4000ff', '8000ff', '0080ff', '00ffff',
        '00ff80', '80ff00', 'daff00', 'ffff00', 'fff500', 'ffda00',
        'ffb000', 'ffa400', 'ff4f00', 'ff2500', 'ff0a00', 'ff00ff',
    ],
    'opacity': 0.7  
}


erosivity_clipped = erosivity.mean().clip(LP)  # 使用 clip 限制数据范围

map_center = [35.0, 105.0]
m = folium.Map(location=map_center, zoom_start=6)  

def add_ee_layer(self, ee_object, vis_params, name):
    map_id_dict = ee_object.getMapId(vis_params)
    folium.raster_layers.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr='Google Earth Engine',
        name=name,
        overlay=True,
        control=True,
        opacity=vis_params.get('opacity', 1.0)
    ).add_to(self)


folium.Map.add_ee_layer = add_ee_layer
m.add_ee_layer(erosivity_clipped, erosivity_visualization, 'Rainfall Erosivity (Clipped)')
m.add_child(folium.LayerControl())
m.save('rainfall_erosivity_map_clipped.html')
m

