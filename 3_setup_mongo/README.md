# install mongo_db
ДЗ отус по установке mongodb     

Создал проект  Google Cloud Platform доступен по ссылке:      
https://console.cloud.google.com/iam-admin/settings?project=mongo2021-19920819       
Создал новую вируталку с ubuntu-2104-hirsute-v20211119	на борту.         
Добавил свой ключ ssh в GCE metadata и подключился под пользователем root со своей        
машины через mobaxterm.
Поставил mongo 5.0 c помощью следующих команд        

```
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org -5.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```       
При такой установке появляется сервис mongod который можно запустить:      
```
systemctl start mongod
```
При таком запуске используется файл конфигурации `/etc/mongod.conf`    

Теперь подключимся и создадим там новую бд и вставим туда данные:            
```        
mongo --port 27017
> use otus
switched to db otus
> show databases
> db.peoples.insert({'Name':'Mickey'})
WriteResult({ "nInserted" : 1 })
> use otus
switched to db otus
> db.peoples.insert({'Name':'Mickey'})
WriteResult({ "nInserted" : 1 })
> db.peoples.insert({'Name':'Mickey'})
WriteResult({ "nInserted" : 1 })
> db.peoples.find()
{ "_id" : ObjectId("61a4e1a06483e9b2f7fe2106"), "Name" : "Mickey" }
{ "_id" : ObjectId("61a4e1a86483e9b2f7fe2107"), "Name" : "Mickey" }
{ "_id" : ObjectId("61a4e1aa6483e9b2f7fe2108"), "Name" : "Mickey" }
> show databases
admin   0.000GB
config  0.000GB
local   0.000GB
otus    0.000GB
```        
Далее для настройки авторизации нам нужно создать нового юзера в бд admin и задать необходимые права:
```       
> use admin;
switched to db admin
> db.createUser( { user: "root", pwd: "otus", roles: [ "userAdminAnyDatabase", "dbAdminAnyDatabase", "readWriteAnyDatabase" ] } )
```       
После все манипуляций для того чтобы mongo работало с авторизацией и была открыта во вне      
файл конфигурации должен выглядеть следующим образом:         
```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1
  bindIpAll: true


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:
```     

После этого сделав рестарт сервера пробуем подключиться без указания авторизации:       
```
root@mongodb:~# mongo --port 27017
MongoDB shell version v5.0.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("d370a46c-1a9c-414a-b560-bd6d3dc38723") }
MongoDB server version: 5.0.4
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
> show databases
>
bye
```       
Видно что без авторизации нам ничего недоступно.     
```
root@mongodb:~# mongo --port 27017 -u root -p otus --authenticationDatabase admin
MongoDB shell version v5.0.4
connecting to: mongodb://127.0.0.1:27017/?authSource=admin&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("fced72a2-8fbe-49d1-a05c-59947b4d04cd") }
MongoDB server version: 5.0.4
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
> show databases
admin   0.000GB
config  0.000GB
local   0.000GB
otus    0.000GB
```     
А вот если мы передадим нашего пользователя то авторизация становится доступной.



