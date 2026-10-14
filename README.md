<div align="center">

# ✂️ url-shortener

### Um encurtador de URLs pequeno no tamanho e grande na arquitetura

*Cada link longo que entra sai com poucos caracteres. Este projeto existe para estudar o que acontece no caminho.*

![Node.js](https://img.shields.io/badge/Node.js-24-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Fastify](https://img.shields.io/badge/Fastify-5-000000?style=for-the-badge&logo=fastify&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-8-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-cluster-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-ECS%20Fargate-FF9900?style=for-the-badge&logo=amazonwebservices&logoColor=white)
![SST](https://img.shields.io/badge/IaC-SST-E27152?style=for-the-badge)

</div>

---

## 🧭 A ideia em uma frase

> **Um contador atômico no Redis gera o ID, o Hashids transforma esse número em base62 embaralhada e o MongoDB guarda o resultado com o próprio número como chave primária.**

Sem gerar códigos aleatórios, sem colisões e sem precisar perguntar ao banco se o código já existe.

```
https://www.exemplo.com/um/caminho/gigantesco?com=muitos&parametros=demais
                               │
                               ▼
                    https://seu.dominio/k3Xa9
```

---

## 🏗️ Arquitetura

### Na nuvem (AWS, provisionado com SST)

```mermaid
flowchart LR
    U([👤 Usuário]) -->|HTTPS| ALB[⚖️ Application<br/>Load Balancer]

    subgraph VPC["🔒 VPC"]
        ALB --> ECS
        subgraph ECS["🐳 ECS Cluster"]
            T1[Task Fargate<br/>on-demand]
            T2[Task Fargate<br/>Spot]
        end
        ECS -->|INCR url:counter| R[(🟥 ElastiCache<br/>Redis Cluster)]
    end

    ECS -->|insertOne / findOne| M[(🍃 MongoDB<br/>Atlas)]
```

| Peça | Papel no sistema |
|---|---|
| **Application Load Balancer** | Ponto de entrada; escuta em 80/443 e encaminha para a porta 8080 dos containers |
| **ECS + Fargate** | Roda a API em containers sem servidores para administrar |
| **Fargate Spot** | Uma task on-demand garante a base; as demais são divididas meio a meio com Spot, que custa bem menos |
| **ElastiCache Redis** | Fonte única do contador global. O `INCR` é atômico, então várias réplicas nunca geram o mesmo ID |
| **MongoDB** | Armazena o mapeamento `id → URL longa` |
| **SST** | Infraestrutura como código em TypeScript, com os segredos injetados via `Resource` |

---

## 🔁 Como uma URL nasce

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant API as Fastify
    participant R as Redis
    participant H as Hashids
    participant M as MongoDB

    C->>API: POST /api/shorten { url }
    API->>API: Zod valida (http/https, ≤ 2048 chars)
    API->>R: INCR url:counter
    R-->>API: 238329
    API->>H: encode(238329)
    H-->>API: "k3Xa9"
    API->>M: insertOne { _id: 238329, shortCode, longUrl }
    API-->>C: 201 { shortCode, shortUrl, longUrl }
```

## ↪️ Como uma URL é seguida

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant API as Fastify
    participant H as Hashids
    participant M as MongoDB

    C->>API: GET /k3Xa9
    API->>H: decode("k3Xa9")
    alt código inválido
        H-->>API: null
        API-->>C: 404 NOT_FOUND
    else código válido
        H-->>API: 238329
        API->>M: findOne { _id: 238329 }
        M-->>API: { longUrl }
        API-->>C: 301 Location: longUrl
    end
```

O redirecionamento **decodifica o código de volta para o número** e busca direto pela chave primária, sem precisar de índice secundário em `shortCode`. Códigos que não decodificam para um único inteiro válido recebem 404 antes de qualquer consulta ao banco.

---

## 🧠 Decisões de arquitetura

<details>
<summary><b>Por que um contador e não códigos aleatórios?</b></summary>

<br/>

Códigos aleatórios trazem a possibilidade de colisão. Para evitá-la, é preciso consultar o banco antes de gravar ou tratar erro de chave duplicada e tentar de novo. Conforme a base cresce, as colisões ficam mais frequentes.

Com um contador, todo ID é único por construção. O `INCR` do Redis é atômico, então duas instâncias da API recebendo requisições ao mesmo tempo nunca pegam o mesmo número.

</details>

<details>
<summary><b>Por que Hashids em cima do contador?</b></summary>

<br/>

Expor `/1`, `/2`, `/3` revela quantos links existem e permite percorrer todos eles. O Hashids, com um *salt* secreto e alfabeto base62 (`0-9a-zA-Z`), transforma a sequência em códigos que não parecem sequenciais e que **continuam reversíveis** sem consultar o banco.

</details>

<details>
<summary><b>Por que o contador começa em 238.328?</b></summary>

<br/>

Porque **238.328 = 62³**. Começando aí, os IDs pequenos (que resultariam em códigos de 1 a 3 caracteres) não são usados, e todos os links têm tamanho parecido desde o primeiro. O valor inicial é definido com `SET ... NX`, então reiniciar a aplicação nunca zera o contador.

</details>

<details>
<summary><b>Por que usar o ID numérico como <code>_id</code> do Mongo?</b></summary>

<br/>

O `_id` já tem índice único. Usar o próprio número do contador como chave elimina um índice extra e faz o redirect ser uma busca por chave primária, com projeção só do campo `longUrl`.

</details>

<details>
<summary><b>Por que 301 e não 302?</b></summary>

<br/>

O 301 (permanente) permite que navegadores e proxies guardem o redirecionamento em cache, o que reduz a carga na API. A contrapartida é perder a contagem de cliques repetidos, então um sistema com analytics provavelmente usaria 302.

</details>

---

## 📡 API

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/` | Health check |
| `POST` | `/api/shorten` | Encurta uma URL |
| `GET` | `/:shortCode` | Redireciona (301) para a URL original |
| `GET` | `/docs` | Documentação interativa (Scalar) |
| `GET` | `/swagger.json` | Especificação OpenAPI |

**Exemplo**

```bash
curl -X POST http://localhost:8080/api/shorten \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Status/301" }'
```

```json
{
  "shortCode": "k3Xa9",
  "shortUrl": "http://localhost:8080/k3Xa9",
  "longUrl": "https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Status/301"
}
```

Os erros sempre seguem o mesmo formato:

```json
{ "error": "Short URL not found", "code": "NOT_FOUND" }
```

---

## 🚀 Rodando o projeto

### Pré-requisitos

- Node.js 24 (`nvm use`)
- pnpm (via `corepack enable`)
- Conta AWS com credenciais configuradas
- Um cluster MongoDB (o Atlas no plano gratuito serve)

### 1. Instalar

```bash
git clone https://github.com/MarceloJael/url-shortener.git
cd url-shortener
corepack enable
pnpm install
```

### 2. Configurar os segredos

A configuração é lida dos *secrets* do SST e validada com Zod na inicialização. Se faltar algo, a aplicação para e mostra exatamente o que está errado.

```bash
npx sst secret set MongoUrl "mongodb+srv://usuario:senha@cluster.mongodb.net/"
npx sst secret set HashIdsSalt "uma-string-longa-e-aleatoria"
npx sst secret set BaseUrl "http://localhost:8080"
```

| Secret | Padrão | Descrição |
|---|---|---|
| `MongoUrl` | — | String de conexão do MongoDB |
| `HashIdsSalt` | — | Salt que embaralha os códigos. **Não mude depois de criar links**, ou os códigos existentes deixam de funcionar |
| `BaseUrl` | `http://localhost:8080` | Prefixo das URLs curtas retornadas |
| `Port` | `8080` | Porta HTTP |
| `NodeEnv` | `development` | `development`, `production` ou `test` |
| `MongoMaxPoolSize` | `100` | Tamanho máximo do pool de conexões |
| `HashIdsMinLength` | `0` | Comprimento mínimo dos códigos |

### 3. Desenvolvimento

```bash
npx sst dev
```

O SST provisiona a VPC e o Redis na AWS e roda `pnpm run dev` localmente com hot reload, já conectado aos recursos. Depois é só abrir **http://localhost:8080/docs**.

### 4. Deploy

```bash
DOMAIN=seu.dominio.com npx sst deploy --stage production
```

No stage `production`, os recursos ficam protegidos contra remoção acidental (`protect` + `removal: retain`). A variável `DOMAIN` é opcional; sem ela, o serviço fica acessível pela URL do Load Balancer.

---

## 🗂️ Estrutura

```
src/
├── index.ts            # bootstrap do Fastify, OpenAPI, CORS, erros e graceful shutdown
├── lib/
│   ├── env.ts          # leitura e validação da configuração com Zod
│   ├── hashids.ts      # encode/decode base62
│   ├── mongo.ts        # cliente e collection tipada
│   └── redis.ts        # cliente do cluster e inicialização do contador
├── routes/
│   └── urls.ts         # POST /api/shorten e GET /:shortCode
└── schemas/
    └── index.ts        # contratos de entrada e saída (Zod → OpenAPI)
sst.config.ts           # toda a infraestrutura AWS em TypeScript
Dockerfile              # imagem usada pelo ECS
```

---

<div align="center">

Feito com ☕ e muitos diagramas, para aprender arquitetura de sistemas na prática.

</div>
