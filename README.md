# todo-list-aws

API REST de lista de tareas (ToDo) construida con **AWS SAM** y desplegada mediante **CI/CD con Jenkins**.
Proyecto del curso de DevOps (UNIR): el objetivo no es la aplicación en sí, que es deliberadamente
simple, sino el ciclo completo de análisis estático, pruebas, despliegue y promoción entre entornos.

## Servicios AWS utilizados

Todos salen de `template.yaml`, que es una plantilla de CloudFormation con el transform
`AWS::Serverless-2016-10-31`:

| Servicio | Uso en el proyecto |
|---|---|
| **AWS Lambda** | Cinco funciones en Python 3.10, timeout de 3 s: `create`, `list`, `get`, `update`, `delete` |
| **Amazon API Gateway** | API REST creada implícitamente por los eventos `Type: Api` de cada función |
| **Amazon DynamoDB** | Tabla `${Stage}-TodosDynamoDbTable`, clave de partición `id` (String), capacidad aprovisionada 1 lectura / 1 escritura |
| **AWS CloudFormation** | Stacks `todo-list-aws-staging` y `todo-list-aws-production`, gestionados por SAM |
| **Amazon S3** | Bucket de artefactos del despliegue, resuelto automáticamente con `--resolve-s3` |
| **AWS IAM** | Las Lambdas asumen un rol existente, `arn:aws:iam::${AWS::AccountId}:role/LabRole` |

Región: `us-east-1`. El parámetro `Stage` admite `default`, `staging` o `production`, y determina tanto
el nombre de la tabla DynamoDB como la configuración de `samconfig.toml` que se aplica.

### Endpoints

| Método | Ruta | Función |
|---|---|---|
| POST | `/todos` | `create.create` |
| GET | `/todos` | `list.list` |
| GET | `/todos/{id}` | `get.get` |
| PUT | `/todos/{id}` | `update.update` |
| DELETE | `/todos/{id}` | `delete.delete` |

La URL base del stack desplegado se publica como el output `BaseUrlApi` de CloudFormation. Los pipelines
la leen con `aws cloudformation describe-stacks` y se la pasan a los tests de integración.

## Estructura

- **src** código de las funciones Lambda. `todoList.py` concentra el acceso a DynamoDB y respeta
  `ENDPOINT_OVERRIDE` para poder apuntar a un DynamoDB local
- **test/unit** pruebas unitarias con `moto` (DynamoDB simulado)
- **test/integration** pruebas de integración contra la API ya desplegada, vía `BASE_URL`
- **pipelines** Jenkinsfiles y scripts compartidos del CI/CD
- **template.yaml** definición de los recursos AWS
- **samconfig.toml** configuración de despliegue por entorno
- **localEnvironment.json** variables para levantar la aplicación en local contra DynamoDB en Docker

## Pipelines de Jenkins

### PIPELINE-FULL-STAGING

`pipelines/PIPELINE-FULL-STAGING/Jenkinsfile`

1. **SetUp** crea el virtualenv e instala radon, flake8, bandit, pytest, boto3, moto, mock y coverage
2. **Test** análisis estático y pruebas unitarias. Publica la cobertura con `publishCoverage` y
   un umbral de línea del 70 % (`failUnhealthy: true`), así que el build cae si baja de ahí
3. **Build** `sam validate --region us-east-1` seguido de `sam build`
4. **Deploy** `sam deploy --config-env staging`
5. **Integration Test after deploy** lee `BaseUrlApi` del stack y lanza la batería completa de tests

### PIPELINE-FULL-PRODUCTION

`pipelines/PIPELINE-FULL-PRODUCTION/Jenkinsfile`

1. **SetUp** virtualenv con awscli, aws-sam-cli, pytest y requests
2. **Build** `sam validate` y `sam build`
3. **Deploy** `sam deploy --config-env production`
4. **Integration Test after deploy** con `TEST_MODE=readonly`, que limita las pruebas a
   `pytest -k "list or get"`. En producción no se crean ni se borran registros

