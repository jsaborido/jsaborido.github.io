===================================================
Despliegues tipo
===================================================

Standalone
===================================================

.. graphviz::

    digraph Standalone {
        rankdir=LR
        node[shape=box]
        "Cliente" -> "Primario";
    }
      



Replica Set
===================================================

* Máximo 1 primario
* Máximo 11 secundarios
* Escrituras solo en primario
* Se pueden hacer lecturas conectándose a un secundario

 .. image:: /images/replica-set-read-write-operations-primary.png


Sharded cluster
===================================================

A través del config server se sabe a que replicaSet (shard) se puede escribir/leer
Config servers posibles:

* 1 - Usado para pruebas
* 3 - Usado para producción. Provee tolerancia a fallos

.. image:: /images/sharded-cluster.png


