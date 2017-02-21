================
Optimización
================

* Se pretende optimizar las queries de lectura para alto rendimiento aunque se penalicen las escrituras.
* MongoDB trabaja con estructuras B-Tree para los índices.
* En la parte de sharding, no se pueden modificar los índices. *Duda: ¿solo shard key o todos?*
* Límite de índices por colección: 64
* La ordenación en memoria sin un índice está limitada a 32MB
* En una query solo se puede utilizar 1 índice, salvo en una query con operador $or.

  * En el caso del $or se puede usar un índice por cada una de las clausulas alternativas.

* Aunque eliminemos una colección, el índice permacene (está asociado al espacio de nombres: fichero .ns)


Explain
============

Lo ejecutamos para la query: ::

    db.users.find().explain()

Cuando no existe un índice nos devuelve un ``BasicCursor`` ::

    > db.users.find({age: {$gt: 80}}).explain()
    { "cursor" : "BasicCursor",
      "isMultiKey" : false,
      "n" : 19101,
      "nscannedObjects" : 100000,
      "nscanned" : 100000,
      "nscannedObjectsAllPlans" : 100000,
      "nscannedAllPlans" : 100000,
      "scanAndOrder" : false,
      "indexOnly" : false,
      "nYields" : 781,
      "nChunkSkips" : 0,
      "millis" : 92,
      "server" : "mongodbvm:27017",
      "filterSet" : false
    }

.. tip ::

    * **"n" : 19101**. Número de documentos que coinciden con el criterio
    * **"nscanned" : 100000**. Índices y/o documentos leídos
    * **"millis" : 92**.
    * **"scanAndOrder"**. Si está a "true" indica que estamos utilizando un índice pero que **NO** nos sirve para ordenar.


Tipos de indexaciones
============================

* Simple (un solo campo)
* Compuesta (varios campos)
* MultiKey (arrays, subdocumentos o arrays de subdocumentos)
* TTL
* Text
* Geo (geoespaciales)
* Hashed Index


Comandos: ::

    //Crear un índice
    > db.users.ensureIndex({age: 1})
    //Con unicidad
    > db.users.ensureIndex({name:1},{unique:1})
    //Elimina los duplicados si existen (dejando el primero)
    > db.users.ensureIndex({name:1},{unique:1, dropDups : 1})
    //Índice disperso (puede o no existir el atributo)
    > db.nums.ensureIndex({z:1},{unique:true, sparse:true})
    //Construye el índice en background
    > db.nums.ensureIndex({z:1},{background:true})
    //Índice TTL
    > db.logs.ensureIndex({creado : 1}, {expireAfterSeconds: 5})
    //Índice de tipo texto
    > db.tweets.ensureIndex({text: "text"})
    //Índice geoespacial (primera versión)
    > db.tweets.ensureIndex({attrib: "2d"})
    //Índice geoespacial (última versión)
    > db.tweets.ensureIndex({attrib: "2dsphere"})
    //Índice por hash
    > db.collection.ensudeIndex( {_id: "hashed"})
    //Para ver los índices existentes
    > db.users.getIndexes()
    //Para borrar un índice
    > db.users.dropIndex("city_1")
    //Borra todos los índices de una colección
    > db.tweets.dropIndexes()

Cuidado con los índices dispersos y ``$exists``, ya que puede evitar el uso del índice u otros problemas...

Se podría insertar un z=null que sería un valor a tener en cuenta, pero no se permitiría otra tupla con z=null (por el índice único sobre z)

Durante la construcción en background podrían existir escrituras en medio, y será mas lento. A partir de la versión 2.6 se puede construir mas de un índice en background al mismo tiempo.


Índice Simple.
============================

* Indicando campo y orden
* Se crea un índice con punteros fuertes a cada documento

