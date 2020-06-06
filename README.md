# Elastic Search and Kibana

## ES concepts
- [ES Concepts](https://logz.io/blog/10-elasticsearch-concepts/)
- [Elasticsearch 101](https://medium.com/velotio-perspectives/elasticsearch-101-fundamentals-core-components-a1fdc6090a5e)

## Installation
  - Download ES from [here](https://www.elastic.co/downloads/elasticsearch)
  - Unzip it and follow instruction in above link to run it
  - Open command prompt and run `curl -XGET localhost:9200`
    ```
      {
        "name" : "OE2vTTb",
        "cluster_name" : "elasticsearch",
        "cluster_uuid" : "o22N02PuQVyfafj0uba5vg",
        "version" : {
          "number" : "5.3.0",
          "build_hash" : "3adb13b",
          "build_date" : "2017-03-23T03:31:50.652Z",
          "build_snapshot" : false,
          "lucene_version" : "6.4.1"
        },
        "tagline" : "You Know, for Search"
      }
    ```
  - Download Kibana from [here](https://www.elastic.co/downloads/kibana)
  - Unzip it and follow instruction in above link to run it

## Shards
Shards help with enabling Elasticsearch to become horizontally scalable. An index can store millions of documents and occupy terabytes of data. This can cause problems with performance, scalability, and maintenance. Let’s see how Shards help achieve scalability.
Indices are divided into multiple units called Shards (Refer below diagram). Shard is a full-featured subset of an index. Shards of the same index now can reside on the same or different nodes of the cluster. Shard decides the degree of parallelism for search and indexing operations. Shards allow the cluster to grow horizontally. [source](https://medium.com/velotio-perspectives/elasticsearch-101-fundamentals-core-components-a1fdc6090a5e)

## Replicas
Hardware can fail at any time. To ensure fault tolerance and high availability ES provides a feature to replicate the data. Shards can be replicated. A shard which is being copied is called as Primary Shard. The copy of the primary shard is called a replica shard or simply a replica. Similar to the number of shards, a number of replication can also be specified at the time of index creation. Replication served two purposes
- High Availability — Replica is never been created on the same node where the primary shard is present. This ensures that even if a complete node is failed data is can be available through the replica shard.
- Performance — Replica can also contribute to search capabilities. The search queries will be executed parallelly across the replicas. [source](https://medium.com/velotio-perspectives/elasticsearch-101-fundamentals-core-components-a1fdc6090a5e)

## Upload dataset to ES
  - Download "shakespeare.json" from [here](https://www.elastic.co/guide/en/kibana/5.5/tutorial-load-dataset.html)
  - Upload the json to ES via POST call, `curl -XPOST "localhost:9200/shakespeare/_bulk?pretty" --data-binary @shakespeare.json`
  - Open Kibana at http://localhost:5601/ abd go to Dev Tools on the left navigation
  - Run, `GET _cat/indices`. You should see shakespeare and the size of the index,
    ```
      shakespeare                                    hdRg6oo1TiqvIN5-JxcLiQ 5 1 111392 0 28.7mb 28.7mb
    ```

## Delete an Index
  - `DELETE shakespeare`

## Queries

  ### Details about fields(properties), mappings, number of shards & replicas
  - `GET shakespeare` (Just extracted required fileds for demonstration)
  ```
   {
    "shakespeare": {
      "aliases": {},
      "mappings": {
        "act": {
          "properties": {
            "line_id": {
              "type": "long"
            },
            "speaker": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            }
          }
        }
      },
      "settings": {
        "index": {
          "creation_date": "1590931118645",
          "number_of_shards": "5",
          "number_of_replicas": "1",
          "uuid": "hdRg6oo1TiqvIN5-JxcLiQ",
          "version": {
            "created": "5030099"
          },
          "provided_name": "shakespeare"
        }
      }
     }
   }
  ```
  
  ### Count of all docs
  ```
  curl -XGET localhost:9200/shakespeare/_count | jq   (jq is to format the output)
  ```
  OR from Kibana
  ```
  GET shakespeare/_count
  ```
  - Output:
  ```
  {
    "count": 111392,
    "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
    }
  }
  ```

  ### Fetch all documents in the index (returns total count and 10 docs, as 10 is the default size)
  - `GET shakespeare/_search` (shorthand querying style) or 
  - Use response body querying style,
  ```
    GET shakespeare/_search
    {
      "query": {
        "match_all": {}
      }
    }
  ```
  - Output: We have a total of 111392 records
  ```
  "hits": {
    "total": 111392,
    "max_score": 1,
  ```
  
  **Using curl**: For above example (This is to show how we can make a request using `curl`)
  ```
  $ curl -XGET localhost:9200/shakespeare/_search/?pretty -d '   (press enter)
  > {
  >   "query": {
  >     "match_all": {}
  >   }
  > }'
  ```
  **NOTE**: You can use `?pretty` or at the end you can append pipe and `jq` => `}' | jq`

  ### match_all, size, from - Pagination
  - By default page size is 10. Using total results count we can query based on result number(using `from` - not page number but result number) and also the number of results (using `size`) you want per page
  ```
    GET shakespeare/_search
    {
      "query": {
        "match_all": {}
      },
      "size": 5,
      "from": 3
    }
  ```

  ### _source - select on certain fields
  - Instead of returning all fields we can use `_source` and display only selected fields (like SELECT in RDBMS)
  ```
  GET shakespeare/_search
  {
    "query": {
      "match_all": {}
    },
    "_source": ["speaker", "text_entry"],
    "size": 5
  }
  ```

  ### Term and Terms (NOTE: Always use term(s) instead of match. The results are faster as we store data by analyzing keywords)
  - "match" and "term" are slightly different the way they are stored in the (inverted index)[]
  - Suppose say the document has a sentence like "The Madison Street". ES configured it in two ways,
    - **Term Query** - Looks for exact string match String(s) in the inverted index
    - **Match Query** - String(s) are analysed before looking up in the inverted index
    - Take sentence like "The Madison Street"
    - **Terms** - "Madison" and "Street" are stored as Keywords (ignore "stop words" like "to", "I", "has", "the", "be", "or", etc. will have little context on their own). So it returns only if given term(s) are matched with any of the keywords in the inverted index
    - **Match** - sentence itself is stored directly in the inverted index (here "The Madison Street"). Results are a bit slow because of the entire string search
  ```
  GET shakespeare/_search
  {
    "query": {
      "term": {
        "speaker": {
          "value": "henry"
        }
      }
    }
  }

  GET shakespeare/_search
  {
    "query": {
      "terms": {
        "speaker": ["henry", "antony"]
      }
    }
  }
  ```

  ### range - apply range on a field(number, date, string)
  ```
  GET shakespeare/_search
  {
    "query": {
      "range": {
        "line_id": {
          "gte": 100,
          "lte": 102
        }
      }
    }
  }
  ```

  ### filter - we always should filter the results for performance reasons
  - So let's say we want to get all speakers with name "henry" but should have line_id greater than 100 and less than 110
  - First filter by condition and then get the speakers => "bool" operator (you will read more about "bool" api later)
  ```
  GET shakespeare/_search
  {
    "query": {
      "bool": {
        "filter": {
          "range": {
            "line_id": {
              "gte": 100,
              "lte": 120
            }
          }
        },
        "must": [
          {
            "term": {
              "speaker": {
                "value": "henry"
              }
            }
          }
        ]
      }
    }
  }
  ```
  - You can also have multiple filters (say all speakers with name "henry" and line_id in range (100, 120) and speaker name should have "prince")
  - Not a good example but this is just a demo
  ```
  GET shakespeare/_search
  {
    "query": {
      "bool": {
        "filter": [
          {
            "range": {
              "line_id": {
                "gte": 100,
                "lte": 120
              }
            }
          },
          {
            "bool": {
              "must": [
                {
                  "term": {
                    "speaker": "prince"
                  }
                }
              ]
            }
          }
        ],
        "must": [
          {
            "term": {
              "speaker": {
                "value": "henry"
              }
            }
          }
        ]
      }
    }
  }
  ```

  ### match - return docs that has the given query-string
  - Search for `Antony` in `play_name`
  - Shorthand: `GET shakespeare/_search?q=play_name:"Antony"`
  - Request body:
    ```
    GET shakespeare/_search
    {
      "query": {
        "match": {
          "play_name": "Antony"
        }
      },
      "_source": ["play_name", "speaker", "text_entry"]
    }
    ```

  ### match_phrase - return docs that matches exact query-string in a field
  ```
  GET shakespeare/_search
  {
    "query": {
      "match_phrase": {
        "text_entry": "Did lately meet in the intestine shock"
      }
    },
    "_source": ["play_name", "speaker", "text_entry"]
  }
  ```

  ### multi_match - return docs that matches query is in any of these fields
  ```
  GET shakespeare/_search
  {
    "query": {
      "multi_match": {
        "query": "antony",
        "fields": ["text_entry", "play_name", "speaker"]
      }
    },
    "_source": ["play_name", "speaker", "text_entry"],
    "size": 1
  }
  ```

  ### match_phrase_prefix - return docs that exactly match has the prefix
  ```
  GET shakespeare/_search
  {
    "query": {
      "match_phrase_prefix": {
        "text_entry": "Even or odd"
      }
    }
  }
  ```

  ### bool - return docs using operators AND/OR/NOT
    #### must - like AND, return docs that contains the query-string in given fields
    - Results count `12` but when compared with above `multi-match` (which is *OR*) it is `4280`
    ```
    GET shakespeare/_search
    {
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "play_name": "antony"
              }
            },
            {
              "match": {
                "text_entry": "antony"
              }
            },
            {
              "match": {
                "speaker": "antony"
              }
            }
          ]
        }
      },
      "_source": ["play_name", "speaker", "text_entry"]
    }
    ```

    #### must_not - like NOT, return docs that must not contain the given query string in given field
    - Must not have given query-string in given field
    ```
    GET shakespeare/_search
    {
      "query": {
        "bool": {
          "must_not": [
            {
              "match": {
                "play_name": "Henry"
              }
            }
          ]
        }
      },
      "_source": ["play_name", "speaker", "text_entry"]
    }
    ```  

    #### should - like OR, return docs that match either of the given conditions
    - Should match either of the given conditions. Also we can mention how many conditions minimum should match using `minimum_number_should_match`
    ```
    GET shakespeare/_search
    {
      "query": {
        "bool": {
          "minimum_number_should_match": 1,
          "should": [
            {
              "match_phrase": {
                "text_entry": "Exeunt, bearing MARK ANTONY"
              }
            },
            {
              "match": {
                "speaker": "antony"
              }
            }
          ]
        }
      },
      "_source": ["play_name", "speaker", "text_entry"]
    }
    ```

  ### query_string - AND/OR/NOT can be done on multiple fields in single line
  ```
  GET shakespeare/_search
  {
    "query": {
      "query_string": {
        "fields": ["play_name", "speech_number"],
        "query": "((Romeo and Juliet) | (Henry IV)) AND (7)"
      }
    }
  }
  ``` 

  ### wildcard - You can specify pattern. '*' for all characters and '?' for one character
  - This is useful when we want docs but we know only a partial string (say middle is missing)
  - NOTE: Use all lower characters in the input else it won't work
  ```
  GET shakespeare/_search
  {
    "query": {
      "wildcard": {
        "text_entry": {
          "value": "*scar*let*"
        }
      }
    }
  }
  ```

  ### regexp - We can specify more patterns using Regular Expressions
  - NOTE: Use all lower characters in the input else it won't work
  ```
  GET shakespeare/_search
  {
    "query": {
      "regexp": {
        "speaker": "nu[a-z]se"
      }
    }
  }
  ```





## Reference
- https://www.elastic.co/blog/a-practical-introduction-to-elasticsearch
- https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl.html
- https://dzone.com/articles/23-useful-elasticsearch-example-queries



