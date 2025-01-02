# Iniciando e testando o servidor Kafka utilizando os binários

## Configurando e iniciando o Kafka
Baixe o arquivo binário do Kafka [aqui](https://kafka.apache.org/downloads).
No momento de escrita desse artigo, a versão mais atual é a 3.9.0.

Após o download, descompacte o arquivo e então copie a pasta _ssl_ onde todos os nossos arquivos foram gerados para dentro da pasta descompactada. A pasta _ssl_ deve ficar no mesmo nível da pasta _bin_, como no screenshot abaixo:

![image](https://github.com/user-attachments/assets/7ff36e20-eef7-4f93-9409-b8f826223d9c)

Agora vamos precisar criar o arquivo de configuração para levantarmos o servidor.

Faça download do arquivo [server-ofb.properties](./confs/server-ofb.properties) e copie o arquivo para a pasta _config/kraft_ que existe dentro da pasta descompactada do Kafka.

>Obs: Edite o arquivo _server-ofb.properties_, editando as senhas _teste123_ pela senha utilizada no seu keystore e truststore.

Agora você pode subir o servidor Kafka com os comandos abaixo:
```
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/kraft/server-ofb.properties

bin/kafka-server-start.sh config/kraft/server-ofb.properties
```

Pronto! Seu servidor Kafka já está funcionando com SSL e aguardando conexões.

## Criando um tópico para testes
Por padrão definimos a opção _auto.create.topics.enable=false_ para que o servidor Kafka não crie tópicos automaticamente quando uma conexão é realizada em um tópico inexistente, portanto precisamos criar um tópico para então testarmos as conexões dos producers e consumers.

O comando abaixo irá criar um tópico chamado ssl_test_topic:
```
bin/kafka-topics.sh --create --topic ssl_test_topic --bootstrap-server localhost:9092
```

## Testando o produtor(producer) Kafka
Os producers são os responsáveis por gravar os eventos nos tópicos. Como a gravação será restrita à agentes internos, faz sentido que os producers se conectem utilizando certificados assinados pela AC interna que criamos anteriormente, então abaixo iremos criar o certificado do producer e em seguida gerar o JKS para conectar ao servidor Kafka.

### Criando o certificado para o produtor (producer)
---

Primeiro vamos gerar a chave privada

```
openssl genrsa -out ssl/kafka-producer.key 2048
```
Vamos agora gerar o pedido de certificado para ser assinado pela nossa AC

```
openssl req -new -key ssl/kafka-producer.key -out ssl/kafka-producer.csr -subj "/CN=producer"
```

Agora vamos assinar o certificado com nossa AC
```
openssl x509 -req -in ssl/kafka-producer.csr -CA ssl/ca-cert -CAkey ssl/ca-key -CAcreateserial -out ssl/kafka-producer.crt -days 3650 -sha256
```

Após executar os comandos acima você deve ter os arquivos _kafka-producer.key_, _kafka-producer.csr_ e _kafka-producer.crt_ no seu diretório _ssl_. Destes 3, apenas 2 são úteis a partir de agora, então você pode apagar o arquivo _kafka-producer.csr_ se quiser:
```
rm ssl/kafka-producer.csr
```
Pronto! Já temos o certificado do producer. Agora precisamos colocar esse certificado dentro de uma keystore para que possamos utilizar ele com o comando _kafka-console-producer_.

### Gerando o arquivo keystore em formato JKS
---
Crie a keystore com uma chave dummy temporária (porque o keytool não permite criar uma keystore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-producer-keystore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a keystore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-producer-keystore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da keystore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-producer-keystore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Para importar o certificado do producer dentro da keystore, precisamos fazer 2 passos. Primeiro temos que unir as chaves pública e privada em um arquivo p12 e então importar na keystore.
Crie o arquivo p12 com o comando abaixo
```
openssl pkcs12 -export -inkey ssl/kafka-producer.key -in ssl/kafka-producer.crt -name kafka-producer -out ssl/kafka-producer.p12 -password pass:teste123
```
_Obs: Altere a senha **teste123** por uma senha temporária, pois o arquivo será descartado em seguida_

Importe o arquivo p12 producer dentro da keystore
```
keytool -importkeystore -destkeystore ssl/kafka-producer-keystore.jks -srckeystore ssl/kafka-producer.p12 -srcstoretype PKCS12 -alias kafka-producer -storepass teste123 -srcstorepass teste123
```
_Obs: Altere a senha **teste123** do atributo -storepass pela senha informada no primeiro comando e a senha **teste123** do atributo -scrstorepass pela senha informada no comando anterior_

Pronto! Já temos a keystore do producer.

### Gerando o arquivo truststore em formato JKS
---
Também precisaremos da truststore, senão o producer não confiará no certificado do servidor e a conexão não será realizada.

Crie a truststore com uma chave dummy temporária (porque o keytool não permite criar uma truststore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-producer-truststore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a truststore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-producer-truststore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da truststore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-producer-truststore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Pronto! Já temos a truststore do producer.

### Testando a conexão do producer
---
Copie o arquivo [producer.properties](./confs/producer.properties) para a pasta _config_ do kafka que você descompactou. 
>Obs: Edite o arquivo _producer.properties_, editando as senhas _teste123_ pela senha utilizada no seu keystore e truststore.

Rode o comando abaixo para se conectar ao servidor Kafka como producer:

```
bin/kafka-console-producer.sh --bootstrap-server localhost:9094 --topic ssl_test_topic --producer.config config/producer.properties
```

Se tudo deu certo você deve estar vendo um prompt aberto como abaixo:
```
>
```
Basta digitar qualquer conteúdo e dar ENTER para que o evento seja enviado.

## Testando o consumidor(consumer) Kafka
**Antes de começar, uma explicação...**

O certificado que será criado agora é apenas para um teste inicial, pois no ambiente final nós iremos permitir que certificados assinados por AC's homologadas no Open Finance ou a AC do Diretório de Participantes (sandbox) possam se conectar como consumidores. Portanto, tudo que iremos fazer nessa sessão, tem apenas a finalidade des teste para garantir que consumers com SSL estão funcionando corretamente.

### Criando o cerfificado para o consumidor (consumer)
---
Primeiro vamos gerar a chave privada

```
openssl genrsa -out ssl/kafka-consumer.key 2048
```
Vamos agora gerar o pedido de certificado para ser assinado pela nossa AC

```
openssl req -new -key ssl/kafka-consumer.key -out ssl/kafka-consumer.csr -subj "/CN=consumer"
```

Agora vamos assinar o certificado com nossa AC
```
openssl x509 -req -in ssl/kafka-consumer.csr -CA ssl/ca-cert -CAkey ssl/ca-key -CAcreateserial -out ssl/kafka-consumer.crt -days 3650 -sha256
```

Após executar os comandos acima você deve ter os arquivos _kafka-consumer.key_, _kafka-consumer.csr_ e _kafka-consumer.crt_ no seu diretório _ssl_. Destes 3, apenas 2 são úteis a partir de agora, então você pode apagar o arquivo _kafka-consumer.csr_ se quiser:
```
rm ssl/kafka-consumer.csr
```
Pronto! Já temos o certificado do consumer. Agora precisamos colocar esse certificado dentro de uma keystore para que possamos utilizar ele com o comando _kafka-console-consumer_.

### Gerando o arquivo keystore em formato JKS
---
Crie a keystore com uma chave dummy temporária (porque o keytool não permite criar uma keystore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-consumer-keystore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a keystore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-consumer-keystore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da keystore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-consumer-keystore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Para importar o certificado do consumer dentro da keystore, precisamos fazer 2 passos. Primeiro temos que unir as chaves pública e privada em um arquivo p12 e então importar na keystore.
Crie o arquivo p12 com o comando abaixo
```
openssl pkcs12 -export -inkey ssl/kafka-consumer.key -in ssl/kafka-consumer.crt -name kafka-consumer -out ssl/kafka-consumer.p12 -password pass:teste123
```
_Obs: Altere a senha **teste123** por uma senha temporária, pois o arquivo será descartado em seguida_

Importe o arquivo p12 do consumer dentro da keystore
```
keytool -importkeystore -destkeystore ssl/kafka-consumer-keystore.jks -srckeystore ssl/kafka-consumer.p12 -srcstoretype PKCS12 -alias kafka-consumer -storepass teste123 -srcstorepass teste123
```
_Obs: Altere a senha **teste123** do atributo -storepass pela senha informada no primeiro comando e a senha **teste123** do atributo -scrstorepass pela senha informada no comando anterior_

Pronto! Já temos a keystore do consumer.

### Gerando o arquivo truststore em formato JKS
---
Também precisaremos da truststore, senão o consumer não confiará no certificado do servidor e a conexão não será realizada.

Crie a truststore com uma chave dummy temporária (porque o keytool não permite criar uma truststore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-consumer-truststore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a truststore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-consumer-truststore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da truststore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-consumer-truststore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Pronto! Já temos a truststore do consumer.

### Testando a conexão do consumer
---
Copie o arquivo [consumer.properties](./confs/consumer.properties) para a pasta _config_ do kafka que você descompactou
>Obs: Edite o arquivo _consumer.properties_, editando as senhas _teste123_ pela senha utilizada no seu keystore e truststore.

Rode o comando abaixo para se conectar ao servidor Kafka como consumer:

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9095 --topic ssl_test_topic --consumer.config config/consumer.properties --from-beginning
```

Se tudo deu certo irá ver todos eventos que foram enviados e o prompt ficará travado esperando novos eventos chegarem.

Se você mantiver este prompt aberto e abrir o prompt do producer, você poderá mandar eventos no producer e os verá chegar no consumer em tempo real.

## Considerações
Se você chegou até aqui e deu tudo certo, ótimo.
Agora existe um ponto que precisa ser melhorado, a segurança.

Com essa configuração que fizemos, um consumer que possua um certificado válido poderá conectar em qualquer tópico e não é isso que queremos. Queremos que cada consumer, baseado no _subject_ de seu certificado, conecte apenas em tópicos específicos. Para isso precisamos implementar as ACL's.

Agora você pode ir para o próximo passo: [Configurando as ACL's](./kafka-acl.md)