# Configurando ACL's

Até aqui você configurou o servidor Kafka e também um producer e um consumer, no entanto, o producer e o consumer, com seus respectivos certificados podem acessar qualquer tópico e não é este o comportamento que queremos, então vamos tornar o servidor mais restritivo.

Edite o arquivo _config/kraft/server-ofb.properties_ e descomente as últimas 3 linhas que existem nele para que fiquem com as opções abaixo ativas.

```properties
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
allow.everyone.if.no.acl.found=false
super.users=User:ANONYMOUS
```

As linhas acima estão definindo que agora o servidor Kafka deve:
- utilizar um autorizador;
- que este autorizador por padrão vai negar qualquer conexão que não tenha recebido explicitamente uma permissão de conexão;
- e definindo o usuário ANONYMOUS como super usuário para que o servidor consiga realizar as tarefas administrativas

Após editar as configurações, reinicie o servidor Kafka para que alterações sejam aplicadas.

Após isso, se você tentar se conectar como producer, o prompt ```>``` será aberto, mas ao tentar enviar qualquer evento, você receberá um erro do tipo ```TOPIC_AUTHORIZATION_FAILED```.

O mesmo acontecerá se você tentar se conectar como consumer, a diferença é que o prompt nem ficará aguardando os eventos.

## Criando ACL para o producer

Agora que já conferimos que as ACL's estão restringindo o acesso, vamos habilitar o acesso do producer com base no subject de seu certificado.

Primeiro, precisamos extrair o subject do certificado, para isso vamos usar o comando abaixo:

```
openssl x509 -in ssl/kafka-producer.crt -noout -subject -nameopt RFC2253
```

A resposta deste comando deve ser:

```subject=CN=producer```

O subject real é essa string porém sem o prefixo **subject=**, portanto no comando iremos usar apenas o restante (**CN=producer**).

Para dar permissão como producer no tópico _ssl_test_topic_ para o usuário com o certificado com o subject _CN=producer_, execute o comando abaixo:

```
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal "User:CN=producer" --producer --topic 'ssl_test_topic'
```

Pronto! Agora você conseguirá se conectar ao tópico novamente como producer e enviar eventos com o mesmo comando que utilizamos anteriormente:

```
bin/kafka-console-producer.sh --bootstrap-server localhost:9094 --topic ssl_test_topic --producer.config config/producer.properties
```

## Criando ACL para o consumer

Vamos fazer o mesmo, porém agora para o consumer.

Primeiro, precisamos extrair o subject do certificado, para isso vamos usar o comando abaixo:

```
openssl x509 -in ssl/kafka-consumer.crt -noout -subject -nameopt RFC2253
```

A resposta deste comando deve ser:

```subject=CN=consumer```

O subject real é essa string porém sem o prefixo **subject=**, portanto no comando iremos usar apenas o restante (**CN=consumer**).

Para dar permissão como consumer no tópico _ssl_test_topic_ para o usuário com o certificado com o subject _CN=consumer_, execute o comando abaixo:

```
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal "User:CN=consumer" --consumer --topic 'ssl_test_topic' --group '*'
```

Pronto! Agora você conseguirá se conectar ao tópico novamente como consumer e receberá os eventos com o mesmo comando que utilizamos anteriormente:

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9095 --topic ssl_test_topic --consumer.config config/consumer.properties --from-beginning
```