# pw-api-shortbeyond

[![Playwright](https://img.shields.io/badge/Playwright-API-2EAD33?logo=playwright&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![k6](https://img.shields.io/badge/Artillery-Performance-FF6600?logo=artillery&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1?logo=postgresql&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![Bruno](https://img.shields.io/badge/Bruno-API_Exploration-orange)](https://github.com/enzoacoelho/pw-api-shortbeyond)

Testes de API com **Playwright** num encurtador de links (ShortBeyond), cobrindo autenticação e gerenciamento completo de links, com massa de dados via banco PostgreSQL e testes de performance com **Artillery** simulando picos de carga.

## Sobre o projeto

A aplicação sob teste é uma API de encurtamento de links com autenticação via JWT. Antes de automatizar, explorei todos os endpoints manualmente com **Bruno**, documentando o contrato de cada rota em `docs/shortbeyond`. Isso me deu contexto suficiente pra desenhar cenários de teste mais completos, incluindo os casos de erro que não ficam óbvios só lendo a documentação da API.

O ambiente roda localmente via Kubernetes (Pod com Postgres, Adminer, API e Web), o que me permitiu isolar o banco de teste do restante e resetar o estado sempre que necessário.

## Estratégia de testes

Cobertura funcional dividida em dois domínios, sempre com cenários positivos e negativos:

**Autenticação**
- Cadastro de usuário com sucesso (valida status, mensagem e que a senha não retorna no payload)
- Cenários negativos de cadastro (dados inválidos/duplicados)
- Login com credenciais corretas
- Login com credenciais incorretas

**Links**
- Criação de link (`POST`)
- Listagem de links (`GET`)
- Atualização de link (`PUT`)
- Remoção de link (`DELETE`)
- Cenários positivos e negativos para cada uma dessas operações (ex: tentar manipular um link sem token, ou um link que não existe)

## Arquitetura

```
pw-api-shortbeyond/
├── docs/shortbeyond/       # Collection Bruno usada na exploração manual da API
├── performance/
│   ├── data/               # Massa de usuários para o teste de carga (usuarios.csv)
│   ├── reports/             # Relatórios gerados pelo Artillery (JSON)
│   └── test/                 # Cenários de carga (Artillery)
├── playwright/
│   ├── e2e/                  # Testes de autenticação e gerenciamento de links
│   └── support/
│       ├── factories/        # Geração de massa de dados com Faker
│       ├── fixtures/         # Extensão do test do Playwright, injeta os services
│       ├── services/          # Camada de abstração das chamadas à API (auth, links)
│       ├── database.js        # Conexão com Postgres, seed e limpeza de dados de teste
│       └── utils.js            # Geração de ULID
├── globalSetup.js              # Limpa dados de teste antes de cada execução
└── .env                          # Configuração de ambiente (API e banco)
```

**Decisões de arquitetura que valem destaque:**

- **Services em vez de chamadas soltas**: toda interação com a API passa por `authService` e `linksService`, que centralizam endpoint, headers e parsing de resposta. Isso evita duplicação e facilita manutenção se um endpoint mudar.
- **Fixtures customizadas via `test.extend`**: os services são injetados como fixtures (`auth`, `links`), então cada teste já recebe as dependências prontas, sem setup manual repetido.
- **Factories com Faker**: `getUser()` e `getUserWithLink()` geram massa de dados realista e isolada por execução, evitando testes dependentes de dados fixos.
- **Isolamento de dados de teste**: usuários gerados nos testes usam o domínio `@enzo.dev`, o que permite ao `globalSetup` limpar exatamente esses registros no Postgres antes de cada execução, sem tocar em dados reais/manuais do ambiente.
- **Seed de massa em escala**: `seedUsers()` insere até 2.000 usuários em lotes de 100, com senha criptografada via bcrypt, e ainda exporta um CSV com credenciais em texto puro — usado como payload no teste de performance.

## Testes de performance (Artillery)

O cenário de carga simula um comportamento realista de tráfego, não só um teste de carga constante:

| Fase | Duração | Carga |
|---|---|---|
| Warmup | 30s | 5 req/s estável |
| Ramp-up | 20s | 5 → 20 req/s |
| Spike | 30s | 100 req/s (pico súbito) |
| Drop | 10s | 100 → 10 req/s |
| Recovery | 10s | 5 req/s |

Três fluxos ponderados por peso, usando os 2.000 usuários pré-cadastrados como payload:

- **Criar Link** (peso 50%) — login + criação de link
- **Listar Links** (peso 30%) — login + listagem
- **Fluxo Completo** (peso 20%) — login + criação + listagem

Esse desenho testa não só o comportamento sob carga estável, mas a resiliência da API num pico repentino de tráfego e sua recuperação logo em seguida — cenário comum em produtos com picos de uso (campanhas, lançamentos, etc).

## Como rodar

```bash
# Instalar dependências
npm install

# Configurar variáveis de ambiente
cp .env.example .env

# Rodar testes de API
npx playwright test

# Popular massa de usuários para o teste de performance
node -e "require('./playwright/support/database').seedUsers()"

# Rodar teste de performance
artillery run performance/test/<arquivo-do-cenario>.yml
```

## Tecnologias

Node.js, Playwright, Artillery, PostgreSQL, Bruno, Faker, ULID, bcrypt, dotenv
