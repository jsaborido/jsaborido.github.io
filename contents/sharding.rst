==========================
Sharding
==========================

Para alto rendimiento deberíamos tener todos los índices cargados en memoria junto con el working set [#workingSet]_.

La forma tradicional ante llenados de memoria es escabilidad vertical (mas memoria o disco). Mongo con los shards nos permite escalabilidad horizontal.

Funcionamiento:

* Para particionar una colección usamos una "shard key"
* Para que una colección se particione automáticamente hay que activar sharding a nivel de base de datos y a nivel de colección.
* Las colecciones que no tengan shard se crean en el shard1 siempre (primary shard).

  * Con ``sh.movePrimary(...)`` podemos cambiar el shard que será el primario (moverá las colecciones)

* Según la shard key, inserta en la partición que corresponda entre "Min value" y "Max value"
* Una partición es un "chunk" no un "shard".
* Cuando una colección es pequeña inicialmente tiene una única partición (chunk) con un tamaño máximo por defecto de 64MB. (-inf .. inf). El chunch es en base a la shard key por lo que solo existe cuando estamos en un shard y la colección está marcada para sharding.
* Al llenarse MongoDB la divide en 2 chunks automáticamente (-inf.. 1000),[1000... inf) si el tamaño era de 2000 documentos todos del mismo tamaño. (realmente divide en 2 chunks de 32MB).
* En la división de chunks se pasa uno de los nuevos chunk al otro shard (parece que no tiene por qué ser siempre, sino que es mongos quien va balanceando los chunks entre shards)


shard key
==========================

* Una shardkey puede tener duplicados, se puede definir un nombre por ejemplo para que las escrituras vayan a varios shards

  * Por la misma razón, el objectId no es bueno para la shardkey porque los timestamp del principio obligaría a escribir todos los últimos en el mismo shard. (incrementales y timestamps no son buena idea)

* Una shardkey solo puede estar en un chunk, es decir, si es un valor muy usado, puede haber chunks con mas de 64MB (por ejemplo datos de los usuarios, con shardkey "santiago") -> jumbochunk.

* Para poder crear una shard key **debe existir un índice sobre el campo o un índice compuesto que empiece por ese campo**.

* Es inmutable

* Los valores son inmutables

* Limitada a 512 bytes de tamaño *¿¿Comprobar??*.

* Es usada para distribuir las queries en un shard.

  * Por lo que se debería elegir un campo que sea usado comunmente en las queries

* can be unique across shards (puede, que no debe) *¿¿Comprobar??*.

  * '_id' field is only unique on individual shards


**Consideraciones sobre la shard key**. Reglas recomendadas a seguir, pero no son obligatorias ni tienen por que ser la solución óptima:

* Cardinality (alta cardinalidad). Veremos que nos pueden ayudar los índices compuestos.
* Write Distributions
* Query Isolation (que una query vaya sobre un shard siempre que sea posible)
* Reliability
* Index Locality

.. important:: *Cardinalidad y que no exista Hotspotting* (por ejemplo fecha actual, que siempre carga el mismo shard)

Estrategias que se suelen usar:

* basado en localización (IPs pueden estar incluidas aquí)
* incremental + (compuesto)
* **Bucketing**. Dependiendo de lo que queramos hacer, hay que potenciar unos elementos u otros.
* **Hash**.

Preguntas a hacer para elegir la clave:

* Crecimiento shard. Cómo esperamos que crecerá nuestro shard (3 nodos shard, o 100 nodos... mejor 3)
* Latencia que sufriremos (zonas geográficas separadas)
* CPU & RAM (tiempo de proceso real). Cómo va a crecer el trabajo
* Ratio de lecturas/escrituras sobre la colección


mongos
==========================

Demonio intermediario al que se conecta el cliente y decide porque shard pasa la query (hace de router).

Para saber a que shard manda la query usa los **config server** (tienen información de todos los chunks y donde están esos chunk).

Sabe a donde mandar la query cuando se usa en la query la shardkey sino hay que consultarlos todos.

En la práctica, se debería poner un mongos en cada máquina cliente.


config server
==========================

El config server debería ser una máquina dedicada (aunque tiene muy poca configuración). Podemos tener 1 o 3, no 2, (los 3 tienen los mismos datos), pero si se cae uno se convierten en solo lectura (de metadatos: no se puede cambiar la definición de los chunks, pero si se pueden realizar escrituras en colecciones)

Para activarlo hay que definir varios shard (puede ser un standalone o un replica set, mejor este último para tener tolerancia a fallos).

El mongos bloquea los config server durante la migración o split de chunks para que otro mongos no pueda escribir.


Configuración.
==========================

**config server**:

* Desde linea de comandos: ``mongod ... --configsvr``
* Desde ficheros de configuración

  * ``configsvr = true`` (antiguo formato)
  * ``sharding.clusterRole: configsvr`` (YAML)

El resto de las configuraciones vistas para mongod son también válidas.


**mongos**:

Desde linea de comandos: ``mongos --configdb <hostname and port of config servers>`` (por defecto: 27017) ::

    mongos ... --configdb mongodbvm:28001 // configuración con 1 config server
    mongos ... --configdb mongodbvm:28001,mongodbvm:28002,mongodbvm:28003 // configuracón con 3 config servers


**shards**:

Cada uno de los shard estará compuesto de un servidor mongod standalone o de un replica set (mejor este caso para tolerancia a fallos)

Cada uno de los ``mongod`` deberán tener la opción ``shardsvr`` activa para indicar que forman parte de un shard cluster.

* Desde linea de comandos: ``mongod ... --shardsvr``
* Desde ficheros de configuración

  * ``shardsvr = true`` (antiguo formato)
  * ``sharding.clusterRole: shardsvr`` (YAML)


**Inicialización**:

Una vez están los demonios ejecutándose, nos conectamos con el comando ``mongo`` al demonio ``mongos``

Una vez conectado, para añadir un shard al cluster: ::

    sh.addShard("localhost:27001") // shard con un mongod standalone.
    sh.addShard("rep0/localhost:27001") // shard con un replica set.

.. note:: En el caso de replica sets, sólo es necesario añadir un nodo. El sistema ya obtiene toda la configuración del shard.

Ahora necesitamos habilitar el sharding para la base de datos y para las colecciones que queramos: ::

    sh.enableSharding("dbname")
    //Field es el campo que será la shard key.
    sh.shardCollection("dbname.collectionname",{field:1}) 



Configuración rápida.
====================================

Al igual que los replica sets, se puede crear un shard rápido de prueba ::

  $ mongo --nodb
  > shard = new ShardingTest({name: "testShard", shards: 3, chunksize: 26,
                  rs: { dbpath:"/data/shard"}})

Info, para mover colecciones entre tags: ::

  mongos> db.runCommand({movePrimary: "tweets", to: "shard0000"})

  mongos> db.tweets.ensureIndex({"user.screen_name": 1})
  //No lo crea automáticamente el shard key, con lo que es bueno hacerlo antes.
  mongos> sh.enableSharding("twitter")
  mongos> sh.shardCollection("twitter.tweets", {"user.screen_name":1})


.. rubric:: Footnotes

.. [#workingSet] **Working set**. Datos que mas utilizamos y que nos interesa que estén en memoria directamente.
