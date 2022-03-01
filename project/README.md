В качестве проектной работы была выбрана геоинформационная система реализованная на      
базе приложения QGIS (`https://www.qgis.org/ru/site/`). В качестве хранилища данных используется mongodb.     
В качестве исходных данных используются данные Open Street Map для Тулузы в формате shape-файлов.       
`https://www.geofabrik.de/data/shapefiles.html` - ссылка на исходники.        
Далее с помощью qgis эти данные можно переконвертировать в формат geojson для импорта        
в mongodb. Для того чтобы можно было подгружать данные в qgis из mongodb нужен специльный плагин:     
`https://plugins.qgis.org/plugins/MongoConnector/`. Поссле его загрузки и настройки коннекта,мы       
можем загружать данные непосредственно из нашей бд.

Так же необходимо открыть порты для соединения с бд в google platform. Создаем новое правило firewall add rule.

Переходим к импорту необходимых слоев

root@mongo1:~# mongoimport -d geodb -h mongo4:27000 -c roads --jsonArray  -u userroot --authenticationDatabase admin --file /root/roads_line.geojson
Enter password:

2022-01-19T12:44:26.629+0000    connected to: mongodb://mongo4:27000/
2022-01-19T12:44:29.629+0000    [#########...............] geodb.roads  11.6MB/30.3MB (38.2%)
2022-01-19T12:44:32.637+0000    [##################......] geodb.roads  23.2MB/30.3MB (76.5%)
2022-01-19T12:44:34.542+0000    [########################] geodb.roads  30.3MB/30.3MB (100.0%)
2022-01-19T12:44:34.542+0000    62236 document(s) imported successfully. 0 document(s) failed to import.


