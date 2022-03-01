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

Таким образом мы загрузили все необходимые нам объекты.


В результате работы, был создан геоинформационнй проект на базе qgis. Доработан плагин для работы mongodb c qgis.     
Найдены и сконвертированы в формат json shape-файлы, которые являются базовыми для работы с qgis.     
Настроены стили отображения пространсвенных данных.