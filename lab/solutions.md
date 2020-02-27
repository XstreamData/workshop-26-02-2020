# Xstream Data - Workshop Elasticsearch



####  <span style="color:green"> SOLUZIONE ROUND1</span> - Indicizzazione Contenuti 



- Creare la configurazione Logstash **xstream.conf** sotto la directory config/
- La pipeline potrebbe essere così fatta:


```
input {
  file {
    path => "C:/Users/nicol/Documents/elasticsearch/stack7.5/logstash-7.5.1/data/xstream/*"
    start_position => "beginning"
    codec => multiline {
      pattern => "^BEGIN:VEVENT"
      negate => true
      what => previous
    }
  }
}

filter {
  mutate { 
    gsub => [ "message", "CN=", "CN:" ]
    gsub => [ "message", ":invalid:nomail", "" ]
  }
  kv {
    field_split => "\r\n"
    value_split => ":"
  }
  if [BEGIN] == "VCALENDAR" {
    drop { }
  }
  mutate {
    rename => ["ATTENDEE;ROLE=REQ-PARTICIPANT;CUTYPE=INDIVIDUAL;CN", "SPEAKERS" ]
  }
}

output {
  stdout { }
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "xstream-fosdem"
  }
}

```

- Eseguire la pipeline con il seguente comando:


  ```
bin\logstash -f config\xstream.conf
  ```

- Accedere sulla proprio istanza Kibana locale **:5601**

- Lanciare una **GET _cat/indices** per vedere la lista degli indici creati

- Verificare i contenuti indicizzati con **GET xstream-fosdem/_search**



-----------



####   <span style="color:green"> SOLUZIONE ROUND2</span> - Modellizzazione Indice Elasticsearch



- Verificare il mapping dell'indice con **GET xstream-fosdem/_mapping**
- Di seguito il mapping ottimizzato:
```
PUT xstream-fosdem-2
{
  "settings": {
    "analysis": {
      "analyzer": {
        "html_analyzer": {
          "char_filter": [
            "my_char_filter"
          ],
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      },
      "char_filter": {
        "my_char_filter": {
          "type": "html_strip"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "@version": {
        "type": "text",
        "index": false
      },
      "BEGIN": {
        "type": "text",
        "index": false
      },
      "CALSCALE": {
        "type": "text",
        "index": false
      },
      "CATEGORIES": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "CLASS": {
        "type": "text",
        "index": false
      },
      "DESCRIPTION": {
        "type": "text",
        "analyzer": "html_analyzer"
      },
      "DTEND": {
        "type": "date",
        "format": "yyyyMMdd'T'HHmmss"
      },
      "DTSTART": {
        "type": "date",
        "format": "yyyyMMdd'T'HHmmss"
      },
      "END": {
        "type": "text",
        "index": false
      },
      "LOCATION": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "METHOD": {
        "type": "text",
        "index": false
      },
      "PRODID": {
        "type": "text",
        "index": false
      },
      "SPEAKERS": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "STATUS": {
        "type": "keyword"
      },
      "SUMMARY": {
        "type": "text"
      },
      "TZID": {
        "type": "text",
        "index": false
      },
      "UID": {
        "type": "text",
        "index": false
      },
      "URL": {
        "type": "text",
        "index": false
      },
      "VERSION": {
        "type": "text",
        "index": false
      },
      "host": {
        "type": "text",
        "index": false
      },
      "message": {
        "type": "text",
        "index": false
      },
      "path": {
        "type": "text",
        "index": false
      },
      "tags": {
        "type": "text",
        "index": false
      }
    }
  }
}
```



- Per re-indicizzare tutti i documenti:

```
POST _reindex
{
  "source": {
    "index": "xstream-fosdem"
  },
  "dest": {
    "index": "xstream-fosdem-2"
  }
}
```



- Verificare una piccola parte delle ottimizzazioni lanciando la seguente queryDSL:

```
GET xstream-fosdem-2/_search
{
  "query": {
    "range": {
      "DTEND": {
        "gte": "20200202T180000"
      }
    }
  }
}
```

#### Quanti talks vi vengono restituiti ? <span style="color:green"> 1!</span>



-----------



####   <span style="color:green"> SOLUZIONE ROUND3</span> - Ricerca di specifici talks



**Query Livello Base**

