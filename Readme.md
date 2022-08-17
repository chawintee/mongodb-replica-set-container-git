# purpose
- note ได้เเพื่อนำมาดูภายหลังครับ อาจจะไม่ถูกต้องและไม่ครบถ้วนครับ
- ถ้ามีอะไรผิดพลาดประการใด รบกวนชี้แนะด้วยครับ

# init keyfile
```
openssl rand -base64 756 > keyfile
chmod 600 keyfile
sudo chown 999 keyfile
sudo chgrp 999 keyfile
```

- init 
```
sh init_keyfile.sh
```

# docker-compose
```
version: "3.8"
services:
  replica-set-git:
    image : mongo:5.0
    container_name: replica-set-git
    hostname: replica-set-git
    restart: on-failure
    environment:
      - PUID=1000
      - PGID=1000
      - MONGO_INITDB_ROOT_USERNAME=usermongo
      - MONGO_INITDB_ROOT_PASSWORD=passmongo
      - MONGO_REPLICA_SET_NAME=replicaset1
    volumes:
      - ./db-replica:/data/db
      - ./keyfile:/opt/keyfile/keyfile
    ports:
      - 27081:27081
    command: "--bind_ip_all --keyFile /opt/keyfile/keyfile --replSet replicaset1 --port 27081"
```

# การ ขึ้น container 
```
docker-compose -f docker-compose-replicaset.yml up -d
```

# การเข้าไป replica-set initiate 
- เข้าไปใน container
    - หา container ที่เราเพิ่งสร้างขึ้น โดยดูจาก container_name ใน docker-compose
        - ดู container ทั้งหมด docker ps
            ```
            docker ps
            ```
        - ผลที่ได้
            ```
            ❯ docker ps
            CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS          PORTS                                                      NAMES
            41e710e0797d   mongo:5.0   "docker-entrypoint.s…"   7 seconds ago   Up 3 seconds    27017/tcp, 0.0.0.0:27081->27081/tcp, :::27081->27081/tcp   replica-set-git
            facecb61fe76   mongo:5.0   "docker-entrypoint.s…"   9 hours ago     Up 13 minutes   27017/tcp, 0.0.0.0:27031->27031/tcp, :::27031->27031/tcp   mongodb-first
            ```

    - เข้าไปใน container 
        - โดยใช้คำสั่ง docker exec -it replica-set-git bash
            ```
            docker exec -it replica-set-git bash
            ```
            - replica-set คือ ชื่อ container_name หรือ container id
            - -it คือ option
            - bash คือ bash shell
        - ผลที่ได้
            ```
            ❯ docker exec -it replica-set-git bash
            root@replica-set-git:/#
            ```
            - แสดงอย่างนี้คือสามารถเข้ามาข้างใน container ได้แล้ว

- เมื่อเข้าไป container ได้ 
    - เป็นดังข้างล่าง
        ```
            ❯ docker exec -it replica-set-git bash
            root@replica-set-git:/#
        ```
    
    - เข้าไปใน mongo shell ได้ใช้คำสั่ง  mongo localhost:27081/admin -u usermongo -p passmongo
        ```
             mongo localhost:27081/admin -u usermongo -p passmongo
        ```
         - localhost     คือ domain ของเครื่องเรา
         - admin         คือ ชื่อ database ที่เราต้องการจะเข้า
         - usermongo     คือ user ของเราตอนที่ทำใน docker-compose คือ MONGO_INITDB_ROOT_USERNAME
         - passmongo     คือ user ของเราตอนที่ทำใน docker-compose คือ MONGO_INITDB_ROOT_PASSWORD
    - ผลที่ได้
        ```
            ❯ docker exec -it replica-set-git bash
            root@replica-set-git:/# docker exec -it replica-set-git bash
            root@replica-set-git:/# mongo localhost:27081/admin -u usermongo -p passmongo
            MongoDB shell version v5.0.6
            connecting to: mongodb://localhost:27081/admin?compressors=disabled&gssapiServiceName=mongodb
            Implicit session: session { "id" : UUID("c6be88a3-5c55-4506-aa54-80646a594508") }
            MongoDB server version: 5.0.6
            ================
            Warning: the "mongo" shell has been superseded by "mongosh",
            which delivers improved usability and compatibility.The "mongo" shell has been deprecated and will be removed in
            an upcoming release.
            For installation instructions, see
            https://docs.mongodb.com/mongodb-shell/install/
            ================
            Welcome to the MongoDB shell.
            For interactive help, type "help".
            For more comprehensive documentation, see
                https://docs.mongodb.com/
            Questions? Try the MongoDB Developer Community Forums
                https://community.mongodb.com
            ---
            The server generated these startup warnings when booting: 
                    2022-08-17T14:10:14.723+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
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
            > 
        ```

