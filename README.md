# GEE
## Este es mi primer post en Github!!

Soy Ingeniero Agronomo y una parte de mi trabajo consiste en hacer ambientaciones de campos para hacer agricultura de precision.

Para hacer mapeos de forma manual el QGIS o para hacer un primer approach de un campo, siempre es una buena practica buscar varias imagenes con distinto nivel de anegamiento, para poder diferenciar las partes mas altas y mas bajas del campo.

Este script sirve para hacer justamente eso. 

Lo que hace es primero calcular el NDWI (un indice que permite diferenciar superficie cubierta de agua) en cada una de las imagenes de la coleccion Sentinel, luego calcula la superficie cubierta de agua en cada imagen (que son los todos los pixeles cuyo valor de NDWI es mayor que 0) y agrega el valor del area anegada a la metadata de cada una. Luego, ordena toda la serie segun el area anegada y selecciona la de mayor, menor, la mediana y dos situaciones intermedias (percentil 75 y 25). Es decir se obtienen 5 imagenes con niveles distintos de anegamiento del area de interes.


```javascript
////// Elegir Carpeta de Drive, Nombre del establecimiento y tolerancia del filtro de nubes

var campo = 'Campo' //sin espacios ni acentos

var carpeta = 'GEE' //sin espacios ni acentos, carpeta de Drive donde lo quiero guardar

var tolerancia_filtro_nubes = 1 //filtra todas las imagenes que tengan por encima de este porcentaje de nubosidad

var imagenes_promediadas = 3  // es la cantidad de imagenes que se promedian

var buffer = 5000  // metros de buffer del lote a evaluar


//EXTRAE UN DEGRADE DE 5 SITUACIONES DE ANEGAMIENTO: MIN, MAX, MEDIA Y DOS SITUACIONES INTERMEDIAS 
//EN BASE A TODA LA SERIE SENTINEL DISPONIBLE DEL ROI ELEGIDO////////

//Filtro de nubes a nivel de pixel
function maskS2clouds(image) {
  var qa = image.select('QA60');
  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask).divide(10000);
}

// Defino el area de interes
var roi = table.map(function(feature) { return feature.buffer(buffer); })


// function addNDVI(image) {
//   var ndvi = image.normalizedDifference(['B8', 'B4']).rename('ndvi');
//   return image.addBands(ndvi);
// }

//Calculo de NDWI
function addNDWI(image) {
  var ndwi = image.normalizedDifference(['B3', 'B8']).rename('ndwi');
  return image.addBands(ndwi);
}

//Coleccion de imagenes, filtro de nubes y calculo de NDVI y NDWI
var dataset = ee.ImageCollection('COPERNICUS/S2')
                  .filterBounds(roi)
                  //.filterDate('2015-05-01', ee.Date(Date.now()))
                  // Pre-filter to get less cloudy granules.
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', tolerancia_filtro_nubes))
                  .map(maskS2clouds)
                  //.map(addNDVI)
                  .map(addNDWI);

//print(dataset)



function addSupAgua(image) {
  var agua = ee.Image(1).mask(image.select('ndwi')).gt(0).clip(roi)
  var areaAgua = agua.multiply(ee.Image.pixelArea())
                     .reduceRegion(ee.Reducer.sum(), table, 15, null, null, false, 1e13)
                     .get('constant','areaAgua')
  //var b_anega = ee.Image.constant(areaAgua) genera la banda constante con el area anegada
  //return image.addBands(b_anega).set({Area_Anegada: areaAgua}) agregaba una banda con el valor de area anegada, no lo use
  return image.set({Area_Anegada: areaAgua})//.clip(table)
}


var datasetAgua = dataset.map(addSupAgua);

//print('datasetAgua', datasetAgua)


var aguaList0 = datasetAgua.aggregate_array('Area_Anegada');

var aguaList = aguaList0.filter(ee.Filter.notEquals('item', 0));

//print('Lista de valores de area anegada con cero', aguaList0)
//print('Lista de valores de area anegada sin cero', aguaList)

var percentiles = aguaList.reduce(ee.Reducer.percentile([10, 25, 50, 75, 100],['IMG_P00','IMG_P25','IMG_P50','IMG_P75','IMG_P100']));

//print('Percentiles', percentiles)


var areaP00 = ee.Dictionary(percentiles).get('IMG_P00');
var areaP25 = ee.Dictionary(percentiles).get('IMG_P25');
var areaP50 = ee.Dictionary(percentiles).get('IMG_P50');
var areaP75 = ee.Dictionary(percentiles).get('IMG_P75');
var areaP100 = ee.Dictionary(percentiles).get('IMG_P100');


// print('Area P00', areaP00)
// print('Area P25', areaP25)
// print('Area P50', areaP50)
// print('Area P75', areaP75)
// print('Area P100', areaP100)


//Hacer coleccion filtrada y ordenada y de esa elegir la primera
var colP00 = ee.ImageCollection(datasetAgua).filter(ee.Filter.lte('Area_Anegada', areaP00)).limit({max: imagenes_promediadas, property: 'Area_Anegada', ascending: false});
var colP25 = ee.ImageCollection(datasetAgua).filter(ee.Filter.lte('Area_Anegada', areaP25)).limit({max: imagenes_promediadas, property: 'Area_Anegada', ascending: false});
var colP50 = ee.ImageCollection(datasetAgua).filter(ee.Filter.lte('Area_Anegada', areaP50)).limit({max: imagenes_promediadas, property: 'Area_Anegada', ascending: false});
var colP75 = ee.ImageCollection(datasetAgua).filter(ee.Filter.lte('Area_Anegada', areaP75)).limit({max: imagenes_promediadas, property: 'Area_Anegada', ascending: false});
var colP100 = ee.ImageCollection(datasetAgua).filter(ee.Filter.lte('Area_Anegada', areaP100)).limit({max: imagenes_promediadas, property: 'Area_Anegada', ascending: false});

//print('Coleccion P00', colP00)


var P00 = colP00.median().clip(roi);
var P25 = colP25.median().clip(roi);
var P50 = colP50.median().clip(roi);
var P75 = colP75.median().clip(roi);
var P100 = colP100.median().clip(roi)


// print('P00', P00)
// print('P25', P25)
// print('P50', P50)
// print('P75', P75)
// print('P100', P100)


var palette = [
  'FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163', '99B718',
  '74A901', '66A000', '529400', '3E8601', '207401', '056201',
  '004C00', '023B01', '012E01', '011D01', '011301'];

var rgbPV = {
  min: 0.0,
  max: 0.3,
  bands: ['B4', 'B3', 'B2'],
};

var ndviPV = {
  min:0, 
  max:0.9, 
  palette: palette
}

var ndwiPV = {
  max: 1.0, 
  min: 0, 
  palette: ['0602ff', '307ef3', '30c8e2', '3ae237', 'd6e21f', 'ffd611',
  'ff8b13', 'ff500d', 'ff0000', 'de0101', 'c21301']
}


Export.image.toDrive({image: P00.select(['B4','B3','B2','B8','B11']), description: campo+'_Anegamiento_P00', folder: carpeta, region: table, scale: 10, maxPixels: 1e12})
Export.image.toDrive({image: P25.select(['B4','B3','B2','B8','B11']), description: campo+'_Anegamiento_P25', folder: carpeta, region: table, scale: 10, maxPixels: 1e12})
Export.image.toDrive({image: P50.select(['B4','B3','B2','B8','B11']), description: campo+'_Anegamiento_P50', folder: carpeta, region: table, scale: 10, maxPixels: 1e12})
Export.image.toDrive({image: P75.select(['B4','B3','B2','B8','B11']), description: campo+'_Anegamiento_P75', folder: carpeta, region: table, scale: 10, maxPixels: 1e12})
Export.image.toDrive({image: P100.select(['B4','B3','B2','B8','B11']), description: campo+'_Anegamiento_P100', folder: carpeta, region: table, scale: 10, maxPixels: 1e12})
```


https://code.earthengine.google.com/132b535f0c011e683083203af9c2be0e
