=================================
Anexo. Ejercicios / Ejemplos
=================================

**cambiar en la BD de tweeter, los campos user.location = "" o null por N/S**

Antes del cambio ::

    > db.tweets.find({$or : [{"user.location":""}, {"user.location":null}]}).count()
    10796

Update: ::

    > db.tweets.update({$or : [{"user.location":""}, {"user.location": null}]},
            {$set: {"user.location":"N/S"}}, {multi:true})
    WriteResult({ "nMatched" : 10796, "nUpserted" : 0, "nModified" : 10796 })
    > db.tweets.find({"user.location":"N/S"}).count()
    10796


**actualizar el campo "created_at" que es String a un tipo Date de BSON** ::

    > var cur = db.tweets.find({},{created_at:1})
    > cur.forEach(function(it) {db.tweets.update({_id: it._id},
            {$set:{created_at: new Date(it.created_at)}})})

**Todos aquellos usuarios que tengan dentro de entidades, menciones de usuarios. Y los 10 usuarios con mas menciones.** ::

    > db.tweets.count({"entities.user_mentions": {$not:{$size:0}}})
    27069

    > var cursor = db.tweets.find({"entities.user_mentions": 
            {$not:{$size:0}}},{"entities.user_mentions":1})
    > cursor.forEach(function(it) {db.tweets.update({_id: it._id}, 
            {$set:{"mentions_count":it.entities.user_mentions.length}})})
    > db.tweets.find({},{"user.screen_name":1,"mentions_count":1}).sort(
            {mentions_count:-1}).limit(10)

.. tip:: hay que tener en cuenta que puede haber mas usuarios que los mostrados con el mismo número de menciones que el mínimo que mostramos. En nuestro caso hay 18 usuarios con 10 menciones.


**Do a regular expression query for all users with screen names beginning with 'a'. Verify that this uses an index** ::

    > db.tweets.ensureIndex({"user.screen_name": 1})
    > db.tweets.find({"user.screen_name": /^a/},
            {"user.screen_name":1, _id: 0}).explain()

**Using the twitter collection, build a simple index on a user's friends count. Then build a separate compound index on both friends count and followers counts. Run a few queries against these fields using explain(), and notice which indexes the query optimizer uses** ::

    > db.tweets.ensureIndex({"user.friends_count":1})
    > db.tweets.ensureIndex({"user.friends_count":1, "user.followers_count":1})

    > db.tweets.find({}).sort({"user.friends_count": -1}).explain()
    > db.tweets.find({}).sort({"user.followers_count": -1}).explain() //ERROR
    > db.tweets.find({}).sort({"user.friends_count":1, 
            "user.followers_count": 1}).explain()


Framework de agregación
========================

* By screen name who tweeted the most? ::

    > db.tweets.aggregate( {$group: {_id: "$user.screen_name", count: {$sum: 1}}},
            {$sort:{count:-1}},{$limit: 1} )
    { "_id" : "behcolin", "count" : 8 }

* Who mentions the most other (unique) tweeters in their tweets? ...
* Ignoring anyone with no friends or no followers:

  1. Who has the highest Followers to Friends (follows) Ratio? ::

        > db.tweets.aggregate({$match: {"user.friends_count": {$gt: 0}}},
                {$match: {"user.followers_count": {$gt: 0}}}, 
                {$project: {"user.screen_name": 1, 
                    "user.friends_count": 1, 
                    "user.followers_count": 1, 
                    _id: 0, 
                    ratio: {$divide:["$user.followers_count",
                                     "$user.friends_count"]}
                }},
                {$sort:{ratio: -1}},
                {$limit: 1})

        { "user" : { "friends_count" : 8, "screen_name" : "Twitterrific", 
                "followers_count" : 153772 }, "ratio" : 19221.5 }

  2. Who has the lowest? ::

        > db.tweets.aggregate({$match: {"user.friends_count": {$gt: 0}}}, 
                {$match: {"user.followers_count": {$gt: 0}}},
                {$project: {
                    "user.screen_name": 1, 
                    "user.friends_count": 1, 
                    "user.followers_count": 1, 
                    _id: 0, 
                    ratio: {$divide:["$user.followers_count", 
                                     "$user.friends_count"]}
                }},
                {$sort:{ratio: 1}},
                {$limit: 1})

        { "user" : { "friends_count" : 121, "screen_name" : "whaider2009",
                "followers_count" : 1 }, "ratio" : 0.008264462809917356 }

  3. What's the average? ::

        > db.tweets.aggregate({$match: {"user.friends_count": {$gt: 0}}},
                {$match: {"user.followers_count": {$gt: 0}}}, 
                {$project: {
                    "user.screen_name": 1, 
                    "user.friends_count": 1,
                    "user.followers_count": 1,
                    _id: 0,
                    ratio: {$divide:["$user.followers_count",
                                     "$user.friends_count"]}
                    }},
                    {$group:{_id: 1, avg: {$avg: "$ratio"}}})

        { "_id" : 1, "avg" : 8.044204413696619 }

  4. Who is closest to the average? ...


