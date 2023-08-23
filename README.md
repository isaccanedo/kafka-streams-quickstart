Início rápido do Quarkus Kafka Streams
========================

Este projeto ilustra como você pode criar aplicativos [Apache Kafka Streams](https://kafka.apache.org/documentation/streams) usando o Quarkus.

## Anatomia

Este início rápido é composto das seguintes partes:

* Apache Kafka e ZooKeeper
* _producer_, um aplicativo Quarkus que publica alguns dados de teste em dois tópicos Kafka: `estações meteorológicas` e `valores de temperatura`
* _aggregator_, um aplicativo Quarkus processando os dois tópicos, usando a API Kafka Streams

O aplicativo _aggregator_ é a peça interessante; isto

* executa um pipeline KStreams, que une os dois tópicos (no id da estação meteorológica),
agrupa os valores por estação meteorológica e emite o valor mínimo/máximo de temperatura por estação para o tópico `temperaturas agregadas`
* expõe um endpoint HTTP para obter os valores mínimos/máximos atuais
para uma determinada estação usando consultas interativas do Kafka Streams.

## Construindo

Para construir os aplicativos _produtor_ e _aggregator_, execute

```bash
mvn clean install
```

## Executando

Um arquivo Docker Compose é fornecido para executar todos os componentes.
Inicie todos os contêineres executando:

```bash
docker-compose up -d --build
```

Agora execute uma instância da imagem _debezium/tooling_ que vem com várias ferramentas úteis como _kafkacat_ e _httpie_:

```bash
docker run --tty --rm -i --network ks debezium/tooling:1.1
```

No contêiner de ferramentas, execute _kafkacat_ para examinar os resultados do pipeline de streaming:

```bash
kafkacat -b kafka:9092 -C -o beginning -q -t temperatures-aggregated
```

Você também pode obter o estado agregado atual para uma determinada estação meteorológica usando _httpie_,
que invocará uma consulta interativa do Kafka Streams para esse valor:

```bash
http aggregator:8080/weather-stations/data/1
```

## Dimensionamento

Os pipelines do Kafka Streams podem ser ampliados, ou seja, a carga pode ser distribuída entre várias instâncias de aplicativos executando o mesmo pipeline.
Para testar isso, dimensione o serviço _agregador_ para três nós:

```bash
docker-compose up -d --scale aggregator=3
```

This will spin up two more instances of this service.
The state store that materializes the current state of the streaming pipeline
(which we queried before using the interactive query),
is now distributed amongst the three nodes.
I.e. when invoking the REST API on any of the three instances, it might either be
that the aggregation for the requested weather station id is stored locally on the node receiving the query,
or it could be stored on one of the other two nodes.

As the load balancer of Docker Compose will distribute requests to the _aggregator_ service in a round-robin fashion,
we'll invoke the actual nodes directly.
The application exposes information about all the host names via REST:

```bash
http aggregator:8080/weather-stations/meta-data
```

Retrieve the data from one of the three hosts shown in the response
(your actual host names will differ):

```bash
http cf143d359acc:8080/weather-stations/data/1
```

If that node holds the data for key "1", you'll get a response like this:

```
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 74
Content-Type: application/json
Date: Tue, 11 Jun 2019 19:16:31 GMT

{
    "avg": 15.7,
    "count": 11,
    "max": 31.0,
    "min": 3.3,
    "stationId": 1,
    "stationName": "Hamburg"
}
```

Otherwise, the service will send a redirect:

```
HTTP/1.1 303 See Other
Connection: keep-alive
Content-Length: 0
Date: Tue, 11 Jun 2019 19:17:51 GMT
Location: http://72064bb97be9:8080/weather-stations/data/2
```

You can have _httpie_ automatically follow the redirect by passing the `--follow` option:

```bash
http --follow aggregator:8080/weather-stations/data/2
```

## TLS 

In case HTTP is disabled via:

```properties
quarkus.http.insecure-requests=disabled
```

The endpoint URL becomes:

```bash
curl -L --insecure https://aggregator:8443/weather-stations/data/2
```

## Running in native

To run the _producer_ and _aggregator_ applications as native binaries via GraalVM,
first run the Maven builds using the `native` profile:

```bash
mvn clean install -Pnative -Dnative-image.container-runtime=docker
```

Then create an environment variable named `QUARKUS_MODE` and with value set to "native":

```bash
export QUARKUS_MODE=native
```

Now start Docker Compose as described above.

## Running locally

For development purposes it can be handy to run the _producer_ and _aggregator_ applications
directly on your local machine instead of via Docker.
For that purpose, a separate Docker Compose file is provided which just starts Apache Kafka and ZooKeeper, _docker-compose-local.yaml_
configured to be accessible from your host system.
Open this file an editor and change the value of the `KAFKA_ADVERTISED_LISTENERS` variable so it contains your host machine's name or ip address.
Then run:

```bash
docker-compose -f docker-compose-local.yaml up

mvn quarkus:dev -f producer/pom.xml

mvn quarkus:dev -Dquarkus.http.port=8081 -f aggregator/pom.xml
```

Any changes done to the _aggregator_ application will be picked up instantly,
and a reload of the stream processing application will be triggered upon the next Kafka message to be processed.
