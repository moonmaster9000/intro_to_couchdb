# Intro to CouchDB

Follow along with this tutorial to grok the basics of CouchDB. 

You'll need to install CouchDB onto your machine. A one-click installer for Mac OS X is available from http://couchbase.com

## Lesson #1: Creating a database

    $ export HOST="http://localhost:5984"
    $ curl -v -X PUT $HOST/couchdb_training_wheels

Try it again! What kind of response do you get now?

    $ curl -v -X PUT $HOST/couchdb_training_wheels

## Lesson #2: Querying all databases

    $ curl -v $HOST/_all_dbs

## Lesson #3: Delete your database

    $ curl -v -X DELETE $HOST/couchdb_training_wheels

Do it again!

    $ curl -v -X DELETE $HOST/couchdb_training_wheels

## Lesson #4: Create a document

All right, now create a database, and DONT delete it: 

    $ curl -X POST $HOST/cms

Now, let's write out a JSON document to file. Open your favorite text editor and create some JSON:

    {
      "type": "article",
      "title": "Matt's cool article",
      "label": "matts-cool-article",
      "published": false
    }

I saved this as "matts-cool-article.js"

Now, let's put it in our database:

    $ curl -v -X POST $HOST/cms -d @matts-cool-article.js

What happened? You likely got an error message telling you that it's a bad content type:

    {"error":"bad_content_type","reason":"Content-Type must be application/json"}

We can fix that. Specify the application/json content type in your request:

    $ curl -v -H "Content-Type:application/json" -X POST $HOST/cms -d @matts-cool-article.js

Notice that CouchDB returned the id of the document it created for you. You can use this ID to access the document. In my case, my database created the id "136a41ce1e3a9e6dd9c52a8b8b7d437d". So now I can look it up at that url:

    $ curl $HOST/cms/136a41ce1e3a9e6dd9c52a8b8b7d437d

I get back the following:

    {
      "_id":      "136a41ce1e3a9e6dd9c52a8b8b7d437d",
      "_rev":     "1-c8cb76e91893fffe282405a280f01460",
      "type":     "article",
      "title":    "Matt's cool article",
      "label":    "matts-cool-article",
      "published":false
    }

## Lesson #5: Update a document

Let's create another document, then update it. 

This time, we'll specify the ID ourselves. Open up the file again and change the title, throw in an author property, and add an "_id" property:

    {
      "_id": "cool-article-9000",
      "type": "article",
      "title": "Matt's cool article is really cool",
      "author": "Matt",
      "label": "matts-cool-article",
      "published": false
    }

Now PUT this into the database:

    $ curl -H "Content-Type:application/json" -X PUT $HOST/cms/cool-article-9000 -d @matts-cool-article.js 

What's the response you got? 

    {"ok":true,"id":"cool-article-9000","rev":"1-f573c35504fa0bf8985fd0c6b6baa156"}

Great! Now change the author name and attempt to put it back: 

    {
      "_id": "cool-article-9000",
      "type": "article",
      "title": "Matt's cool article is really cool",
      "author": "Matt Parker",
      "label": "matts-cool-article",
      "published": false
    }

    $ curl -H "Content-Type:application/json" -X PUT $HOST/cms/cool-article-9000 -d @matts-cool-article.js 

What response did you get?

    {"error":"conflict","reason":"Document update conflict."}

What the hell? Basically, CouchDB wants to be certain that you are updating the revision of the document you think you're updating. Add the "_rev" property into your document:

    {
      "_rev": "1-c8cb76e91893fffe282405a280f01460",
      "_id": "cool-article-9000",
      "type": "article",
      "title": "Matt's cool article is really cool",
      "author": "Matt Parker",
      "label": "matts-cool-article",
      "published": false
    }

Now try again:
    
    $ curl -X PUT $HOST/cms/136a41ce1e3a9e6dd9c52a8b8b7d437d -d @matts-cool-article.js 

Success!

    {
      "ok":true, 
      "id":"cool-article-9000", 
      "rev":"2-f573c35504fa0bf8985fd0c6b6baa156"
    }


## Lesson #6: Deleting a document

This is a simple one:

    $ curl -X DELETE $HOST/cms/cool-article-9000

Got a conflict error again? No prob: just add the rev as a query string parameter:

    $ curl -X DELETE $HOST/cms/cool-article-9000?rev=2-f573c35504fa0bf8985fd0c6b6baa156


## Lesson #7: Attachments!

Let's add an attachment to a document. First, put a document in the database:

    $ curl -v -H "Content-Type:application/json" -X PUT $HOST/cms/cool-article-9000 -d @matts-cool-article.js
      {"ok":true,"id":"cool-article-9000","rev":"1-937c7a8da4d27461d08a853b66f0d422"}

Next, write out a text file:

    $ echo "some editor notes about this article" > matts-notes.txt

Next, post it:
    
    $ curl -X PUT $HOST/cms/cool-article-9000/matts-notes.txt?rev=1-937c7a8da4d27461d08a853b66f0d422 -d @matts-notes.txt -H "Content-Type:text/plain"

Next, retrieve it:

    $ curl $HOST/cms/cool-article-9000/matts-notes.txt

Done!


## Lesson #8: Querying

Make sure this document is in your database:

    {
      "_id": "cool-article-9000",
      "type": "article",
      "title": "Matt's cool article is really cool",
      "author": "Matt",
      "label": "matts-cool-article",
      "published": false
    }

Now, let's create a special design document called "_design/article":

    {
      "_id":  "_design/article",
      "views": {
        "published_by_label": {
          "map": "function(doc){ if (doc.type == 'article' && doc.published == true){ emit(doc.label, null) }}"
        }
      }
    }

Now PUT it in your database:

    $ curl -X PUT $HOST/cms/_design/article -d @article_design.js

Now query for published articles:

    $ curl $HOST/cms/_design/article/_view/published_by_label

No results? Create a new document, and set "published" to true on it:

    {
      "type": "article",
      "title": "Matt's cool published article is really cool",
      "author": "Matt",
      "label": "matts-cool-published-article",
      "_id": "matts-cool-published-article-id",
      "published": true
    }

    $ curl -H "Content-Type:application/json" -X PUT $HOST/cms/matts-cool-published-article-id -d @matts-cool-published-article.js

Now query the view:

    $ curl $HOST/cms/_design/article/_view/published_by_label
      
      {
        "total_rows":1,
        "offset":0,
        "rows":[
          {
            "id":"matts-cool-published-article-id",
            "key":"matts-cool-published-article",
            "value":null
          }
        ]
      }

Now query the view with include_docs=true:

    $ curl $HOST/cms/_design/article/_view/published_by_label?include_docs=true

Next, update your design document to add a reduce onto it:

    {
      "_id":  "_design/article",
      "_rev":"1-10af4ef66c0fac8e71f3a6c8dd46104b",
      "views": {
        "published_by_label": {
          "map": "function(doc){ if (doc.type == 'article' && doc.published == true){ emit(doc.label, null) }}",
          "reduce": "_count"
        }
      }
    }

    $ curl -H "Content-Type:application/json" -X PUT $HOST/cms/_design/article -d @article_design.js

Now query with "reduce=true":

    $ curl $HOST/cms/_design/article/_view/published_by_label?reduce=true

      {
        "rows": [
          {
            "key":null,
            "value":1
          }
        ]
      }

Lastly, query by label:

    $ curl "$HOST/cms/_design/article/_view/published_by_label?reduce=false&key=\"some-label\""
    $ curl "$HOST/cms/_design/article/_view/published_by_label?reduce=false&key=\"matts-cool-published-article\""
