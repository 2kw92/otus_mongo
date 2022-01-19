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

Теперь скачаем и загрузим данные.      
```
root@mongo1:~# wget https://dl.dropboxusercontent.com/s/p75zp1karqg6nnn/stocks.zip
--2022-01-19 13:12:39--  https://dl.dropboxusercontent.com/s/p75zp1karqg6nnn/stocks.zip
Resolving dl.dropboxusercontent.com (dl.dropboxusercontent.com)... 162.125.3.15, 2620:100:601b:15::a27d:80f
Connecting to dl.dropboxusercontent.com (dl.dropboxusercontent.com)|162.125.3.15|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 88115591 (84M) [application/zip]
Saving to: ‘stocks.zip’

stocks.zip                            100%[========================================================================>]  84.03M  32.7MB/s    in 2.6s

2022-01-19 13:12:42 (32.7 MB/s) - ‘stocks.zip’ saved [88115591/88115591]

root@mongo1:~# unzip -qo stocks.zip
root@mongo1:~# mongorestore -d otus -h mongo4:27000  -u userroot --authenticationDatabase admin /root/dump/stocks/values.bson                          Enter password:

2022-01-19T13:15:32.008+0000    checking for collection data in /root/dump/stocks/values.bson
2022-01-19T13:15:32.072+0000    restoring otus.values from /root/dump/stocks/values.bson
2022-01-19T13:15:35.010+0000    [........................]  otus.values  23.2MB/715MB  (3.2%)
2022-01-19T13:15:38.006+0000    [#.......................]  otus.values  45.0MB/715MB  (6.3%)
2022-01-19T13:15:41.006+0000    [##......................]  otus.values  65.6MB/715MB  (9.2%)
2022-01-19T13:15:44.006+0000    [##......................]  otus.values  86.4MB/715MB  (12.1%)
2022-01-19T13:15:47.006+0000    [###.....................]  otus.values  111MB/715MB  (15.5%)
2022-01-19T13:15:50.005+0000    [####....................]  otus.values  131MB/715MB  (18.3%)
2022-01-19T13:15:53.006+0000    [#####...................]  otus.values  154MB/715MB  (21.5%)
2022-01-19T13:15:56.005+0000    [#####...................]  otus.values  175MB/715MB  (24.4%)
2022-01-19T13:15:59.007+0000    [######..................]  otus.values  198MB/715MB  (27.7%)
2022-01-19T13:16:02.013+0000    [#######.................]  otus.values  221MB/715MB  (30.8%)
2022-01-19T13:16:05.009+0000    [#######.................]  otus.values  238MB/715MB  (33.3%)
2022-01-19T13:16:08.006+0000    [########................]  otus.values  257MB/715MB  (36.0%)
2022-01-19T13:16:11.008+0000    [#########...............]  otus.values  280MB/715MB  (39.2%)
2022-01-19T13:16:14.006+0000    [##########..............]  otus.values  303MB/715MB  (42.3%)
2022-01-19T13:16:17.006+0000    [##########..............]  otus.values  326MB/715MB  (45.5%)
2022-01-19T13:16:20.005+0000    [###########.............]  otus.values  347MB/715MB  (48.5%)
2022-01-19T13:16:23.009+0000    [############............]  otus.values  369MB/715MB  (51.6%)
2022-01-19T13:16:26.006+0000    [#############...........]  otus.values  391MB/715MB  (54.6%)
2022-01-19T13:16:29.011+0000    [#############...........]  otus.values  412MB/715MB  (57.6%)
2022-01-19T13:16:32.006+0000    [##############..........]  otus.values  436MB/715MB  (61.0%)
2022-01-19T13:16:35.006+0000    [###############.........]  otus.values  453MB/715MB  (63.3%)
2022-01-19T13:16:38.006+0000    [###############.........]  otus.values  472MB/715MB  (66.0%)
2022-01-19T13:16:41.013+0000    [################........]  otus.values  492MB/715MB  (68.8%)
2022-01-19T13:16:44.006+0000    [#################.......]  otus.values  515MB/715MB  (72.1%)
2022-01-19T13:16:47.006+0000    [##################......]  otus.values  537MB/715MB  (75.1%)
2022-01-19T13:16:50.006+0000    [##################......]  otus.values  555MB/715MB  (77.6%)
2022-01-19T13:16:53.006+0000    [###################.....]  otus.values  575MB/715MB  (80.4%)
2022-01-19T13:16:56.005+0000    [####################....]  otus.values  599MB/715MB  (83.7%)
2022-01-19T13:16:59.006+0000    [####################....]  otus.values  616MB/715MB  (86.1%)
2022-01-19T13:17:02.006+0000    [#####################...]  otus.values  637MB/715MB  (89.1%)
2022-01-19T13:17:05.007+0000    [######################..]  otus.values  658MB/715MB  (92.0%)
2022-01-19T13:17:08.006+0000    [######################..]  otus.values  680MB/715MB  (95.0%)
2022-01-19T13:17:11.006+0000    [#######################.]  otus.values  702MB/715MB  (98.1%)
2022-01-19T13:17:12.783+0000    [########################]  otus.values  715MB/715MB  (100.0%)
2022-01-19T13:17:12.783+0000    finished restoring otus.values (4308303 documents, 0 failures)
2022-01-19T13:17:12.783+0000    4308303 document(s) restored successfully. 0 document(s) failed to restore.

```      


