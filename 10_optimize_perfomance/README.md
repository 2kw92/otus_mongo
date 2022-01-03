# optimization mongo_db
ДЗ отус оптимизация производительности     

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


Теперь включим раширенное профилирование запросов:
```
db.setProfilingLevel(2)
{ "was" : 0, "slowms" : 100, "sampleRate" : 1, "ok" : 1 }
```      

Теперь мы можем посомтреть информацию о выполняемых запросах:
```
db.system.profile.find().sort({$natural:-1});
```
Получим следующий вывод        
```     
{ "op" : "query", "ns" : "otus.system.profile", "command" : { "find" : "system.profile", "filter" : {  }, "sort" : { "$natural" : -1 }, "lsid" : { "id" : UUID("fb3489bf-228d-434d-8313-2845970698fe") }, "$db" : "otus" }, "keysExamined" : 0, "docsExamined" : 15, "cursorExhausted" : true, "numYield" : 0, "nreturned" : 15, "locks" : { "Global" : { "acquireCount" : { "r" : NumberLong(1) } }, "Mutex" : { "acquireCount" : { "r" : NumberLong(1) } } }, "flowControl" : {  }, "responseLength" : 11606, "protocol" : "op_msg", "millis" : 0, "planSummary" : "COLLSCAN", "execStats" : { "stage" : "COLLSCAN", "nReturned" : 15, "executionTimeMillisEstimate" : 0, "works" : 17, "advanced" : 15, "needTime" : 1, "needYield" : 0, "saveState" : 0, "restoreState" : 0, "isEOF" : 1, "direction" : "backward", "docsExamined" : 15 }, "ts" : ISODate("2022-01-02T15:39:51.956Z"), "client" : "127.0.0.1", "appName" : "MongoDB Shell", "allUsers" : [ { "user" : "root", "db" : "admin" } ], "user" : "root@admin" }
{ "op" : "query", "ns" : "otus.system.profile", "command" : { "find" : "system.profile", "filter" : {  }, "sort" : { "$natural" : -1 }, "lsid" : { "id" : UUID("fb3489bf-228d-434d-8313-2845970698fe") }, "$db" : "otus" }, "keysExamined" : 0, "docsExamined" : 14, "cursorExhausted" : true, "numYield" : 0, "nreturned" : 14, "locks" : { "Global" : { "acquireCount" : { "r" : NumberLong(1) } }, "Mutex" : { "acquireCount" : { "r" : NumberLong(1) } } }, "flowControl" : {  }, "responseLength" : 10789, "protocol" : "op_msg", "millis" : 0, "planSummary" : "COLLSCAN", "execStats" : { "stage" : "COLLSCAN", "nReturned" : 14, "executionTimeMillisEstimate" : 0, "works" : 16, "advanced" : 14, "needTime" : 1, "needYield" : 0, "saveState" : 0, "restoreState" : 0, "isEOF" : 1, "direction" : "backward", "docsExamined" : 14 }, "ts" : ISODate("2022-01-02T15:38:45.708Z"), "client" : "127.0.0.1", "appName" : "MongoDB Shell", "allUsers" : [ { "user" : "root", "db" : "admin" } ], "user" : "root@admin" }
{ "op" : "command", "ns" : "otus.vine", "command" : { "explain" : { "find" : "vine", "filter" : { "pH" : { "$lt" : 3 } }, "projection" : { "pH" : 1, "quality" : 1, "_id" : 0 } }, "verbosity" : "queryPlanner", "lsid" : { "id" : UUID("fb3489bf-228d-434d-8313-2845970698fe") }, "$db" : "otus" }, "numYield" : 0, "locks" : { "Global" : { "acquireCount" : { "r" : NumberLong(1) } }, "Mutex" : { "acquireCount" : { "r" : NumberLong(1) } } }, "flowControl" : {  }, "responseLength" : 1183, "protocol" : "op_msg", "millis" : 0, "ts" : ISODate("2022-01-02T15:38:37.378Z"), "client" : "127.0.0.1", "appName" : "MongoDB Shell", "allUsers" : [ { "user" : "root", "db" : "admin" } ], "user" : "root@admin" }
Type "it" for more
```

