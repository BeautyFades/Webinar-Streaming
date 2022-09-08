# Webinar - Construindo um fluxo de dados em streaming
![webinar-frontpage](https://i.imgur.com/iYZcb1R.png)

# Disclaimer
Este repositório complementa o conteúdo abordado [neste webinar](https://www.youtube.com/watch?v=aLZkynt8pPg). Aqui estarão listados alguns dos códigos usados e um passo a passo do setup necessário para replicar o que foi apresentado na parte prática do webinar.

Estou usando Windows 10 como sistema operacional. Alguns comandos ou passos podem ser diferentes de acordo com o seu SO.

# Requisitos

- Docker instalado e funcional na sua máquina;
- Uma conta da GCP ativa (pode usar o trial gratuito de 3 meses com $300 USD em créditos!);
- Saber emitir as chaves de acesso em formato .json para contas de serviço na GCP (veja [aqui](https://www.youtube.com/watch?v=SDhMwyyd9_0) ou [aqui](https://developers.google.com/identity/protocols/oauth2/service-account)).

# Prática passo a passo
No passo a passo, vamos seguir a ordem de criação das aplicações de acordo com o apresentado no webinar. **É MUITO IMPORTANTE SEGUIR CORRETAMENTE AS NOMENCLATURAS DOS CAMPOS DAS TABELAS, E PRINCIPALMENTE DO NOME DO TÓPICO DO PUB/SUB, DO NOME DA SUBSCRIPTION E DO NOME DO SCHEMA!** 

Isso se dá porque por padrão o Debezium vai automaticamente repassar as mensagens do banco de dados para um tópico no Pub/Sub cujo nome deve ser EXATAMENTE ```<nomeDoServidor>.<nomeDoBanco>.<nomeDaTabela>```. Ou seja, como as configurações do Debezium estão para um servidor com nome *mysql*, banco *webinar* e tabela *produtos* (vide arquivo ```Debezium_MySQL/conf/application.properties```), se alterar os nomes das coisas do Pub/Sub você vai ter que ajeitar toda a estrutura de nomenclatura. 

**TL;DR**: não troque o nome de nada a menos que saiba o que está fazendo!

## Comandos GCP
1. Primeiro vamos criar uma tabela no BigQuery para receber os dados. Entre na interface do BQ, crie um dataset de nome ```webinar_streaming_01``` e depois abra uma nova aba para executar a seguinte query:
```
CREATE OR REPLACE TABLE webinar_streaming_01.raw_produtos_structured (
  id INT64 NOT NULL,
  name STRING,
  description STRING,
  price float64,
  created_timestamp INT64,
  qty_in_stock INT64,
  __op STRING,
  __table STRING,
  __source_ts_ms INT64,
  __deleted STRING
);
```

2. Abra o Cloud Shell na parte superior da interface da GCP e salve a variável $PROJECT_ID com o ID do seu projeto na GCP executando o comando:
```
export PROJECT_ID=$(gcloud config get-value project)
```

3. Execute o seguinte comando para criar o *schema* do Pub/Sub:
```
gcloud pubsub schemas create webinar.produtos.produtos-schema \
--type=AVRO \
--definition='{"type":"record","name":"SchemaProdutosWebinar","fields":[{"type":"int","optional":false,"name":"id"},{"type":"string","optional":false,"name":"name"},{"type":"string","optional":false,"name":"description"},{"type":"float","optional":false,"name":"price"},{"type":"int","optional":false,"name":"created_timestamp"},{"type":"int","optional":false,"name":"qty_in_stock"},{"type":"string","optional":true,"name":"__op"},{"type":"string","optional":true,"name":"__table"},{"type":"long","optional":true,"name":"__source_ts_ms"},{"type":"string","optional":true,"name":"__deleted"}]}'
```

4. Execute em seguida o seguinte comando para criar o tópico:
```
gcloud pubsub topics create mysql.webinar.produtos --message-encoding=json --schema=webinar.produtos.produtos-schema
```

5. Em seguida, crie a inscrição do BQ no tópico que foi criado com o seguinte comando:
```
gcloud pubsub subscriptions create \
 mysql.webinar.produtos-bq-subscription \
 --topic mysql.webinar.produtos \
 --bigquery-table=$PROJECT_ID:webinar_streaming_01.raw_produtos_structured \
 --use-topic-schema
```

Pronto! A parte GCP está pronta, agora precisamos ajeitar o servidor local com o MySQL e o Debezium!

## Comandos Docker para Debezium e MySQL

1. Clone ou baixe este repositório e salve localmente no seu computador. Tenha certeza que o Docker está rodando na sua máquina.

2. Ajeite o arquivo ```demo-sa.json``` para que o mesmo contenha os dados da sua conta de serviço com permissões, como explicado anteriormente. Não troque o nome do arquivo pois o docker-compose está procurando pela chave com esse nome "demo-sa.json" especificamente. Apenas substitua o conteúdo do arquivo pelo conteúdo da chave baixada.

3. Edite o arquivo ```Debezium_MySQL/conf/application.properties``` trocando apenas a segunda linha do mesmo. Troque o termo "bix-tecnologia-dev" pelo ID do seu projeto na GCP. Neste arquivo você vai poder depois configurar mais um monte de coisa do Debezium, quando tiver mais familiarizado com o mesmo.

4. Abra a linha de comando na pasta ```Debezium_MySQL/``` (pasta que contém o arquivo *docker-compose.yaml*) e execute o comando:
```
docker-compose up
```

5. Pronto, está tudo rodando! Agora você deverá usar sua interface de conexão com BDs preferida (uso DBeaver), conectar no seu banco usando as credenciais:
```
Server host: localhost
Port: 3306
Username: root
Password: debezium
```

6. Como falado anteriormente, o Debezium leva em consideração o nome da conexão do servido, do banco e da tabela para definir para qual tópico vai enviar a mensagem. Dessa forma, será necessário criar um novo banco de dados chamado ```webinar```, e dentro dele uma tabela chamada ```produtos```. Esta tabela deve possuir o schema apresentado no webinar. Sendo assim, deve ficar algo exatamente como o apresentado:
![schema](https://i.imgur.com/4hT07gU.png)

Pronto! Tudo configurado. Agora você pode executar queries, inserir dados, mexer no que quiser na tabela de produtos no banco, e as mudanças serão capturadas por CDC pelo Debezium e enviadas ao BQ automaticamente em tempo real, assim como visto no webinar!

# Comandos extras para executar no MySQL para testar

```sql
-- Inserir dado na tabela de produtos
INSERT INTO webinar.produtos 
(id, name, description, price, created_timestamp, qty_in_stock) VALUES 
(1, 'Garrafinha de água', 'Garrafa térmica 500ml na cor azul', 25.99, 1662589282, 30)


-- Deletar dado na tabela de produtos
DELETE FROM webinar.produtos WHERE id = 1


-- Atualizar dado na tabela de produtos
UPDATE webinar.produtos
SET 
	name = 'Garrafinha de água editada', 
	description = 'Promoção de garrafa térmica 500ml na cor azul',
	price = 15.99
WHERE id = 1
```

```sql
-- Cria procedure para inserir dados automaticamente na tabela produtos
DROP PROCEDURE IF EXISTS webinar.insert_data;

CREATE PROCEDURE webinar.insert_data()
BEGIN
    INSERT INTO produtos (
    	name, 
    	description, 
    	price, 
    	created_timestamp, 
    	qty_in_stock) VALUES (
			LEFT(UUID(), 5), 
			LEFT(UUID(), 20), 
			(RAND() * 1000), 
			UNIX_TIMESTAMP(), 
			FLOOR(1 + RAND() * 50));
END


-- Deleta evento do scheduler do MySQL
DROP EVENT IF EXISTS webinar.evento_insercao


-- Cria evento do scheduler do MySQL a cada 5 segundos
CREATE EVENT webinar.evento_insercao
    ON SCHEDULE EVERY 5 SECOND
    DO
      CALL webinar.insert_data;
     

-- Ativa ou desativa o scheduler
SET GLOBAL event_scheduler = 1;
SET GLOBAL event_scheduler = 0;
```

# Comandos no BQ mostrados:
```sql
-- Consulta tabela de dados no BQ crua com as informações completas de CDC
SELECT * FROM webinar_streaming_01.raw_produtos_structured;
```

```sql
-- Cria view que remonta os dados mais atualizados do BQ
CREATE OR REPLACE VIEW `webinar_streaming_01.vw_replica` AS (
  SELECT t1.* FROM `webinar_streaming_01.raw_produtos_structured` AS t1
    WHERE t1.__source_ts_ms = (
      SELECT(MAX(t2.__source_ts_ms)) 
      FROM `webinar_streaming_01.raw_produtos_structured` AS t2 
      WHERE t1.id = t2.id)
    AND __deleted = 'false'
);

-- Select na view para observar os dados
SELECT * FROM `webinar_streaming_01.vw_replica`;
```
