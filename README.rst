###########
ETL mit EAI
###########

======
Angabe
======

"The ETL (Extract, Transform, Load) is a mechanism for loading data into systems or databases using some kind of Data Format from a variety of sources; often files then using Pipes and Filters, Message Translator and possible other Enterprise Integration Patterns.
So you could query data from various Camel Components such as File, HTTP or JPA, perform multiple patterns such as Splitter or Message Translator then send the messages to some other Component.
To show how this all fits together, try the ETL Example." [1]

ETL ist ein wichtiger Prozess bei einem Datawarehouse. Zeigen Sie wie Enterprise Integration Patterns [2] dabei eingesetzt werden können (8 Punkte, nur jene, die in dem Beispiel vorkommen). 
Verwenden Sie dazu das ETL Example [3]. 
Dokumentieren Sie die Implementierung sowie alle notwendigen Schritte ausführlich in einem Protokoll (8 Punkte). 

Fügen Sie den verwendeten Code nach den Metaregeln an und geben Sie alles als ZIP-Archiv (Gesamtes Framework mit Anleitung, wie das System gestartet werden kann) ab.

============
Installation
============

Zuerst wird Maven installiert:

.. code:: bash

    sudo apt-get update && sudo apt-get install maven

Nun in den Download Ordner wechseln, Camel herunterladen und entpacken:

.. code:: bash

    cd Downloads/
    wget ftp://gd.tuwien.ac.at/pub/infosys/servers/http/apache/dist/camel/apache-camel/2.12.3/apache-camel-2.12.3.tar.gz
    tar xfvz apache-camel-2.12.3.tar.gz

Die Installation ist damit abgeschlossen.

==============
Implementation
==============

**Starten**

Wir navigieren in das examples/camel-example-etl Verzeichnis:

.. code:: bash

    cd ~/Downloads/apache-camel-2.12.3/examples/camel-example-etl

Nun kann das Beispiel direkt ausgeführt werden:

.. code:: bash

    mvn camel:run

Beim ersten Ausführen wird noch eine Fehlernachricht ausgegeben, da die Daten noch nicht in der Datenbank vorhanden sind:

.. code:: plain

    ...
    thread #0 - file://src/data] CustomerTransformer            INFO  Created a new CustomerEntity Customer[userName: james firstName: null surname: null] as no matching persisted entity found.
    ...
    thread #0 - file://src/data] CustomerTransformer            INFO  Created a new CustomerEntity Customer[userName: hiram firstName: null surname: null] as no matching persisted entity found.
    ...

Wenn wir das Programm erneut starten, kommt nun folgende Meldung (erfolg):

.. code:: plain

    ...
    thread #0 - file://src/data] CustomerTransformer            INFO  Found a matching CustomerEntity Customer[userName: james firstName: James surname: Strachan] having the userName james.
    ...
    thread #0 - file://src/data] CustomerTransformer            INFO  Found a matching CustomerEntity Customer[userName: hiram firstName: Hiram surname: Chirino] having the userName hiram.
    ...

**Wichtige Quellcode Zeilen**

EtlRoutes:

.. code:: java

    public class EtlRoutes extends SpringRouteBuilder {
        public void configure() throws Exception {
     
            from("file:src/data?noop=true")
                .convertBodyTo(PersonDocument.class)
                .to("jpa:org.apache.camel.example.etl.CustomerEntity");
     
            // the following will dump the database to files
            from("jpa:org.apache.camel.example.etl.CustomerEntity?consumeDelete=false&delay=3000&consumeLockEntity=false")
                .setHeader(Exchange.FILE_NAME, el("${in.body.userName}.xml"))
                .to("file:target/customers");
        }
    }

Die oben angegebene Datei erstellt eine Route vom "src/data" Verzeichnis. Der "noop" Modus von File verhindert, dass die Dateien gelöscht oder verschoben werden, während sie verarbeitet werden.
Dadurch können sie nach einem Neustart erneut verarbeitet werden.
Diese EtlRoute gibt an, von wo das ETL/EAI seine Ausgangsdaten lesen soll.

