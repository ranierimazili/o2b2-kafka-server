# o2b2-kafka-server
Este projeto tem como objetivo demonstrar a configuração básica de um servidor Kafka que atenda aos requisitos estabelecidos no Open Finance Brasil.

O desenho abaixo é um exemplo (sem todos os componentes) de como essa arquitetura poderia ser desenhada e ilustrar em que parte iremos trabalhar.

Neste repositório iremos trabalhar nos componentes que ficam na DMZ, que são o Servidor Kafka e as APIs de gerenciamento dos tópicos.

![ofb-kafka drawio](https://github.com/user-attachments/assets/3fcfb3d9-9d73-413a-8c45-0475b65af58e)

## Índice
1. [Configurando a Autoridade Certificadora interna](./docs/kafka-ca.md)
2. [Iniciando e testando o servidor Kafka](./docs/kafka-binaries.md)
3. [Configurando as ACL's](./docs/kafka-acl.md)
4. [Adicionando as Autoridades Certificadoras do Open Finance Brasil](./docs/kafka-ca-ofb.md)
5. [Testando a conexão com certificados do Open Finance Brasil](./docs/kafka-test-ca-ofb.md)