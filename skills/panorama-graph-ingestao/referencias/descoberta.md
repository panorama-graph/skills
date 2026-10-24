# Onde descobrir cada fato

Este arquivo diz onde procurar cada assunto, no código e na configuração, e em cada acesso. Comece pelo que a fonte declara: configuração, anotações, manifestos, registros. Só depois leia o fluxo do código.

Anote cada achado no `DESCOBERTA.md` com a evidência:
- **No repositório:** `caminho#L<linha>` ou `caminho#<função>`, relativo à raiz.
- **Fora do repositório:** a URL do registro, ou o objeto lido (`<cluster>/<namespace>/<Kind>/<nome>`).

O que nenhuma seção cobre não entra. Registre no `LACUNAS.md` o que viu e por que não resolveu.

## Acessos

Faça o inventário antes de descobrir. Cada acesso só lê:

| Acesso | Para quê | Leituras | Nunca |
|---|---|---|---|
| Repositório local | Tudo o que está no código | `git remote get-url origin`, `git rev-parse HEAD`, `git status --porcelain`, ler arquivos | Escrever, rodar o build ou instalar dependências sem a pessoa permitir |
| `gh` / `glab` | Repositórios que não estão na máquina, `CODEOWNERS`, tópicos e descrição dos repositórios, membros dos grupos | `gh api repos/<o>/<r>/contents/<caminho>`, `gh repo list <org> --json`, `glab api projects/<id>/repository/files/...` | Qualquer chamada que não seja GET |
| `kubectl` | Ambientes, recursos, rotas, implantações | `kubectl get ns,deploy,statefulset,svc,ingress,httproute -o json`, `kubectl get configmap -o json` | `apply`, `edit`, `delete`, `exec`, `port-forward`, ler `secret` |
| `helm` | Values de cada release | `helm list -A -o json`, `helm get values <release> -n <ns> -o json` | Levar os values para a carga: eles podem trazer senhas. Extraia só host, nome do banco e tópico |
| `az`, `aws`, `gcloud` | Bancos, brokers, buckets e gateways gerenciados | `list` e `describe`/`show` | Criar, alterar ou ler chaves e connection strings |
| Diretório | Times, níveis e pessoas | `az ad group list`, `az ad group member list`, `ldapsearch` (só leitura), o export que a pessoa der | Ler atributos além de nome, login, e-mail, cargo e grupo |
| Catálogo (Backstage, CMDB) | Domínios, sistemas, donos | `GET /api/catalog/entities` do Backstage, o export do CMDB | — |
| MCPs da sessão | Depende do servidor (catálogo, repositórios, nuvem) | Só ferramentas de leitura | Ferramentas que criam ou alteram |

Um documento de texto (wiki, apresentação, README de outro time) diz o que alguém escreveu, e não o que a fonte afirma. Ele serve de pista para procurar a fonte. Só vira evidência quando a pessoa confirmar que ele é o registro oficial daquele fato (o catálogo de domínios da organização, por exemplo).

## Identidade do componente (`build-manifest`)

| Fato | Onde | Vai para |
|---|---|---|
| Caminho e host do repositório | `git remote get-url origin`. O caminho é o que vem depois do host, sem `.git` (`git@git.example.com:shop/orders-api.git` → `shop/orders-api`). Tire usuário e token da URL | `repo:<host>/<caminho>` (`url` sem credencial), `HOSTED_IN` |
| Commit e versão do artefato | `git rev-parse HEAD`. A versão vem do `pom.xml`, do `build.gradle`, do `package.json` ou da tag. Com mudanças não commitadas, avise | `revision` dos retratos do componente |
| Tipo | Serviço HTTP ou de eventos: `service`. BFF de um frontend: `bff`. SPA ou app: `frontend`. Roda e termina (CronJob, batch): `job`. Publicado para outros importarem: `library` | `Component.kind` |
| Nome de serviço | Spring: `spring.application.name` (`application.yml`, `bootstrap.yml`). Outras stacks: o nome no chart, no `Service` do Kubernetes ou no host com que os outros o chamam. Sem nada disso, o fim do caminho do repositório | É a chave do componente, `component:<serviço>`, e nomeia a API e os endpoints: `api:<serviço>/http` |
| Pacote publicado | `group`/`artifactId` com publicação configurada (`maven-publish`, `distributionManagement`), `name` de um `package.json` publicado | `lib:...`, `PRODUCES` (`version`) |

