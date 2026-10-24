# O que pode ser importado

O grafo responde perguntas sobre a organização e a arquitetura:
- quem chama quem;
- quem é dono do quê;
- que stack e que bibliotecas cada componente usa;
- que bancos, eventos e rotas existem;
- o que roda em cada ambiente.

Cada assunto abaixo é um conjunto de nós e relações do modelo. Em cada assunto estão a fonte e o escopo de que esses fatos são afirmados, e onde eles costumam estar. Os tipos, as propriedades e os pares válidos estão em [../produto/MODELO.md](../produto/MODELO.md).

## Assuntos

| Assunto | Nós e relações | Fonte | Escopo | Onde costuma estar |
|---|---|---|---|---|
| Estrutura organizacional | `OrgLevel`, `Group`, `Person`, `AT_LEVEL`, `PART_OF` (grupo em grupo), `MEMBER_OF` (`role`, `lead`) | `directory` | `{ tenant: <organização> }` | Diretório corporativo (LDAP, Entra ID, Google Workspace), export do RH, `catalog-info.yaml` do tipo `Group` |
| Catálogo | `Domain`, `System`, `PART_OF` (sistema no domínio, componente no sistema), `OWNED_BY` (componente, sistema, domínio ou API do time), componentes de infraestrutura (`Component` com `kind: infra`) | `service-catalog` | `{ catalog: <organização> }` | Backstage e `catalog-info.yaml`, CMDB, portal interno, `CODEOWNERS` (só como pista de dono) |
| Identidade e stack de um componente | `Component` (`name`, `kind`), `Repository`, `HOSTED_IN`, `Technology`, `BUILT_WITH` (`version`), `Library` e `PRODUCES` (`version`) quando o build publica um pacote | `build-manifest` | `{ component: <nome> }` | O build (`pom.xml`, `build.gradle`, `package.json`, `go.mod`, `pyproject.toml`) e o git |
| Bibliotecas | `Library`, `DEPENDS_ON` (`version`, `scope`, `resolved`, `direct`, `via`) | `sbom` | `{ component }` | Um SBOM CycloneDX, gerado por ferramenta. O lockfile lido à mão não basta |
| Uso das bibliotecas | `DEPENDS_ON` (`declared`, `used`) | `dependency-usage` | `{ component }` | Só de uma análise de bytecode (o plugin Gradle). Sem ela, fica de fora |
| API HTTP oferecida | `Api` (`kind: http`, `version`), `Endpoint` (`method`, `path`, `name`, `operationId`, `contract`), `PROVIDES`, `HAS_ENDPOINT`. Com um documento OpenAPI no repositório, também `Contract` (`openapi`) e `SPECIFIED_BY` | `openapi` | `{ component }` | Os controllers e as rotas, ou a especificação contract-first |
| Eventos | `Topic` (`kind`, `version`), `PUBLISHES`, `SUBSCRIBES`. Com um documento AsyncAPI, também `Contract` (`asyncapi`) e `SPECIFIED_BY` | `asyncapi` | `{ component }` | Listeners, produtores e a configuração que resolve os nomes dos canais |
| Quem o componente chama | `CONSUMES` (a API do outro), `CALLS` (cada operação) | `http-clients` | `{ component }` | Clientes HTTP declarados (Feign, `@HttpExchange`, clientes gerados) e chamadas com base URL de configuração |
| Contratos de consumidor | `Contract` (`pact`), `CONSUMES`, `CALLS`, `SPECIFIED_BY` | `pact` | `{ component }` | Pacts no repositório ou no Pact Broker |
| Dados | `Dataset` (`kind`, `store`), `OWNS_DATA`, `READS`, `WRITES` | `data-access` | `{ component }` | Entidades e repositórios do ORM, SQL literal, chamadas ao driver |
| Infraestrutura | `Environment` (`kind`), `Resource` (servidor, broker, cache, gateway, banco, bucket), `IN_ENVIRONMENT`, `RUNS` (`version`), `HOSTED_ON`, `INSTANCE_OF` | `k8s-infra` | `{ cluster }`, ou `{ cluster, namespace }` quando a leitura é por namespace | Manifestos e values do Helm, o cluster (`kubectl`), o console ou a CLI da nuvem |
| Rotas de entrada | `ROUTES` (`host`, `path`), `REACHES_VIA` | `k8s-routes` | `{ cluster, namespace }` | `Ingress`, `HTTPRoute` (Gateway API), configuração do API gateway |
| Configuração de cada serviço | `USES_RESOURCE` (`access`), `HOSTED_ON` (o banco no servidor, o tópico no broker) | `service-config` | `{ component, environment }` | Variáveis de conexão nos values, nos manifestos ou no `application-<ambiente>.yml` |
| O que roda em cada ambiente | `Environment`, `DEPLOYED_IN` (`version`, `commit`, `image`, `replicas`), e a versão de API exposta na infraestrutura (`Api` com `version`, `PROVIDES`, `VERSION_OF`) | `k8s-deployments` | `{ cluster, namespace }` | O cluster: Deployments e Services |