Помимого этого мы можем использовать метод explain, который может содержать параметр verbose      
Есть 3 режима: "queryPlanner", "executionStats"и "allPlansExecution"       
Режим по умолчанию "queryPlanner".        
Посмотрим на план запроса, котрый покажет нам вина у, которых pH строго меньше 2.9        
```
> db.vine.explain().find({"pH" : {$lt : 2.9}},{pH :1 , quality:1, _id : 0 })
{
        "explainVersion" : "1",
        "queryPlanner" : {
                "namespace" : "otus.vine",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "pH" : {
                                "$lt" : 2.9
                        }
                },
                "queryHash" : "18EE869B",
                "planCacheKey" : "A525489A",
                "maxIndexedOrSolutionsReached" : false,
                "maxIndexedAndSolutionsReached" : false,
                "maxScansToExplodeReached" : false,
                "winningPlan" : {
                        "stage" : "PROJECTION_SIMPLE",
                        "transformBy" : {
                                "pH" : 1,
                                "quality" : 1,
                                "_id" : 0
                        },
                        "inputStage" : {
                                "stage" : "COLLSCAN",
                                "filter" : {
                                        "pH" : {
                                                "$lt" : 2.9
                                        }
                                },
                                "direction" : "forward"
                        }
                },
                "rejectedPlans" : [ ]
        },
        "command" : {
                "find" : "vine",
                "filter" : {
                        "pH" : {
                                "$lt" : 2.9
                        }
                },
                "projection" : {
                        "pH" : 1,
                        "quality" : 1,
                        "_id" : 0
                },
                "$db" : "otus"
        },
        "serverInfo" : {
                "host" : "mongodb",
                "port" : 27017,
                "version" : "5.0.4",
                "gitVersion" : "62a84ede3cc9a334e8bc82160714df71e7d3a29e"
        },
        "serverParameters" : {
                "internalQueryFacetBufferSizeBytes" : 104857600,
                "internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
                "internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
                "internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
                "internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
                "internalQueryProhibitBlockingMergeOnMongoS" : 0,
                "internalQueryMaxAddToSetBytes" : 104857600,
                "internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
        },
        "ok" : 1
}
```      

Теперь посмотрим на время выполнения запроса по поиску вин по хлоридам без индекса:        
```     
> db.vine.explain("executionStats").find({"chlorides" : 0.078})
{
        "explainVersion" : "1",
        "queryPlanner" : {
                "namespace" : "otus.vine",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "chlorides" : {
                                "$eq" : 0.078
                        }
                },
                "maxIndexedOrSolutionsReached" : false,
                "maxIndexedAndSolutionsReached" : false,
                "maxScansToExplodeReached" : false,
                "winningPlan" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "chlorides" : {
                                        "$eq" : 0.078
                                }
                        },
                        "direction" : "forward"
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 51,
                "executionTimeMillis" : 3,
                "totalKeysExamined" : 0,
                "totalDocsExamined" : 1599,
                "executionStages" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "chlorides" : {
                                        "$eq" : 0.078
                                }
                        },
                        "nReturned" : 51,
                        "executionTimeMillisEstimate" : 0,
                        "works" : 1601,
                        "advanced" : 51,
                        "needTime" : 1549,
                        "needYield" : 0,
                        "saveState" : 1,
                        "restoreState" : 1,
                        "isEOF" : 1,
                        "direction" : "forward",
                        "docsExamined" : 1599
                }
        },
        "command" : {
                "find" : "vine",
                "filter" : {
                        "chlorides" : 0.078
                },
                "$db" : "otus"
        },
        "serverInfo" : {
                "host" : "mongodb",
                "port" : 27017,
                "version" : "5.0.4",
                "gitVersion" : "62a84ede3cc9a334e8bc82160714df71e7d3a29e"
        },
        "serverParameters" : {
                "internalQueryFacetBufferSizeBytes" : 104857600,
                "internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
                "internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
                "internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
                "internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
                "internalQueryProhibitBlockingMergeOnMongoS" : 0,
                "internalQueryMaxAddToSetBytes" : 104857600,
                "internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
        },
        "ok" : 1
}
```

