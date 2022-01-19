ДЗ ОТУС бэкапирование шардированного кластера

Разворачиваем наши вирутальные машины
```
gcloud beta compute --project=mongo2021-19920819 instances create mongo1 --zone=us-central1-a --machine-type=e2-medium --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=793214010684-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-2104-hirsute-v20220111 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=mongo1 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
gcloud beta compute --project=mongo2021-19920819 instances create mongo2 --zone=us-central1-a --machine-type=e2-medium --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=793214010684-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-2104-hirsute-v20220111 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=mongo2 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
gcloud beta compute --project=mongo2021-19920819 instances create mongo3 --zone=us-central1-a --machine-type=e2-medium --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=793214010684-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-2104-hirsute-v20220111 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=mongo3 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
gcloud beta compute --project=mongo2021-19920819 instances create mongo4 --zone=us-central1-a --machine-type=e2-medium --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=793214010684-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-2104-hirsute-v20220111 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=mongo4 --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

Устновим монгу на всех машинах:
```
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```       
При такой установке появляется сервис mongod который можно запустить:      
```
systemctl start mongod
```
При таком запуске используется файл конфигурации `/etc/mongod.conf`       

Создадим папки для бд и назначим владельца:       
```
mkdir /var/lib/mongodb{_1,_2,_3} && chown mongodb:mongodb -R /var/lib/mongodb{_1,_2,_3}
```

Шардированный кластер у нас будет иметь следующую архитектуру:      
кластер из 3 кластерных нод( по 3 инстанса с репликацией) и с кластером конфига(3 инстанса);
```
vm1=config1 + primary1 + slave2 + slave3
vm2=config2 + slave1 + primary2 + slave3
vm3=config3 + slave1 + slave2 + primary3
vm4=mongos1
```
vm1 - это нода с названием mongo1        
vm2 - это нода с названием mongo2        
vm3 - это нода с названием mongo3       
vm4 - это нода с названием mongo4

В папках conf и service лежат конфиги и сервисы как для кластера конфига (`mongod.conf`) так и для всех инстансов (`mongod{_1,_2,_3}.conf`).      
Для того чтобы работала авторизация по ключу нам сначало нужно создать этот ключ на серваке mongo1:
```
openssl rand -base64 756 > /home/mongodb/keyfile
chmod 400 /home/mongodb/keyfile
```
А после этого раскидать его на все остальные серваки.     
Чтобы заупстить с авторизацией нужно добавить строки в конфиг      
```
security:
  keyFile: /home/mongodb/keyfile
```
Но мы для начала запустим без ключа на 3 серверах:
```
systemctl start mongod
```   
И проинициализируем репликасет конфиг кластера:   
```
root@mongo1:/var/log/mongodb# mongo --port 27001
> rs.initiate({"_id" : "rs0", configsvr: true, members : [{"_id" : 0, host : "mongo1:27001"},{"_id" : 1, host : "mongo2:27001"},{"_id" : 2, host : "mongo3:27001"}]});
{
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : Timestamp(1642505473, 1),
                "electionId" : ObjectId("000000000000000000000000")
        },
        "lastCommittedOpTime" : Timestamp(1642505473, 1)
}
```

Создадим несколько юзеров       
```
rs0:PRIMARY> use admin
switched to db admin
rs0:PRIMARY> db.createUser({user: "userdbadmin",pwd: "otus", roles: [ { role: "dbAdmin", db: "*" } ]})
Successfully added user: {
        "user" : "userdbadmin",
        "roles" : [
                {
                        "role" : "dbAdmin",
                        "db" : "*"
                }
        ]
}
rs0:PRIMARY> db.createUser({user: "userroot",pwd: "otus", roles: [ "root" ]})
Successfully added user: { "user" : "userroot", "roles" : [ "root" ] }
rs0:PRIMARY>
```

Чтобы заупстить с авторизацией нужно добавить строки в конфиг      
```
security:
  keyFile: /home/mongodb/keyfile
