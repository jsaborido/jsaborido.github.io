==================================================
Anexo. Ejemplos de despliegues en la misma máquina
==================================================

Ejemplos con máquinas vagrant. Usan el directorio ``/home/vagrant/mongodata`` para almacenar los ficheros de datos.

Los ejemplos de linea de comandos pueden ser pasados a ficheros de configuración con el antiguo formato.

Para lanzar los demonios usando los ficheros de configuración (YAML o antiguo formato) usar los comandos ``-f`` ó ``--config``.

Estos despliegues crearán todos los demonios en la misma máquina usando distintos puertos


Linea de comandos
================================

Standalone
---------------------------------

::

    mkdir -p /home/vagrant/mongodata/standalone/db

    mongod --dbpath=/home/vagrant/mongodata/standalone/db --journal \
        --fork --logpath=/home/vagrant/mongodata/standalone/db.log --logappend

    mongo


Replica Set
---------------------------------

::

    mkdir -p /home/vagrant/mongodata/replicaSet/{node1,node2,node3}/db

    mongod --dbpath=/home/vagrant/mongodata/replicaSet/node1/db --journal \
        --fork --logpath=/home/vagrant/mongodata/replicaSet/node1/db.log \
        --logappend --replSet set0 --port 28001
    mongod --dbpath=/home/vagrant/mongodata/replicaSet/node2/db --journal \
        --fork --logpath=/home/vagrant/mongodata/replicaSet/node2/db.log \
        --logappend --replSet set0 --port 28002
    mongod --dbpath=/home/vagrant/mongodata/replicaSet/node3/db --journal \
        --fork --logpath=/home/vagrant/mongodata/replicaSet/node3/db.log \
        --logappend --replSet set0 --port 28003

    mongo --port 28001

**Inicialización del replica set** ::

    config = { "_id": "set0", "members" : [
        {"_id" : 0, "host" : "precise32:28001" },
        {"_id" : 1, "host" : "precise32:28002" },
        {"_id" : 2, "host" : "precise32:28003" } ]
    }
    rs.initiate(config)


Shard
---------------------------------

::

    mkdir -p /home/vagrant/mongodata/shard/rs{1,2}/node{1,2,3}/db
    mkdir -p /home/vagrant/mongodata/shard/config_server_{1,2,3}

    mongod --dbpath=/home/vagrant/mongodata/shard/rs1/node1/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs1/node1/db.log \
        --logappend --replSet set1 --port 28001 --shardsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/rs1/node2/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs1/node2/db.log \
        --logappend --replSet set1 --port 28002 --shardsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/rs1/node3/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs1/node3/db.log \
        --logappend --replSet set1 --port 28003 --shardsvr

    mongod --dbpath=/home/vagrant/mongodata/shard/rs2/node1/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs2/node1/db.log \
        --logappend --replSet set2 --port 28011 --shardsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/rs2/node2/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs2/node2/db.log \
        --logappend --replSet set2 --port 28012 --shardsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/rs2/node3/db --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/rs2/node3/db.log \
        --logappend --replSet set2 --port 28013 --shardsvr

    mongod --dbpath=/home/vagrant/mongodata/shard/config_server_1 --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/config_server_1/mongod.log \
        --logappend --port 28101 --configsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/config_server_2 --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/config_server_2/mongod.log \
        --logappend --port 28102 --configsvr
    mongod --dbpath=/home/vagrant/mongodata/shard/config_server_3 --journal \
        --fork --logpath=/home/vagrant/mongodata/shard/config_server_3/mongod.log \
        --logappend --port 28103 --configsvr

    mongos --configdb precise32:28101,precise32:28102,precise32:28103 \
        --logpath=/home/vagrant/mongodata/shard/mongos.log --logappend \
        --fork --port 28000