Теперь создадим индекс и сделаем бд шардированной:
```
mongos> db.values.createIndex({stock_symbol: 1})
{
        "raw" : {
                "rs3/mongo1:27031,mongo2:27031,mongo3:27031" : {
                        "numIndexesBefore" : 1,
                        "numIndexesAfter" : 2,
                        "createdCollectionAutomatically" : false,
                        "commitQuorum" : "votingMembers",
                        "ok" : 1
                }
        },
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642598372, 4),
                "signature" : {
                        "hash" : BinData(0,"3v+/3hFcajt5p5VNvOOp9wAq5Lo="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642598372, 4)
}
mongos> sh.enableSharding("otus")
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642598396, 2),
                "signature" : {
                        "hash" : BinData(0,"/6n79W141oG8jeSiY/Niho4TkVs="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642598396, 2)
}
mongos> sh.shardCollection("otus.values",{ stock_symbol: 1 })
{
        "collectionsharded" : "otus.values",
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642598418, 35),
                "signature" : {
                        "hash" : BinData(0,"Tqa82tkg16hxBXqferbwDykEn+M="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642598418, 31)
}

``` 
И паралельно посмотрим как данные будут разъежаться по шардам:
```
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
                otus.values started at Wed Jan 19 2022 13:20:58 GMT+0000 (UTC)
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours:
                686 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                rs1     342
                                rs2     341
                                rs3     341
                        too many chunks to print, use verbose if you want to force print
        {  "_id" : "otus",  "primary" : "rs3",  "partitioned" : true,  "version" : {  "uuid" : UUID("ffc4d3f0-2782-4545-bea3-784fd8387b09"),  "timestamp" : Timestamp(1642598131, 1),  "lastMod" : 1 } }
                otus.values
                        shard key: { "stock_symbol" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                rs1     2
                                rs2     2
                                rs3     7
                        { "stock_symbol" : { "$minKey" : 1 } } -->> { "stock_symbol" : "AMIC" } on : rs2 Timestamp(2, 0)
                        { "stock_symbol" : "AMIC" } -->> { "stock_symbol" : "ATRO" } on : rs1 Timestamp(3, 0)
                        { "stock_symbol" : "ATRO" } -->> { "stock_symbol" : "BKUNA" } on : rs1 Timestamp(4, 0)
                        { "stock_symbol" : "BKUNA" } -->> { "stock_symbol" : "CALM" } on : rs2 Timestamp(5, 0)
                        { "stock_symbol" : "CALM" } -->> { "stock_symbol" : "CNIC" } on : rs3 Timestamp(5, 1)
                        { "stock_symbol" : "CNIC" } -->> { "stock_symbol" : "DAGM" } on : rs3 Timestamp(1, 5)
                        { "stock_symbol" : "DAGM" } -->> { "stock_symbol" : "ENDO" } on : rs3 Timestamp(1, 6)
                        { "stock_symbol" : "ENDO" } -->> { "stock_symbol" : "FMBI" } on : rs3 Timestamp(1, 7)
                        { "stock_symbol" : "FMBI" } -->> { "stock_symbol" : "MACE" } on : rs3 Timestamp(1, 8)
                        { "stock_symbol" : "MACE" } -->> { "stock_symbol" : "MTCT" } on : rs3 Timestamp(1, 9)
                        { "stock_symbol" : "MTCT" } -->> { "stock_symbol" : { "$maxKey" : 1 } } on : rs3 Timestamp(1, 10)
        {  "_id" : "test",  "primary" : "rs1",  "partitioned" : true,  "version" : {  "uuid" : UUID("6179a109-5d28-4dcc-8a45-f80825148a67"),  "timestamp" : Timestamp(1642598383, 2),  "lastMod" : 1 } }
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
        Currently running: no
        Collections with active migrations:
                otus.values started at Wed Jan 19 2022 13:21:20 GMT+0000 (UTC)
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours:
                687 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                rs1     342
                                rs2     341
                                rs3     341
                        too many chunks to print, use verbose if you want to force print
        {  "_id" : "otus",  "primary" : "rs3",  "partitioned" : true,  "version" : {  "uuid" : UUID("ffc4d3f0-2782-4545-bea3-784fd8387b09"),  "timestamp" : Timestamp(1642598131, 1),  "lastMod" : 1 } }
                otus.values
                        shard key: { "stock_symbol" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                rs1     2
                                rs2     3
                                rs3     6
                        { "stock_symbol" : { "$minKey" : 1 } } -->> { "stock_symbol" : "AMIC" } on : rs2 Timestamp(2, 0)
                        { "stock_symbol" : "AMIC" } -->> { "stock_symbol" : "ATRO" } on : rs1 Timestamp(3, 0)
                        { "stock_symbol" : "ATRO" } -->> { "stock_symbol" : "BKUNA" } on : rs1 Timestamp(4, 0)
                        { "stock_symbol" : "BKUNA" } -->> { "stock_symbol" : "CALM" } on : rs2 Timestamp(5, 0)
                        { "stock_symbol" : "CALM" } -->> { "stock_symbol" : "CNIC" } on : rs2 Timestamp(6, 0)
                        { "stock_symbol" : "CNIC" } -->> { "stock_symbol" : "DAGM" } on : rs3 Timestamp(6, 1)
                        { "stock_symbol" : "DAGM" } -->> { "stock_symbol" : "ENDO" } on : rs3 Timestamp(1, 6)
                        { "stock_symbol" : "ENDO" } -->> { "stock_symbol" : "FMBI" } on : rs3 Timestamp(1, 7)
                        { "stock_symbol" : "FMBI" } -->> { "stock_symbol" : "MACE" } on : rs3 Timestamp(1, 8)
                        { "stock_symbol" : "MACE" } -->> { "stock_symbol" : "MTCT" } on : rs3 Timestamp(1, 9)
                        { "stock_symbol" : "MTCT" } -->> { "stock_symbol" : { "$maxKey" : 1 } } on : rs3 Timestamp(1, 10)
        {  "_id" : "test",  "primary" : "rs1",  "partitioned" : true,  "version" : {  "uuid" : UUID("6179a109-5d28-4dcc-8a45-f80825148a67"),  "timestamp" : Timestamp(1642598383, 2),  "lastMod" : 1 } }
```
Данные разъехались по шардам.