```
И сделать рестарт сервиса mongod.     
После этого зайдем с авторизацией:     
```
root@mongo1:~# mongo --port 27001 -u userroot -p otus --authenticationDatabase admin
```    
И прверим статус кластера:       
```
rs0:PRIMARY> rs.status()
{
        "set" : "rs0",
        "date" : ISODate("2022-01-18T18:31:57.316Z"),
        "myState" : 1,
        "term" : NumberLong(5),
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "configsvr" : true,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "votingMembersCount" : 3,
        "writableVotingMembersCount" : 3,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1642530716, 1),
                        "t" : NumberLong(5)
                },
                "lastCommittedWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1642530716, 1),
                        "t" : NumberLong(5)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1642530716, 1),
                        "t" : NumberLong(5)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1642530716, 1),
                        "t" : NumberLong(5)
                },
                "lastAppliedWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                "lastDurableWallTime" : ISODate("2022-01-18T18:31:56.652Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1642530688, 1),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "stepUpRequestSkipDryRun",
                "lastElectionDate" : ISODate("2022-01-18T18:27:56.527Z"),
                "electionTerm" : NumberLong(5),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(1642530474, 1),
                        "t" : NumberLong(4)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1642530475, 1),
                        "t" : NumberLong(4)
                },
                "numVotesNeeded" : 2,
                "priorityAtElection" : 1,
                "electionTimeoutMillis" : NumberLong(10000),
                "priorPrimaryMemberId" : 2,
                "numCatchUpOps" : NumberLong(0),
                "newTermStartDate" : ISODate("2022-01-18T18:27:56.539Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2022-01-18T18:27:57.544Z")
        },
        "electionParticipantMetrics" : {
                "votedForCandidate" : true,
                "electionTerm" : NumberLong(4),
                "lastVoteDate" : ISODate("2022-01-18T11:38:53.799Z"),
                "electionCandidateMemberId" : 2,
                "voteReason" : "",
                "lastAppliedOpTimeAtElection" : {
                        "ts" : Timestamp(1642505902, 1),
                        "t" : NumberLong(2)
                },
                "maxAppliedOpTimeInSet" : {
                        "ts" : Timestamp(1642505902, 1),
                        "t" : NumberLong(2)
                },
                "priorityAtElection" : 1
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongo1:27001",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 24825,
                        "optime" : {
                                "ts" : Timestamp(1642530716, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2022-01-18T18:31:56Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1642530476, 1),
                        "electionDate" : ISODate("2022-01-18T18:27:56Z"),
                        "configVersion" : 1,
                        "configTerm" : 5,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongo2:27001",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 24797,
                        "optime" : {
                                "ts" : Timestamp(1642530715, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1642530715, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2022-01-18T18:31:55Z"),
                        "optimeDurableDate" : ISODate("2022-01-18T18:31:55Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "lastHeartbeat" : ISODate("2022-01-18T18:31:56.653Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-18T18:31:56.702Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "mongo1:27001",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 5
                },
                {
                        "_id" : 2,
                        "name" : "mongo3:27001",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 0,
                        "optime" : {
                                "ts" : Timestamp(1642530716, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1642530716, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2022-01-18T18:31:56Z"),
                        "optimeDurableDate" : ISODate("2022-01-18T18:31:56Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T18:31:56.652Z"),
                        "lastHeartbeat" : ISODate("2022-01-18T18:31:56.848Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-18T18:31:56.545Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "mongo2:27001",
                        "syncSourceId" : 1,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 5
                }
        ],
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : Timestamp(0, 0),
                "electionId" : ObjectId("7fffffff0000000000000005")
        },
        "lastCommittedOpTime" : Timestamp(1642530716, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642530716, 1),
                "signature" : {
                        "hash" : BinData(0,"BK4TcBJKrepxQKdut/8cK5IL5lY="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642530716, 1)
}
```
Если запустим без авторизации и прверим статус то получим соотвествующую ошибку:      
```
rs0:PRIMARY> rs.status()
{
        "ok" : 0,
        "errmsg" : "command replSetGetStatus requires authentication",
        "code" : 13,
        "codeName" : "Unauthorized",
        "lastCommittedOpTime" : Timestamp(1642530873, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642530873, 1),
                "signature" : {
                        "hash" : BinData(0,"qNkw9nvEMKIOVVQo+82c3BhKQa0="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642530873, 1)
}
rs0:PRIMARY>
```

Для того чтобы реплика была доступна на чтение:        
rs.secondaryOk(true)      

Далее будем строить нашу архитектуру.         
Пример сервиса mongod vi/usr/lib/systemd/system/mongod_1.service
```
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network-online.target
Wants=network-online.target

[Service]
User=mongodb
Group=mongodb
EnvironmentFile=-/etc/default/mongod
ExecStart=/usr/bin/mongod --config /etc/mongod_1.conf
PIDFile=/var/run/mongodb/mongod_1.pid
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

# Recommended limits for mongod as specified in
# https://docs.mongodb.com/manual/reference/ulimit/#recommended-ulimit-settings

[Install]
WantedBy=multi-user.target
```

Пример Конфигурационного файла:
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
  bindIpAll : true


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
#  authorization: enabled
  keyFile: /home/mongodb/keyfile
#operationProfiling:

replication:
  replSetName: rs0
```
Теперь нам нужно запустить все интсансы mongod, для которых у нас есть сервисы. Пока запустим без авторизации.       
Для это в файлах /etc/mongod_{1,2,3}.conf закомментируем строчки:     
```
security:
#  keyFile: /home/mongodb/keyfile
```
После того как мы запустили сервисы mongod_1 на всех серверах (mongo1,mongo2,mongo3) на сервере mongo1 проинициализируем репликасет:      
```
root@mongo1:/var/lib# mongo --port 27011
> rs.initiate({"_id" : "rs1", members : [{"_id" : 0, priority : 3, host : "mongo1:27011"},{"_id" : 1, host : "mongo2:27011"},{"_id" : 2, host : "mongo3:27011"}]});
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642500440, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1642500440, 1)
```
Проверим статус:
```        
rs1:PRIMARY> rs.status()
{
        "set" : "rs1",
        "date" : ISODate("2022-01-18T10:10:22.271Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "votingMembersCount" : 3,
        "writableVotingMembersCount" : 3,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1642500620, 1),
                        "t" : NumberLong(1)
                },
                "lastCommittedWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1642500620, 1),
                        "t" : NumberLong(1)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1642500620, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1642500620, 1),
                        "t" : NumberLong(1)
                },
                "lastAppliedWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                "lastDurableWallTime" : ISODate("2022-01-18T10:10:20.651Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1642500610, 1),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "electionTimeout",
                "lastElectionDate" : ISODate("2022-01-18T10:07:30.585Z"),
                "electionTerm" : NumberLong(1),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(1642500440, 1),
                        "t" : NumberLong(-1)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1642500440, 1),
                        "t" : NumberLong(-1)
                },
                "numVotesNeeded" : 2,
                "priorityAtElection" : 3,
                "electionTimeoutMillis" : NumberLong(10000),
                "numCatchUpOps" : NumberLong(0),
                "newTermStartDate" : ISODate("2022-01-18T10:07:30.628Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2022-01-18T10:07:31.961Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongo1:27011",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 940,
                        "optime" : {
                                "ts" : Timestamp(1642500620, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2022-01-18T10:10:20Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1642500450, 1),
                        "electionDate" : ISODate("2022-01-18T10:07:30Z"),
                        "configVersion" : 1,
                        "configTerm" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongo2:27011",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 181,
                        "optime" : {
                                "ts" : Timestamp(1642500620, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1642500620, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2022-01-18T10:10:20Z"),
                        "optimeDurableDate" : ISODate("2022-01-18T10:10:20Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "lastHeartbeat" : ISODate("2022-01-18T10:10:20.665Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-18T10:10:22.175Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "mongo1:27011",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongo3:27011",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 181,
                        "optime" : {
                                "ts" : Timestamp(1642500620, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1642500620, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2022-01-18T10:10:20Z"),
                        "optimeDurableDate" : ISODate("2022-01-18T10:10:20Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "lastDurableWallTime" : ISODate("2022-01-18T10:10:20.651Z"),
                        "lastHeartbeat" : ISODate("2022-01-18T10:10:20.669Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-18T10:10:22.175Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "mongo1:27011",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 1
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642500620, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1642500620, 1)
}
```
Видим что все ок, добавим несколько пользователей
```
rs1:PRIMARY> use admin
switched to db admin
rs1:PRIMARY> db.createUser({user: "userdbadmin",pwd: "otus", roles: [ { role: "dbAdmin", db: "*" } ]})
Successfully added user: {
        "user" : "userdbadmin",
        "roles" : [
                {
                        "role" : "dbAdmin",
                        "db" : "*"
                }
        ]
}
rs1:PRIMARY> db.createUser({user: "userroot",pwd: "otus", roles: [ "root" ]})
Successfully added user: { "user" : "userroot", "roles" : [ "root" ] }
```
Тоже самое проделаем на хостах mongo2 и mongo3, но предварительно нужно запустить сервисы mongod_2 и mongod_3 на всех кластерах :
```
systemctl start mongod_2
systemctl start mongod_3
```
Проверим что все поднялось:      
```
root@mongo2:/var/lib# ps aux | grep mongo
mongodb    11273  0.9  3.1 1842364 124736 ?      Ssl  10:07   0:10 /usr/bin/mongod --config /etc/mongod_1.conf
mongodb    11787  2.6  2.9 1678848 119644 ?      Ssl  10:24   0:01 /usr/bin/mongod --config /etc/mongod_2.conf
mongodb    11843  2.9  3.0 1678848 121268 ?      Ssl  10:24   0:01 /usr/bin/mongod --config /etc/mongod_3.conf
root       11896  0.0  0.0   8392   916 pts/0    S+   10:25   0:00 grep --color=auto mongo
```
И теперь проинициализиреум репликасеты:       
```
root@mongo2:/var/lib# mongo --port 27021
MongoDB shell version v5.0.5
connecting to: mongodb://127.0.0.1:27021/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("537acb23-904a-4a31-bb1e-4394e96270b2") }
MongoDB server version: 5.0.5
================
Warning: the "mongo" shell has been superseded by "mongosh",
which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
an upcoming release.
For installation instructions, see
https://docs.mongodb.com/mongodb-shell/install/
================
---
The server generated these startup warnings when booting:
        2022-01-18T10:24:46.062+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
        2022-01-18T10:24:46.745+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
---
---
        Enable MongoDB's free cloud-based monitoring service, which will then receive and display
        metrics about your deployment (disk utilization, CPU, operation statistics, etc).

        The monitoring data will be available on a MongoDB website with a unique URL accessible to you
        and anyone you share the URL with. MongoDB may use this information to make product
        improvements and to suggest MongoDB products and deployment options to you.

        To enable free monitoring, run the following command: db.enableFreeMonitoring()
        To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---
> rs.initiate({"_id" : "rs2", members : [{"_id" : 0, host : "mongo1:27021"},{"_id" : 1, priority : 3, host : "mongo2:27021"},{"_id" : 2, host : "mongo3:27021"}]});
{ "ok" : 1 }
rs2:OTHER>
rs2:PRIMARY>
```
Создадим юзеров:
```
rs2:PRIMARY> db.createUser({user: "userdbadmin",pwd: "otus", roles: [ { role: "dbAdmin", db: "*" } ]})
Successfully added user: {
        "user" : "userdbadmin",
        "roles" : [
                {
                        "role" : "dbAdmin",
                        "db" : "*"
                }
        ]
}
rs2:PRIMARY> db.createUser({user: "userroot",pwd: "otus", roles: [ "root" ]})
Successfully added user: { "user" : "userroot", "roles" : [ "root" ] }
```

И теперь на ноде mongo3 проделаем тоже самое:      
```
> rs.initiate({"_id" : "rs3", members : [{"_id" : 0, host : "mongo1:27031"},{"_id" : 1, host : "mongo2:27031"},{"_id" : 2, priority : 3, host : "mongo3:27031"}]});
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642502363, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1642502363, 1)
}
rs3:SECONDARY>
rs3:SECONDARY>
rs3:PRIMARY>
rs3:PRIMARY> use admin
switched to db admin
rs3:PRIMARY> db.createUser({user: "userroot",pwd: "otus", roles: [ "root" ]})
Successfully added user: { "user" : "userroot", "roles" : [ "root" ] }
rs3:PRIMARY> db.createUser({user: "userdbadmin",pwd: "otus", roles: [ { role: "dbAdmin", db: "*" } ]})
Successfully added user: {
        "user" : "userdbadmin",
        "roles" : [
                {
                        "role" : "dbAdmin",
                        "db" : "*"
                }
        ]
}
```
После этого можем запустить реплики с авторизацией и ключом. Для этого в файлай конифгурации /etc/mongod.conf расскоментим:
```
security:
  keyFile: /home/mongodb/keyfile
```
И перезапустим все сервисы mongod.

Теперь запустим mongos на хосте mongo4:
```
root@mongo4:~#  mongos --keyFile /home/mongo/keyfile --configdb rc0/mongo1:27001,mongo2:27001,mongo3:27001 --bind_ip_all --port 27000 --logpath /home/mongo/dbms/dbs.log --pidfilepath /home/mongo/dbms/dbs.pid --fork
about to fork child process, waiting until server is ready for connections.
forked process: 16068
child process started successfully, parent exiting
```

И попробуем подрубиться:
```
root@mongo4:~# mongo --port 27000
mongos> rs.status();
{
        "ok" : 0,
        "errmsg" : "command replSetGetStatus requires authentication",
        "code" : 13,
        "codeName" : "Unauthorized",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642582323, 1),
                "signature" : {
                        "hash" : BinData(0,"+jxizH6xVXVF4Cey9vNIGgNEqFY="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642582323, 1)

```
Видим что без авторизации нам недоступна функция

Подрубимся с авторизацией c первого хоста и снова проверим:
```
mongo --port 27000 -u userroot -p otus --authenticationDatabase admin --host mongo4
mongos> rs.status()
{
        "info" : "mongos",
        "ok" : 0,
        "errmsg" : "replSetGetStatus is not supported through mongos",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642582944, 1),
                "signature" : {
                        "hash" : BinData(0,"A3JSJkrsFUYputV2oiwGEUPpEH8="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642582944, 1)
```
Видим все что ок, функция доступна, но ее нельзя вызывать через mongos. Осталось добавить шарды:       
Подключимся к нашему mongos под пользователем userdbadmin      
```
mongo --port 27000 -u "userroot" -p "otus" --authenticationDatabase "admin"
mongos> sh.addShard("rs1/mongo1:27011,mongo2:27011,mongo3:27011")
mongos> sh.addShard("rs1/mongo1:27011,mongo2:27011,mongo3:27011")
{
        "shardAdded" : "rs1",
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642588858, 4),
                "signature" : {
                        "hash" : BinData(0,"QL4CEsmll66p3ZG5PwvapM9S748="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642588858, 4)
}
mongos> sh.addShard("rs2/mongo1:27021,mongo2:27021,mongo3:27021")
{
        "shardAdded" : "rs2",
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642591035, 4),
                "signature" : {
                        "hash" : BinData(0,"0Ns+vffSxGIasZptSJtlvIoIanQ="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642591035, 4)
}
{
        "shardAdded" : "rs3",
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642591128, 5),
                "signature" : {
                        "hash" : BinData(0,"OlbGJdY5nvt5+Nf019WRoPdhzmk="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642591128, 4)
}
mongos> sh.status()
--- Sharding Status ---
  sharding version: {
        "_id" : 1,
        "minCompatibleVersion" : 5,
        "currentVersion" : 6,
        "clusterId" : ObjectId("61e6a50c7666c7fbb860e4a6")
  }
  shards:
        {  "_id" : "rs1",  "host" : "rs1/mongo1:27011,mongo2:27011,mongo3:27011",  "state" : 1,  "topologyTime" : Timestamp(1642588858, 1) }
        {  "_id" : "rs2",  "host" : "rs2/mongo1:27021,mongo2:27021,mongo3:27021",  "state" : 1,  "topologyTime" : Timestamp(1642591035, 2) }
        {  "_id" : "rs3",  "host" : "rs3/mongo1:27031,mongo2:27031,mongo3:27031",  "state" : 1,  "topologyTime" : Timestamp(1642591128, 2) }
  active mongoses:
        "5.0.5" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled: yes
        Currently running: yes
        Collections with active migrations:
                config.system.sessions started at Wed Jan 19 2022 11:19:16 GMT+0000 (UTC)
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours:
                100 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                rs1     923
                                rs2     78
                                rs3     23
                        too many chunks to print, use verbose if you want to force print
mongos>
```

Мы построили шардированный кластер.    

Теперь загрузим данные. Будем подгружать геоданные содержащий информацию об интересных местах Тулузы.      


mongoimport -d geodb -c church_polygon2 --jsonArray  -u root --authenticationDatabase admin --file /root/church_polygon_edit.geojson