Wir konvertieren dann mithilfe des PersonDocument die XML Datei in eine Objekt repräsentation. Durch die Angabe von @XmlRootElement stößt diese den JAXB (Java API for XML Processing) zum umwandeln des XML an.

Anschließend wird diese Nachricht mit dem PersonDocument Inhalt (body) an einen JPA (Java Persistance API) Endpoint gesendet.
Durch die Angabe des erwarteten Datentyps (in diesem Fall CustomerEntity) in der CustomerTransformer Klasse, wodurch automatisch versucht wird, in diesen Datentyp umzuwandeln.
Die @Converter Methoden geben die Methoden an, welche automatisch aufgerufen werden sollen.

.. code:: java

    @Converter
    public final class CustomerTransformer {
     
        private static final Logger LOG = LoggerFactory.getLogger(CustomerTransformer.class);
     
        private CustomerTransformer() {
        }
     
        /**
         * A transformation method to convert a person document into a customer
         * entity
         */
        @Converter
        public static CustomerEntity toCustomer(PersonDocument doc, Exchange exchange) throws Exception {
            EntityManager entityManager = exchange.getIn().getHeader(JpaConstants.ENTITYMANAGER, EntityManager.class);
            TransactionTemplate transactionTemplate = exchange.getContext().getRegistry().lookupByNameAndType("transactionTemplate", TransactionTemplate.class);
     
            String user = doc.getUser();
            CustomerEntity customer = findCustomerByName(transactionTemplate, entityManager, user);
     
            // let's convert information from the document into the entity bean
            customer.setUserName(user);
            customer.setFirstName(doc.getFirstName());
            customer.setSurname(doc.getLastName());
            customer.setCity(doc.getCity());
     
            LOG.info("Created object customer: {}", customer);
            return customer;
        }
     
        /**
         * Finds a customer for the given username
         */
        private static CustomerEntity findCustomerByName(TransactionTemplate transactionTemplate, final EntityManager entityManager, final String userName) throws Exception {
            return transactionTemplate.execute(new TransactionCallback<CustomerEntity>() {
                public CustomerEntity doInTransaction(TransactionStatus status) {
                    entityManager.joinTransaction();
                    List<CustomerEntity> list = entityManager.createNamedQuery("findCustomerByUsername", CustomerEntity.class).setParameter("userName", userName).getResultList();
                    CustomerEntity answer;
                    if (list.isEmpty()) {
                        answer = new CustomerEntity();
                        answer.setUserName(userName);
                        LOG.info("Created a new CustomerEntity {} as no matching persisted entity found.", answer);
                    } else {
                        answer = list.get(0);
                        LOG.info("Found a matching CustomerEntity {} having the userName {}.", answer, userName);
                    }
     
                    return answer;
                }
            });
        }
     
    }

Dadurch wird die Konvertierung durchgeführt und speichert dieses "Bean" in der Datenbak.

Also Zusammengefasst:

*XML -> PersonDocument -> CustomerEntity -> Datenbank*

============
EAI Patterns
============

~~~~~~~~~~~~~~~~~~~~~~~~
Domain specific language
~~~~~~~~~~~~~~~~~~~~~~~~

Camel verwedet die DSL *fluid builder*.

.. code:: java

    RouteBuilder builder = new RouteBuilder() {
        public void configure() {
            errorHandler(deadLetterChannel("mock:error"));
     
            from("direct:a")
                .choice()
                    .when(header("foo").isEqualTo("bar"))
                        .to("direct:b")
                    .when(header("foo").isEqualTo("cheese"))
                        .to("direct:c")
                    .otherwise()
                        .to("direct:d");
        }
    };
    
DSL ist hierbei der Ausdruck welcher mit dem Aufruf der Methode anfängt. 
                
