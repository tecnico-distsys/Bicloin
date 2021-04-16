# Guião de Demonstração

(incompleto -- **compete ao grupo completar o guião** -- ver TODOs)

## 1. Preparação do sistema

Para testar o sistema e todos os seus componentes, é necessário preparar um ambiente com dados para proceder à verificação dos testes.

### 1.1. Lançar o *registry*

Para lançar o *ZooKeeper*, ir à pasta `zookeeper/bin` e correr o comando  
`./zkServer.sh start` (Linux) ou `zkServer.cmd` (Windows).

É possível também lançar a consola de interação com o *ZooKeeper*, novamente na pasta `zookeeper/bin` e correr `./zkCli.sh` (Linux) ou `zkCli.cmd` (Windows).

### 1.2. Compilar o projeto

Primeiramente, é necessário compilar e instalar todos os módulos e suas dependências --  *rec*, *hub*, *app*, etc.
Para isso, basta ir à pasta *root* do projeto e correr o seguinte comando:

```sh
$ mvn clean install -DskipTests
```

### 1.3. Lançar e testar o *rec*

Para proceder aos testes, é preciso em primeiro lugar lançar o servidor *rec* .
Para isso basta ir à pasta *rec* e executar:

```sh
$ mvn compile exec:java
```

Este comando vai colocar o *rec* no endereço *localhost* e na porta *8091*.

Para confirmar o funcionamento do servidor com um *ping*, fazer:

```sh
$ cd rec-tester
$ mvn compile exec:java
```

Para executar toda a bateria de testes de integração, fazer:

```sh
$ mvn verify
```

Todos os testes devem ser executados sem erros.


### 1.4. Lançar e testar o *hub*

Devido a nesta entrega ainda não ter de ser implementado a possibilidade de existir réplicas do Hub, só é possível iniciar uma instância do mesmo, com o comando initRec ativo, tal que basta executar:

```sh
$ mvn compile exec:java
```

Este comando corresponde a "$ hub localhost 8891 localhost 8081 1 users.csv stations.csv initRec", sendo que estes argumentos estão especificados no ficheiro pom.xml e são passados através do maven.

Para executar toda a bateria de testes de integração, fazer:

```sh
$ mvn verify
```

### 1.5. *App*

Iniciar a aplicação com a utilizadora alice:

```sh
$ app localhost 2181 alice +35191102030 38.7380 -9.3000
```

**Nota:** Para poder correr o script *app* diretamente é necessário fazer `mvn install` e adicionar ao *PATH* ou utilizar diretamente os executáveis gerados na pasta `target/appassembler/bin/`.

Abrir outra consola, e iniciar a aplicação com o utilizador bruno.

Depois de lançar todos os componentes, tal como descrito acima, já temos o que é necessário para usar o sistema através dos comandos.

## 2. Teste dos comandos

Nesta secção vamos correr os comandos necessários para testar todas as operações do sistema.
Cada subsecção é respetiva a cada operação presente no *hub*.

### 2.1. *balance*

O comando balance do Hub recebe uma mensagem BalanceRequest (composta por uma string username, que corresponde ao id do usário) e começa por fazer a verificação das propriedades da mensagem, falhando caso o username dado tenha null value, esteja vazio ou não esteja presente no map que contém os vários usados registados. Após a verificação, é efetuada a leitura do registo que contém o balance do usário e é enviada uma mensagem BalanceResponse de volta ao mesmo com este valor.
.
### 2.2 *top-up*

O comando top-up do Hub recebe uma mensagem TopUpRequest (composta por uma string username, um google.type.Money money e uma string phonenumber) e começa por fazer a verificação das propriedas, sendo a do username igual à verificação ocorrida no comando balance, a do tipo money verifica se o código ISO da moeda está vazio ou se corresponde a uma moeda não aceite pela Bicloin para conversão em BIC, se o valor convertido das unidades não é válido (entre 10 a 200 BIC) e se recebeu qualquer valor em cêntimos, a verificação do phonenumber verifica se este tem valor null, se é uma string vazia e se corresponde ao phonenumber associado com o usário dado, falhando caso alguma desta verificações seja verdadeira. Depois, é obtido do registo o valor atual do balance do usário e é escrito no registo o valor atual mais o do carregamento e é retornada uma mensagem TopUpResponse com este valor atualizado.

### 2.3 *infoStation*

