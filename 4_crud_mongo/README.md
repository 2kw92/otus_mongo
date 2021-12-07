# CRUD mongo_db
ДЗ отус по основным командам mongodb     

В качестве тестовых данных был выбран датасет содержащий информацию о вине доступный по адресу:        
`https://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-red.csv`      

После того как он был скачан, импортируем его в бд otus на машине  mongodb в GCP по адресу:       
`https://console.cloud.google.com/iam-admin/settings?project=mongo2021-19920819`        
С помощью следующей команды:       
```
root@mongodb:~# mongoimport --type csv -d otus -c vine --headerline /root/winequality.csv -u root --authenticationDatabase admin
Enter password:

2021-12-06T15:08:19.139+0000    connected to: mongodb://localhost/

2021-12-06T15:08:19.195+0000    1599 document(s) imported successfully. 0 document(s) failed to import.
```         
Видим что данные успешно загружены.       

Подключаемся к нашей бд и проверяем что действительно загружено 1599 документов:       
```
root@mongodb:~# mongo --port 27017 -u root -p otus --authenticationDatabase admin
MongoDB shell version v5.0.4
connecting to: mongodb://127.0.0.1:27017/?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("31061248-ebfd-4e1b-be3f-fa7cd3f2451f") }
MongoDB server version: 5.0.4
================
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
For installation instructions, see
https://docs.mongodb.com/mongodb-shell/install/
================
> use otus
switched to db otus
> db.vine.count();
1599
```       

Посмотрим содержимое документов используя лимит:      
```
> db.vine.find().limit(5);
{ "_id" : ObjectId("61ae276360b74fc4d4ae801a"), "fixed acidity" : 7.8, "volatile acidity" : 0.88, "citric acid" : 0, "residual sugar" : 2.6, "chlorides" : 0.098, "free sulfur dioxide" : 25, "total sulfur dioxide" : 67, "density" : 0.9968, "pH" : 3.2, "sulphates" : 0.68, "alcohol" : 9.8, "quality" : 5 }
{ "_id" : ObjectId("61ae276360b74fc4d4ae801b"), "fixed acidity" : 7.4, "volatile acidity" : 0.7, "citric acid" : 0, "residual sugar" : 1.9, "chlorides" : 0.076, "free sulfur dioxide" : 11, "total sulfur dioxide" : 34, "density" : 0.9978, "pH" : 3.51, "sulphates" : 0.56, "alcohol" : 9.4, "quality" : 5 }
{ "_id" : ObjectId("61ae276360b74fc4d4ae801c"), "fixed acidity" : 7.8, "volatile acidity" : 0.76, "citric acid" : 0.04, "residual sugar" : 2.3, "chlorides" : 0.092, "free sulfur dioxide" : 15, "total sulfur dioxide" : 54, "density" : 0.997, "pH" : 3.26, "sulphates" : 0.65, "alcohol" : 9.8, "quality" : 5 }
{ "_id" : ObjectId("61ae276360b74fc4d4ae801d"), "fixed acidity" : 11.2, "volatile acidity" : 0.28, "citric acid" : 0.56, "residual sugar" : 1.9, "chlorides" : 0.075, "free sulfur dioxide" : 17, "total sulfur dioxide" : 60, "density" : 0.998, "pH" : 3.16, "sulphates" : 0.58, "alcohol" : 9.8, "quality" : 6 }
{ "_id" : ObjectId("61ae276360b74fc4d4ae801e"), "fixed acidity" : 7.4, "volatile acidity" : 0.7, "citric acid" : 0, "residual sugar" : 1.9, "chlorides" : 0.076, "free sulfur dioxide" : 11, "total sulfur dioxide" : 34, "density" : 0.9978, "pH" : 3.51, "sulphates" : 0.56, "alcohol" : 9.4, "quality" : 5 }
```           
Видим что лимит работает корректно и отображает данные.        