No hay etapa de análisis estático ni de pruebas unitarias: ya pasaron en staging, y a producción
solo llega código que superó ese pipeline.

### PIPELINE-FULL-CD

`pipelines/PIPELINE-FULL-CD/Jenkinsfile`, que enlaza los dos anteriores:

1. **Staging** invoca PIPELINE-FULL-STAGING
2. **Merge** fusiona `develop` en `master` y hace push con la credencial SSH `github`
3. **Production** invoca PIPELINE-FULL-PRODUCTION

### Jenkinsfile_agentes

Variante del pipeline completo repartida entre **agentes distribuidos**, con `agent none` a nivel de
pipeline y una etiqueta distinta por etapa: `default` para el código, la construcción y el despliegue,
`static-test` para el análisis estático e `integration-test` para las pruebas de integración. El código
viaja entre agentes con `stash`/`unstash`, y la URL de la API se pasa igual, como el stash `api-url`.

Acepta los parámetros `ENVIRONMENT` (staging o production) y `BRANCH`. La etapa **Promote**, que fusiona
`develop` en `master`, solo se ejecuta cuando `ENVIRONMENT` es staging. Todos los pipelines terminan con
`cleanWs()` en el bloque `post { always }`.

## Calidad del código

El análisis estático (`pipelines/PIPELINE-FULL-STAGING/static_test.sh`) encadena cuatro herramientas y
aborta a la primera que falle:

```bash
radon cc src -nc     # complejidad ciclomática: ningún bloque por debajo de C
radon mi src -nc     # índice de mantenibilidad
flake8 src/*.py      # estilo
bandit src/*.py      # seguridad
```

Las pruebas unitarias (`unit_test.sh`) se ejecutan contra la tabla `todoUnitTestsTable` y miden
cobertura solo sobre `src/todoList.py`, que es donde está la lógica.

## Despliegue manual

Requisitos: [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html),
Python 3 y [Docker](https://hub.docker.com/search/?type=edition&offering=community).

```bash
sam build
sam deploy --config-env staging
sam deploy --config-env production
```

La primera vez, o para regenerar `samconfig.toml`:

```bash
sam deploy --guided
```

## Ejecución en local

Levanta DynamoDB en Docker y apunta las Lambdas contra él:

```bash
docker network create sam
docker run -p 8000:8000 --network sam --name dynamodb -d amazon/dynamodb-local
aws dynamodb create-table --table-name local-TodosDynamoDbTable \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
  --endpoint-url http://localhost:8000
sam build
sam local start-api --port 8080 --env-vars localEnvironment.json --docker-network sam
```

## Tests

Unitarios:

```bash
python -m pip install boto3 moto mock==4.0.2 coverage==4.5.4
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
export DYNAMODB_TABLE=todoUnitTestsTable
python test/unit/TestToDo.py
```

Integración, contra una API ya desplegada:

```bash
python -m pip install pytest requests
export BASE_URL=https://<id>.execute-api.us-east-1.amazonaws.com/Prod
pytest -s test/integration/todoApiTest.py
```

`BASE_URL` se lee siempre del entorno. No se guarda ninguna URL de API en el repositorio.

## Logs

```bash
sam logs -n GetTodoFunction --stack-name todo-list-aws-staging
```

## Limpieza

```bash
aws cloudformation delete-stack --stack-name todo-list-aws-staging
aws cloudformation delete-stack --stack-name todo-list-aws-production
```

## Nota sobre versiones de Python

`template.yaml` declara `Runtime: python3.10` para las Lambdas, mientras que los `setup.sh` de los
pipelines crean el virtualenv con `python3.7`. Son entornos distintos: uno es el runtime de ejecución en
AWS y el otro el del agente de Jenkins que construye y prueba. Conviene unificarlos.
