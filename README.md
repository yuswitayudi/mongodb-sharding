Untuk membangun sebuah cluster dengan metode sharding pada MongoDB dibutuhkan minimal 5 service.
Masing masing sebagai berikut :
- 2 service Shard
- 1 service mongos
- 2 service config server

-------------------------------------------
## Step by Step

### 1. Configure Hosts File
    
   Atur disetiap server untuk menyamakan nama hostname dan ip pada file `` /etc/hosts `` contoh sebagai berikut.
```
192.0.2.1    mongo-config-1
192.0.2.2    mongo-config-2
192.0.2.3    mongo-config-3
192.0.2.4    mongo-query-router
192.0.2.5    mongo-shard-1
192.0.2.6    mongo-shard-2
```

### 2. Set Up MongoDB Authentication

- Create an Administrative User

Masuk ke mongoshell dari config server yang akan dijadikan PRIMARY replica set, dan buat user admin dengan privilege root.

```
- use admin

- db.createUser({user: "mongo-admin", pwd: "password", roles:[{role: "root", db: "admin"}]})
```

- Generate a Key file

```
- openssl rand -base64 756 > /home/yuswitayudi/mongodb.key
- sudo chown mongodb:mongodb /home/yuswitayudi/mongodb.key
- sudo chmod 600 /home/yuswitayudi/mongodb.key
```

Setelah itu tambahkan ke config file dari mongodb, ex: /etc/mongod.conf
```
security:
 keyFile: /home/yuswitayudi/mongodb.key
```
simpan dan restart mongodb ``sudo systemctl restart mongod``

### 3. Initialize Config Servers

- Untuk setup config server menggunakan CSRS (Config Server Replica Set) jadi disetiap file config dari masing masing config server sesuaikan untuk mengarah ke *replicaset* yang sama. Dan sesuaikan bindIp agar bisa terhubung ke service mongoDB yang lain.

```
port: 27017
bindIp: 192.0.2.1
```

```
replication:
  replSetName: configReplSet
```

*Uncomment* sharding dan sesuaikan nama sharding
```
sharding:
  clusterRole: configsvr
```
- Setelah dikira semuai config file telah sesuai, restart MongoDB
- Masuk ke salah satu config server yang akan dijadikan primary *replica set*

``mongo mongo-config-1:27019 -u mongo-admin -p --authenticationDatabase admin``

- inisiasi *replica set* dan menambahkan server config yang lain

``rs.initiate( { _id: "configReplSet", configsvr: true, members: [ { _id: 0, host: "mongo-config-1:27019" }, { _id: 1, host: "mongo-config-2:27019" }, { _id: 2, host: "mongo-config-3:27019" } ] } )``

pastikan proses berhasil dan cek dengan perintah ``rs.status()``

### 4. Configure Query Router

- Masuk ke server yang akan digunakan sebagai router
- Buat file config baru  ``/etc/mongos.conf`` kemudian tambahkan dan sesuaikan dengan kebutuhan

```
# where to write logging data.
systemLog:
destination: file
logAppend: true
path: /var/log/mongodb/mongos.log

# network interfaces
net:
port: 27017
bindIp: 192.0.2.4

security:
keyFile: /home/
dh/mongodb.key

sharding:
configDB: configReplSet/mongo-config-1:27019,mongo-config-2:27019,mongo-config-3:27019
```

- Buat *systemd unit file* yang baru untuk menjadi service mongos pada */lib/systemd/system/mongos.service*

```
[Unit]
Description=Mongo Cluster Router
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
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
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

[Install]
WantedBy=multi-user.target
```
- Simpan kemudian stop mongod apabila sebelumnya sudah running
- Kemudian enable mongos dan start. Pastikan berjalan dengan lancar

```
sudo systemctl enable mongos.service
sudo systemctl start mongos
```

### 5. Add Shards to the Cluster
Pada tahap ini pastikan systemd yang jalan sudah ditambahkan *--shardsvr*
- Masuk ke setiap *server shard* dan bindIp ke ip yang bisa diakses dari server lain
```
bindIp: 192.0.2.5
```

- Dari salah satu server shard masuk ke shell mongo *mongo-query-router* 

``
mongo mongo-query-router:27017 -u mongo-admin -p --authenticationDatabase admin
``
- Dan tambahkan semua server yang dibutuhkan menjadi *shard*

``
sh.addShard( "mongo-shard-1:27017" )
sh.addShard( "mongo-shard-2:27017" )
``

- *Optionally* jika shading menggunakan *replica set* gunakan perintah berikut ini
``
sh.addShard( "rs0/mongo-repl-1:27017,mongo-repl-2:27017,mongo-repl-3:27017" )
``
Pastikan ketika menambahkan sharding berjalan dengan lancar

### 6. Configure Sharding
- Enable Sharding at Database Level
   - Akses mongos shell query router
   - Dari mongos shell buat database baru

     ``use exampleDB``
   - Kemudian enableSharding database
     ``sh.enableSharding("exampleDB")``
   - Untuk verifikasi bahwa database sudah berhasil di sharding bisa masuk ke database config dan dilanjutkan perintah berikut

     ``db.databases.find()``
   - Dari perintah diatas akan menampilkan database dengan informasinya, seperti dibawah ini

     ``{ "_id" : "exampleDB", "primary" : "shard0001", "partitioned" : true }``

- Enable Sharding at Collection Level
   - Akses mongos shell query router
   - Masuk ke database *exampleDB* yang sudah dibuat
   - Tambahkan collection baru dan *hash* _id key

      ``db.exampleCollection.ensureIndex( { _id : "hashed" } )``
   - Kemudian shard collection

      ``sh.shardCollection( "exampleDB.exampleCollection", { "_id" : "hashed" } )``

### 7. Test Your Cluster
- Akses mongos shell query router dan masuk ke database *exampleDB*
- Jalankan *code* sperti dibawah ini

```
for (var i = 1; i <= 500; i++) db.exampleCollection.insert( { x : i } )
```
harusnya muncul output seperti ini

```
Shard shard0000 at mongo-shard-1:27017
 data : 8KiB docs : 265 chunks : 2
 estimated data per chunk : 4KiB
 estimated docs per chunk : 132

Shard shard0001 at mongo-shard-2:27017
 data : 7KiB docs : 235 chunks : 2
 estimated data per chunk : 3KiB
 estimated docs per chunk : 117

Totals
 data : 16KiB docs : 500 chunks : 4
 Shard shard0000 contains 53% data, 53% docs in cluster, avg obj size on shard : 33B
 Shard shard0001 contains 47% data, 47% docs in cluster, avg obj size on shard : 33B
``` 

**Dan Finish**

NB: apabila ada kesalahan penulisan akan diupdate dikemudian hari (:

[![license](https://img.shields.io/github/license/DAVFoundation/captain-n3m0.svg?style=flat-square)](https://github.com/yuswitayudi/mongodb-sharding/blob/master/LICENSE)