::

    > db.users.ensureIndex({age: 1})
    {   
        "createdCollectionAutomatically" : false,
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "ok" : 1
    }

    > db.users.find({age: {$gt: 80}}).explain()
    {
        "cursor" : "BtreeCursor age_1",
        "isMultiKey" : false,
        "n" : 19101,
        "nscannedObjects" : 19101,
        "nscanned" : 19101,
        "nscannedObjectsAllPlans" : 19101,
        "nscannedAllPlans" : 19101,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "nYields" : 149,
        "nChunkSkips" : 0,
        "millis" : 81,
        "indexBounds" : {
            "age" : [
                [
                    80,
                    Infinity
                ]
            ]
        },
        "server" : "mongodbvm:27017",
        "filterSet" : false
    }


    > db.users.ensureIndex({city: 1})

    > db.users.find({city: /^B/i}).explain()
    {
        "cursor" : "BtreeCursor city_1",
        "isMultiKey" : false,
        "n" : 22210,
        "nscannedObjects" : 22210,
        "nscanned" : 100000,
        "nscannedObjectsAllPlans" : 22210,
        "nscannedAllPlans" : 100000,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "nYields" : 781,
        "nChunkSkips" : 0,
        "millis" : 230,
        "indexBounds" : {
            "city" : [
                [
                    "",
                    {

                    }
                ],
                [
                    /^B/i,
                    /^B/i
                ]
            ]
        },
        "server" : "mongodbvm:27017",
        "filterSet" : false
    }

.. tip:: El índice no funciona porque en este caso se está usando una expresión regular.

Ante varios índices el planificador usa el índice que cree mas conveniente: ::

    > db.users.find({city:"Barcelona",age: {$gt: 80}}).explain()
    {
        "cursor" : "BtreeCursor city_1",
        "isMultiKey" : false,
        "n" : 2137,
        "nscannedObjects" : 11192,
        "nscanned" : 11192,
        "nscannedObjectsAllPlans" : 11704,
        "nscannedAllPlans" : 12220,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "nYields" : 95,
        "nChunkSkips" : 0,
        "millis" : 48,
        "indexBounds" : {
            "city" : [
                [
                    "Barcelona",
                    "Barcelona"
                ]
            ]
        },
        "server" : "mongodbvm:27017",
        "filterSet" : false
    }



Índice Compuesto
============================

::

    > db.users.ensureIndex({city: 1, age: 1})

Si se busca por un campo del índice se puede seguir usando un índice compuesto
siempre que este sea el primer nivel del índice (el orden importa a la hora de crear los índices compuestos).

.. note::
    Si se recupera solo la información que está en el índice, es una query muy rápida y se denomina **"Query Covered"**. Para esto los campos de la búsqueda y de la proyección deben estar en el índice: ::

        > db.users.find({city:"Barcelona"}, {age:1, _id:0}).explain()

Podemos añadir mas niveles al índice ::

    > db.users.ensureIndex({city: 1, age: 1, name:1}, {name:"byCityAgeAndName"}) //Con nombre

    > db.users.find({city: "Barcelona", name : /9$/}).explain()
    {
        "cursor" : "BtreeCursor city_1_age_1_name_1",
        "isMultiKey" : false,
        "n" : 1098,
        "nscannedObjects" : 1098,
        "nscanned" : 11192,
        "nscannedObjectsAllPlans" : 3150,
        "nscannedAllPlans" : 13246,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "nYields" : 103,
        "nChunkSkips" : 0,
        "millis" : 77,
        "indexBounds" : {
            "city" : [
                [
                    "Barcelona",
                    "Barcelona"
                ]
            ],
            "age" : [
                [
                    {
                        "$minElement" : 1
                    },
                    {
                        "$maxElement" : 1
                    }
                ]
            ],
            "name" : [
                [
                    "",
                    {

                    }
                ],
                [
                    /9$/,
                    /9$/
                ]
            ]
        },
        "server" : "mongodbvm:27017",
        "filterSet" : false
    }



Índice Multikey
============================

Cuando se hace sobre un atributo que es un array, un subdocumento o arrays de subdocumentos.

Se pueden hacer índices multikey combinandolas con "simple", pero es mejor que el primer nivel sea el simple

