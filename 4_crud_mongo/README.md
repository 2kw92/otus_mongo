# CRUD mongo_db
ДЗ отус по основным командам mongodb     

В качестве тестовых данных был выбран датасет доступный по адресу:        
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
Видим что все отображается.