Mit *fluid builder* lassen sich Strukturen wie Router einfach und in Java Code umsetzen. 
Camel bietet noch einige andere wege die zum gleichen Ziel führen. So sieht etwa die gleiche
Struktur als XML aus:

.. code:: XML

    <camelContext errorHandlerRef="errorHandler" xmlns="http://camel.apache.org/schema/spring">
        <route>
            <from uri="direct:a"/>
            <choice>
                <when>
                    <xpath>$foo = 'bar'</xpath>
                    <to uri="direct:b"/>
                </when>
                <when>
                    <xpath>$foo = 'cheese'</xpath>
                    <to uri="direct:c"/>
                </when>
                <otherwise>
                    <to uri="direct:d"/>
                </otherwise>
            </choice>
        </route>
    </camelContext>
    
~~~~~~~~~~~~~~
Type Converter
~~~~~~~~~~~~~~

Es wird ein TypeConverter verwendet um PersonDocument nach CustomerEntity umzuwandeln. 
Der Converter ist durch die Annotation ``Converter`` markiert. 

.. code:: java

    @Converter
    public static CustomerEntity toCustomer(PersonDocument doc, Exchange exchange) throws Exception {
        EntityManager entityManager = exchange.getIn().getHeader(JpaConstants.ENTITYMANAGER, EntityManager.class);
        TransactionTemplate transactionTemplate = exchange.getContext().getRegistry().lookupByNameAndType("transactionTemplate", TransactionTemplate.class);

        String user = doc.getUser();
        CustomerEntity customer = findCustomerByName(transactionTemplate, entityManager, user);

        // let's convert information from the document into the entity bean
        customer.setUserName(user);
        customer.setFirstName(doc.getFirstName());
        customer.setSurname(doc.getLastName());
        customer.setCity(doc.getCity());

        LOG.info("Created object customer: {}", customer);
        return customer;
    }
    
Code aus ``org.apache.camel.example.etl.CustomerTransformer``.

~~~~~~~~
Messages
~~~~~~~~

.. image:: http://www.enterpriseintegrationpatterns.com/img/MessageSolution.gif
    :width: 70%

Bei einer Message wird eine Information über eine Route geführt. 

Im Beispielcode wurde die Route mit der *fluid builder* DSL bestimmt.

.. code:: java

    from("file:src/data?noop=true")
        .convertBodyTo(PersonDocument.class)
        .to("jpa:org.apache.camel.example.etl.CustomerEntity");

    // the following will dump the database to files
    from("jpa:org.apache.camel.example.etl.CustomerEntity?consumeDelete=false&delay=3000&consumeLockEntity=false")
        .setHeader(Exchange.FILE_NAME, el("${in.body.userName}.xml"))
        .to("file:target/customers");

 
=======
Quellen
=======

[1] Extract Transform Load (ETL);Apache Camel; Online: http://camel.apache.org/etl.html; abgerufen 27.02.2014

[2] Enterprise Integration Patterns; G.Hohpe, B.Woolf; 2003; Online: http://www.enterpriseintegrationpatterns.com/toc.html; abgerufen 27.02.2014

[3] Extract Transform Load (ETL) Example; Apache Camel; Online: http://camel.apache.org/etl-example.html; abgerufen 27.02.2014

[4] Java DSL; Apache Camel; Online: https://camel.apache.org/java-dsl.html; abgerufen 27.02.2014

[5] Type Converter; Apache Camel; Online: https://camel.apache.org/type-converter.html; abgerufen 27.02.2014

[6] Messages; Apache Camel; Online: https://camel.apache.org/message.html; abgerufen 27.02.2014

.. header::

    +-------------+-------------------+------------+
    | Titel       | Autor             | Date       |
    +=============+===================+============+
    | ###Title### | Andreas Willinger | 27.02.2014 |
    |             | -- Jakob Klepp    |            |
    +-------------+-------------------+------------+

.. footer::

    ###Page### / ###Total###
