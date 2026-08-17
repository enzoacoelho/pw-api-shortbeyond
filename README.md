# pw-api-shortbeyond

[![Playwright](https://img.shields.io/badge/Playwright-API-2EAD33?logo=playwright&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![Artillery](https://img.shields.io/badge/Artillery-Performance-FF6600?logo=artillery&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1?logo=postgresql&logoColor=white)](https://github.com/enzoacoelho/pw-api-shortbeyond)
[![Bruno](https://img.shields.io/badge/Bruno-API_Exploration-orange)](https://github.com/enzoacoelho/pw-api-shortbeyond)

Testes de API com **Playwright** num encurtador de links (ShortBeyond), cobrindo autenticação e gerenciamento de links, com massa de dados via PostgreSQL e teste de carga com **Artillery**.

## O que foi testado

**Autenticação**
- Cadastro de usuário (positivo e negativo)
- Login (credenciais corretas e incorretas)

**Links**
- Criação (`POST`)
- Listagem (`GET`)
- Remoção (`DELETE`)
- Cenários positivos e negativos em cada operação

Antes de automatizar, explorei os endpoints manualmente com **Bruno** (collection em `docs/shortbeyond`).

## Estrutura

```
pw-api-shortbeyond/
├── docs/shortbeyond/       # Collection Bruno
├── performance/
│   ├── data/               # Massa de usuários (usuarios.csv)
│   ├── reports/             # Relatórios do Artillery (JSON)
│   └── test/                 # Cenários de carga
├── playwright/
│   ├── e2e/                  # Testes de autenticação e links
│   └── support/
│       ├── factories/        # Geração de massa com Faker
│       ├── fixtures/         # test.extend, injeta os services
│       ├── services/          # Chamadas à API (auth, links)
│       ├── database.js        # Conexão Postgres, seed e limpeza
│       └── utils.js            # Geração de ULID
├── globalSetup.js              # Limpa dados de teste antes da execução
└── .env
```

Services centralizam as chamadas à API (`authService`, `linksService`) e são injetados via fixture, evitando duplicação entre os testes. Os dados de teste usam o domínio `@enzo.dev`, o que permite ao `globalSetup` limpar só esses registros no Postgres antes de cada execução.

## Performance (Artillery)

O cenário simula warmup, rampa de carga, um pico súbito e queda, usando os usuários pré-cadastrados como payload. Três fluxos ponderados: Criar Link (50%), Listar Links (30%) e Fluxo Completo — login + criação + listagem (20%).

## Como rodar

```bash
npm install
# edite o .env com suas credenciais locais

npx playwright test

# popular massa de usuários para o teste de performance
node -e "require('./playwright/support/database').seedUsers()"

artillery run performance/test/<arquivo-do-cenario>.yml
```

## Tecnologias

Node.js, Playwright, Artillery, PostgreSQL, Bruno, Faker, ULID, bcrypt, dotenv