- Recuperare tutti i talks in cui si parla del linguaggio di programmazione GO.

```
GET xstream-fosdem-2/_search
{
  "query": {
    "match": {
      "CATEGORIES": "go"
    }
  }  
}

```

- Recuperare *solo* il *SUMMARY* dei talks tenuti da *Schimanski*.

```
GET xstream-fosdem/_search
{
  "_source": "SUMMARY", 
  "query": {
    "match": {
      "SPEAKERS": "Schimanski"
    }
  }  
}
```




- Recuperare *solo* il numero dei talks che iniziano dopo le ore 15:30 del 02/02/2020
```
GET xstream-fosdem-2/_search
{
  "size": 0, 
  "query": {
    "range": {
      "DTSTART": {
        "gte": "20200202T153000"
      }
    }
  }
}
```

**Query Livello Avanzato**

- Recuperare i talks che iniziano dalle ore 9:00 alle ore 10:00 del 01/02/2020 e che sia di tipo *Keynotes*.

```
GET xstream-fosdem-2/_search
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "DTSTART": {
            "gte": "20200201T090000",
            "lte": "20200201T100000"
          }
        }
      }, 
      "must": [
        {
          "term": {
            "CATEGORIES.keyword": "Keynotes"
          }
        }
      ]
    }
  }
}
```

- L'utente vuole cercare tutti i talks in cui si parla di LINUX, ma per errore digita sulla barra di ricerca *LINUS*. Implementare una ricerca che permetta la ricerca di questi talks nonostante questo errore.

```
GET xstream-fosdem-2/_search
{
  "query": {
    "match": {
      "DESCRIPTION": {
        "query": "LINUS",
        "fuzziness": 1
      }
    }
  }
}
```
- Recuperare tutti i talks in cui si parla di LINUX. Mettere però in risalto quelli che trattano l'argomento *SECURITY*


```
GET xstream-fosdem-2/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "DESCRIPTION":  {
              "query": "Linux"
            }
          }
        }
      ],
      "should": [
        {
          "match": {
            "CATEGORIES": "security"
          }
        }
      ]
    }
  }
}
```





----------------



####   <span style="color:green">ROUND4</span> - Analisi dei talks

> 

**Aggs Livello Base**

- Trovare lo speaker che ha tenuto il maggiore numero di talks

```
GET xstream-fosdem-2/_search
{
  "size": 0, 
  "aggs": {
    "speaker_aggs": {
      "terms": {
        "field": "SPEAKERS.keyword",
        "size": 1
      }
    }
  }
}
```

- Trovare i TOP 3 speakers che hanno tenuto il maggiore numero di talks il giorno 02/02/2020

```
GET xstream-fosdem-2/_search
{
  "size": 0, 
  "query": {
    "range": {
      "DTSTART": {
        "gte": "20200202T080000",
        "lte": "20200202T230000"
      }
    }
  }, 
  "aggs": {
    "speaker_aggs": {
      "terms": {
        "field": "SPEAKERS.keyword",
        "size": 3
      }
    }
  }
}
```
- Capire quale è stato il giorno con il maggior numero di talks. 
```
GET xstream-fosdem-2/_search
{
  "size": 0, 
  "aggs": {
    "count_talks_aggs": {
      "date_histogram": {
        "field": "DTSTART",
        "interval": "day"
      }
    }
  }
}
```
**Aggs Livello Avanzato**

- Si vogliono cercare, per fascia oraria, gli speaker che tengono talks relativi  a *DOCKER*.
  - Sono significative solo le fasce orarie con almeno 1 talk su questo argomento

```
GET xstream-fosdem-2/_search
{
  "size": 0,
  "query": {
    "match": {
      "DESCRIPTION": "docker"
    }
  }, 
  "aggs": {
    "NAME": {
      "date_histogram": {
        "field": "DTSTART",
        "interval": "hour",
        "min_doc_count": 1
      },
      "aggs": {
        "NAME": {
          "terms": {
            "field": "SPEAKERS.keyword",
            "size": 10
          }
        }
      }
    }
  }
}
```

------------------



####   <span style="color:green">FINAL ROUND</span> - Kibana!

- Ecco una dashboard di esempio basata sul dataset Fosdem

![dashboard xstream](C:\Users\nicol\Documents\meetup-workshop\xstreamdata_26022020\lab\dashboard xstream.png)