Видим что `"needTime" : 1549` и `"stage" : "COLLSCAN"`       
Теперь добавим индекс по полю поиска:          
```
> db.vine.createIndex({chlorides : 1})
{
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
```
И теперь снова проверим план и времы выполнения запроса:       
```
> db.vine.explain("executionStats").find({"chlorides" : 0.078})
{
        "explainVersion" : "1",
        "queryPlanner" : {
                "namespace" : "otus.vine",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "chlorides" : {
                                "$eq" : 0.078
                        }
                },
                "maxIndexedOrSolutionsReached" : false,
                "maxIndexedAndSolutionsReached" : false,
                "maxScansToExplodeReached" : false,
                "winningPlan" : {
                        "stage" : "FETCH",
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "keyPattern" : {
                                        "chlorides" : 1
                                },
                                "indexName" : "chlorides_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "chlorides" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "chlorides" : [
                                                "[0.078, 0.078]"
                                        ]
                                }
                        }
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 51,
                "executionTimeMillis" : 2,
                "totalKeysExamined" : 51,
                "totalDocsExamined" : 51,
                "executionStages" : {
                        "stage" : "FETCH",
                        "nReturned" : 51,
                        "executionTimeMillisEstimate" : 0,
                        "works" : 52,
                        "advanced" : 51,
                        "needTime" : 0,
                        "needYield" : 0,
                        "saveState" : 0,
                        "restoreState" : 0,
                        "isEOF" : 1,
                        "docsExamined" : 51,
                        "alreadyHasObj" : 0,
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "nReturned" : 51,
                                "executionTimeMillisEstimate" : 0,
                                "works" : 52,
                                "advanced" : 51,
                                "needTime" : 0,
                                "needYield" : 0,
                                "saveState" : 0,
                                "restoreState" : 0,
                                "isEOF" : 1,
                                "keyPattern" : {
                                        "chlorides" : 1
                                },
                                "indexName" : "chlorides_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "chlorides" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "chlorides" : [
                                                "[0.078, 0.078]"
                                        ]
                                },
                                "keysExamined" : 51,
                                "seeks" : 1,
                                "dupsTested" : 0,
                                "dupsDropped" : 0
                        }
                }
        },
        "command" : {
                "find" : "vine",
                "filter" : {
                        "chlorides" : 0.078
                },
                "$db" : "otus"
        },
        "serverInfo" : {
                "host" : "mongodb",
                "port" : 27017,
                "version" : "5.0.4",
                "gitVersion" : "62a84ede3cc9a334e8bc82160714df71e7d3a29e"
        },
        "serverParameters" : {
                "internalQueryFacetBufferSizeBytes" : 104857600,
                "internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
                "internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
                "internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
                "internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
                "internalQueryProhibitBlockingMergeOnMongoS" : 0,
                "internalQueryMaxAddToSetBytes" : 104857600,
                "internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
        },
        "ok" : 1
}
```

Видим что индекс ускорил работу `"needTime" : 0` и теперь у нас `"stage" : "IXSCAN"`       

Рассмотрим еще один запрос, по 2 условиям:        
```
> db.vine.explain("executionStats").find({ $and: [{"density" : 0.998},{"sulphates" : 0.51}]})
{
        "explainVersion" : "1",
        "queryPlanner" : {
                "namespace" : "otus.vine",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "$and" : [
                                {
                                        "density" : {
                                                "$eq" : 0.998
                                        }
                                },
                                {
                                        "sulphates" : {
                                                "$eq" : 0.51
                                        }
                                }
                        ]
                },
                "maxIndexedOrSolutionsReached" : false,
                "maxIndexedAndSolutionsReached" : false,
                "maxScansToExplodeReached" : false,
                "winningPlan" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "$and" : [
                                        {
                                                "density" : {
                                                        "$eq" : 0.998
                                                }
                                        },
                                        {
                                                "sulphates" : {
                                                        "$eq" : 0.51
                                                }
                                        }
                                ]
                        },
                        "direction" : "forward"
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 2,
                "executionTimeMillis" : 1,
                "totalKeysExamined" : 0,
                "totalDocsExamined" : 1599,
                "executionStages" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "$and" : [
                                        {
                                                "density" : {
                                                        "$eq" : 0.998
                                                }
                                        },
                                        {
                                                "sulphates" : {
                                                        "$eq" : 0.51
                                                }
                                        }
                                ]
                        },
                        "nReturned" : 2,
                        "executionTimeMillisEstimate" : 0,
                        "works" : 1601,
                        "advanced" : 2,
                        "needTime" : 1598,
                        "needYield" : 0,
                        "saveState" : 1,
                        "restoreState" : 1,
                        "isEOF" : 1,
                        "direction" : "forward",
                        "docsExamined" : 1599
                }
        },
        "command" : {
                "find" : "vine",
                "filter" : {
                        "$and" : [
                                {
                                        "density" : 0.998
                                },
                                {
                                        "sulphates" : 0.51
                                }
                        ]
                },
                "$db" : "otus"
        },
        "serverInfo" : {
                "host" : "mongodb",
                "port" : 27017,
                "version" : "5.0.4",
                "gitVersion" : "62a84ede3cc9a334e8bc82160714df71e7d3a29e"
        },
        "serverParameters" : {
                "internalQueryFacetBufferSizeBytes" : 104857600,
                "internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
                "internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
                "internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
                "internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
                "internalQueryProhibitBlockingMergeOnMongoS" : 0,
                "internalQueryMaxAddToSetBytes" : 104857600,
                "internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
        },
        "ok" : 1
}
```
Видим `"needTime" : 1598`    
Теперь создадим составной индекс:       
```
> db.vine.createIndex({density : 1, sulphates : 1})
{
        "numIndexesBefore" : 2,
        "numIndexesAfter" : 3,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
```

