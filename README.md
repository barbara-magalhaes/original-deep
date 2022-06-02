<h1  align="center">Deep Log Service</h1>

[![CircleCI](https://circleci.com/gh/pelando/deep-log-service.svg?style=shield&circle-token=4225d6a0e6f96c69a73d3a9dcbbb969da0b8f452)](https://circleci.com/gh/pelando/deep-log-service)


<div  align="center">
Service for tracking URLs of deals.
</div>

## Description

This service aims to describe the following items:

- Manage rules between Affiliate Networks and Partner Stores.
- Integrate with the Affiliate Networks APIs to generate Tracked URLs.
- Collect purchase transactions on Affiliate Networks.

## Prerequisite

Start at [link confluence](adicionar)

Install `build-essential` [tutorial](https://linuxhint.com/install-build-essential-ubuntu/)

Install `cmake` [tutorial](https://linuxhint.com/install-cmake-on-ubuntu)

Install `yarn` [tutorial](https://linuxize.com/post/how-to-install-yarn-on-ubuntu-20-04/)

Install `nodejs` [tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-20-04-pt)

Install `docker` [tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04-pt)

Install `docker-compose` [tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04-pt)


## Usage

1. Clone project:

```bash
git clone git@github.com:pelando/deep-log-service.git
cd deep-log-service
```

2. [Download](https://pelandobr.atlassian.net/wiki/spaces/EN/pages/2621506/Deeplog) the local `.env` file for this repository.

3. Build and start the application:

```bash
sudo make up
```

or (detached)

```bash
make up-silent
```

4. After the startup is complete, open a browser and visit [http://localhost/status](http://localhost/status).

5. On a new shell, create the database, migrate and populate it:

```bash

make db-reset

```

## Other commands

Build docker image:

```bash

make build

```

Run tests:

```bash

make test
make test-all

```

Inspect the container:

```bash

make shell

```

## Dependencies

Attention when updating this repository check if there is any relationship with the [API](https://github.com/pelando/api/blob/development/k8s/dev/api.template.yml). Deep Log integrates directly with the same


## Deep Log como Aplicação

1. Redes de Afiliados

Atualmente esse serviço possui as seguintes redes:

- [Afilio](https://v2.afilio.com.br/Manual/manuais-v2.html)
- [Awin](https://wiki.awin.com/index.php/Publisher_API)
- Direto
- [Lomadee](https://developer.lomadee.com/afiliados/)
- MagazineVoce
- [Rakuten](https://api.rakuten.co.jp/docs/docs/documenting-an-api/)

As redes Direto e MagazineVoce não são redes de afiliados, são apenas modeladas dessa forma pois de certa maneira também são capazes de gerar deeplink.

Algumas lojas oferecem a possibilidade de trackear uma URL apenas adicionando parâmetros na URL do produto. Esse é o caso em que a "rede" Direto atua.

O caso da Magazine Luiza é diferente pois não passa em nenhuma rede de afiliados e tampouco é possível passar pelo Direto. Por essa razão foi implementado a "rede" MagazineVoce que consiste de um programa da Magazine Luiza para parceiros que fazem vendas via afiliados.

Na prática o link trackeado é um endpoint disponibilizado pela loja, mas que precisa do SKU do produto como parâmetro. Para obter esse dado esse serviço faz uma requisição à página do produto no site do Magazine Luiza e entrega o conteúdo HTML que possui o SKU desejado.

Awin e Zanox são a mesma rede, Zanox é a antiga API que ficou obsoleta.

2. Ordem de Prioridades nas Regras

Uma regra é um relacionamento entre uma loja e uma ou mais redes. A ordem de prioridade diz respeito sobre qual rede deve gerar o link trackeado. Essa ordem é calculada de forma decrescente, isto é, a rede com maior prioridade tem valor 0. Uma loja, por exemplo, pode passar com prioridade 1 na Awin e prioridade 2 na Lomadee, e assim por diante. Dessa forma, é esperado que o deeplink seja gerado pela API da Awin e se por algum motivo falhar, é tentado a rede a seguir.

Note que isso pode ser lento pois há comunicação com API de terceiros e no pior caso esse processo pode ser cascateado até a última prioridade. Vale revisitar essa solução e adicionar um timeout para evitar uma experência ruim para o usuário.

Nos casos em que o deeplog não é capaz de entregar uma URL trackeada, seja por eventual falha na comunicação com as APIs das redes ou simplesmente por não existir regra para aquela loja/url, é feito um redirecionamento para o digidip, ferramenta equivalente ao deeplog mas desenvolvida pelo time do Pepper.

3. Como Atualizar Regras via endpoint

Existem casos em que o time comercial fecha acordo direto com a loja/rede e assim nasce a necessidade de alterar a ordem de prioridade de uma regra. Isso pode ser feito através de queries no banco ou das rotas a seguir.

A URL base de cada ambiente é:

- http://localhost: local
- http://development.deeplog.pelando.com.br: development
- http://www.deeplog.pelando.com.br: production

Para obter a regra ativa para uma loja, você precisa do id da loja no banco do deeplog:

```
curl -X GET \
  '<base_url>/rule?store_id=<store_id>' \
  -H 'Authorization: 4u1h-/t0k3n-/Qu4n1!c0-/d0-/Gu514c0' \
  -H 'cache-control: no-cache' | json_pp
```

Para modificar a regra ativa para uma loja, você precisa do id da loja no banco do deeplog e enviar um objeto com o id da rede no deeplog e a prioridade que ela deve ter:

```
curl -X POST \
  <base_url>/rule \
  -H 'Accept: */*' \
  -H 'Authorization: 4u1h-/t0k3n-/Qu4n1!c0-/d0-/Gu514c0' \
  -H 'Cache-Control: no-cache' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/json' \
  -d '{
	"store_id": <store_id>,
	"network_priorities": [
            {
                "network_id": <network_id>,
                "order_priority": 1
            },
            {
                "network_id": <network_id>,
                "order_priority": 2
            },
            {
                "network_id": <network_id>,
                "order_priority": 3
            }
        ]
}' | json_pp
```

Sempre que uma regra é atualizada através dessa rota é feito um soft-delete na regra anterior e escrito uma nova regra no banco. Uma regra contém uma ou mais linhas na tabela `rule` e é caracterizada como regra ativa quando o campo `ended_at` possui valor igual a `9999-12-31T23:59:59`. Os campos `started_at` e `ended_at` fazem o controle de quando a regra deve ser ativa, isso é importante para casos em que uma regra deve ser ativa no futuro além de manter o histórico de regras no banco.

O campo `network_priorities` deve conter todas as novas prioridades, uma regra com três redes escreve três entradas na tabele `rule` uma para cada rede/prioridade. Feito isso o campo `started_at` recebe o valor now() e o campo `ended_at` recebe `9999-12-31T23:59:59`. A regra anterior (se existir) tem o campo `ended_at` setado com now() para indicar que aquela regra foi finalizada.

4. Como Gerar um Link Trackeado

Você pode efetuar um clickout através do seguinte endpoint:

- <base_url>/visit?url=<product_url_encoded>

A aplicação busca no banco o id da loja através da URL do produto. Então, procura na tabela de regras se existe uma regra ativa para essa loja. No caso de não existir, o link trackeado é gerado pela API da rede e escrito no banco na tabela de URL. Nos demais casos o link trackeado é recuperado do banco e é realizado o redirecionamento.

A URL trackeada pode ser cacheada no banco ou através da função que tem como responsabilidade obter a URL trackeada. No segundo caso o cache ocorrre utilizando a técnica de "memoization". Esse cache é mapeado de acordo com os parâmetros que recebe e é invalidado por tempo de vida (o default no projeto atualmente é de 5 minutos).

## Deep Log como Worker

1. Como Disparar Worker?

Atualmente os workers desse serviço rodam de forma automática na AWS no [ECS](https://console.aws.amazon.com/ecs/home?region=us-east-1#/clusters/workers-deep-log-api-production/scheduledTasks). Caso seja necessário a execução de forma manual, isso pode ser feito editando o schedule do worker desejado.

2. Quando os Workers rodam

Existem três tipos de workers em execução:

- Coleta de transações nas Redes: Afilio, Awin, Lomadee e Rakuten
- Verifica se uma Loja foi adicionada a uma Rede
- Verifica se uma Loja foi removida de uma Rede

Os workers de coleta de transação rodam de 1h em 1h. Os demais rodam 1 vez por dia em horário específico.

3. Transações

Uma transação é uma compra efetuada por um usuário. Quando um usuário realiza uma compra através de um link trackeado por esse serviço, a rede de afiliados disponibiliza através das suas API's a transação referente àquela compra. Isso é importante porque traz dados referentes a data de compra, valor, comissão a ser recebida, dentre outros.

Uma forma de mapear o clique do pelando que originou a compra à transação é através de parâmetros que as redes disponibilizam.

- Afilio utiliza: `aff_xtra`
- Awin utiliza: `clickref`
- Lomadee utiliza: `mdasc`
- Rakuten utliza: `u1`

Um exemplo simples é uma URL no formato https://deeplink.com.br?clickref=123 onde 123 é o id do clique no pelando. Dessa forma, ao coletar a transação da Awin é esperado que exista um campo que contenha o valor 123 e assim é relacionado o clique à compra.

4. Clickouts

Toda requisição à rota `/visit` escreve um novo registro na tabela clickout, mesmo os casos em que a URL não é trackeada. Isso é muito importante para o negócio pois informa quais produtos tem mais ou menos atenção pelos usuários.

5. Como Regras são Atualizadas de forma Automática

As regras podem ser alteradas através da rota descrita no item 3, direto no banco ou de forma automática pelos workers que verificam se lojas foram adicionadas ou removidas de redes.

Nos casos em que uma Loja passa a pertencer a uma Rede, se não existia regra ativa para essa Loja o worker adiciona a regra. Se já existia regra ativa para essa loja, o worker adiciona à regra existente a nova rede no final da fila.

Por exemplo, vamos imaginar que Magalu passa pela Awin e Afilio. Nesse caso, a regra é Awin com prioridade 1 e Afilio com prioridade 2. O worker é executado e verifica que essa loja agora está presente também na Lomadee, nesse caso, a nova regra passa a ser Awin, Afilio e Lomadee.

O ideal é que nunca seja inserida uma regra direto no banco pois os workers garantem que se a loja passa por uma rede, a regra existe. Por outro lado é natural que surja a necessidade de alterar a ordem de prioridade da rede para a loja, nesse caso, o ideal é usar o endpoint `/rule`.

6. Como Lojas são Adicionadas de forma Automática

Quando o worker que identifica novas lojas adicionadas pelas redes é executado ele pode encontrar na lista de lojas ativas da rede uma loja que não está cadastrada no banco do deeplog. Nesse caso, ele cadastra a loja na tabela store e o domínio da loja na tabela store_url.

Novamente o ideal é que lojas não sejam cadastradas direto no banco através de queries sql uma vez que isso é feito de forma automática pelo worker. Se por algum motivo o cadastro não ocorreu, deve ser investigado no código o possível motivo que impediu o cadastro.

## Project Structure

| Name                          | Description                                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **.circleci**/                | CircleCI configuration                                                                                       |
| **.github**/                  | Github configuration for workflows, PR templates and others                                                  |
| **app/command**/              | Commands called by the manager or others                                                                     |
| **app/core**                  | Abstract class, database connection, etc                                                                     |
| **app/enum**/                 | Enum should be implemented here                                                                              |
| **app/helper**/               | Helpers should be implemented here                                                                           |
| **app/manager**/              | Manager of all project                                                                                       |
| **app/network**/              | The implementation of all affiliate networks is found here                                                   |
| server.py                     | Starts the flask server                                                                                      |
| **utils**/                    | Auxiliary functions                                                                                          |
| **migrations**/               | Database versioning migrations                                                                               |
| **scripts**/                  | Auxiliary scripts                                                                                            |
| **tests**/                    | Application unit and integration tests                                                                       |
| .dockerignore                 | Folder and files ignored by docker usage                                                                     |
| .env.template                 | Your API keys, tokens, passwords and database URI                                                            |
| .gitignore                    | Folder and files ignored by git                                                                              |
| config.py                     | Env var config, database parameters, general configuration                                                   |
| docker-compose.yml            | Docker compose configuration file                                                                            |
| docker-compose.ci.yml         | Docker compose configuration file for ci                                                                     |
| docker-compose.daemon.yml     | Docker compose configuration file for pavement task like daemon                                              |
| Dockerfile                    | Dockerfile image blueprint                                                                                   |
| Makefile                      | Helpers and shortcuts to start the application                                                               |
| migrator.py                   | File that instances database for migration                                                                   |
| pavement.py                   | A hybrid declarative/imperative [system](https://paver.github.io/paver/pavement.html) for getting stuff done |
| README.md                     | This very file. It is meant for being read                                                                   |
| requirements                  | List of all packages that need to install                                                                    |
| uwsgi.ini                     | uWSGI configuration file                                                                                     |