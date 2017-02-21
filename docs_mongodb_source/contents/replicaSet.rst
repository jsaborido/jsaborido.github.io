===================================================
Replica Set
===================================================

**Sistema de votos para la elección de primarios**:

* Para que un nodo pase a primario tiene que obtener la mayoría de votos (mayor que la mitad)
* Mínimo 3 nodos para poder tener recuperación automática. Sino no se puede decidir un nuevo primario.
* **Árbitro**: es un nodo secundario sin almacenamiento y solo usado para las votaciones.
* Un secundario puede tener **derecho de veto**. Cuando un secundatio tiene una fecha de actualización mayor.
* Máximo número de nodos que pueden votar: 7

  * Un nodo que no puede votar, si puede poner 1 veto.

* Los secundarios tienen prioridad, por defecto = 1.


**Oplog.rs**

En los nodos del replica set, existirá la colección ``oplog.rs`` dentro de la base de datos ``local``.
Esta colección se crea con las escrituras en el primario y se envía a los secundarios.

Es una colección de tamaño máximo donde se almacenan las operaciones de una forma idempotente.


**Lecturas en secundarios**

Hacer lecturas en los secundarios puede ser una buena idea si estamos haciendo operaciones pesadas y no nos importa que los datos no sean justo los últimos.

Al conectarse a un secundario con el comando "mongo", no nos permite lecturas por defecto, para habilitarlas debemos ejecutar::

    rs.slaveOk()

**Mantenimientos y upgrades**.

Para mantenimientos de los servidores y actualizaciones del software, primero se deberían hacer en los secundarios.


Write concerns
==========================

* **w:0**. Mando y espero solo el ack de la red
* **w:1**. *Valor por defecto* en standalone y primarios de replica set. Cuando la escritura ha sido hecha en la memoria del servidor, aunque no en disco ni replicada.
* **w:n (siendo n>1)**. Cuando se ha confirmado en n nodos (primario y n-1 secundarios).
* **w:"majority"**. Cuando se ha confirmado en la mayoría de los miembros del replica set.
* **w:<tag set>**. Cuando se ha confirmado en los nodos configurados con el tag dado.
* **j:true**. Espera a la escritura en el journal para devolver la confirmación.
* **wtimeout:<ms>**. Timeout en milisegundos que se espera para confirmación de secundarios. Solo tiene sentido con w>1.


Read Preference modes
==========================

* **primary**. Valor por defecto. Siempre se lee en el primario
* **primaryPrefered**. Siempre en el primario a no ser que no esté disponible, en este caso se permiten secundarios.
* **secondary**. Solo se lee de secundarios.
* **secondaryPrefered**. Se lee de secundarios a no ser que ninguno esté disponible.
* **nearest**. Lee del mas cercano (nodo con menos latencia en la red) sea primario o secundario.


Datacenters
==========================

Como ya se ha dicho, para auto-recuperación de un replica set necesitamos al menos 3 nodos (si hay solo 2 y se cae el primario, el secundario no tiene votos suficientes para pasar a primario). Repartiendo los nodos en datacenters:

.. graphviz:: 

    graph ejemplos {

        subgraph clusterSingleDataCenter {
            label="1 Datacenter";

            subgraph clusterA {
                label="Datacenter";
                "1DC Node 1" -- "1DC Node 2"
                "1DC Node 1" -- "1DC Node 3"
                "1DC Node 2" -- "1DC Node 3"
            }
        }

        subgraph clusterDosDataCenter {
            label="2 Datacenter";

            subgraph clusterA {
                label="Datacenter 1";
                "2DC Node 1"
                "2DC Node 2"
            }

            subgraph clusterB {
                label="Datacenter 2";
                "2DC Node 3"
            }

            "2DC Node 1" -- "2DC Node 2"
            "2DC Node 1" -- "2DC Node 3"
            "2DC Node 2" -- "2DC Node 3"
        }

        subgraph clusterTresDataCenter {
            label="3 Datacenter";

            subgraph clusterA {
                label="Datacenter 1";
                "3DC Node 1"
            }

            subgraph clusterB {
                label="Datacenter 2";
                "3DC Node 2"
            }

            subgraph clusterC {
                label="Datacenter 3";
                "3DC Node 3"
            }

            "3DC Node 1" -- "3DC Node 2"
            "3DC Node 1" -- "3DC Node 3"
            "3DC Node 2" -- "3DC Node 3"
        }
    }