* Sacar de forma ordenada decreciente la diferencia entre followers 
  y friends (mas followers que friends) <- **Asegurar un índice!!** ::

    > db.tweets.aggregate({$match: {"user.friends_count": {$gt: 0}}}, 
            {$match: {"user.followers_count": {$gt: 0}}}, 
            {$project: {
                "user.screen_name": 1, 
                "user.friends_count": 1, 
                "user.followers_count": 1, 
                _id: 0, 
                ratio: {$divide:["$user.followers_count", "$user.friends_count"]}, 
                diferencia: {$subtract: [
                    "$user.followers_count","$user.friends_count"]}
            }},
            {$sort:{ratio: -1}},
            {$limit: 10})

    { "user" : { "friends_count" : 8, "screen_name" : "Twitterrific",
            "followers_count" : 153772 }, "ratio" : 19221.5,
            "diferencia" : 153764 }
    { "user" : { "friends_count" : 2, "screen_name" : "steve_berra",
            "followers_count" : 34248 }, "ratio" : 17124,
            "diferencia" : 34246 }
    { "user" : { "friends_count" : 1, "screen_name" : "2dopeboyz",
            "followers_count" : 16847 }, "ratio" : 16847,
            "diferencia" : 16846 }
    { "user" : { "friends_count" : 9, "screen_name" : "backstreetboys",
            "followers_count" : 125048 }, "ratio" : 13894.222222222223,
            "diferencia" : 125039 }
    { "user" : { "friends_count" : 3, "screen_name" : "tedouumdado",
            "followers_count" : 39467 }, "ratio" : 13155.666666666666,
            "diferencia" : 39464 }
    { "user" : { "friends_count" : 1, "screen_name" : "3in1of2",
            "followers_count" : 10962 }, "ratio" : 10962,
            "diferencia" : 10961 }
    { "user" : { "friends_count" : 1, "screen_name" : "craigslistjobs",
            "followers_count" : 8742 }, "ratio" : 8742,
            "diferencia" : 8741 }
    { "user" : { "friends_count" : 1, "screen_name" : "LaJornada",
            "followers_count" : 8720 }, "ratio" : 8720,
            "diferencia" : 8719 }
    { "user" : { "friends_count" : 6, "screen_name" : "trampos",
            "followers_count" : 35508 }, "ratio" : 5918,
            "diferencia" : 35502 }
    { "user" : { "friends_count" : 1, "screen_name" : "StyleCareers",
            "followers_count" : 3661 }, "ratio" : 3661,
            "diferencia" : 3660 }

* Obtener una colección tipo: ::

    {_id: hashtag, tweets: [ {id: "", text: "", screen_name: ""}, ...]}

  Es decir, tag y los tweets relacionados con el tweet

  Con push: ::

    > db.tweets.aggregate( {$match: {"entities.hashtags": {$not: {$size: 0} } }},
            {$project: {
                _id: 0, 
                id:1, 
                text: 1, 
                "entities.hashtags": 1, 
                "user.screen_name":1}},
            {$unwind:"$entities.hashtags"},
            {$group:{
                _id: "$entities.hashtags.text",
                tweets: {$push: {
                            id: "$id", 
                            text: "$text", 
                            screen_name: "$user.screen_name"}}
            }}, ,
            {$out: "hashtags_tweets"})

  Con addToSet (si no existe lo crea como array, si existe como array lo añade, **si existe y no es un array da un error**): ::

    > db.tweets.aggregate( {$match: {"entities.hashtags": {$not: {$size: 0} } }}, 
            {$project: {
                _id: 0, 
                id:1, 
                text: 1, 
                "entities.hashtags": 1, 
                "user.screen_name":1}},
            {$unwind:"$entities.hashtags"},
            {$group:{
                _id: "$entities.hashtags.text",
                tweets: {$addToSet: {
                            id: "$id", 
                            text: "$text", 
                            screen_name: "$user.screen_name"}}
            }},
            {$out: "hashtags_tweets"}
        ).pretty()