O nome de serviço liga provedor e consumidores. É o nome do código, sem a versão que vá no host. Se os clientes chamam `orders-api-v2`, o serviço continua `orders-api` (veja [Versões](#versões-de-apis-e-tópicos)).

**Stack** (`BUILT_WITH`, com a versão resolvida). Use um item por tecnologia, com nomes curtos e minúsculos: `java`, `kotlin`, `gradle`, `maven`, `spring-boot`, `typescript`, `node`, `fastify`, `react`, `go`, `python`, `fastapi`.

| Stack | Onde |
|---|---|
| Java/Kotlin | Toolchain ou `sourceCompatibility` no build. Spring Boot no bloco `plugins` ou no parent do `pom.xml`. Gradle em `gradle/wrapper/gradle-wrapper.properties`; Maven em `.mvn/wrapper/maven-wrapper.properties` |
| Node | `engines`, `packageManager`, os frameworks nas `dependencies`, com a versão do lockfile |
| Go | Diretiva `go` do `go.mod`; os frameworks nos `require` |
| Python | `requires-python`; os frameworks nas dependências do `pyproject.toml` ou do `requirements.txt` |

Só entra o que o projeto usa de fato: uma biblioteca que chega como dependência de outra não é stack.

Às vezes a versão vem de um plugin de convenção ou de um BOM interno, e não aparece no build do projeto. Procure no artefato publicado (o POM no repositório Maven, ou o cache do Gradle em `~/.gradle/caches/modules-2`). Sem acesso, a tecnologia entra sem versão, e a versão vira lacuna.

## Bibliotecas (`sbom`)

Um retrato de `sbom` sai de um SBOM CycloneDX:
- `Library` pelo `purl`, com a chave das receitas;
- `DEPENDS_ON` com `version`, `scope: runtime`, `resolved: true`, `direct` e `via` (as diretas que trazem cada transitiva, pela árvore `dependencies`).

Gerar o SBOM é rodar uma ferramenta no projeto. Faça só com a permissão da pessoa, e sempre com a saída fora do repositório:
- `npx @cyclonedx/cyclonedx-npm --omit dev --output-file <carga>/bom.json`;
- `cyclonedx-gomod app -json`;
- `cyclonedx-py environment`;
- o plugin CycloneDX do Maven ou do Gradle.

Sem ferramenta, não monte bibliotecas à mão a partir do lockfile. Deixe o assunto de fora e registre a lacuna.

## API HTTP oferecida (`openapi`)

Se o repositório tem a especificação (contract-first: `openapi.yaml`, `api/*.yaml`), use-a. Ela dá a API, os endpoints, o `Contract` (`openapi`) e o `SPECIFIED_BY`. Se não tem, leia o código. Nesse caso, não há `Contract`.

| Stack | Onde |
|---|---|
| Spring MVC e WebFlux | `@RestController` + `@RequestMapping` da classe (prefixo); `@GetMapping`, `@PostMapping`... (método e caminho; vários na mesma anotação são uma operação cada); `server.servlet.context-path`, `spring.mvc.servlet.path`, `spring.webflux.base-path` no `application.yml` padrão; `RouterFunction`; interface gerada (`implements OrdersApi`): siga até ela |
| Fastify | `fastify.get/post/...`, `fastify.route({ method, url })`, prefixo do `register(plugin, { prefix })` |
| Express | `app.get/post/...`, `router.<método>`, prefixo de `app.use('/prefixo', router)` |
| NestJS | `@Controller('prefixo')` + `@Get()`, `@Post()`... |
| Go | `http.HandleFunc`/`mux.Handle` (`"GET /orders/{id}"` no Go 1.22+), `chi` (`r.Get`, `r.Route`), `gin` (`r.GET`, `r.Group`), `echo` |
| FastAPI | `@app.get`, `@router.post`, `APIRouter(prefix=...)` |
| Flask | `@app.route(..., methods=[...])`, `Blueprint(url_prefix=...)` |

Para cada operação:
- **`Endpoint`:**
  - a chave pela receita, com os parâmetros sem nome (`endpoint:orders-api/GET /orders/{}`);
  - `method`;
  - `path` com os nomes (`/orders/{id}`);
  - `name` (`GET /orders/{id}`);
  - `operationId`, que é o nome do método no código.
- **`HAS_ENDPOINT`** da API para o endpoint, com a evidência na linha do mapeamento.
- **`contract`** (opcional): o fragmento OpenAPI da operação, em texto JSON. Leva `format: "openapi-3"`, os parâmetros, o corpo, as respostas e os `components.schemas` que eles usam. Vale a pena quando os DTOs são claros: o console e o `contract_get` mostram o contrato. Os tipos JSON seguem os campos:
  - `String` → `string`;
  - `int`/`long` → `integer`;
  - `BigDecimal`/`double` → `number`;
  - `List<T>` → `array`;
  - `enum` → `enum`;
  - `LocalDate` → `date`; `Instant` → `date-time`; `UUID` → `uuid`.
  
  `required` vem de `@NotNull`, `@NotBlank` e `@NotEmpty`. O retorno define a resposta de sucesso, e o status vem de `@ResponseStatus` ou do `ResponseEntity`; sem nada, `200`. Os erros vêm dos `@ExceptionHandler`.

**A API:** `Api` (`kind: http`, `name`) e `PROVIDES` do componente, com a evidência na classe da aplicação ou na configuração do OpenAPI. Quando houver, `@OpenAPIDefinition(info = @Info(title, description))` dá o nome e a descrição.

Ficam de fora:
- `/actuator/**`, `/error`, o swagger-ui e os controllers de teste;
- operações com caminho montado em execução, que vão para as lacunas.

As anotações do springdoc e do Swagger (`@Operation`, `@ApiResponse`, `@Schema`) são a melhor fonte quando existem.

## Eventos (`asyncapi`)

O grafo guarda o **endereço do canal** e a **direção**: `Topic` com `kind` (`topic`, ou `queue` para fila), `PUBLISHES` e `SUBSCRIBES`.

| Onde | Direção |
|---|---|
| `@KafkaListener(topics)`, `@RabbitListener(queues)`, `@JmsListener(destination)`, `@SqsListener` | `SUBSCRIBES` |
| `KafkaTemplate.send(topico, ...)`, `RabbitTemplate.convertAndSend(exchange, ...)`, `JmsTemplate.convertAndSend(destino, ...)` | `PUBLISHES` |
| Spring Cloud Stream: bean `Consumer`/`Function`/`Supplier`, com `spring.cloud.stream.bindings.<função>-in-0.destination` (consome) e `-out-0.destination` (publica). Também `StreamBridge.send` | Pela binding |
| Node (`kafkajs`, `amqplib`), Go (`sarama`, `franz-go`, `kafka-go`), Python (`confluent-kafka`, `aiokafka`, `pika`) | Pela chamada |

**Placeholders.** Resolva `${...}` pelo `application.yml` padrão, inclusive o valor default (`${app.topic:orders.created}` é `orders.created`). Se o valor só existir num perfil, numa variável de ambiente ou num values do Helm, o canal vai para as lacunas, com o nome da propriedade. Nunca use `${...}` num endereço.

A evidência do tópico é a linha da configuração que dá o nome. A da relação é a chamada ou o listener.

No RabbitMQ, quem consome lê uma fila e quem publica escreve num exchange. Use a fila em `SUBSCRIBES` e o exchange em `PUBLISHES`. Se a declaração do binding estiver no código, registre nas lacunas a ligação entre os dois.

## Quem o componente chama (`http-clients`)

| Onde | Serviço chamado e operações |
|---|---|
| `@FeignClient(name, url, path)` com métodos `@GetMapping`... | O serviço vem do `name` ou do host da `url`; as operações, dos métodos |
| `@HttpExchange`/`@GetExchange` em interfaces | O serviço vem do grupo ou da base URL; as operações, dos métodos |
| `RestTemplate`, `RestClient`, `WebClient`, `axios`, `fetch`, `requests`, `httpx`, com base URL de configuração | O serviço vem do host da base URL; as operações, das chamadas com caminho literal |

Para cada serviço chamado:
- `CONSUMES` do componente para `api:<serviço>/http`, com a evidência no cliente;
- `CALLS` para cada `endpoint:<serviço>/<MÉTODO> <caminho>`, com a evidência no método que chama.

O caminho é o que o provedor vê. Se a base URL tem caminho (`http://catalog-api:8080/v1`), ele entra no começo de cada operação.

**Cliente que a tabela não cobre.** Parta da configuração, que diz para onde o componente fala:
- propriedades e variáveis com URL ou host (`*.url`, `*.base-url`, `*_URL`, `*_HOST`);
- `lb://<serviço>`;
- blocos por cliente (`spring.cloud.openfeign.client.config.<nome>.url`).

Siga cada uma até o código que a lê, e daí até as chamadas. O destino que só existe num perfil ou numa variável de ambiente vira lacuna.

Chamadas a serviços de fora da organização (gateway de pagamento, API pública) entram do mesmo jeito.

## Contratos de consumidor (`pact`)

Os pacts vêm do repositório (`pacts/`, `target/pacts/`, `build/pacts/`) ou do Pact Broker (`GET /pacts/provider/<p>/consumer/<c>/latest`). Para cada um:
- `Contract` (`kind: pact`, chave `contract:pact/<consumidor>→<provedor>`);
- `SPECIFIED_BY` da API do provedor para o contrato;
- `CONSUMES`;
- `CALLS` para cada interação que case com uma operação conhecida.

Interação sem operação conhecida fica só no `CONSUMES`. Não gere pact: ele é resultado de teste.

## Dados (`data-access`)

| Onde | O que extrair |
|---|---|
| JPA: `@Entity` + `@Table(name)`; Spring Data `JpaRepository` | `store: relational`, `kind: table` |
| Spring Data MongoDB: `@Document(collection)`, `MongoRepository`, `MongoTemplate` | `store: document`, `kind: collection` |
| JDBC/R2DBC, `JdbcTemplate` com SQL literal | `relational`, pela tabela no SQL |
| Redis (`RedisTemplate`, `@RedisHash`) | `store: key-value` |
| Prisma (`model` com `@@map`), TypeORM (`@Entity`), Mongoose (`model('X')`), Drizzle | Pelo modelo |
| GORM (`TableName()`), `database/sql` com SQL literal, `mongo-driver` (`Collection("x")`) | Pelo modelo ou pela chamada |
| SQLAlchemy (`__tablename__`), Django (`Meta.db_table`), `pymongo` | Pelo modelo ou pela chamada |

- **`Dataset`:** a chave `dataset:<componente>/<nome>`.
- **`OWNS_DATA`:** só quando o mapeamento é do próprio componente, com a evidência na entidade.
- **`READS` e `WRITES`:** pelos métodos usados no código, com a evidência na linha da primeira chamada.
- **Fica de fora:** SQL montado em execução e nome que não é literal, que vão para as lacunas.
- **Migrações:** confirmam nomes de tabela, mas não são acesso.

## Organização e catálogo (`directory`, `service-catalog`)

| Fato | Onde | Cuidado |
|---|---|---|
| Times, níveis e pessoas | O diretório. Também `catalog-info.yaml` do tipo `Group` (`spec.type`, `spec.parent`, `spec.members`) e os grupos do GitLab/GitHub, se a pessoa confirmar que refletem os times | Pessoa entra pelo login da fonte (`person:<login>`), sem e-mail se a fonte não o tiver. Níveis (`OrgLevel`) só os que a fonte mostra |
| Dono (`OWNED_BY`) | `catalog-info.yaml` (`spec.owner`), o catálogo, o CMDB. Também `CODEOWNERS` com um time (`@org/time`) | Um usuário em `CODEOWNERS` não é um time. Sem time explícito, é lacuna, com o que achou como pista |
| Domínio e sistema (`PART_OF`) | `catalog-info.yaml` (`spec.system`, `spec.domain`), o catálogo | A estrutura de grupos do git só se a pessoa confirmar que ela é a do negócio |
| Componentes de infraestrutura | O catálogo: servidores de banco, brokers e gateways mantidos como produto, com `kind: infra` | — |

## Infraestrutura, rotas e configuração (`k8s-infra`, `k8s-routes`, `service-config`)

| Fato | Onde | Cuidado |
|---|---|---|
| Ambientes | Namespaces e clusters (`kubectl get ns`, a label `panorama-graph.io/environment`), pastas e arquivos por ambiente (`values-dev.yaml`, `k8s/prd/`) | O nome é livre (`dev`, `hml`, `prd`). O `kind` é fixo (`development`, `test`, `staging`, `production`): pergunte se não for óbvio. Um `docker-compose` de teste local não é ambiente |
| Servidores, brokers, caches | StatefulSets e Services do cluster, charts de dependência, a CLI da nuvem (`az postgres flexible-server list`, `aws rds describe-db-instances`) | Instância: `resource:<ambiente>/<nome>`, com `IN_ENVIRONMENT` e `RUNS` (a tecnologia e a versão da imagem ou do serviço gerenciado) |
| Bancos, buckets | Variáveis de conexão por ambiente (`SPRING_DATASOURCE_URL`, `MONGODB_URI`), a CLI da nuvem | Recurso lógico: `resource:<ambiente>/<instância>/<nome>`, com `HOSTED_ON` na instância. Só o servidor e o nome, nunca usuário ou senha |
| Que recurso o componente usa | Variáveis do Deployment ou dos values por ambiente (`*_URL`, `*_HOST`, `KAFKA_BOOTSTRAP_SERVERS`) | `USES_RESOURCE` com `access` (`read`, `write`, `read-write`) e a evidência no nome da variável. Um retrato por componente e ambiente (`service-config`) |
| Tópico no broker | A configuração do componente aponta o broker, e o código dá o tópico | `HOSTED_ON` do tópico para o broker, no retrato de `service-config` |
| Rotas | `Ingress` (host, path, backend), `HTTPRoute`, rotas do API gateway | `ROUTES` do gateway ou ingress para o componente, com `host` e `path`. `REACHES_VIA` quando um componente chama outro pelo gateway |

## Implantação e versões expostas

O que roda em cada ambiente vem do `graph-deployments`, que lê o cluster. À mão, só quando o Deployment tem as anotações `panorama-graph.io/component` e `panorama-graph.io/commit`. Sem o commit, registre a lacuna.

Uma versão de API exposta à parte (o Service `orders-api-v2`, que seleciona os pods do componente) pode entrar sem o `DEPLOYED_IN`, num retrato da fonte do adaptador:

```json
{
  "source": "k8s-deployments",
  "scope": { "cluster": "prd", "namespace": "orders" },
  "observedAt": "2026-10-23T12:00:00Z",
  "evidenceBase": "prd/orders/Service/orders-api-v2",
  "nodes": [{ "key": "api:orders-api/v2/http", "type": "Api", "props": { "kind": "http", "version": "v2", "name": "orders-api-v2" } }],
  "relations": [
    { "type": "PROVIDES", "from": "component:orders-api", "to": "api:orders-api/v2/http" },
    { "type": "VERSION_OF", "from": "api:orders-api/v2/http", "to": "api:orders-api/http" }
  ]
}
```

Se o `sources_status` já mostra `k8s-deployments` nesse escopo, não envie: o adaptador já afirma.

## Versões de APIs e tópicos

Sempre que achar a versão de uma operação, de uma API ou de um tópico, em qualquer padrão, leve-a à carga. Ela pode estar no caminho, no host, no nome do tópico, num cabeçalho, no media type ou numa anotação.

O formato é `v<N>` (só a maior). A release do componente não é versão de interface. Interface sem versão é o caso comum: não invente uma.

| Versionamento | Na carga |
|---|---|
| No caminho (`/v2/orders`) | `api:<serviço>/v2/http` com `version: v2`, e os endpoints `endpoint:<serviço>/v2/<MÉTODO> /v2/orders`. Uma API por versão, no provedor e nos clientes |
| No host (`orders-api-v2`) | O cliente consome `api:orders-api/v2/http`, mas chama os endpoints do código, sem versão (`endpoint:orders-api/GET /orders`). O provedor só afirma a API do código. A versão exposta vem da implantação (seção acima) |
| No nome do tópico (`order-created.v2`) | `topic:order-created.v2`, com `version: v2` |
| Outro padrão (cabeçalho, media type) | A versão na chave e em `version`, com a evidência onde ela aparece. Se não tiver certeza de que ela separa interfaces que convivem, registre a lacuna |

As regras completas estão em [../produto/MODELO.md](../produto/MODELO.md#versões-de-apis-e-tópicos). Quando os lados discordam (o cliente chama a v2 e o provedor não tem v2), use o que cada fonte diz e registre a diferença.