Podemos ver que la única configuración para poder recuperarse automáticamente ante la caída/destrucción de un data center (DR = Disaster Recovery) es usar como mínimo 3 datacenters.

Si usamos 1 y se pierde, perdemos todo.

Si usamos 2, podríamos recuperarnos en el caso de que el datacenter que se pierda sea el que contiene 1 nodo, en otro caso, no.


Configuración.
==========================

**Configurando un replica set añadiendo secundarios**:

* Se levantan los servidores con el nombre del replica set

  * ``$ mongod ... --replSet set0``. Si se configura desde linea de comandos. "set0" es el nombre del replica set
  * ``replSet=set0``. Si se configura en fichero de configuración con formato antiguo
  * ``replication.replSetName: set0``. Si se configura en el fichero de configuración con formato YAML

* Nos conectamos al nodo que será el primario

  * Con ``rs.status()`` podemos comprobar que no está inicializado el replica set
  * Con ``rs.conf()`` también veremos datos de la configuración del replica set

* Con ``rs.initiate()`` iniciamos el replica set que solo tendrá este nodo actualmente
  y el shell nos debería cambiar a ``set0:PRIMARY>``.

* Para añadir otros nodos ``rs.add("<hostname>:<port>")``. Ejemplo: ``rs.add("precise32:28002")``.

  * También es posible pasar un documento en vez de un string con la configuración del nodo.
  * Los secundarios deben ser añadidos desde el primario


**Configurando un replica set pasando la configuración de todos los nodos:**

Para ello, en vez de inicializar con una configuración vacía, pasamos al comando "initiate" la configuración::

    config = { "_id": "set0", "members" : [
        {"_id" : 0, "host" : "localhost:27017" },
        {"_id" : 1, "host" : "localhost:27018" },
        {"_id" : 2, "host" : "localhost:27019" } ]
    }
    
    rs.initiate(config)

Valores posibles de configuración: ::

    {
      _id: <string>,
      version: <int>,
      members: [
        {
          _id: <int>,
          host: <string>,
          arbiterOnly: <boolean>,
          buildIndexes: <boolean>,
          hidden: <boolean>,
          priority: <number>,
          tags: <document>,
          slaveDelay: <int>,
          votes: <number>
        },
        ...
      ],
      settings: {
        getLastErrorDefaults : <document>,
        chainingAllowed : <boolean>,
        getLastErrorModes : <document>,
        heartbeatTimeoutSecs: <int>
      }
    }

**Reconfigurar un replica set**: ::

    rs.reconfig(<config>)


Configuración rápida.
==========================

Existe una forma de crear un replica set de forma rápida para pruebas y desarrollo, para ello necesitamos el directorio ``/data/db`` ::

  # mkdir -p /data/db
  # chown -R student:student /data
  $ mongo --nodb
  > var rsTest = new ReplSetTest({name: "replicaTest", nodes: 3})
  > rsTest.startSet() && rsTest.initiate()

.. note:: Una vez salgamos de la consola el réplica set se finaliza

Conectarse desde consola: ::

  $ mongo --nodb
  > conn = new Mongo("localhost:31000") // Conectamos al primario
  > db = conn.getDB("test")
  // Para conectarse al secundario
  replicaTest:PRIMARY> conn = new Mongo("localhost:31001")
  replicaTest:PRIMARY> db = conn.getDB("test")

  // Volvemos al primario para insertar datos
  > replicaTest:PRIMARY> use test // Esta puede no ser necesaria