И теперь посмотрим на план выполнения запросов:
```
> db.vine.explain("executionStats").find({ $and: [{"density" : 0.998},{"sulphates" : 0.51}]})
{
        "explainVersion" : "1",
        "queryPlanner" : {
                "namespace" : "otus.vine",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "$and" : [
                                {
                                        "density" : {
                                                "$eq" : 0.998
                                        }
                                },
                                {
                                        "sulphates" : {
                                                "$eq" : 0.51
                                        }
                                }
                        ]
                },
                "maxIndexedOrSolutionsReached" : false,
                "maxIndexedAndSolutionsReached" : false,
                "maxScansToExplodeReached" : false,
                "winningPlan" : {
                        "stage" : "FETCH",
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "keyPattern" : {
                                        "density" : 1,
                                        "sulphates" : 1
                                },
                                "indexName" : "density_1_sulphates_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "density" : [ ],
                                        "sulphates" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "density" : [
                                                "[0.998, 0.998]"
                                        ],
                                        "sulphates" : [
                                                "[0.51, 0.51]"
                                        ]
                                }
                        }
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 2,
                "executionTimeMillis" : 1,
                "totalKeysExamined" : 2,
                "totalDocsExamined" : 2,
                "executionStages" : {
                        "stage" : "FETCH",
                        "nReturned" : 2,
                        "executionTimeMillisEstimate" : 0,
                        "works" : 3,
                        "advanced" : 2,
                        "needTime" : 0,
                        "needYield" : 0,
                        "saveState" : 0,
                        "restoreState" : 0,
                        "isEOF" : 1,
                        "docsExamined" : 2,
                        "alreadyHasObj" : 0,
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "nReturned" : 2,
                                "executionTimeMillisEstimate" : 0,
                                "works" : 3,
                                "advanced" : 2,
                                "needTime" : 0,
                                "needYield" : 0,
                                "saveState" : 0,
                                "restoreState" : 0,
                                "isEOF" : 1,
                                "keyPattern" : {
                                        "density" : 1,
                                        "sulphates" : 1
                                },
                                "indexName" : "density_1_sulphates_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "density" : [ ],
                                        "sulphates" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "density" : [
                                                "[0.998, 0.998]"
                                        ],
                                        "sulphates" : [
                                                "[0.51, 0.51]"
                                        ]
                                },
                                "keysExamined" : 2,
                                "seeks" : 1,
                                "dupsTested" : 0,
                                "dupsDropped" : 0
                        }
                }
        },
        "command" : {
                "find" : "vine",
                "filter" : {
                        "$and" : [
                                {
                                        "density" : 0.998
                                },
                                {
                                        "sulphates" : 0.51
                                }
                        ]
                },
                "$db" : "otus"
        },
        "serverInfo" : {
                "host" : "mongodb",
                "port" : 27017,
                "version" : "5.0.4",
                "gitVersion" : "62a84ede3cc9a334e8bc82160714df71e7d3a29e"
        },
        "serverParameters" : {
                "internalQueryFacetBufferSizeBytes" : 104857600,
                "internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
                "internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
                "internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
                "internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
                "internalQueryProhibitBlockingMergeOnMongoS" : 0,
                "internalQueryMaxAddToSetBytes" : 104857600,
                "internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
        },
        "ok" : 1
}
```
Видим что `"needTime" : 0`    

Таким образом мы посомтрели на работу индексов и планировщик запросов.
