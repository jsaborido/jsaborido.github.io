==================================================
Desarrollo. Introducción.
==================================================

Tanto el diseño como a la hora de trabajar, mongodb es muy dependiente de la aplicación.

Si no se pone "_id" en el insert se genera ObjectId que está compuesto de: timestamp+idMachine+pid+sequencia => byte

JSON admite de tipos de datos:

* double
* string
* array
* sub-docs
* boolean
* null

Para tener otros tipos de datos usamos BSON, que es mucho mas rico.

Todo lo que hagamos desde la consola, aunque sea JSON se serealiza en BSON (http://bsonspec.org)

Como "_id" también se puede usar un documento: ::

    {_id: {state: " ", city: " "}}

Hay que tener en cuenta que en estos casos influyen campos, orden y valor como clave primaria ya que lo que se tienen en cuenta como unicidad es el hash.

Esto está permitido, siendo 2 documentos distintos: ::

    > db.lines.insert({_id: {number: 01, label:"002"} , origen:"or2", destino:"destino2"})
    > db.lines.insert({_id: {label:"002", number:01} , origen:"or2", destino:"destino2"})

Se pueden insertar arrays directamente que es mas eficiente que insertar uno por uno ::

    > var array = [
    ... {number:1, label:"001"},
    ... {number:2, label:"002"}
    ... ]
    > db.lines.insert(array)
    BulkWriteResult({
    "writeErrors" : [ ],
    "writeConcernErrors" : [ ],
    "nInserted" : 2,
    "nUpserted" : 0,
    "nMatched" : 0,
    "nModified" : 0,
    "nRemoved" : 0,
    "upserted" : [ ]
    })

En una lista, si hay un error en las operaciones, como la transaccionalidad es a nivel de documento, pueden quedar una parte de ellos insertada y la otra no.
