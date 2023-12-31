Início rápido do Quarkus Kafka Streams
========================

Este projeto ilustra como você pode criar aplicativos [Apache Kafka Streams](https://kafka.apache.org/documentation/streams) usando o Quarkus.

## Anatomia da Aplicação

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

Isso irá gerar mais duas instâncias deste serviço.
O armazenamento de estado que materializa o estado atual do pipeline de streaming
(que consultamos antes de usar a consulta interativa),
agora está distribuído entre os três nós.
ou seja ao invocar a API REST em qualquer uma das três instâncias, pode ser
que a agregação para o ID da estação meteorológica solicitada seja armazenada localmente no nó que recebe a consulta,
ou pode ser armazenado em um dos outros dois nós.

Como o balanceador de carga do Docker Compose distribuirá solicitações para o serviço _agregador_ em um estilo round-robin,
invocaremos os nós reais diretamente.
A aplicação expõe informações sobre todos os nomes de host via REST:

```bash
http aggregator:8080/weather-stations/meta-data
```

Recupere os dados de um dos três hosts mostrados na resposta
(seus nomes de host reais serão diferentes):

```bash
http cf143d359acc:8080/weather-stations/data/1
```

Se esse nó contiver os dados da chave "1", você receberá uma resposta como esta:

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

Caso contrário, o serviço enviará um redirecionamento:

```
HTTP/1.1 303 See Other
Connection: keep-alive
Content-Length: 0
Date: Tue, 11 Jun 2019 19:17:51 GMT
Location: http://72064bb97be9:8080/weather-stations/data/2
```

Você pode fazer com que _httpie_ siga automaticamente o redirecionamento passando a opção `--follow`:
```bash
http --follow aggregator:8080/weather-stations/data/2
```

## TLS 

Caso o HTTP esteja desativado por meio de:

```properties
quarkus.http.insecure-requests=disabled
```

O URL do terminal se torna:

```bash
curl -L --insecure https://aggregator:8443/weather-stations/data/2
```

## Executando em nativo

Para executar os aplicativos _produtor_ e _agregador_ como binários nativos via GraalVM,
primeiro execute as compilações do Maven usando o perfil `nativo`:

```bash
mvn clean install -Pnative -Dnative-image.container-runtime=docker
```

Em seguida, crie uma variável de ambiente chamada `QUARKUS_MODE` e com valor definido como "nativo"

```bash
export QUARKUS_MODE=native
```

Agora inicie o Docker Compose conforme descrito acima.

## Executando localmente

Para fins de desenvolvimento, pode ser útil executar os aplicativos _produtor_ e _agregador_
diretamente na sua máquina local em vez de via Docker.
Para esse propósito, é fornecido um arquivo Docker Compose separado que apenas inicia o Apache Kafka e o ZooKeeper, _docker-compose-local.yaml_
configurado para ser acessível a partir do seu sistema host.
Abra este arquivo em um editor e altere o valor da variável `KAFKA_ADVERTISED_LISTENERS` para que contenha o nome ou endereço IP da sua máquina host.
Então corra:

```bash
docker-compose -f docker-compose-local.yaml up

mvn quarkus:dev -f producer/pom.xml

mvn quarkus:dev -Dquarkus.http.port=8081 -f aggregator/pom.xml
```

Quaisquer alterações feitas no aplicativo _agregador_ serão detectadas instantaneamente,
e uma recarga do aplicativo de processamento de fluxo será acionada na próxima mensagem Kafka a ser processada.
