===================
Desarrollo. Driver
===================

Podemos descarganos el driver según el lenguage necesario en: http://docs.mongodb.org/ecosystem/contents

MongoClient -> Mongo (driver)

La app se comunicará con ayuda del driver al mongod (primario o secundario) o al mongoS.

MongoClient nos ayuda ya con problemas de conexion (pool) o concurrencia. Nos abre un pool de conexiones.

El objeto ``DBObject`` es usado para manejar JSON, pero es abstracto. ``BasicDBObject`` será la instancia normal a usar.


**Ejemplos trabajando con groovy**

.. code-block:: groovy

    import com.mongodb.*;
    import com.mongodb.util.*;

    class TestMongoGroovyClient {

        static void main(def args) {
            MongoClient client = new MongoClient("localhost", 27017)

            println client.getDatabaseNames() //show dbs

            DB twitterDB = client.getDB("tweeter")

            println twitterDB.getCollectionNames() // show collections

            DBCollection tweets = twitterDB.getCollection("tweets")

            BasicDBObject firstTweet = tweets.findOne()

            // Buscar los usuarios que son de Santiago de compostela
            DBObject query = new BasicDBObject(
                    "user.location", "Santiago de Compostela")

            tweets.find(query).each { println it.text }

            // Todos aquellos usuarios que tienen mas de 100 followers
            def query2 = new BasicDBObject("user.followers_count", 
                    new BasicDBObject('$gt', 100))
            println tweets.count(query2)

            def query3 = query2.append("user.location", "London")
            println tweets.count(query3)

            String queryStr = "{'user.location': 'London', 
                    'user.followers_count': {\$gt: 100}}"
            BasicDBObject json = (BasicDBObject) JSON.parse(queryStr)
            println tweets.count(json)
            
            QueryBuilder query4 = QueryBuilder.start(
                    "user.followers_count").greaterThan(100)
            println tweets.count(query4.get())
            
            //proyecciones
            tweets.find(query4.get(), new BasicDBObject("text", 1)).limit(10).each {
                println it
            }

            // BasicDBList para manejar arrays

        }
    }

.. code-block:: groovy

    import java.nio.channels.ReadPendingException;

    import com.mongodb.*;
    import com.mongodb.util.*;

    class TestWithReplicaSet {

        static void main(def args) {
            // No hay que indicar cual es el primario ni nada
            MongoClient client = new MongoClient([
                new ServerAddress("localhost", 31000),
                new ServerAddress("localhost", 31001),
                new ServerAddress("localhost", 31002),
                ])

            println client.getDatabaseNames()
            
            DB db = client.getDB 'test'
            DBCollection users = db.getCollection 'user'
            
            users.find().each { println it }
            
            // El insert básico no garantiza que se lanza una excepción si falla
            // hay que modificar el write concern
            //users.insert(new BasicDBObject("username", "juan"))
            
            
            /*
             * Para modificar las preferencias de lectura
             * ReadPreference.primary() // Solo de primario
             * ReadPreference.primaryPreferred() // Primario preferido, 
             *                                      pero podemos leer de secundarios
             * ReadPreference.secondary() // Solo de secundarios
             * ReadPreference.secondaryPreferred() // Secundario preferido
             * 
             * Por defecto es primaryPreferred
             */
            
            //Si el primario se cae no quiero leer de ninguno
            users.setReadPreference ReadPreference.primary()
            
            /*
             * El writeConcern se puede hacer a nivel de cliente, 
             * collección, insert, update...
             */
            // Devuelve el acuse de recibo cuando replique en la mayoría
            users.setWriteConcern(WriteConcern.MAJORITY)
            // Definir a cuantos hay que escribir. W=0 (0 es el valor por defecto)
            users.setWriteConcern(WriteConcern.NORMAL) //W=0
            //W=1 (inserción en el primario pero sin llegar al journal)
            //WriteConcern.SAFE
            //WriteConcern.ACKNOWLEDGED //también W=1
            
            // Si queremos personalizar el WriteConcern
            try {
                // 4 nodos, 2000ms timeout, <?>, esperamos al journal = true (W=4|j=1)
                users.insert(new BasicDBObject("username", "juan"), 
                        new WriteConcern(4, 2000, true, true))
            } catch (WriteConcernException e) {
                e.printStackTrace()
            }
            
            try {
                users.insert(new BasicDBObject("username", "juan"), 
                        new WriteConcern(4, 2000, true, true))
            } catch (e) {
                e.printStackTrace()
            }
            
        }
    }

