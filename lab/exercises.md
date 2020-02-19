# Xstream Data - Workshop Elasticsearch



####  <span style="color:red"> ROUND1</span> - Indicizzazione Contenuti 

> Il primo task prevede l'indicizzazione di contenuti su Elasticsearch. A tale scopo è stato fornito un file .ics in cui sono definiti una serie di eventi (talks fatti al FOSDEM 2020).

###### Ad esempio:


```
BEGIN:VEVENT
METHOD:PUBLISH
UID:9246@FOSDEM20@fosdem.org
TZID:Europe-Brussels
DTSTART:20200202T090000
DTEND:20200202T094000
SUMMARY:Continuous Delivery starts with Continuous Infrastructure
DESCRIPTION: <p>Most organisations start their journey towards Continuous Delivery with their development teams, or often their web or mobile teams. I’ve seen many of these journeys fail because “ops” was not included in the picture.  The organisation assumed DevOps didn’t need ops. So the team didn’t adapt, didn’t provide the right stacks, couldn’t support the tools. I’ve started a number of successful journeys with the ops teams doing Continuous Delivery of their infrastructure as code. They changed their mindset, allowing them to understand, support and onboard the development teams.  This talk will document that approach with some supporting cases and examples.</p><p>Taking one step further we'll showcase a on how to do Continuous Delivery of your Infrastructure as Code,    obviously with Open Source tools</p>
CLASS:PUBLIC
STATUS:CONFIRMED
CATEGORIES:Continuous Integration and Continuous Deployment
URL:https:/fosdem.org/2020/schedule/event/continuous_delivery_starts_with_continuous_infrastructure/
LOCATION:UB4.136
ATTENDEE;ROLE=REQ-PARTICIPANT;CUTYPE=INDIVIDUAL;CN="Kris Buytaert":invalid:nomail
END:VEVENT
```

###### Per indicizzare questo tipo di contenuti utilizziamo Logstash.



 Il primo obiettivo è quindi quello di :

 - Creare una pipeline <input, filter, output>, che riesce a leggere in modo corretto gli eventi. Si precisa che ogni evento dovrà essere indicizzato come document.
 - **Base**: visualizzare su *stdout* gli eventi ben strutturati
 - **Avanzato**: Indicizzare gli eventi strutturati in Elasticsearch. 
    - Fare riferimento al proprio Elasticsearch locale
    - Su indice: **xstream-fosdem**
   - Verificare l'indicizzazione dall'istanza locale Kibana - *dev tools*



##### TIPS

- I dati risiedono su di un file. E' necessario utilizzare l'opportuno plugin in Input.

- Ogni evento è rappresentato su più righe del file. 

  Di default, Logstash gestisce ogni riga come singolo evento. Verificare come utilizzare il *Multiline*.

- Gli eventi sono definiti come KEY=VALUE. 

  Utilizzare il corretto Filter per strutturare l'evento come oggetto JSON <key,value>

- Il *CN* rappresenta lo speaker del talk. Un talk potrebbe avere 1+ speaker.



-----------



####   <span style="color:red"> ROUND2</span> - Modellizzazione Indice Elasticsearch

> Logstash è stato fondamentale per indicizzare i nostri eventi in Elasticsearch.                                    Purtroppo, non sempre l'*auto index creation* di Elasticsearch ci permette di costruire un'indice ottimale.



Il prossimo task, prevede l'ottimizzazione dell'indice elasticsearch creato in automatico.

**Steps**

- Verificare il mapping dell'indice - *utilizzare Kibana*
- Per ciascun field testuale, decidere quando utilizzare solo il type *text*, solo il type *keyword*, oppure entrambi.
- Verificare se tutti i fields presentano il datatype corretto - *probabilmente no*

Individuate le modifiche da fare all'indice:

- Creare un secondo indice **xstream-fosdem-2**
- Copiare tutti i dati dal primo indice a questo nuovo - *Si tratta di una operazione di re-indicizzazione*
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

#### Quanti talks vi vengono restituiti ? 

> Dovrebbe esserci solo un talk che soddisfa questo criterio. Viceversa, rivedere l'ottimizzazione del field *DTEND*.



-----------



####   <span style="color:red"> ROUND3</span> - Ricerca di specifici talks

> E' il momento di fare delle SEARCH! Elasticsearch utilizza il framework delle QueryDSL. 



In questo round, si chiede di implementare alcune search che potrebbe voler fare un partecipante al FOSDEM per trovare i talks di suo maggiore interesse.



**Query Livello Base**

- Recuperare tutti i talks in cui si parla del linguaggio di programmazione GO.
  - **TIPS**: potreste fare riferimento alla categoria del talk...
- Recuperare *solo* il *SUMMARY* dei talks tenuti da *Schimanski*.
- Recuperare *solo* il numero dei talks che iniziano dopo le ore 15:30 del 02/02/2020
  - **TIPS**: Attenzione al formato...

**Query Livello Avanzato**

- Recuperare i talks che iniziano dalle ore 9:00 alle ore 10:00 del 01/02/2020 e che sia di tipo *Keynotes*.
- L'utente vuole cercare tutti i talks in cui si parla di LINUX, ma per errore digita sulla barra di ricerca *LINUS*. Implementare una ricerca che permetta la ricerca di questi talks nonostante questo errore.
- Recuperare tutti i talks in cui si parla di LINUX. Mettere però in risalto quelli che trattano l'argomento *SECURITY*



----------------


####   <span style="color:red">ROUND4</span> - Analisi dei talks

> Iniziamo a lavorare con il framework delle Aggregations!



In questo round si chiede di creare delle aggregations su cui si potrebbero basare filtri di ricerca e visualizzazioni in Kibana.



**Aggs Livello Base**

- Trovare lo speaker che ha tenuto il maggiore numero di talks
- Trovare i TOP 3 speakers che hanno tenuto il maggiore numero di talks il giorno 02/02/2020
- Capire quale è stato il giorno con il maggior numero di talks. 

**Aggs Livello Avanzato**

- Si vogliono cercare, per fascia oraria, gli speaker che tengono talks relativi  a *DOCKER*.
  - Sono significative solo le fasce orarie con almeno 1 talk su questo argomento



------------------

####   <span style="color:red">ROUND5</span> - Kibana!

> Ultimo Round. Creiamo una dashboard!



**Tasks**

- Creare l'index-pattern riferito all'indice ottimizzato

- Creare *5* visualizations a piacere
- Creare una Dashboard che le contiene