Sobre atributos de un subdocumento (no lo toma como un multiKey, pero se categoriza aquí): ::

    > db.tweets.ensureIndex({"user.followers_count" : 1, "user.friends_count" : 1})

    > db.tweets.find({"user.followers_count" : {$gt : 1000}}).explain()
    {
        "cursor" : "BtreeCursor user.followers_count_1_user.frinds_count_1",
        "isMultiKey" : false,  <----- multikey
        "n" : 5862,
        ...



Índices TTL
============================

Estos son los índices con tiempo de vida, como por ejemplo logs rotativos, o almacenamiento de sesiones

Existen 3 restricciones para este tipo de índices:

*   El campo a indexar tiene que ser un ISODate (obligatorio).
*   Tiene que estar en un campo limpio, sin que esté usado en un prefijo de un índice (sin que se use en otro índice).
*   No puede ser una clave compuesta.

Ej.: ::

    > use logger
    //No se puede garantizar que sea justo a los 5s
    > db.logs.ensureIndex({creado : 1}, {expireAfterSeconds: 5})
    > db.logs.insert({creado: new Date(),
            msg: "Prueba 001", detalles: "Prueba 001", nivel_log: "FINE"})
    > db.logs.find() // después de 5s desaparece el documento

Otra alternativa, expire en 0 y añadiendo la fecha de expiración en el insert: ::

    > db.subastas.ensureIndex({fecha_final:1},{expireAfterSeconds: 0})
    > db.subastas.insert({id:90, product: {id: 4, precio: 56.8}, pujas:[], 
            fecha_final: new Date("20:15:00")})



Índices TEXT
============================

Indice sobre textos complejos.

Como son lentos, es normal montarlos en background. Cuando trabajemos sobre índices TEXT siempre trabajaremos con accesos a disco.

A la hora de definir un índice, en vez de un "1", estableceremos un "text". ::

    > db.tweets.ensureIndex({text: "text"})

Para hacer búsquedas sobre texto será el operador $text (tanto en CRUD como en el framework de agregación). ::

    > db.tweets.find({$text: {$search: "preciso"}}).explain()

.. note:: Siempre es un OR, no existe un Y.

**Restricción**: Solo puede existir un único índice de tipo "texto".

Se puede escoger un idioma o "none" (por defecto es el del sistema): ::

    > db.tweets.ensureIndex({text: "text"},{default_language: "fr"})
    > db.tweets.ensureIndex({text: "text"},{default_language: "none"})

    // Busca con OR
    > db.tweets.find({$text: {$search: "depende de la velocidad"}})

    // Busqueda por frase
    > db.tweets.find({$text: {$search: "\"depende de la velocidad\""}})

    // Busqueda por frase AND otro término (Entre frases y palabras AND, entre palabras OR)
    > db.tweets.find({$text: {$search: "\"depende de la velocidad\" cosaDistinta"}})

    //Especificando idioma
    > db.tweets.find({$text: {$search: "\"depende de la velocidad\" cosa distinta", 
            $language: "es"}}) 



Índices geoespaciales
============================

**Tipo "2d"**.

*   Plano
*   Los primeros en aparecer en Mongo.
*   Las coordenadas son sobre 100,100 (si queremos mas valores hay que especificar una opción)
*   La estructura a mantener es un array simple donde añadiremos la longitud y la latitudes ([long, latitud])
*   Se utiliza como entorno legacy

**Tipo "2dsphere"**.

*   El segundo que se añadió y el que se debería utilizar en la mayoría de los casos
*   Sigue la especificación de GeoJSON
*   http://docs.mongodb.org/manual/tutorial/build-a-2dsphere-index/
*   La localización es un subdocumento donde se define el tipo geométrico que vamos a tener ("Point", "Line", "Point[]", "Polygon", *  "Circle")

    *   ``{loc: {type : "Point", coordinates: [long, lat]}}``

.. important:: Con "2dsphere" las unidades son en metros, mientras que con "2d" son en radianes.

::

    > db.<collection>.ensureIndex({<location field>:"2d", <additional field>: <value>}, ...)

    > db.<collection>.find( { <location field> : 
            { $geoWithin: {$box|$polygon|$center : <coordinates>}}})

Se puede usar $near (en los 2dsphere solo??) como alternativa a $geoWithin (cerca vs dentro de una figura)

Con $near sale un listado de documentos ordenados por cercanía ::

    {
      $near: {
         $geometry: {
            type: "Point" ,
            coordinates: [ <longitude> , <latitude> ]
         },
         $maxDistance: <distance in meters>,
         $minDistance: <distance in meters>
      }
    }

    > db.tweets.find({coordinates : {$near: {$geometry : 
            {type : "Point", coordinates : [107.119031, -6.301964] }}}}).count()
    > db.tweets.find({coordinates : {$near: {$geometry : 
            {type : "Point", coordinates : [107.119031, -6.301964] }, 
            $maxDistance : 20000}}}).count() //20km de distincia máxima



Índices Hash
============================

Desde la versión 2.4 de MongoDB

Podemos utilizar el campo que queramos y marcarlo dentro del index como "hashed".

En vez de trabajar con un documento de texto grande, se trabajaría con un hash para realizar la indexación. Es una buena opción para elegir como clave de shard key en determinadas colecciones. ::

    > db.collection.ensudeIndex( {_id: "hashed"})

