# o2b2-kafka-server
Este projeto tem como objetivo demonstrar a configuração básica de um servidor Kafka que atenda aos requisitos estabelecidos no Open Finance Brasil.

O desenho abaixo é um exemplo (sem todos os componentes) de como essa arquitetura poderia ser desenhada e ilustrar em que parte iremos trabalhar.

Neste repositório iremos trabalhar nos componentes que ficam na DMZ, que são o Servidor Kafka e as APIs de gerenciamento dos tópicos.

![ofb-kafka drawio](https://github.com/user-attachments/assets/3fcfb3d9-9d73-413a-8c45-0475b65af58e)

# Configurando os certificados
No Open Finance Brasil foi definido que a comunicação entre o servidor Kafka e seus clientes seria exclusivamente utilizando mTLS, portanto a geração de certificado do servidor Kafka e a configuração da truststore para permitir a utilização de certificados BRCAC(também conhecidos como certificado de transporte ou aplicação) para os clientes (consumers) é parte fundamental deste processo.

Ainda não foi definido se o certificado do servidor Kafka precisa ser um certificado ICP Brasil (assim como é para os authorization servers), portanto neste exemplo utilizamos certificados auto-assinados para o servidor Kafka e para os produtores (producers).
Os clients (consumers) precisarão apresentar um certificado BRCAC emitido por uma das autoridades certificadores homologadas para o Open Finance Brasil.

## Criando a Autoridade Certificadora para certificados auto-assinados
Antes de começar, vamos criar uma pasta onde os arquivos serão gerados:

```
mkdir ssl
```

Como iremos trabalhar com certificados auto-assinado tanto para o servidor Kafka quanto para os produtores (producers), precisamos gerar o certificado da AC que irá assinar os demais certificados.

```
openssl \
    req -new -x509 \
    -keyout ssl/ca-key \
    -out ssl/ca-cert \
    -days 3650 \
    -subj "/CN=Autoridade Certificadora de Auto-assinados RaniBank" \
    -nodes
```

O comando acima deve ter criado a chave privada (ca-key) da nossa AC que será usada para assinar os demais certificados e também deve ter criado a chave pública (ca-cert) da AC, que será utilizado posteriomente na truststore para que permita conexões com certificados que foram assinados por essa AC.

## Criando o certificado para o servidor Kafka
Primeiro vamos gerar a chave privada

```
openssl genrsa -out ssl/kafka-server.key 2048
```
Vamos agora gerar o pedido de certificado para ser assinado pela nossa AC

```
openssl req -new -key ssl/kafka-server.key -out ssl/kafka-server.csr -subj "/CN=kafka.ranieri.dev.br"
```
_Obs: Altere a CN do comando acima para o seu host_

Agora vamos assinar o certificado com nossa AC
```
openssl x509 -req -in ssl/kafka-server.csr -CA ssl/ca-cert -CAkey ssl/ca-key -CAcreateserial -out ssl/kafka-server.crt -days 3650 -sha256
```

Após executar os comandos acima você deve ter os arquivos _kafka-server.key_, _kafka-server.csr_ e _kafka-server.crt_ no seu diretório _ssl_. Destes 3, apenas 2 são úteis a partir de agora, então você pode apagar o arquivo _kafka-server.csr_ se quiser:
```
rm ssl/kafka-server.csr
```
Pronto! Já temos o certificado do servidor Kafka. Agora precisamos coloca-los no formato correto (JKS) para que seja possível sua utilização.

## Gerando o arquivo keystore em formato JKS
Por padrão, o Kafka utiliza arquivos JKS para configuração de keystore e truststore, então iremos agora fazer a criação destes arquivos e a inserção das chaves que geramos nos passos anteriores dentro dos arquivos JKS.

A keystore é o arquivo que contém o certificado do servidor Kafka, portanto iremos fazer a criação dele e em seguida inserir o certificado do servidor Kafka que foi gerado anteriormente.

Crie a keystore com uma chave dummy temporária (porque o keytool não permite criar uma keystore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-server-keystore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a keystore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-server-keystore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da keystore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-server-keystore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Para importar o certificado do servidor Kafka dentro da keystore, precisamos fazer 2 passos. Primeiro temos que unir as chaves pública e privada em um arquivo p12 e então importar na keystore.
Crie o arquivo p12 com o comando abaixo
```
openssl pkcs12 -export -inkey ssl/kafka-server.key -in ssl/kafka-server.crt -name kafka-server -out ssl/kafka-server.p12 -password pass:teste123
```
_Obs: Altere a senha **teste123** por uma senha temporária, pois o arquivo será descartado em seguida_

Importe o arquivo p12 do servidor Kafka dentro da keystore
```
keytool -importkeystore -destkeystore ssl/kafka-server-keystore.jks -srckeystore ssl/kafka-server.p12 -srcstoretype PKCS12 -alias kafka-server -storepass teste123 -srcstorepass teste123
```
_Obs: Altere a senha **teste123** do atributo -storepass pela senha informada no primeiro comando e a senha **teste123** do atributo -scrstorepass pela senha informada no comando anterior_

Pronto! Já temos a keystore do servidor Kafka.

## Gerando o arquivo truststore em formato JKS
A truststore é o arquivo que contêm os certificados das AC's que o servidor Kafka entenderá como confiáveis para emissão de certificados dos consumidores (consumers) e produtores (producers). Ou seja, quem tiver um certificado emitido por um AC que conste na truststore poderá ser conectar ao servidor Kafka. Neste primeiro momento iremos adicionar apenas nossa AC, mas em um passo futuro, iremos adicionar as ACs homologadas pelo Open Finance Brasil.

Crie a truststore com uma chave dummy temporária (porque o keytool não permite criar uma truststore vazia)
```
keytool -genkeypair -alias dummy -keyalg RSA -keystore ssl/kafka-server-truststore.jks -storepass teste123 -dname "CN=Dummy" -keypass teste123
```
_Obs: Altere a senha **teste123** por uma senha de sua preferência_

Deixa a trustore vazia apagando a chave dummy
```
keytool -delete -alias dummy -keystore ssl/kafka-server-truststore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada no comando anterior_

Importe o certificado da nossa AC dentro da truststore
```
keytool -importcert -alias ca-cert -file ssl/ca-cert -keystore ssl/kafka-server-truststore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada no primeiro comando_

Pronto! Já temos a truststore do servidor Kafka.

## Iniciando o servidor Kafka
Existem várias formas de se levantar um servidor Kafka, principalmente se considermos os serviços providos por cloud providers. Neste tutorial vou mostrar duas formas (execução dos binários ou docker) que são factiveis em provedores cloud e também para testes em sua própria máquina. Você só precisa executar uma das opções abaixo.

### Utilizando os binário
- [Iniciando e testando o servidor Kafka](kafka-binaries.md)
- Configurando ACL's

### Utilizandoo docker
- TODO


