# Testando a conexão com certificados do Open Finance Brasil

Para realizar os testes você precisará de um certificado BRCAC (também conhecido como certificado de transporte ou aplicação) para continuar. Como na etapa anterior eu adicionei as AC's de sandbox, irei utilizar também um certificado BRCAC de sandbox para realizar os testes.

Durante os próximos passos, irei me referir a chave privada como _brcac.key_ e a chave pública como _brcac.pem_, pois estes são os nomes padrões dos arquivos gerados pelo diretório de sandbox, portanto altere a referência a estes arquivos para o nome dos seus arquivos.

Também copiei os arquivos _brcac.key_ e _brcac.pem_ para o diretório ssl do servidor Kafka para facilitar os comandos. Leve isso em consideração quando estiver rodando os comandos.

Os comandos serão executados levando em consideração que estamos no diretório raiz do servidor Kafka.

## Criando a keystore do novo consumer
Agora que temos um novo consumer, precisaremos criar uma keystore onde as chaves _brcac.key_ e _brcac.pem_ serão armazenadas para utilizarmos na conexão com o servidor Kafka.

Crie a keystore com uma chave dummy temporária (porque o keytool não permite criar uma keystore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-consumer-ofb-keystore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a keystore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-consumer-ofb-keystore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Para importar o certificado BRCAC dentro da keystore, precisamos fazer 2 passos. Primeiro temos que unir as chaves pública e privada em um arquivo p12 e então importar na keystore.
Crie o arquivo p12 com o comando abaixo
```
openssl pkcs12 -export -inkey ssl/brcac.key -in ssl/brcac.pem -name consumer-ofb -out ssl/consumer-ofb.p12 -password pass:teste123
```
_Obs: Altere a senha **teste123** por uma senha temporária, pois o arquivo será descartado em seguida_

Importe o arquivo p12 do servidor Kafka dentro da keystore
```
keytool -importkeystore -destkeystore ssl/kafka-consumer-ofb-keystore.jks -srckeystore ssl/consumer-ofb.p12 -srcstoretype PKCS12 -alias consumer-ofb -storepass teste123 -srcstorepass teste123
```
_Obs: Altere a senha **teste123** do atributo -storepass pela senha informada no primeiro comando e a senha **teste123** do atributo -scrstorepass pela senha informada no comando anterior_

Pronto! Já temos a keystore do consumer com certificado do Open Finance Brasil.

## Criando a truststore do novo consumer
A truststore deste consumer é a mesma do anterior, portanto faremos apenas uma cópia para termos arquivos separados:

```
cp ssl/kafka-consumer-truststore.jks ssl/kafka-consumer-ofb-truststore.jks
```

## Criando o arquivo consumer-ofb.properties
Vamos também criar um novo arquivo de pametrização para esse novo consumer.

Faça download do arquivo [consumer-ofb.properties](consumer-ofb.properties) e salve no diretório _config_ do seu servidor Kafka.

## Criando a ACL para o novo consumer
Se você apenas tentar conectar no tópico _ssl_test_topic_ com o comando abaixo, receberá uma mensagem de erro do tipo **TOPIC_AUTHORIZATION_FAILED**:

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9095 --topic ssl_test_topic --consumer.config config/consumer-ofb.properties --from-beginning
```

Isso acontece porque apesar de agora aceitarmos certificados emitidos por outras AC's e o BRCAC ter sido assinado por uma destas AC's, a gente ainda não definiu qual/quais tópicos esse certificado pode acessar.


**Antes de irmos para o próximo passo, você precisa entender uma questão sobre o subject de certificados e subject de certificados do Open Finance Brasil.**

Quando extraímos o subject de um certificado através do comando openssl que usamos anteriormente, ele irá mostrar o certificado em formato _human readable(legível para humanos)_:

```
openssl x509 -in ssl/brcac.pem -noout -subject -nameopt RFC2253
```

A saída desse comando será algo parecido com:

```
subject=UID=91630aa7-0537-486c-851c-39791314538a,organizationIdentifier=OFBBR-7292c33f-e95a-5fe7-8f27-gg7a95c68b55,jurisdictionC=BR,businessCategory=Private Organization,serialNumber=54246410000166,CN=meubanco.com.br,O=MEU BANCO S.A.,L=SAO PAULO,ST=SP,C=BR
```

No entanto, não é exatamente esse subject que servidor Kafka irá fazer a comparação. Ele irá usar outros atributos do certificado para entender quais atributos do subject devem ser interpretados como _human readable_ e quais devem ser interpretados como ASN.1.

No caso do Open Finance Brasil, os atributos abaixo devem ser mantidos no formato ASN.1:
- organizationIdentifier
- jurisdictionC
- businessCategory
- serialNumber

Sabendo dessa informação, precisaremos montar o subject para o comando kafka-acl manualmente. Vamos lá:

Primeiro, rode o comando abaixo para gerar o subject quebrado linha a linha com todos os atributos no formato _human readable_:

```
openssl x509 -subject -noout -nameopt rfc2253 -nameopt sep_multiline -in ssl/brcac.pem
```

O resultado será algo semlhante a isto:
```
subject=
    UID=91630aa7-0537-486c-851c-39791314538a
    organizationIdentifier=OFBBR-7292c33f-e95a-5fe7-8f27-gg7a95c68b55
    jurisdictionC=BR
    businessCategory=Private Organization
    serialNumber=54246410000166
    CN=meubanco.com.br
    O=MEU BANCO S.A.
    L=SAO PAULO
    ST=SP
    C=BR
```

Segundo, rode o comando abaixo para gerar o subject quebrado linha a linha com todos os atributos no formato _ASN.1_:

```
openssl x509 -subject -noout -nameopt rfc2253 -nameopt dump_all -nameopt oid -nameopt sep_multiline -in ssl/brcac.pem
```
O resultado será algo semlhante a isto:
```
subject=
    0.9.2342.19200300.100.1.1=#132439313633306161372D303533372D343836632D383531632D333937393133313435333861
    2.5.4.97=#132A4F464242522D38323932633333652D643935612D356665372D386632372D646437613935633638623535
    1.3.6.1.4.1.311.60.2.1.3=#13024252
    2.5.4.15=#131450726976617465204F7267616E697A6174696F6E
    2.5.4.5=#130E3435323436343130303030313535
    2.5.4.3=#131262616E636F67656E69616C2E636F6D2E6272
    2.5.4.10=#131142414E434F2047454E49414C20532E412E
    2.5.4.7=#130E52494F204445204A414E4549524F
    2.5.4.8=#1302524A
    2.5.4.6=#13024252
```
>Note que a ordem dos atributos é mesma, a diferença é apenas o formato, portanto o atributo da primeira linha em _human readable_ é o atributo da primeira linha no formato _ASN.1_.

Terceiro, vamos pegar o subject em formato _human readable_ e substituir os atributos que precisam estar em _ASN.1_ (organizationIdentifier,jurisdictionC,businessCategory,serialNumber) para seu resultado no segundo subject. Fazendo isso manualmente, seu resultado deve algo parecido com o abaixo:

```
subject=
    UID=91630aa7-0537-486c-851c-39791314538a
    2.5.4.97=#132A4F464242522D38323932633333652D643935612D356665372D386632372D646437613935633638623535
    1.3.6.1.4.1.311.60.2.1.3=#13024252
    2.5.4.15=#131450726976617465204F7267616E697A6174696F6E
    2.5.4.5=#130E3435323436343130303030313535
    CN=meubanco.com.br
    O=MEU BANCO S.A.
    L=SAO PAULO
    ST=SP
    C=BR
```

Agora precisamo descartar a primeira linha (subject=) e nas demais, vamos remover os espaços vázios a esquerda e trocar a quebra de linha por vírgula (,), assim você terá um subject semelhante ao abaixo:

```
UID=91630aa7-0537-486c-851c-39791314538a,2.5.4.97=#132A4F464242522D38323932633333652D643935612D356665372D386632372D646437613935633638623535,1.3.6.1.4.1.311.60.2.1.3=#13024252,2.5.4.15=#131450726976617465204F7267616E697A6174696F6E,2.5.4.5=#130E3435323436343130303030313535,CN=meubanco.com.br,O=MEU BANCO S.A.,L=SAO PAULO,ST=SP,C=BR
```

Pronto, agora sim podemos rodar o comando kafka-acl para permitir que o detentor deste certificado possa se conectar ao tópico _ssl_test_topic_ como consumer:

```
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal "User:UID=91630aa7-0537-486c-851c-39791314538a,2.5.4.97=#132A4F464242522D38323932633333652D643935612D356665372D386632372D646437613935633638623535,1.3.6.1.4.1.311.60.2.1.3=#13024252,2.5.4.15=#131450726976617465204F7267616E697A6174696F6E,2.5.4.5=#130E3435323436343130303030313535,CN=meubanco.com.br,O=MEU BANCO S.A.,L=SAO PAULO,ST=SP,C=BR" --consumer --topic 'ssl_test_topic' --group '*'
```

## Testando a conexão do consumer
Agora você pode rodar o comando abaixo e caso tudo esteja certo, ele irá consumir os eventos que já foram enviados e os próximos que virão:

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9095 --topic ssl_test_topic --consumer.config config/consumer-ofb.properties --from-beginning
```