- ใน mongo shell ใน admin database 
    - check status
        - rs.status()
        - check status ของ replicaSet
        ```
            > rs.status()
            {
                "ok" : 0,
                "errmsg" : "no replset config has been received",
                "code" : 94,
                "codeName" : "NotYetInitialized"
            }
        ```
        - อันนี้ คือ replica set ยังไม่ได้ initiate หรือ config
    - check config
        - rs.config()
        - check การ config ของ replica set นี้
        ```
            > rs.config()
            uncaught exception: Error: Could not retrieve replica set config: {
                "ok" : 0,
                "errmsg" : "no replset config has been received",
                "code" : 94,
                "codeName" : "NotYetInitialized"
            } :
            rs.conf@src/mongo/shell/utils.js:1713:11
            @(shell):1:1
        ```

    - การทำ initiate 
        - rs.initiate()
        - ตาม doc
        ```
            rs.initiate(
            {
                _id: "myReplSet",
                version: 1,
                members: [
                    { _id: 0, host : "mongodb0.example.net:27017" },
                    { _id: 1, host : "mongodb1.example.net:27017" },
                    { _id: 2, host : "mongodb2.example.net:27017" }
                ]
            }
            )
        ```
        - แต่ของเราจะให้มีตัวเดียว 
        ```
             rs.initiate(
            {
                _id: "replicaset1",
                version: 1,
                members: [
                    { _id: 0, host : "localhost:27081" },
                ]
            }
            )
        ```
        - initiate replica
            - replicaset1 คือ ชื่อที่เราตั้งเอง ดูได้จาก docker-compose คือ MONGO_REPLICA_SET_NAME
            - localhost คือ domain เครื่องของเรา 
            - 27081 คือ port

            - ย่อเหลือ บรรทัดเดียว
            ```
                rs.initiate({ _id: "replicaset1", version: 1, members: [{ _id: 0, host : "localhost:27081" }]})
            ```

        - สรุป copy ข้างล่าง เปลียน "replicaset1", "localhost:27080" เป็นของเราเอง
        ```
            rs.initiate({ _id: "replicaset1", version: 1, members: [{ _id: 0, host : "localhost:27081" }]})
        ```
        - ผลที่ได้ 
        ```
            > rs.initiate({ _id: "replicaset1", version: 1, members: [{ _id: 0, host : "localhost:27081" }]})
            { "ok" : 1 }
            replicaset1:SECONDARY> 
        ```
         - อันนี้ คือ ทำ replicaSet สำเร็จ เรียบร้อย
        
           - คำสั่งที่ควรรู้ที่ mongo shell
            ```
                mongo localhost:27081/admin -u usermongo -p passmongo
                mongo localhost:27081/admin 
                mongo localhost:27081
                mongo 
            ```
            ```
                rs.initiate()
                rs.status()
                rs.config()
                rs.reconfig()
            ```

# การ rs.reconfig() 
- ตาม link ข้างล่างเลย
- https://www.mongodb.com/community/forums/t/replica-set-primary-in-state-other/124079/12

# สรุปคำสั่งที่ ใช้ ทั้งหมด
  - docker
    ```
        docker ps
        docker ps -a
        docker network ls
        docker pull image
        docker-compose up -d
        docker-compose down
        docker-compose pull
        docker-compose -f docker-compose-file-name.yml up -d
        docker-compose -f docker-compose-file-name.yml down
        docker-compose -f docker-compose-file-name.yml pull
        docker exec -it contianer_name bash
    ```
  - mongo
    ```
        mongo localhost:27081/admin -u usermongo -p passmongo
        mongo localhost:27081/admin 
        mongo localhost:27081
        mongo
        rs.initiate()
        rs.status()
        rs.config()
        rs.reconfig() 
    ```


# reference
- Atthavit Wannasakwong (ขออนุญาติมา ณ ที่นี้ด้วย)
- https://stackoverflow.com/questions/61486024/mongo-container-with-a-replica-set-with-only-one-node-in-docker-compose
- https://www.mongodb.com/docs/manual/reference/method/rs.initiate/
- https://blog.me-idea.in.th/mongodb-docker-compose-up-%E0%B8%9B%E0%B8%B8%E0%B9%8A%E0%B8%9A%E0%B8%AA%E0%B8%A3%E0%B9%89%E0%B8%B2%E0%B8%87-mongo-database-%E0%B8%9B%E0%B8%B1%E0%B9%8A%E0%B8%9A-d27004a9fd78
- https://www.mongodb.com/community/forums/t/replica-set-primary-in-state-other/124079/12\