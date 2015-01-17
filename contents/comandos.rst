========================
Línea de comandos
========================

:mongod:        primary daemon process for the MongoDB system
:mongos:        “MongoDB Shard,” routing service for MongoDB shard configurations
:mongo:         interactive JavaScript shell interface to MongoDB
:mongodump:     utility for creating a binary export
:mongorestore:  writes data from a binary database dump created by mongodump to a MongoDB instance
:bsondump:      The bsondump converts BSON files into human-readable formats, including JSON
:mongooplog:    simple tool that polls operations from the replication oplog of a remote server, and applies them to the local server
:mongoimport:   import content from a JSON, CSV, or TSV export created by mongoexport
:mongoexport:   produces a JSON or CSV export of data stored in a MongoDB instance
:mongostat:     quick overview of the status of a currently running mongod or mongos instance
:mongotop:      method to track the amount of time a MongoDB instance spends reading and writing data.
:mongosniff:    low-level operation tracing/sniffing view into database activity in real time
:mongoperf:     check disk I/O performance independently of MongoDB
:mongofiles:    makes it possible to manipulate files stored in your MongoDB instance in GridFS objects from the command line


mongod
----------------------
::

    [--dbpath]                          default= /data/db, c:\data\db
    [--port]                            default=27017
    [--journal | --nojournal]           active by default on 64 bits
    [--smallfiles]      
    [--nssize <MB>]                     max 2GB
    [--noprealloc]
    [--logpath]
    [--logappend]
    [--fork]                            Not supported on windows
    [--config|-f <ASCII configFile>]
    [--replSet <rsName>]
    [--oplogSize <MG>]                  default=5% free space
    [--shardsvr]
    [--configsvr]
    [--auth]
    [--keyFile <path to file>]
    [--syncdelay 45]                    seconds between disk syncs
    [-vvvvv]                            Verbose level

**Data File Allocation**.

Each database will have at least two data files, one ending in .ns and the rest in integers starting with 0.

The .ns file stores metadata about namespaces (collections and indexes). The number of namespaces is
proportional to the size of the .ns file. Each database can have up to 24,000 namespaces by default, although
the size of these files, and thus the number of namespaces, can be increased with the ``--nssize`` option (up
to 2GB).

By default, datafiles start at 64MB and double in size with each additional datafile, up to 2GB. Additionally,
on some platforms, mongod allocates one more numbered data file than it needs, to improve throughput.
Thus, it’s possible for allocated size to be much larger than data size. If this presents a problem, you can
use some combination of server options: ::

    --smallfiles    // quarters the sizes of data files
    --noprealloc    // inhibits preallocation of extra files



**The Lock File**

``mongod.lock`` contains PID. When the server shuts down cleanly, it truncates the lock file to length 0.




**Journal Subdirectory**

Activated by default on x86_64 machines. Directory ``journal``.

*Nota: Todos los servidores comparten el mismo journal.*



**Log Files**

Default: standard output.

``--logpath /var/mongodb/mongodb.log --logappend`` allows to write the log into a file.

The database includes a command to rotate log files::

    db.adminCommand( { "logRotate" : 1 } ) ;


**Config files**

All of these options can be specified in a config file. Any option that takes an argument is specified as option
= argument. Options that don’t take arguments are specified as option = true::

    fork = true
    # vvv = true
    logpath = /var/mongodb/mongodb.log

Since mongodb 2.6 version, a new YALM format is introduced. Example:

.. code-block:: yaml

    systemLog:
      destination: file
      path: /home/vagrant/mongodata/singledb/db.log
      logAppend: true
    processManagement:
      fork: true
    net:
      port: 27017
      http.enabled: true
    storage:
      dbPath: /home/vagrant/mongodata/singledb/db
      preallocDataFiles: false
      smallFiles: true
      journal:
        enabled: true

    # Otras secciones no probadas (obtenidas wiki online)
    setParameter:
    security:
      ...
    operationProfiling:
      ...
    replication:
      ...
    sharding:
      ...
    auditLog:
      ...
    snmp:
      ...





mongorestore
----------------------
::

    mongorestore (.exe) -d digg sampledata/dump/digg
    mongorestore (.exe) -d training -c scores sampledata/dump/training/scores.bson


mongoimport
----------------------
::

    mongoimport (.exe) -d twitter -c tweets sampledata/twitter.json