**Linha principal.** Os fatos de um componente lidos do código são os da linha principal: o commit lido, que deve ser o do branch padrão. O retrato leva `revision` (o componente, o commit e a versão), sem `commit` no escopo ([retratos.md](retratos.md#revision)). Nunca monte um retrato congelado (com `commit` no escopo). Ele é do adaptador que vê o commit rodando.

**O que roda em cada ambiente.** Cada `DEPLOYED_IN` exige o `commit` em execução, e o cluster só o mostra pela anotação `panorama-graph.io/commit`. Sem ela, a implantação é uma lacuna. A tag da imagem não é commit. A versão exposta na infraestrutura (o Service `orders-api-v2`) pode entrar sozinha, como no exemplo de [descoberta.md](descoberta.md#implantação-e-versões-expostas).

## Fontes que um adaptador do produto publica

As fontes de build (`build-manifest`, `sbom`, `dependency-usage`, `openapi`, `asyncapi`, `http-clients`, `pact`, `data-access`) são publicadas pelo `graph-publish` e pelo `graph-receiver`. A `k8s-deployments` é publicada pelo `graph-deployments`.

Se o `sources_status` já mostra uma dessas fontes no escopo, com `producer` dos adaptadores, o assunto já está coberto: não envie. Se não mostra, a carga desta skill vale até o adaptador chegar. O primeiro envio dele, com a mesma fonte e o mesmo escopo, a substitui.

As demais fontes (`directory`, `service-catalog`, `k8s-infra`, `k8s-routes`, `service-config`) são os nomes que um adaptador próprio da organização usaria. Use-os do mesmo jeito.

## Escopos que valem para a organização inteira

`directory`, `service-catalog` e `k8s-infra` (com `{ cluster }`) cobrem a organização, ou o ambiente inteiro, num retrato só. Cada envio substitui o anterior. Por isso, o retrato precisa ter tudo o que a fonte afirma, e não só o que esta rodada leu.

Antes de montar um desses retratos:
- veja se a pasta de carga já tem o arquivo de uma rodada anterior;
- veja no `sources_status` se o escopo existe.

Acrescente ao que já foi carregado. O dry-run mostra no `stale` o que a carga tiraria.

Se o escopo é de outro produtor, não envie. Se é uma leitura parcial por natureza (um namespace só), use o escopo mais estreito que a fonte permite (`{ cluster, namespace }`).

## O que não cabe no modelo

O modelo é fechado. Não há lugar, por exemplo, para:
- incidentes, SLAs e custos;
- tickets, ADRs e documentação;
- habilidades das pessoas;
- métricas de uso.

Não force nada disso em `description` ou em `name`: o grafo perderia a confiança de que cada campo diz o que diz. Explique à pessoa que o assunto não é importável hoje. A seção "Quando algo não cabe no modelo" do [MODELO.md](../produto/MODELO.md#quando-algo-não-cabe-no-modelo) mostra onde cada conceito comum já tem lugar e como o modelo muda, numa versão nova da API.