**Inicialización de los replica set** ::

    # Conectar antes a un nodo del replica set (mongo --port 28001)
    config = { "_id": "set1", "members" : [
        {"_id" : 0, "host" : "precise32:28001" },
        {"_id" : 1, "host" : "precise32:28002" },
        {"_id" : 2, "host" : "precise32:28003" } ]
    }
    rs.initiate(config)

    # Conectar antes a un nodo del replica set (mongo --port 28011)
    config = { "_id": "set2", "members" : [
        {"_id" : 0, "host" : "precise32:28011" },
        {"_id" : 1, "host" : "precise32:28012" },
        {"_id" : 2, "host" : "precise32:28013" } ]
    }
    rs.initiate(config)

**Inicialización de los shard** ::

    # Conectar antes a mongos (mongo --port 28000)
    sh.addShard("set1/precise32:28001")
    sh.addShard("set2/precise32:28011")

**Activando sharding en una base de datos y tabla** ::

    sh.enableSharding("mydb")
    #Se necesita un índice sobre la shard key si no es _id
    db.testtb.ensureIndex({dummy:1}) 
    sh.shardCollection("mydb.testtb", {dummy:1})

YAML
=============

Nuevo formato de configuración. Estos ficheros .yml son los ficheros equivalentes a los ejemplos anteriores.

Sustituyen el lanzamiento del demonio ``mongod`` para pasarle solo el fichero de configuración: ::

  mongod -f configfile.yml


Standalone
---------------------------------

.. code-block:: yaml

    systemLog:
      destination: file
      path: /home/vagrant/mongodata/standalone/db.log
      logAppend: true
    processManagement:
      fork: true
    storage:
      dbPath: /home/vagrant/mongodata/standalone/db
      journal:
        enabled: true

Replica Set
---------------------------------

.. code-block:: yaml

    # node1_conf.yml file
    systemLog:
      destination: file
      path: /home/vagrant/mongodata/replicaSet/node1/db.log
      logAppend: true
    processManagement:
      fork: true
    net:
      port: 28001
    storage:
      dbPath: /home/vagrant/mongodata/replicaSet/node1/db
      journal:
        enabled: true
    replication:
      replSetName: set0

    # node2_conf.yml file
    systemLog:
      destination: file
      path: /home/vagrant/mongodata/replicaSet/node2/db.log
      logAppend: true
    processManagement:
      fork: true
    net:
      port: 28002
    storage:
      dbPath: /home/vagrant/mongodata/replicaSet/node2/db
      journal:
        enabled: true
    replication:
      replSetName: set0

    # node3_conf.yml file
    systemLog:
      destination: file
      path: /home/vagrant/mongodata/replicaSet/node3/db.log
      logAppend: true
    processManagement:
      fork: true
    net:
      port: 28003
    storage:
      dbPath: /home/vagrant/mongodata/replicaSet/node3/db
      journal:
        enabled: true
    replication:
      replSetName: set0


Shard
---------------------------------

.. code-block:: yaml

  #rs1_node1_conf.yml
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/shard/rs1/node1/db.log
    logAppend: true
  processManagement.fork: true
  net.port: 28001
  storage:
    dbPath: /home/vagrant/mongodata/shard/rs1/node1/db
    journal.enabled: true
  replication.replSetName: set1
  sharding.clusterRole: shardsvr

  #... (Similar en el resto de nodos del mismo replica set 1)

  #rs2_node1_conf.yml
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/shard/rs2/node1/db.log
    logAppend: true
  processManagement.fork: true
  net.port: 28011
  storage:
    dbPath: /home/vagrant/mongodata/shard/rs2/node1/db
    preallocDataFiles: false
    smallFiles: true
    journal.enabled: true
  replication.replSetName: set2
  sharding.clusterRole: shardsvr

  #... (Similar en el resto de nodos del mismo replica set 2)

  #config_server1_conf.yml
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/shard/config_server_1/mongod.log
    logAppend: true
  processManagement.fork: true
  net.port: 28101
  storage:
    dbPath: /home/vagrant/mongodata/shard/config_server_1
    journal.enabled: true
  sharding.clusterRole: configsvr

  #... (Similar en el resto de nodos de config servers)

  #mongos.yml
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/shard/mongos.log
    logAppend: true
  processManagement.fork: true
  net.port: 28000
  sharding.configDB: "precise32:28101,precise32:28102,precise32:28103"
