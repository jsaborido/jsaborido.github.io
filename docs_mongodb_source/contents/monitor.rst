==================================
Monitorización y rendimiento
==================================

**mongostat**

* opcounters
* resident memory size
* page faults (cuando no lo encuentra en memoria y pasa a buscarlo a disco) Tener pocos es normal, tener muchos no es bueno, puede indicar que el working set no coje en memoria.
* Porcentaje de bloqueos y número de operaciones esperando en colas de escritura/lectura (ar/qr)
* (Mirar en la documentación de mongo online para ver una descripción de todos los campos)

**mongotop**

**db.currentOp(true)**. Me enseña las operaciones actuales.

**db.killOp(op_id)**. Para matar una operación que no está bloqueando o tardando demasiado.

**db.serverStatus()**. Estado general del servidor, información del flush (veces, tiempo que tarda de media, tiempo del último), número de conexiones, información del journal

**db.stats()**

**db.<collection>.stats()**

**Database profiling**

``db.setProfilingLevel()``. Permite establecer estadísticas de las queries que está ejecutando. Por defecto tiene level 0 (no está activado). level 1 -> queries lentas. level 2 -> todas las queries

Por defecto una query se considera lenta si tarda mas de 100ms.

``db.setProfilingLevel(2, -1)``. Con -1 para que muestre todas las queries, 0 puede dejar queries sin mostrar por los redondeos.

Esto crea una colleción llamada ``system.profile`` en la base de datos local (no se soporta sobre un sharding)

Tendrá una sobrecarga de acceso a disco, por lo que no es normal activarlo siempre, o como mucho las queries lentas. Dejando activado todo solo en temas de log
