В качестве проектной работы была выбрана геоинформационная система реализованная на      
базе приложения QGIS (`https://www.qgis.org/ru/site/`). В качестве хранилища данных используется mongodb.     
В качестве исходных данных используются данные Open Street Map для Тулузы в формате shape-файлов.       
`https://www.geofabrik.de/data/shapefiles.html` - ссылка на исходники.        
Далее с помощью qgis эти данные можно переконвертировать в формат geojson для импорта        
в mongodb. Для того чтобы можно было подгружать данные в qgis из mongodb нужен специльный плагин:     
`https://plugins.qgis.org/plugins/MongoConnector/`. Поссле его загрузки и настройки коннекта,мы       
можем загружать данные непосредственно из нашей бд.

Тут необходимо чтобы в питоне с которым работает qgis был установле модуль pymongo для винды он находится по пути
```
C:\Program Files\QGIS 3.16\apps\Python37
```
И именно этим итепритатором необходимо выполнить:
```
python.exe -m pip install pymongo
```

Так же необходимо открыть порты для соединения с бд в google platform. Создаем новое правило firewall add rule.

Переходим к импорту необходимых слоев

Импортиурем полигональный слой со сроениями (жилого и нежилого характера)
```
root@mongodb:~# mongoimport -d geodb -c buildings --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/buildings.geojson   2022-03-01T18:24:53.430+0000    connected to: mongodb://localhost/
2022-03-01T18:24:56.430+0000    [#########...............] geodb.buildings      13.2MB/31.8MB (41.5%)
2022-03-01T18:24:59.430+0000    [####################....] geodb.buildings      26.6MB/31.8MB (83.7%)
2022-03-01T18:25:00.833+0000    [########################] geodb.buildings      31.8MB/31.8MB (100.0%)
2022-03-01T18:25:00.833+0000    66963 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем точеченый слой, который укажет нам где находятся различные развлекательные заведения        
(Арт пространсва, бассейны, пекарни, музеи, отели и т.д) 
```
root@mongodb:~# mongoimport -d geodb -c entertaiment_point.geojson --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/entertaiment_point.geojson
2022-03-01T18:31:48.227+0000    connected to: mongodb://localhost/
2022-03-01T18:31:48.558+0000    6608 document(s) imported successfully. 0 document(s) failed to import.
```
Импортируем точеченый слой, который укажет нам где находятся различные развлекательные и образовательные учережения  и не только    
(Отели, кафе, школы, университеты рестораны и т.д)
```
root@mongodb:~# mongoimport -d geodb -c entertaiment_polygon.geojson --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/entertaiment_polygon.geojson
2022-03-01T18:39:25.108+0000    connected to: mongodb://localhost/
2022-03-01T18:39:25.386+0000    1749 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем полигональный слой c памятными и интересными местами
```
root@mongodb:~# mongoimport -d geodb -c interested_place_polygon.geojson --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/interested_place_polygon.geojson
2022-03-01T18:53:18.980+0000    connected to: mongodb://localhost/
2022-03-01T18:53:19.027+0000    82 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем линейный слой с линиями электропередач
```
root@mongodb:~# mongoimport -d geodb -c power_lines --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/power_lines.geojson
2022-03-01T18:59:03.497+0000    connected to: mongodb://localhost/
2022-03-01T18:59:03.519+0000    10 document(s) imported successfully. 0 document(s) failed to import.
```
Импортируем точечный слой с объектами электрической инфраструктуры
```
root@mongodb:~# mongoimport -d geodb -c power_point --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/power_point.geojson
2022-03-01T19:00:56.039+0000    connected to: mongodb://localhost/
2022-03-01T19:00:56.062+0000    38 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем полигональный слой с церквями
```
root@mongodb:~# mongoimport -d geodb -c church_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/powf_polygon.geojson
2022-03-01T19:03:36.107+0000    connected to: mongodb://localhost/
2022-03-01T19:03:36.140+0000    51 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем линейный слой с железыми дорогами
```
root@mongodb:~# mongoimport -d geodb -c church_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/powf_polygon.geojson
2022-03-01T19:03:36.107+0000    connected to: mongodb://localhost/
2022-03-01T19:03:36.140+0000    51 document(s) imported successfully. 0 document(s) failed to import.
```

Импортируем слой с дорогами:

```
root@mongodb:~# mongoimport -d geodb -c road_line --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/roads_line.geojson
2022-03-01T19:10:42.206+0000    connected to: mongodb://localhost/
2022-03-01T19:10:43.273+0000    10497 document(s) imported successfully. 0 document(s) failed to import.
```
Импортируем слой с велосипедными и пешими маршрутами:
```
root@mongodb:~# mongoimport -d geodb -c route_line --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/routes_line.geojson
2022-03-01T19:11:56.506+0000    connected to: mongodb://localhost/
2022-03-01T19:11:56.608+0000    13 document(s) imported successfully. 0 document(s) failed to import
```
Полигональынй слой с электроподстанциями:
```
root@mongodb:~# mongoimport -d geodb -c substation_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/substation_polygon.geojson
2022-03-01T19:13:17.931+0000    connected to: mongodb://localhost/
2022-03-01T19:13:17.956+0000    5 document(s) imported successfully. 0 document(s) failed to import
```
Загружаем слой с заправками и парковками
```
root@mongodb:~# mongoimport -d geodb -c traffic_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/traffic_polygon.geojson
2022-03-01T19:21:26.918+0000    connected to: mongodb://localhost/
2022-03-01T19:21:26.973+0000    282 document(s) imported successfully. 0 document(s) failed to import
```

Загружаем точечный слой с объектами транспортной инфраструктуры
```
root@mongodb:~# mongoimport -d geodb -c transport_point --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/transport_point.geojson
2022-03-01T19:25:17.436+0000    connected to: mongodb://localhost/
2022-03-01T19:25:17.494+0000    832 document(s) imported successfully. 0 document(s) failed to import
```
Загружаем полигональный слой с объектами транспортной инфраструктуры
```
root@mongodb:~# mongoimport -d geodb -c transport_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/transport_polygon.geojson
2022-03-01T19:27:34.590+0000    connected to: mongodb://localhost/
2022-03-01T19:27:34.614+0000    23 document(s) imported successfully. 0 document(s) failed to import.
```

Линейный слой с водными объектами
```
root@mongodb:~# mongoimport -d geodb -c transport_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/transport_polygon.geojson
2022-03-01T19:27:34.590+0000    connected to: mongodb://localhost/
2022-03-01T19:27:34.614+0000    23 document(s) imported successfully. 0 document(s) failed to import.
```

Полигональный слой с водными объектами
```
root@mongodb:~# mongoimport -d geodb -c water_polygon --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/water_polygon.geojson
2022-03-01T19:30:00.389+0000    connected to: mongodb://localhost/
2022-03-01T19:30:00.428+0000    63 document(s) imported successfully. 0 document(s) failed to import.
```

И загрузим кастомный слой с районами он нам нужен будет для демонстрации работы пространтвенного поиска.

```
root@mongodb:~# mongoimport -d geodb -c neighborhoods --jsonArray  -u root -p otus --authenticationDatabase admin --file /root/geojson/neighborhoods.geojson
2022-03-02T13:40:55.907+0000    connected to: mongodb://localhost/
2022-03-02T13:40:55.932+0000    3 document(s) imported successfully. 0 document(s) failed to import.
```

Таким образом мы загрузили все необходимые нам объекты.


Создадим пространсвенный индекс на коллеции с даннми транспортной ифраструк:
```
> db.transport_point.createIndex( {geometry: "2dsphere"} )
{
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
```

Теперь можем найти например автобусные остановки, который находятся на расстояние от 1 метра до 300 от заданной точки:
```
> db.transport_point.find({geometry:{ $near:{$geometry: { type: "Point",  coordinates: [1.44217,43.60335] }, $minDistance: 1,$maxDistance: 300}}})
{ "_id" : ObjectId("621e731de3645b7b48dd05be"), "type" : "Feature", "properties" : { "osm_id" : "2483297512", "lastchange" : "2013-10-05T08:46:20Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Rémusat" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.443796, 43.605236 ] } }
{ "_id" : ObjectId("621e731de3645b7b48dd02dd"), "type" : "Feature", "properties" : { "osm_id" : "248533209", "lastchange" : "2013-10-03T20:04:48Z", "code" : 5601, "fclass" : "railway_station", "geomtype" : "N", "name" : "Capitole" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.445382, 43.6044765 ] } }
```

Отсортируем объекты по увелчиению расстояния от заданной точки
```
> db.transport_point.aggregate( [{$geoNear: {near: { type: "Point", coordinates: [ 1.44217,43.60335] },spherical: true,distanceField: "calcDistance"}}] )
{ "_id" : ObjectId("621e731de3645b7b48dd05c7"), "type" : "Feature", "properties" : { "osm_id" : "2483309109", "lastchange" : "2013-10-05T08:58:23Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Gambetta" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.442168, 43.603353 ] }, "calcDistance" : 0.3708349008718456 }
{ "_id" : ObjectId("621e731de3645b7b48dd05be"), "type" : "Feature", "properties" : { "osm_id" : "2483297512", "lastchange" : "2013-10-05T08:46:20Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Rémusat" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.443796, 43.605236 ] }, "calcDistance" : 247.50143705815427 }
{ "_id" : ObjectId("621e731de3645b7b48dd02dd"), "type" : "Feature", "properties" : { "osm_id" : "248533209", "lastchange" : "2013-10-03T20:04:48Z", "code" : 5601, "fclass" : "railway_station", "geomtype" : "N", "name" : "Capitole" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.445382, 43.6044765 ] }, "calcDistance" : 287.6846107333694 }
{ "_id" : ObjectId("621e731de3645b7b48dd030f"), "type" : "Feature", "properties" : { "osm_id" : "474675725", "lastchange" : "2013-05-20T17:41:14Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4439563, 43.6004537 ] }, "calcDistance" : 353.1074880165126 }
{ "_id" : ObjectId("621e731de3645b7b48dd0589"), "type" : "Feature", "properties" : { "osm_id" : "2063891820", "lastchange" : "2013-05-06T09:27:09Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4439666, 43.6003883 ] }, "calcDistance" : 360.1002978559844 }
{ "_id" : ObjectId("621e731de3645b7b48dd030d"), "type" : "Feature", "properties" : { "osm_id" : "474675719", "lastchange" : "2009-08-26T17:59:34Z", "code" : 5641, "fclass" : "taxi", "geomtype" : "N", "name" : null }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4438174, 43.6002537 ] }, "calcDistance" : 369.37463429162364 }
{ "_id" : ObjectId("621e731de3645b7b48dd0581"), "type" : "Feature", "properties" : { "osm_id" : "2063842798", "lastchange" : "2013-05-06T09:27:09Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4444705, 43.6004219 ] }, "calcDistance" : 375.0141654471736 }
{ "_id" : ObjectId("621e731de3645b7b48dd051d"), "type" : "Feature", "properties" : { "osm_id" : "1737101673", "lastchange" : "2013-05-20T17:51:20Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.441918, 43.599981 ] }, "calcDistance" : 375.58295732221006 }
{ "_id" : ObjectId("621e731de3645b7b48dd030e"), "type" : "Feature", "properties" : { "osm_id" : "474675724", "lastchange" : "2013-05-20T17:41:14Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4444912, 43.6003449 ] }, "calcDistance" : 383.29969899337675 }
{ "_id" : ObjectId("621e731de3645b7b48dd02de"), "type" : "Feature", "properties" : { "osm_id" : "248533272", "lastchange" : "2015-12-10T16:20:35Z", "code" : 5601, "fclass" : "railway_station", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4439146, 43.6001467 ] }, "calcDistance" : 383.31837299950445 }
{ "_id" : ObjectId("621e731de3645b7b48dd02df"), "type" : "Feature", "properties" : { "osm_id" : "248533272", "lastchange" : "2015-12-10T16:20:35Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Esquirol" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4439146, 43.6001467 ] }, "calcDistance" : 383.31837299950445 }
{ "_id" : ObjectId("621e731de3645b7b48dd037e"), "type" : "Feature", "properties" : { "osm_id" : "1149065269", "lastchange" : "2013-05-20T17:51:20Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4415376, 43.5999212 ] }, "calcDistance" : 385.0794023746599 }
{ "_id" : ObjectId("621e731de3645b7b48dd058c"), "type" : "Feature", "properties" : { "osm_id" : "2063891826", "lastchange" : "2012-12-12T22:11:40Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4415602, 43.5998601 ] }, "calcDistance" : 391.58928964707025 }
{ "_id" : ObjectId("621e731de3645b7b48dd05c8"), "type" : "Feature", "properties" : { "osm_id" : "2483309356", "lastchange" : "2013-10-05T08:58:38Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Quai de la Daurade" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.440178, 43.600055 ] }, "calcDistance" : 400.4053707645651 }
{ "_id" : ObjectId("621e731de3645b7b48dd0586"), "type" : "Feature", "properties" : { "osm_id" : "2063842813", "lastchange" : "2012-12-12T21:35:33Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4409102, 43.599703 ] }, "calcDistance" : 418.48897257096377 }
{ "_id" : ObjectId("621e731de3645b7b48dd036e"), "type" : "Feature", "properties" : { "osm_id" : "1149065200", "lastchange" : "2013-05-20T17:51:20Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.44094, 43.5996226 ] }, "calcDistance" : 426.61230291340644 }
{ "_id" : ObjectId("621e731de3645b7b48dd038c"), "type" : "Feature", "properties" : { "osm_id" : "1152145881", "lastchange" : "2011-02-14T02:00:29Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : null }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4462806, 43.600595 ] }, "calcDistance" : 451.50281625112626 }
{ "_id" : ObjectId("621e731de3645b7b48dd038e"), "type" : "Feature", "properties" : { "osm_id" : "1152145893", "lastchange" : "2011-02-14T02:00:29Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : null }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4465943, 43.6006006 ] }, "calcDistance" : 469.96918970150745 }
{ "_id" : ObjectId("621e731de3645b7b48dd051e"), "type" : "Feature", "properties" : { "osm_id" : "1737101674", "lastchange" : "2013-05-20T17:51:21Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4405233, 43.5992973 ] }, "calcDistance" : 470.26588030609804 }
{ "_id" : ObjectId("621e731de3645b7b48dd0520"), "type" : "Feature", "properties" : { "osm_id" : "1737101675", "lastchange" : "2013-05-20T17:48:07Z", "code" : 5621, "fclass" : "bus_stop", "geomtype" : "N", "name" : "Pont Neuf" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.440534, 43.5990487 ] }, "calcDistance" : 496.64609013589 }
Type "it" for more
```

Создадим пространственный индекс на коллекции с развлекательными местами:
```
> db.entertaiment_point.geojson.createIndex( {geometry: "2dsphere"} )
{
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
```
И на кастомной коллекции эмитирующей нам границе районов:
```
> db.neighborhoods.createIndex( {geometry: "2dsphere"} )
{
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
```

Теперь предположим что человек находится в точке с координатами 1.41662,43.60875.Найдем для начала район в котором находится этот человек:
```
> db.neighborhoods.findOne( { geometry: { $geoIntersects: { $geometry: { type: "Point", coordinates: [ 1.41662,43.60875 ] } } } } )
{
        "_id" : ObjectId("621f73e7f67e96264fad322b"),
        "type" : "Feature",
        "properties" : {
                "id" : 1,
                "name" : "Первый район"
        },
        "geometry" : {
                "type" : "MultiPolygon",
                "coordinates" : [
                        [
                                [
                                        [
                                                1.407876323008902,
                                                43.635400593558344
                                        ],
                                        [
                                                1.439247221305361,
                                                43.63581702141184
                                        ],
                                        [
                                                1.438969602736365,
                                                43.58223663759576
                                        ],
                                        [
                                                1.408292750862394,
                                                43.58251425616476
                                        ],
                                        [
                                                1.407876323008902,
                                                43.635400593558344
                                        ]
                                ]
                        ]
                ]
```
Район найден верно.
Зададим переменную:
```
var neighborhood = db.neighborhoods.findOne( { geometry: { $geoIntersects: { $geometry: { type: "Point", coordinates: [ 1.41662,43.60875 ] } } } })
```
И найдем количество интересных мест в этом районе
```
> db.entertaiment_point.geojson.find( { geometry: { $geoWithin: { $geometry: neighborhood.geometry } } } ).count()
1710
```


Теперь найдем объекты определенного типа и их расстояние от заданной точки:
```
db.entertaiment_point.geojson.aggregate( [{$geoNear: {near: { type: "Point", coordinates: [ 1.44217,43.60335] },spherical: true,query: {"properties.fclass":"museum"},distanceField: "calcDistance"}}])

{ "_id" : ObjectId("621e6694d3dafbf4260ddcb6"), "type" : "Feature", "properties" : { "osm_id" : "247081244", "lastchange" : "2013-01-27T01:42:45Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Musée du Vieux Toulouse" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4431296, 43.6023148 ] }, "calcDistance" : 138.7919177957004 }
{ "_id" : ObjectId("621e6694d3dafbf4260df493"), "type" : "Feature", "properties" : { "osm_id" : "1562254", "lastchange" : "2017-04-09T08:58:40Z", "code" : 2722, "fclass" : "museum", "geomtype" : "R", "name" : "Église des Jacobins" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.440231824532923, 43.603565057998935 ] }, "calcDistance" : 158.0585804363102 }
{ "_id" : ObjectId("621e6694d3dafbf4260df48f"), "type" : "Feature", "properties" : { "osm_id" : "1562255", "lastchange" : "2017-01-16T20:42:29Z", "code" : 2722, "fclass" : "museum", "geomtype" : "R", "name" : "Ensemble conventuel des Jacobins" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.44009895349773, 43.60362629766737 ] }, "calcDistance" : 169.75523205223448 }
{ "_id" : ObjectId("621e6694d3dafbf4260defd5"), "type" : "Feature", "properties" : { "osm_id" : "216048", "lastchange" : "2017-01-16T20:42:30Z", "code" : 2722, "fclass" : "museum", "geomtype" : "R", "name" : "Hôtel d'Assézat" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.441986891725654, 43.600385334680674 ] }, "calcDistance" : 330.3530465282429 }
{ "_id" : ObjectId("621e6694d3dafbf4260de271"), "type" : "Feature", "properties" : { "osm_id" : "1982847379", "lastchange" : "2017-05-15T13:57:42Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Fondation Bemberg" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4420464, 43.6001991 ] }, "calcDistance" : 350.8960340081876 }
{ "_id" : ObjectId("621e6694d3dafbf4260def5b"), "type" : "Feature", "properties" : { "osm_id" : "22718219", "lastchange" : "2017-01-16T20:43:30Z", "code" : 2722, "fclass" : "museum", "geomtype" : "W", "name" : "Musée des Augustins" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.446336081524159, 43.60103131119118 ] }, "calcDistance" : 423.5634022923198 }
{ "_id" : ObjectId("621e6694d3dafbf4260de0d4"), "type" : "Feature", "properties" : { "osm_id" : "1533753894", "lastchange" : "2016-10-20T06:29:26Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "galerie Alain Daudet" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4449735, 43.5995448 ] }, "calcDistance" : 480.10728623680586 }
{ "_id" : ObjectId("621e6694d3dafbf4260df01d"), "type" : "Feature", "properties" : { "osm_id" : "61565556", "lastchange" : "2017-03-29T16:34:51Z", "code" : 2722, "fclass" : "museum", "geomtype" : "W", "name" : "Château d'eau Charles Laganne" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4369308108834, 43.59869722767629 ] }, "calcDistance" : 668.3099238426131 }
{ "_id" : ObjectId("621e6694d3dafbf4260de13d"), "type" : "Feature", "properties" : { "osm_id" : "1649207016", "lastchange" : "2013-01-27T01:42:45Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Musée de La Médecine" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4363753, 43.5989613 ] }, "calcDistance" : 675.9303497751765 }
{ "_id" : ObjectId("621e6694d3dafbf4260de13a"), "type" : "Feature", "properties" : { "osm_id" : "1649207015", "lastchange" : "2012-02-26T19:01:14Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Musée Paul Dupuy" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4468993, 43.5969677 ] }, "calcDistance" : 806.2983423664398 }
{ "_id" : ObjectId("621e6694d3dafbf4260def6b"), "type" : "Feature", "properties" : { "osm_id" : "22896377", "lastchange" : "2017-04-03T10:47:21Z", "code" : 2722, "fclass" : "museum", "geomtype" : "W", "name" : "Les Abattoirs" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.429599020046863, 43.60091765984273 ] }, "calcDistance" : 1048.9107696550925 }
{ "_id" : ObjectId("621e6694d3dafbf4260de139"), "type" : "Feature", "properties" : { "osm_id" : "1649207014", "lastchange" : "2017-04-18T14:06:47Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Musée de l'Affiche de Toulouse" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4305059, 43.5986552 ] }, "calcDistance" : 1075.7537727295241 }
{ "_id" : ObjectId("621e6694d3dafbf4260ded0b"), "type" : "Feature", "properties" : { "osm_id" : "4011776228", "lastchange" : "2016-02-17T21:42:52Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Quai des savoirs" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4510813, 43.5945205 ] }, "calcDistance" : 1217.4373095229862 }
{ "_id" : ObjectId("621e6694d3dafbf4260df45c"), "type" : "Feature", "properties" : { "osm_id" : "134123637", "lastchange" : "2015-11-21T01:27:32Z", "code" : 2722, "fclass" : "museum", "geomtype" : "W", "name" : "Museum d'Histoire Naturelle" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.449406413020514, 43.59330995739437 ] }, "calcDistance" : 1260.7361327030164 }
{ "_id" : ObjectId("621e6694d3dafbf4260df03a"), "type" : "Feature", "properties" : { "osm_id" : "63801451", "lastchange" : "2014-01-09T15:55:23Z", "code" : 2722, "fclass" : "museum", "geomtype" : "W", "name" : "Musée Georges Labit" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.458466954289851, 43.59111218785981 ] }, "calcDistance" : 1892.6143304529953 }
{ "_id" : ObjectId("621e6694d3dafbf4260de13b"), "type" : "Feature", "properties" : { "osm_id" : "1649207017", "lastchange" : "2013-12-12T22:40:29Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Musée de La Résistance et de la Déportation" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4566916, 43.5889001 ] }, "calcDistance" : 1989.4734084609152 }
{ "_id" : ObjectId("621e6694d3dafbf4260de2b8"), "type" : "Feature", "properties" : { "osm_id" : "2083634308", "lastchange" : "2016-10-28T13:32:11Z", "code" : 2722, "fclass" : "museum", "geomtype" : "N", "name" : "Les Jardins du Muséum" }, "geometry" : { "type" : "Point", "coordinates" : [ 1.4517509, 43.6323938 ] }, "calcDistance" : 3324.042200555872 }
```

В результате работы, был создан геоинформационнй проект на базе qgis. Доработан плагин для работы mongodb c qgis.     
Найдены и сконвертированы в формат json shape-файлы, которые являются базовыми для работы с qgis.     
Настроены стили отображения пространственных данных. Построены пространсвенные индексы и так же осуществлен поиск    
объектов на основании этих индексов.