O comando infoStation do Hub recebe uma mensagem InfoStationRequest (composta por uma string stationId) e inicialmente faz uma verificação análoga à do username de um usário, a única diferença sendo que neste caso o map a que terá de conter o id é o que contém as vários estações registadas, a chamada remota falha nas mesmas condições da de um username. Após a verificação, é obtido através do registo o número de bicicletas presentes na estação e as estatísticas da mesma (i.e. o número acumulado de bicicletas levantadas e retornadas), em seguida e construída uma mensagem InfoStationResponse com o nome da estação (string name), as coordenadas da mesma (double nsCoordinates e weCoordinates), o número de docas para bicicletas (uint32 dockCapacity), o número de bicicleas presentes (uint32 bikesAvailable) e as estatísticas (uint32 acumulatedTakeouts e acumulatedReturns) e por fim esta mensagem é retornada.

### 2.4 *locateStation*

O comando locateStation do Hub recebe uma mensagem LocateStationRequest (composta por um uint32 numberToList, que indica o número de estações a enviar na mensagem de retorno, coordenadas double nsCoordinates e weCoordinates) e faz a verificação dos valores recebidos, sendo que o número de estações a listar não deve ser inferior a 1 e as coordenadas recebidas devem ser válidas, caso contrário a chamada falha, é também verificado se o número de estações a listar é superior ao números de estações registadas, se tal ocorrer, não ocorre qualquer erro, mas o número de estações a listar passa a ser o número total de estações registadas e não o recebido na mensagem. Por fim, é criada uma lista com o tamanho anteriormente definido com os identificadores das estações a listar, por ordem crescente de distância (calculada através da função de haversine) em relação às coordenadas recebidas e posteriormente é enviada numa mensagem LocateStationResponse.

### 2.5 *bikeUp*

O comando bikeUp do Hub recebe uma mensagem BikeUpRequest (composta por uma string username, coordenadas double nsCoordinates e weCoordinates e uma string stationId), fazendo a verificação do recebido, sendo a do username igual à ocorrida no comando balance, a das coordenadas igual à ocorrida no comando LocateStation e a do stationId à ocorrida no comando infoStation, verificando ainda se a distância do utilizador à estação dada é superior a 200 metros (pela função de haversine) e a chamada falha em qualquer um destes casos. Após esta verificação inicial, é obtido a indicação de se o usário ainda tem uma bicicleta por entregar, o número de bicicletas na estação e o número de moedas do usário, sendo depois feita uma 2ª verificação que falha caso o usário ainda tenha uma bicicleta por entregar, a estação não tenha bicicletas disponíveis ou o usário não tenha moedas suficientes para o aluguer. Se estas 2 verificações forem passadas, é escrito no registo que o utilizador retirou uma bicleta, é incrementado o número acumulado de bicicletas levantadas da estação, é removido 10 BIC do balance do usário e 1 bicleta da estação, caso um destes pedidos de escrita ao registo falhe, a chamada é desfeita, caso contrário é retornada uma mensagem BikeUpResponse com o valor SUCCESS.

### 2.6 *bikeDown*

O comando bikeDown do Hub recebe uma mensagem BikeDownRequest (que tem a mesma especificação da BikeUpRequest) e é em quase tudo análogo ao comando bikeUp nas verificações feitas, com as exceções de que nas 2ª validações não ocorre a verificação monetária e ocorrem falhas casos o usário já tenha uma bicicleta ativa ou já estejam presentes o número máximo de bicicletas nesta estação. Após as validações, é incrementado o valor acumulado de bicicletas retornadas á estação, o número de bicicletas presentes na mesma, o usário tem o seu balance aumentado de acordo com o valor do prémio da estação e é escrito no registo que o usário já não tem uma bicleta ativa e como no bikeUp, a chamada é desfeita caso alguma escrita no registo falhe, caso contrário é retornada uma mensagem BikeUpResponse com o valor SUCCESS.

### 2.7 *ping*

O comando ping do Hub recebe uma mensagem PingRequest (composta por uma string input), verifica se este input tem valor null ou está vazia, retornando erro nesse caso. Retorna uma mensagem PingResponse que é uma composição de "Hello", o input recebido e um ponto de exclamação, servindo isto para sinalizar que Hub está ativo.

### 2.8 *sysStatus*



----

## 3. Considerações Finais

Estes testes não cobrem tudo, pelo que devem ter sempre em conta os testes de integração e o código.
