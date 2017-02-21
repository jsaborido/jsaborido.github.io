=============
Seguridad
=============

MongoDB tiene la autenticación/autorización **desactivada por defecto**.

Esto es debido a que la primara opción de seguridad debería ser **establecer la red de cluster en una red privada protegida por cortafuegos**.

------------

**Cifrado**

La comunicación entre los distintos nodos y cliente no está cifrada, ni dispone de cifrado alternativo. Se podría usar aplicaciones de terceros (tunel ssh, ...). Hay que tener en cuenta que el cifrado puede penalizar el rendimiento.

Tampoco hay cifrado de datos en disco.

Lo único que hay cifrado es la contraseña de los usuarios (tanto en el servidor como en la red).

-------------

**Autenticación:**

Tenemos 2 tipos de autenticación:

Para un **cliente**, especificando usuario y contraseña. Asociada a esta autenticación tenemos privilegios de usuarios.

Los usuarios se crean a nivel de base de datos.

* ``use twitter; db.addUser("...")``. También ``db.createUser(...)`` desde la 2.6. Con este último comando podemos crear un usuario para varias BD.
* Se guardan en la colección ``system.users`` en cada una de las BD.
* Debemos activar la autenticación con ``--auth`` en el arranque del demonio.
* Desde localhost se permite conectar sin autenticación si no hay usuarios definidos en ``admin.system.users``.

Para usar **autenticación con un shard o réplica set**, primero debemos generar una clave: ::

    $ openssl rand -base64 741 > mongodb-keyfile

Y cambiar los permisos ::

    $ chmod 600 mongodb-keyfile

Lo que nos queda es arrancar los replicaSet y shard con la opción ``--keyFile`` (implica ``--auth``)

La clave es para evitar suplantaciones de identidad de un réplica set, lo cual sería posible si se cae un elemento del réplica set y se suplanta la IP.

-------------

**Ejemplo configuración con clave** ::

    # Config servers
    mongod --dbpath=data/config_server_1 --fork \
         --logpath=data/config_server_1/mongod.log --logappend \
         --port 28001 --configsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/config_server_2 --fork \
         --logpath=data/config_server_2/mongod.log --logappend \
         --port 28002 --configsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/config_server_3 --fork \
         --logpath=data/config_server_3/mongod.log --logappend \
         --port 28003 --configsvr --keyFile data/mongodb-keyfile

    # RS set0
    mongod --dbpath=data/rs1/db1 --journal --fork \
         --logpath=data/rs1/db1/mongod.log --logappend --replSet set0 --port 27001 \
         --shardsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/rs1/db2 --journal --fork \
         --logpath=data/rs1/db2/mongod.log --logappend --replSet set0 --port 27002 \
         --shardsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/rs1/db3 --journal --fork \
         --logpath=data/rs1/db3/mongod.log --logappend --replSet set0 --port 27003 \
         --shardsvr --keyFile data/mongodb-keyfile

    # RS set1
    mongod --dbpath=data/rs2/db1 --journal --fork \
         --logpath=data/rs2/db1/mongod.log --logappend --replSet set1 --port 27101 \
         --shardsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/rs2/db2 --journal --fork \
         --logpath=data/rs2/db2/mongod.log --logappend --replSet set1 --port 27102 \
         --shardsvr --keyFile data/mongodb-keyfile
    mongod --dbpath=data/rs2/db3 --journal --fork \
         --logpath=data/rs2/db3/mongod.log --logappend --replSet set1 --port 27103 \
         --shardsvr --keyFile data/mongodb-keyfile

    # MongoS
    mongos --configdb mongodbvm:28001,mongodbvm:28002,mongodbvm:28003 \
         --logpath=data/mongos1/mongos.log --logappend --fork --port 28000 \
         --keyFile data/mongodb-keyfile
