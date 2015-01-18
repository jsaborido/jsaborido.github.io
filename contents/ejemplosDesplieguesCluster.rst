==================================================
Ejemplos de despliegues en cluster
==================================================

Ejemplo de despligue de un shard cluster con 2 shards y 3 nodos en cada replica set usando distintas máquinas.

**Información de la red**:

* 3 máquinas:

  * nodo1:              ip: "192.168.50.101"
  * nodo2:              ip: "192.168.50.102"
  * nodo3:              ip: "192.168.50.103"

* En cada máquina:

  * shard_server set1   port: 27001
  * shard_server set2   port: 27002
  * configserver        port: 28000
  * mongos              port: 27017 (default port)

**Montaje de los replica set**:

Nos conectamos a un nodo de cada uno de los replica set e inicializamos los replica set. ::

  # Replica set 1 (mongo --host 192.168.50.101:27001)
  config = { "_id": "set1", "members" : [
      {"_id" : 0, "host" : "192.168.50.101:27001" },
      {"_id" : 1, "host" : "192.168.50.102:27001" },
      {"_id" : 2, "host" : "192.168.50.103:27001" } ]
  }
  rs.initiate(config)

  # Replica set 2 (mongo --host 192.168.50.101:27002)
  config = { "_id": "set2", "members" : [
      {"_id" : 0, "host" : "192.168.50.101:27002" },
      {"_id" : 1, "host" : "192.168.50.102:27002" },
      {"_id" : 2, "host" : "192.168.50.103:27002" } ]
  }
  rs.initiate(config)

Nos conectamos a mongos e inicializamos el shard. ::

  sh.addShard("set1/192.168.50.101:27001")
  sh.addShard("set2/192.168.50.101:27002")

**Plantillas de aprovisionamiento**:

.. code-block:: yaml

  # mongo db shardsvr config
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/{{set_name}}/db.log
    logAppend: true
  processManagement:
    fork: true
  net:
    port: {{port_number}}
  storage:
    dbPath: /home/vagrant/mongodata/{{set_name}}
    journal:
      enabled: true
  replication:
    replSetName: {{set_name}}
  sharding:
    clusterRole: shardsvr

  # mongo db configsvr
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/configsvr/db.log
    logAppend: true
  processManagement:
    fork: true
  net:
    port: 28000
  storage:
    dbPath: /home/vagrant/mongodata/configsvr
    journal:
      enabled: true
  sharding:
    clusterRole: configsvr

  # mongos
  systemLog:
    destination: file
    path: /home/vagrant/mongodata/mongos/db.log
    logAppend: true
  processManagement:
    fork: true
  net:
    port: 27017
  sharding:
    configDB: {{shard_configDB}}
