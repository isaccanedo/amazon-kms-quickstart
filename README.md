# Demonstração do Quarkus: Amazon KMS Client

Este exemplo mostra como usar o cliente AWS KMS com o Quarkus. Como pré-requisito, instale a [interface de linha de comando da AWS](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

# Instância local do AWS KMS

Basta executá-lo da seguinte maneira para iniciar o KMS localmente:
`docker run --rm --name local-kms -p 8011:4599 -e SERVICES=kms -e START_WEB=0 -d localstack/localstack:0.11.1`
KMS listens on `localhost:8011` for REST endpoints.

Crie um perfil da AWS para sua instância local usando a AWS CLI:

```
$ aws configure --profile localstack
AWS Access Key ID [None]: test-key
AWS Secret Access Key [None]: test-secret
Default region name [None]: us-east-1
Default output format [None]:
```

## Criar chave mestra do KMS

Crie uma chave mestra e armazene o ARN na variável de ambiente `MASTER_KEY_ARN`
```
$> MASTER_KEY_ARN=`aws kms create-key --profile localstack --endpoint-url=http://localhost:8011 | cut -f3`
```
Gere dados de chave como chave simétrica de 256 bits (AES_256)
```
$> aws kms generate-data-key --key-id $MASTER_KEY_ARN --key-spec AES_256 --profile localstack --endpoint-url=http://localhost:8011
```

# Execute a demonstração no modo dev

- Run `./mvnw clean package` and then `java -Dkey.arn=$MASTER_KEY_ARN -jar ./target/quarkus-app/quarkus-run.jar`
- In dev mode `./mvnw clean quarkus:dev -Dkey.arn=$MASTER_KEY_ARN`

## Criptografe seu texto
```
curl -XPOST -H"Content-type: text/plain" http://localhost:8080/sync/encrypt -d'Quarkus is awsome'
```
E o resultado semelhante a esta saída:
```
S2Fybjphd3M6a21zOnVzLWVhc3QtMTowMDAwMDAwMDAwMDA6a2V5LzZmYzAwOWZhLWYwMDUtNGI4My04ZDc1LTk4OGVhZTk4ZTM1NwAAAAAfC2HyHrHBXLNFomHLdH9PlMKWQKofyhJjbY2TUovEaBuc4Hj+Lb2BSoYTa/c=
```
## Descriptografar sua mensagem
Agora você pode descriptografar uma mensagem criptografada anteriormente

```
curl -XPOST -H"Content-type: text/plain" http://localhost:8080/sync/decrypt -d '<encrypted-message>'
```

Repita o mesmo usando endpoints assíncronos. Criptografar
```
curl -XPOST -H"Content-type: text/plain" http://localhost:8080/async/encrypt -d 'Quarkus is awsome'
```
E então descriptografar
```
curl -XPOST -H"Content-type: text/plain" http://localhost:8080/async/decrypt -d '<encrypted-message>'
```

# Executando em nativo

Você pode compilar o aplicativo em um binário nativo usando:

`./mvnw clean install -Pnative`

e execute com:

`./target/amazon-kms-quickstart-1.0.0-SNAPSHOT-runner -Dkey.arn=$MASTER_KEY_ARN` 


# Executando nativo em contêiner

Crie uma imagem nativa no contêiner executando:
`./mvnw package -Pnative -Dnative-image.docker-build=true`

Build a docker image:
`docker build -f src/main/docker/Dockerfile.native -t quarkus/amazon-kms-quickstart .`

Create a network that connect your container with localstack
`docker network create localstack`

Stop your localstack container you started at the beginning
`docker stop local-kms`

Inicie o localstack e conecte-se à rede
`docker run --rm --network=localstack --name localstack -p 8011:4599 -e SERVICES=kms -e START_WEB=0 -d localstack/localstack:0.11.1`

Crie uma chave mestra e armazene o ARN na variável de ambiente `CMK_ARN`
```
$> MASTER_KEY_ARN=`aws kms create-key --profile localstack --endpoint-url=http://localhost:8011 | cut -f3`
```
Gere dados de chave como chave simétrica de 256 bits (AES_256)
```
$> aws kms generate-data-key --key-id $MASTER_KEY_ARN --key-spec AES_256 --profile localstack --endpoint-url=http://localhost:8011
```
Run quickstart container connected to that network (note that we're using internal port of the localstack)
`docker run -i --rm --network=localstack -p 8080:8080 quarkus/amazon-kms-quickstart -Dquarkus.kms.endpoint-override=http://localstack:4599`