Теперь давайте остановим сервис mongod на первой ноде:
```
root@mongo1:~# systemctl status mongod
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: https://docs.mongodb.org/manual
```
И со второй ноды проверим статус реплики:
```
rs0:SECONDARY> rs.status
function() {
    return db._adminCommand("replSetGetStatus");
}
rs0:SECONDARY> rs.status()
{
        "set" : "rs0",
        "date" : ISODate("2022-01-19T13:31:49.510Z"),
        "myState" : 2,
        "term" : NumberLong(8),
        "syncSourceHost" : "mongo3:27001",
        "syncSourceId" : 2,
        "configsvr" : true,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "votingMembersCount" : 3,
        "writableVotingMembersCount" : 3,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1642599108, 263),
                        "t" : NumberLong(8)
                },
                "lastCommittedWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1642599108, 263),
                        "t" : NumberLong(8)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1642599108, 263),
                        "t" : NumberLong(8)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1642599108, 263),
                        "t" : NumberLong(8)
                },
                "lastAppliedWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                "lastDurableWallTime" : ISODate("2022-01-19T13:31:48.347Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1642599099, 1),
        "electionParticipantMetrics" : {
                "votedForCandidate" : true,
                "electionTerm" : NumberLong(8),
                "lastVoteDate" : ISODate("2022-01-19T13:21:19.614Z"),
                "electionCandidateMemberId" : 2,
                "voteReason" : "",
                "lastAppliedOpTimeAtElection" : {
                        "ts" : Timestamp(1642598468, 1),
                        "t" : NumberLong(7)
                },
                "maxAppliedOpTimeInSet" : {
                        "ts" : Timestamp(1642598468, 1),
                        "t" : NumberLong(7)
                },
                "priorityAtElection" : 1,
                "newTermStartDate" : ISODate("2022-01-19T13:21:20.463Z"),
                "newTermAppliedDate" : ISODate("2022-01-19T13:21:20.466Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongo1:27001",
                        "health" : 0,
                        "state" : 8,
                        "stateStr" : "(not reachable/healthy)",
                        "uptime" : 0,
                        "optime" : {
                                "ts" : Timestamp(0, 0),
                                "t" : NumberLong(-1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(0, 0),
                                "t" : NumberLong(-1)
                        },
                        "optimeDate" : ISODate("1970-01-01T00:00:00Z"),
                        "optimeDurableDate" : ISODate("1970-01-01T00:00:00Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-19T13:21:06.351Z"),
                        "lastDurableWallTime" : ISODate("2022-01-19T13:21:06.351Z"),
                        "lastHeartbeat" : ISODate("2022-01-19T13:31:49.438Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-19T13:21:33.819Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "Error connecting to mongo1:27001 (10.128.0.6:27001) :: caused by :: Connection refused",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 7
                },
                {
                        "_id" : 1,
                        "name" : "mongo2:27001",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 10399,
                        "optime" : {
                                "ts" : Timestamp(1642599108, 263),
                                "t" : NumberLong(8)
                        },
                        "optimeDate" : ISODate("2022-01-19T13:31:48Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                        "lastDurableWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                        "syncSourceHost" : "mongo3:27001",
                        "syncSourceId" : 2,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "configTerm" : 8,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 2,
                        "name" : "mongo3:27001",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 10397,
                        "optime" : {
                                "ts" : Timestamp(1642599108, 263),
                                "t" : NumberLong(8)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1642599108, 263),
                                "t" : NumberLong(8)
                        },
                        "optimeDate" : ISODate("2022-01-19T13:31:48Z"),
                        "optimeDurableDate" : ISODate("2022-01-19T13:31:48Z"),
                        "lastAppliedWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                        "lastDurableWallTime" : ISODate("2022-01-19T13:31:48.347Z"),
                        "lastHeartbeat" : ISODate("2022-01-19T13:31:49.338Z"),
                        "lastHeartbeatRecv" : ISODate("2022-01-19T13:31:48.702Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1642598479, 1),
                        "electionDate" : ISODate("2022-01-19T13:21:19Z"),
                        "configVersion" : 1,
                        "configTerm" : 8
                }
        ],
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : Timestamp(0, 0),
                "electionId" : ObjectId("000000000000000000000000")
        },
        "lastCommittedOpTime" : Timestamp(1642599108, 263),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642599108, 263),
                "signature" : {
                        "hash" : BinData(0,"fw/q5pfgXIdYilzLdCwO516bu/8="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642599108, 263)
}

```
Видим что у нас primary переехал на 3 ноду