Посомторим количество вин, определенной крепости:      
```
> db.vine.aggregate([{$group : {_id: "$alcohol", count_vine : {$sum : 1}}} ] )
{ "_id" : 9.3, "count_vine" : 59 }
{ "_id" : 9.95, "count_vine" : 1 }
{ "_id" : 11.0666666666667, "count_vine" : 1 }
{ "_id" : 9.5, "count_vine" : 139 }
{ "_id" : 12.2, "count_vine" : 12 }
{ "_id" : 12.7, "count_vine" : 9 }
{ "_id" : 11.95, "count_vine" : 1 }
{ "_id" : 10.0333333333333, "count_vine" : 2 }
{ "_id" : 8.8, "count_vine" : 2 }
{ "_id" : 9.05, "count_vine" : 1 }
{ "_id" : 9.23333333333333, "count_vine" : 1 }
{ "_id" : 13.2, "count_vine" : 1 }
{ "_id" : 9, "count_vine" : 30 }
{ "_id" : 13.5666666666667, "count_vine" : 1 }
{ "_id" : 8.4, "count_vine" : 2 }
{ "_id" : 13.6, "count_vine" : 4 }
{ "_id" : 10.5, "count_vine" : 67 }
{ "_id" : 9.25, "count_vine" : 1 }
{ "_id" : 10.7, "count_vine" : 27 }
{ "_id" : 12.9, "count_vine" : 9 }
```       

Теперь давайте обновим крепость у вина с `_id=ObjectId("61ae276360b74fc4d4ae801e")`      
```
> db.vine.find({"_id" : ObjectId("61ae276360b74fc4d4ae801e")})
{ "_id" : ObjectId("61ae276360b74fc4d4ae801e"), "fixed acidity" : 7.4, "volatile acidity" : 0.7, "citric acid" : 0, "residual sugar" : 1.9, "chlorides" : 0.076, "free sulfur dioxide" : 11, "total sulfur dioxide" : 34, "density" : 0.9978, "pH" : 3.51, "sulphates" : 0.56, "alcohol" : 9.4, "quality" : 5 }

> db.vine.update({"_id" : ObjectId("61ae276360b74fc4d4ae801e")},{$set: {"alcohol":10}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

> db.vine.find({"_id" : ObjectId("61ae276360b74fc4d4ae801e")})
{ "_id" : ObjectId("61ae276360b74fc4d4ae801e"), "fixed acidity" : 7.4, "volatile acidity" : 0.7, "citric acid" : 0, "residual sugar" : 1.9, "chlorides" : 0.076, "free sulfur dioxide" : 11, "total sulfur dioxide" : 34, "density" : 0.9978, "pH" : 3.51, "sulphates" : 0.56, "alcohol" : 10, "quality" : 5 } 
```      
Видим что данные успешно обновились.       

Теперь обновим несколько документов. Например мы хотим quality у всех вин у которых pH строго меньше 2.9
```
> db.vine.update({"pH" : {$lt : 2.9}},{$set : {"quality":9}},{multi : true})
WriteResult({ "nMatched" : 9, "nUpserted" : 0, "nModified" : 9 })
```      
И теперь проверяем:
```
> db.vine.find({"pH" : {$lt : 2.9}},{pH :1 , quality:1, _id : 0 })
{ "pH" : 2.74, "quality" : 9 }
{ "pH" : 2.88, "quality" : 9 }
{ "pH" : 2.86, "quality" : 9 }
{ "pH" : 2.87, "quality" : 9 }
{ "pH" : 2.89, "quality" : 9 }
{ "pH" : 2.89, "quality" : 9 }
{ "pH" : 2.89, "quality" : 9 }
{ "pH" : 2.89, "quality" : 9 }
{ "pH" : 2.88, "quality" : 9 }
```
Видим что данные успешно обновились.


Теперь я хочу удалить все элементый у которых "citric acid" <=1. Для этого выполняю запрос:
```
> db.vine.deleteMany({"citric acid" : {$lte : 1}})
{ "acknowledged" : true, "deletedCount" : 1599 }
```
Так получилось что этим запросом мы удалили все данные из колекции.Тепеь можно удлаить и саму коллекцию        
```
> db.vine.drop();
true
```

И удалить саму базу
```
> db.dropDatabase()
{ "ok" : 1 }
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```
Таким образом мы оттачили навыки работы с CRUD в Mongodb