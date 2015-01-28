===========================
CRUD - MongoDB Shell
===========================

Se accede con el comando ``mongo``.

La base de datos por defecto es ``test``.

Si se pasa la función sin paréntisis, el shell pinta la definición. Ej.: ``db.startup_log.findOne``

Ejemplos comandos básicos::

    show dbs | db.adminCommand({"listDatabases": 1})
    use <dbname>
    show collections | show tables | db.getCollectionNames()
    db.<col>.findOne()
    db.<col>.find()
    db.<col>.insert(<jsonObject>)



**Document Data Types**

It’s sometimes useful to see the size of an object in BSON. You can do that like so ::

    Object.bsonsize( { "user.name" : "Emma Smith" } ) ;
    Object.bsonsize( db.tweets.findOne( { "user.name" : "Emma Smith" } ) ) ;

You can also query objects by BSON type. Here’s how to find all stories where href is a string. ::

    use digg
    db.stories.find( { "href" : { "$type" : 2 } } ) ;

Cargar desde un fichero: ::

    load("testIndex.js")


Inserts
============

::

    db.people.insert( { "name" : "Smith", "age": 30 } ) ;


Queries
============

::

    db.scores.find( { "score" : { "$gte" : 70 } } ) ;

**Operators**:

* $gt / $gte / $lt / $lte
* $ne
* $in / $nin. puede ser usado tanto con números, como cualquier tipo de dato (strings, ...)
* $mod
* $regex/$options,
* $all
* $size. Consulta si el array es del tamaño que pasemos
* $exists. Comprueba si existe un campo
* $type. Comprueba si un campo es de un tipo concreto, hay que usar el número del tipo (BSON).
* $not
* $or / $nor
* $elemMatch,
* $where (try not to use $where !) **Full collection scan**. En breve quedará obsoleto y siempre tendrá valores javascript (aunque estemos desarrollando en otro lenguage)



In order to query into subdocuments, you must use "Dot notation": ``db.collection.find( { "x.y" : 10 } );``



**Queries and arrays**

Queries that examine arrays will compare each element in the array with the argument to the matching
predicate. For example, both ::

    db.collection.find( { "x" : 2 } ) ;

and ::

    db.collection.find( { "x" : { "$gt" : 4 } } ) ;

will match documents where x is the array [ 1, 2, 5 ]. (In the first case, 2 is present in the array; in the
second case, at least one element of the array is greater than 4.)



**Subset of fields (projections)** ::

    db.scores.find( {} , { "score" : 1 } ) ; // only includes score and _id
    db.scores.find( {} , { "score" : 0 }) ; // includes everything but score



**Cursors**

To query MongoDB is to instantiate a cursor. You can also get more control using ``hasNext()``, ``next()`` and ``forEach()`` ::

    var cursor = db.scores.find() ;
    while ( cursor.hasNext() ) printjson( cursor.next() ) ;
    db.scores.find().forEach( function(x){ printjson( x ) } ) ;
    db.scores.find().forEach( printjson ) ;
    cursor.forEach // without () prints source

With ``sort()``, ``skip()``, and ``limit()`` you can control the results of the cursor

.. note:: el cursor tiene un tiempo de vida que suelen ser unos 10 minutos.


Updates
============

.. DANGER::

    Without ``$set`` operator, full document is replaced

    .. parsed-literal::

        db.stuff.update( { "_id" : 123 } , { "hello" : "world" } ) ; // Replace full document
        db.stuff.update( { "_id" : 123 } ,
                { **"$set"** : { "hello" : "world" } } ) ; // Updates hello attribute

**Operators**

* $set
* $unset
* $inc
* $push
* $pushAll
* $pull
* $pullAll
* $pop
* $addToSet
* $rename
* $bit
* $ positional operator

**Upserts and multi-update**

``upsert``: updates or creates if not exists.

By default, only one document is updated, you must use ``multi`` in order to update several documents ::

    db.people.update( { "name" : "Jones" } , { "$set" : { "age" : 50 } } , { "upsert" : true } ) ;
    db.people.update( { } , { "$set" : { "city" : "NYC" } } , { "multi" : true } ) ;