Если мы погасим сервисы отевчающие за репликасет rs1 а это сервисы mongod_1, хотя бы на 2-ух нодах, то при запросе через mongos поулчим:
```
mongos> db.values.count()
uncaught exception: Error: count failed: {
        "ok" : 0,
        "errmsg" : "failed on: rs1 :: caused by :: Could not find host matching read preference { mode: \"primary\" } for set rs1",
        "code" : 133,
        "codeName" : "FailedToSatisfyReadPreference",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642599498, 1),
                "signature" : {
                        "hash" : BinData(0,"0c9JzqVu3/bpk20fNIK43EkTPnY="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642599498, 1)
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
DBCollection.prototype.count@src/mongo/shell/collection.js:1406:15
@(shell):1:1
```

Соотвественно если погами сервис для репликасета rs2, получим ошибку:
```
mongos> db.values.count()
uncaught exception: Error: count failed: {
        "ok" : 0,
        "errmsg" : "failed on: rs2 :: caused by :: Could not find host matching read preference { mode: \"primary\" } for set rs2",
        "code" : 133,
        "codeName" : "FailedToSatisfyReadPreference",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1642599648, 1),
                "signature" : {
                        "hash" : BinData(0,"RhUDzBtVdUar69XLJ0cFwXfD624="),
                        "keyId" : NumberLong("7054507337280651287")
                }
        },
        "operationTime" : Timestamp(1642599646, 2)
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
DBCollection.prototype.count@src/mongo/shell/collection.js:1406:15
@(shell):1:1
```

Из этого можно сделать, что наш кластер может потерять по одному инстансу из каждого репликасета и оставаться в рабочем состоянии.
