===================================================
Introducción
===================================================

**Características sobre MongoDB**


* No JOINS
* Orientado a documentos (agrupación de ditintos campos)
* Existen varios tipos de datos declarados en el documento y cada atributo puede ser de distinto tipo en cada documento
* Preparado para alto rendimiento. Intenta mapear lo mas posible en memoria
* Limitaciones

  * En los sistemas de 32bit tiene una limitación de 2GB
  * En Windows < 8 tiene una limitación de 4TB
  * Linux x64 alcanza hasta 64TB

* Usan BSON (http://bsonspec.org) para almacenamiento. Formato binario de JSON.
* Escalabale horizontalmente.

  * Shard (particiones). No tienen porque estar en el mismo CPD

* Mas features que otras BD NoSQL

  * rich queries
  * geoespacial
  * text search
  * agregación
  * map reduce

--------

**Documentos**

* Un documento no puede estar fragmentado en disco
* El diseño para alto rendimiento está pensado en la forma de presentar los datos en la pantalla
* El **tamaño límite** de un documento son 16MB
* Los ficheros de metadatos (ej.: local.ns) pueden ocupar hasta 16MB, esto equivale a unos 24000 namespaces

---------

**Equivalencia de conceptos SQL**

=============== ==============
SQL             MongoDB
=============== ==============
Table, view     Collection
Row             Document
Index           =
Join            Embedded Document
FK              Reference
Partition       Shard
=============== ==============

