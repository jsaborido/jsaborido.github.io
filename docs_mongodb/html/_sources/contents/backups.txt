==================================================
Backups y restauración
==================================================

DR (Disaster recovery) vs HA (High availability)

Depende de las necesidades del negocio, ¿podemos perder datos de un día? ¿cuanto tiempo podemos estar offline?...

Si perdemos un nodo, al conectar un nodo limpio se replican los datos desde el primario, pero esto significa una carga de la red al transpasar datos. Por lo que puede ser interesante cargar de un dump (o por si perdemos todo el replicaSet).


Backups lógico: mongodump/mongorestore.
==================================================

2 opciones:

* Conectándose a mongod
* Leyendo datos del dbpath directamente

Genera para cada colección 2 ficheros:

* documentos en formato BSON
* metadatos en formato JSON (estos metadatos son basicamente la definición de los índices)

Con el comando ``bsondump`` podemos ver los datos existentes en los ficheros bson.

Ejemplos:

* Para crear un dump completo de todas las BD del replica set (menos BD "local") ::

    $ mongodump --port 27002

* Dump de una sola base de datos indicando en que directorio se generará (por defecto se genera en "dump") ::

    $ mongodump --port 27002 -d twitter --out backup/misbackups/twitter

Para restaurar hay que conectarse al primario o sacar el secundario fuera del replicaset para cargar los datos ::

    $ mongorestore -d twitter --port 27001 backup/misbackups/twitter/twitter

.. NOTE:: Si se restaura sobre una BD existente, crea los documentos nuevos si no existen (por clave primaria) o da un error (que no se muestra en consola)

**Este backup puede ser inconsistente**, ya que pueden estar modificándose datos mientras se está haciendo el backup. Es decir, no bloquea la BD. Para tener un backup consistente hay 2 opciones:

* Bloquear: ``db.fsyncLock()/db.fsyncUnlock()`` (Esta es una buena opción para no tener que parar el servidor) o parar el servidor. 
* ``mongodump`` con ``--oplog``. Guardo las bases de datos y las operaciones ejecutadas durante el dump de la base de datos.

El fsyncLock además de bloquear hace una sincronización de memoria con disco, para que los ficheros de datos tengan los datos actualizados.

Mongodump es el mecanisco de backup mas lento. El backup físico es mas rápido, por lo que ante servidores standalone, ya que hay que bloquear para hacer backups existentes, es mejor un backup físico.

Para la opción --oplog tiene que ser un backup completo. ::

    $ mongodump --port 27002 --out backup/misbackups/withOpLog --oplog

Esto creará además de los archivos bson y json anteriores, un nuevo fichero oplog.bson

Para restaurar un dump con oplog: ::

    $ mongorestore --port XXX --oplogReplay dumpFolder

Se podrían simular backups incrementales haciendo backups del oplog (no existe una manera nativa de backups incrementales) ::

    $ mongodump --port 27002 -d local -c oplog.rs --out outfolder
    $ mv oplog.rs.bson oplog.bson
    $ mongorestore --port XXX --oplogReplay ...

Para filtros selectivos en el dump tenemos "``--query/filter``"

Si queremos restaurar en una base de datos con distinto nombre simplemente: ::

    $ mongorestore --port XXX -d nuevoNombre pathToDump



Backups físicos
==================================================

Esta es una copia de ficheros. Y para que sean consistentes, tenemos varias opciones:

* shutdown / fsyncLock() ó...
* snapshot del sistema de ficheros y luego copia de estos ficheros. Minimiza los tiempos de backup con lo que son consistentes.

  .. important::
    Importante tener el journaling activado (para poder hacerlo sin apagar el servidor) y que el journal esté en el mismo sistema de ficheros.

    Si el journal está en otro volumen, primero deberíamos hacer un bloqueo, hacer snapshots de todos los filesystems y quito el bloqueo. Luego se puede hacer la copia de ficheros a partir del snapshots.

Con la copia física hay que copiar una BD completa siempre. No es posible copiar una sola colección.


En un shard
-------------------------------

#. Hacer backup de cada uno de los replica set y de los config server.
#. Paramos balanceador
#. Paramos un config server (para que no se modifiquen los metadatos de los chunk)
#. Backup de cada shard y de un config server
#. Reiniciamos config server
#. Reiniciamos balanceador

*Info: La gente de MongoDB tiene MMS Backup. De esta forma la gente de Mongo es quien hace el backup.*

