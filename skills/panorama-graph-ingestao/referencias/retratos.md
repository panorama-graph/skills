# Montar os retratos

Um retrato é o corpo do `POST /v1/ingest`: tudo o que uma fonte afirma num escopo. O formato campo a campo, as respostas da API e um exemplo de cada relação estão em [../produto/ADAPTADOR.md](../produto/ADAPTADOR.md). Este arquivo diz como aplicá-lo nesta skill.

## Um arquivo por fonte e escopo

`carga-grafo/retratos/<fonte>--<valores do escopo>.json`, com `/` trocada por `-`. Exemplos:
- `openapi--shop-orders-api.json`;
- `service-config--shop-orders-api--dev.json`;
- `directory--example.json`.

O `enviar.sh` envia os arquivos em ordem de nome. A ordem entre fontes não importa: uma relação pode apontar para um nó que outra fonte ainda vai declarar.

```json
{
  "source": "openapi",
  "scope": { "component": "orders-api" },
  "observedAt": "2026-10-23T12:00:00Z",
  "evidenceBase": "shop/orders-api@3f2c9e1a7b5d4c3e2f1a0b9c8d7e6f5a4b3c2d1e:",
  "revision": { "component": "component:orders-api", "commit": "3f2c9e1a7b5d4c3e2f1a0b9c8d7e6f5a4b3c2d1e", "version": "1.4.0" },
  "nodes": [
    { "key": "api:orders-api/http", "type": "Api", "props": { "name": "orders-api", "kind": "http" },
      "evidence": "src/main/java/com/example/shop/orders/OrdersApplication.java" },
    { "key": "endpoint:orders-api/GET /orders/{}", "type": "Endpoint",
      "props": { "method": "GET", "path": "/orders/{id}", "name": "GET /orders/{id}", "operationId": "getOrder" },
      "evidence": "src/main/java/com/example/shop/orders/OrderController.java#L42" }
  ],
  "relations": [
    { "type": "PROVIDES", "from": "component:orders-api", "to": "api:orders-api/http",
      "evidence": "src/main/java/com/example/shop/orders/OrdersApplication.java" },
    { "type": "HAS_ENDPOINT", "from": "api:orders-api/http", "to": "endpoint:orders-api/GET /orders/{}",
      "evidence": "src/main/java/com/example/shop/orders/OrderController.java#L42" }
  ]
}
```

## Campos

| Campo | Nesta skill |
|---|---|
| `source`, `scope` | Os do assunto, pela tabela de [o-que-importar.md](o-que-importar.md). O escopo é um objeto de textos |
| `observedAt` | O momento da leitura, em ISO 8601 com fuso, o mesmo em todos os retratos da rodada. Numa rodada nova, um momento novo. Ele precisa ser mais novo que a última carga do escopo (o `sources_status` mostra a última), ou a API recusa com `409` |
| `evidenceBase` | O prefixo comum das evidências do retrato. A evidência de cada fato é a concatenação dele com a do item |
| `revision` | Só nos retratos das fontes de build (`build-manifest`, `openapi`, `asyncapi`, `http-clients`, `data-access`, `sbom`, `pact`), com o componente, o commit lido e a versão do artefato. Diz de que versão o retrato é. O escopo continua sem `commit`: é a linha principal |
| `producer` | Não use. Ele diz que programa montou o retrato e em que versão, e é dos adaptadores |
| `nodes[]` | `key`, `type` (o do prefixo da chave), `props` e `evidence`. Um nó só entra aqui quando a fonte afirma alguma propriedade dele. Para só ligar, basta a chave na relação |
| `relations[]` | `type`, `from`, `to`, `props` (só as da relação no modelo) e `evidence` |

## Evidência

| Onde o fato foi visto | `evidenceBase` | `evidence` do item |
|---|---|---|
| Num repositório | `<caminho do repositório>@<commit>:` | `caminho/do/arquivo#L42` ou `caminho#metodo` |
| No cluster | `<cluster>/<namespace>/` | `<Kind>/<nome>`, com o campo quando ajuda (`Deployment/orders-api#env.SPRING_DATASOURCE_URL`) |
| Num registro (catálogo, diretório, broker) | A URL base do sistema | O caminho do registro (`groups/catalog-team`) |

Num retrato com fatos de repositórios diferentes, como o catálogo lido de vários `catalog-info.yaml`, deixe o `evidenceBase` de fora. Ponha a evidência completa em cada item.

## Regras que a API aplica

Qualquer uma destas, violada, recusa o retrato inteiro com `422`:

- **Toda afirmação tem evidência.**
- **Propriedades só do modelo, dentro de `props`.** Cada uma com o tipo certo e, quando há lista fechada, com um valor aceito. Uma propriedade com `null` é ignorada. Uma fora do modelo recusa o retrato.
- **Nada repetido.** O mesmo nó duas vezes, ou a mesma relação (mesmo tipo, mesmas pontas e mesmas propriedades de identidade) duas vezes, recusa o retrato.
- **Pares válidos.** `from` e `to` precisam ser de um dos pares da relação, pelo tipo do nó: `OWNED_BY` vai do componente para o time, e nunca ao contrário.
- **Sem `state`.** Tudo o que entra pelo `/ingest` é `observed`.

## Chaves

As receitas estão em [../produto/MODELO.md](../produto/MODELO.md#receitas-de-chave), e as regras de normalização em [../produto/ADAPTADOR.md](../produto/ADAPTADOR.md#receitas-de-chave). Os pontos em que mais se erra:

- **Componente:** o nome de serviço, único na organização. **Repositório:** o caminho, com o host.
- **API e endpoint:** pelo nome de serviço, o mesmo da chave do componente.
- **Parâmetros do endpoint:** viram `{}` na chave (`/orders/{}`) e mantêm o nome na propriedade `path` (`/orders/{id}`).
- **Biblioteca:** sem versão. A versão vai na relação (`DEPENDS_ON.version`).
- **Contrato de pact:** usa a seta `→` (U+2192), e não `->`.
- **Recurso:** o ambiente é o primeiro segmento da chave, e o mesmo do `env:`.

Em dúvida, confira a chave contra os casos de [../produto/key-recipes.json](../produto/key-recipes.json). Dois nomes para a mesma coisa viram dois nós. Se o grafo já tem a coisa com outra chave, use a dele.

## Repartir os fatos pelas fontes

Os fatos de um componente lidos do código não vão todos num retrato só. Cada um vai na fonte que o adaptador usaria para ele, porque é essa fonte que o adaptador vai substituir depois:

| Fonte | Leva |
|---|---|
| `build-manifest` | `Component`, `Repository`, `HOSTED_IN`, `Technology`, `BUILT_WITH`, `Library` e `PRODUCES` |
| `openapi` | `Api` (`http`), `Endpoint`, `PROVIDES`, `HAS_ENDPOINT` e, com o documento, `Contract` e `SPECIFIED_BY` |
| `asyncapi` | `Topic`, `PUBLISHES`, `SUBSCRIBES` e, com o documento, `Contract` e `SPECIFIED_BY` |
| `http-clients` | `CONSUMES` e `CALLS` |
| `data-access` | `Dataset`, `OWNS_DATA`, `READS` e `WRITES` |
| `sbom` | `Library` e `DEPENDS_ON` |

O exemplo completo, um retrato de cada, está em [../exemplos/carga-grafo/retratos/](../exemplos/carga-grafo/retratos/).
