# Adicionando as Autoridades Certificadoras do Open Finance Brasil ao servidor Kafka

Agora que já temos tudo configurado, precisamos adicionar as Autoridades Certificadoras do Open Finance como _confiáveis_ no servidor Kafka para que ele aceite conexões utilizando certificados que foram emitidos pela autoridades certificadoras homologadas pelo Open Finance.

Para buscar essa lista de certificados, vamos utilizar o projeto [o2b2-update-acs](https://github.com/ranierimazili/o2b2-update-acs/) que já faz a extração das autoridades certificadoras a partir do Diretório de Participantes.

Fora da pasta do servidor Kafka, clone o repositório do projeto:
```
git clone https://github.com/ranierimazili/o2b2-update-acs.git
```

Entre no diretório e instale as dependências:
```
cd o2b2-update-acs
npm install
```

Execute o comando para gerar os certificados das AC's:
```
node index.js sandbox --explode
```
> Obs: No caso acima, iremos usar as AC's de sandbox(homologação) para testar contra certificados assinados pelo diretório, em um cenário de produção, substitua **sandbox** por **production** para gerar os certificados de AC's de produção.

Após rodar o comando acima, na pasta _certs_ você encontrará uma série de certificados com a extensão _.crt_, são estes arquivos que precisaremos importar na keystore do servidor Kafka.

Para facilitar um pouco, copie a pasta _certs_ do projeto o2b2-update-acs para dentro da pasta _ssl_ do servidor Kafka. No meu cenário fica algo parecido com:

```
cp -r certs/ ../kafka_2.13-3.9.0/ssl/
```

Agora entre no diretório _ssl_ do servidor Kafka

```
cd ../kafka_2.13-3.9.0/ssl/
```

E importe os arquivos com a extensão _.crt_ para a truststore.
Você pode importar os arquivos um a um com o comando abaixo ou importar todos de uma vez se estiver usando o bash do linux:

#### Modo 1 a 1
Para cada arquivo com a extensão _.crt_, rode o comando abaixo:
```
keytool -importcert -alias ca-cert -file certs/<NOME DO CERTIFICADO> -keystore kafka-server-truststore.jks -storepass teste123 -noprompt
```
_Obs: Altere a senha **teste123** pela senha informada da sua truststore e o \<NOME DO CERTIFICADO> para o nome do arquivo .crt que está sendo importado_

#### Importar todos com apenas um comando (bash)
Basta colar o shell script abaixo que todos os arquivos com extensão _.crt_ serão importados

```
for cert in `ls certs/*.crt`; do
    ALIAS=`basename $cert .crt`
    keytool -importcert -alias $ALIAS -file $cert -keystore kafka-server-truststore.jks -storepass teste123 -noprompt;
done
```
_Obs: Altere a senha **teste123** pela senha informada da sua truststore_

Se tudo deu certo, você poderá verificar que as AC's foram incluídas na truststore através do comando _list_ do _keytool_:
```
keytool -list -keystore kafka-server-truststore.jks -storepass teste123
```
_Obs: Altere a senha **teste123** pela senha informada da sua truststore_

Pronto! Agora basta iniciar/reiniciar o servidor Kafka para que as alterações sejam aplicadas.

Agora você pode ir para o próximo passo: [Testando a conexão com certificados do Open Finance Brasil](./kafka-test-ca-ofb.md)