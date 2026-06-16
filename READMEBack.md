# API de Exames

API REST para gerenciamento de exames médicos: cadastro de tipos de exame, solicitação de exames vinculados a uma consulta (validada em um sistema externo chamado **G4**), atualização de status e registro de resultados.

Construída em **Node.js + Express 5**, com **Sequelize** sobre **PostgreSQL** e autenticação via **JWT**.

---

## Sumário

- [Visão geral](#visão-geral)
- [Arquitetura e estrutura de pastas](#arquitetura-e-estrutura-de-pastas)
- [Tecnologias](#tecnologias)
- [Pré-requisitos](#pré-requisitos)
- [Instalação e execução](#instalação-e-execução)
- [Variáveis de ambiente](#variáveis-de-ambiente)
- [Banco de dados](#banco-de-dados)
- [Autenticação](#autenticação)
- [Integração com o G4](#integração-com-o-g4)
- [Endpoints da API](#endpoints-da-api)
- [Modelo de dados](#modelo-de-dados)
- [Recursos do banco (view, procedures, triggers)](#recursos-do-banco-view-procedures-triggers)
- [Observações e melhorias sugeridas](#observações-e-melhorias-sugeridas)

---

## Visão geral

O sistema organiza o fluxo de exames de um atendimento clínico em torno de três entidades principais:

1. **Tipo de Exame** (`tipo_exame`) — catálogo de exames disponíveis (ex.: Hemograma, Glicemia, Raio-X).
2. **Solicitação de Exame** (`solicitacao_exame`) — pedido de um exame para uma determinada consulta, com justificativa clínica e um ciclo de status.
3. **Resultado de Exame** (`resultado_exame`) — laudo/resultado vinculado a uma solicitação (relação 1:1).

Ao criar uma solicitação, a API valida se a `consulta_id` (um UUID) existe em um sistema externo chamado **G4** antes de persistir o registro.

O ciclo de status de uma solicitação é: `PENDENTE` → `EM_ANDAMENTO` → `CONCLUIDO`, com possibilidade de `CANCELADO`.

---

## Arquitetura e estrutura de pastas

O projeto segue uma separação em camadas no padrão **Model–Controller**, com Sequelize como camada de acesso a dados.

```
Exames/
├── index.js                       # Ponto de entrada: configura Express, middlewares e rotas
├── banco.js                       # Instância/conexão Sequelize com o PostgreSQL
├── gerarToken.js                  # Utilitário para gerar um JWT de teste
├── package.json
├── .env.example                   # Modelo das variáveis de ambiente
├── .gitignore
│
├── middleware/
│   └── Auth.js                    # Middleware de autenticação JWT (Bearer token)
│
├── controllers/
│   ├── TipoExame.Controller.js
│   ├── SolicitacaoExame.Controller.js
│   └── ResultadoExame.Controller.js
│
├── models/
│   ├── TipoExame.Model.js
│   ├── SolicitacaoExame.Model.js
│   └── ResultadoExame.Model.js
│
└── bancoPostgres/
    ├── Script_banco.sql           # Criação de tabelas, índices, seeds, view, SP e trigger
    └── Script_SP_VIEW_TRIGGER.sql # View, procedure e trigger adicionais
```

Responsabilidade de cada camada:

- **Models** definem o mapeamento Sequelize de cada tabela.
- **Controllers** contêm a lógica de cada operação (validação de entrada, chamada ao banco, integração externa e resposta HTTP).
- **Middleware** intercepta as requisições protegidas e valida o token JWT.
- **`index.js`** amarra tudo: registra os middlewares globais (`cors`, `express.json`), conecta ao banco e mapeia as rotas para os métodos dos controllers.

---

## Tecnologias

| Dependência     | Versão    | Uso                                              |
|-----------------|-----------|--------------------------------------------------|
| express         | ^5.2.1    | Framework web / roteamento                       |
| sequelize       | ^6.37.8   | ORM para PostgreSQL                              |
| pg / pg-hstore  | ^8.21.0   | Driver PostgreSQL                                |
| jsonwebtoken    | ^9.0.3    | Geração e verificação de tokens JWT             |
| axios           | ^1.17.0   | Cliente HTTP para chamar a API externa do G4    |
| cors            | ^2.8.6    | Liberação de CORS                               |
| dotenv          | ^17.4.2   | Carregamento de variáveis de ambiente           |

O projeto usa **ES Modules** (`"type": "module"` no `package.json`), portanto a sintaxe é `import`/`export`.

---

## Pré-requisitos

- **Node.js** (versão compatível com Express 5; recomendado Node 18+).
- **PostgreSQL** rodando localmente.
- Acesso à **API do G4** (ou um endpoint compatível) para validação de consultas.

---

## Instalação e execução

```bash
# 1. Instalar dependências
npm install

# 2. Criar o arquivo .env a partir do modelo
cp .env.example .env
# edite o .env preenchendo JWT_SECRET, BEARER_G4 e G4_API_URL

# 3. Criar o banco de dados "Exames" no PostgreSQL e executar os scripts
psql -U postgres -d Exames -f bancoPostgres/Script_banco.sql
psql -U postgres -d Exames -f bancoPostgres/Script_SP_VIEW_TRIGGER.sql

# 4. Subir a API
node index.js
```

Ao iniciar, a aplicação tenta autenticar a conexão com o banco e sobe um servidor HTTP na **porta 6742**.

> Não há script `start` configurado no `package.json` (apenas o `test` padrão). A execução é feita diretamente com `node index.js`.

Para gerar um token de teste rapidamente:

```bash
node gerarToken.js
```

Esse utilitário assina um JWT com payload `{ userId: 1, nome: 'Exames' }`, válido por 1 hora, usando o `JWT_SECRET` do `.env`, e imprime o token no console.

---

## Variáveis de ambiente

Definidas no arquivo `.env` (veja `.env.example`):

| Variável     | Descrição                                                                 |
|--------------|---------------------------------------------------------------------------|
| `JWT_SECRET` | Segredo usado para assinar e verificar os tokens JWT.                     |
| `BEARER_G4`  | Token Bearer usado para autenticar as chamadas à API do G4.              |
| `G4_API_URL` | URL base da API de consultas do G4 (ex.: `http://localhost:3000/api/consultations/`). |

> Os dados de conexão com o PostgreSQL (banco `Exames`, usuário `postgres`, senha `postgres`, host `localhost`) estão **fixos no código** em `banco.js`, e não em variáveis de ambiente.

---

## Banco de dados

A conexão é configurada em `banco.js`:

```js
const banco = new Sequelize('Exames', 'postgres', 'postgres', {
    host: 'localhost',
    dialect: 'postgres',
    define: {
        timestamps: false,
        freezeTableName: true
    }
});
```

Pontos relevantes da configuração:

- `timestamps: false` — o Sequelize **não** cria automaticamente `createdAt`/`updatedAt`. O controle de datas é feito por campos manuais (`criado_em`, `solicitado_em`, `registrado_em`).
- `freezeTableName: true` — o nome da tabela é exatamente o passado em `define(...)`, sem pluralização automática.

---

## Autenticação

A autenticação é feita pelo middleware `middleware/Auth.js`, aplicado **globalmente após a rota raiz** em `index.js`:

```js
app.get('/', ...);   // rota pública (boas-vindas)
app.use(auth);       // a partir daqui, todas as rotas exigem token
```

Funcionamento do middleware:

1. Lê o cabeçalho `Authorization`.
2. Exige o formato `Bearer <token>`; caso contrário responde **401** com `{ error: 'Token não informado.' }`.
3. Verifica o token com `jwt.verify(token, JWT_SECRET)`.
4. Em caso de sucesso, anexa o payload decodificado em `req.user` e segue (`next()`).
5. Se o token for inválido/expirado, responde **401** com `{ error: 'Token inválido ou expirado.' }`.

Ou seja: **todas as rotas, exceto `GET /`, exigem um header** `Authorization: Bearer <token>`.

---

## Integração com o G4

Ao criar uma solicitação de exame, o controller valida a consulta no sistema externo G4 antes de gravar:

- A `consulta_id` deve ser um **UUID válido** (validado por regex). Caso contrário, retorna **422**.
- A API faz um `GET ${G4_API_URL}${consulta_id}` via `axios`, enviando `Authorization: Bearer ${BEARER_G4}`.
- Se o G4 não encontrar a consulta (ou a chamada falhar), retorna **422** com a mensagem `Consulta <id> não encontrada no G4.`.
- Só após a validação bem-sucedida a solicitação é persistida.

---

## Endpoints da API

Base URL: `http://localhost:6742`

Salvo a rota raiz, **todas exigem** `Authorization: Bearer <token>`.

### Geral

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| GET | `/` | Não | Mensagem de boas-vindas da API. |

### Tipo de Exame

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/TipoExame` | Lista todos os tipos de exame. |
| POST | `/TipoExame` | Cria um novo tipo de exame. |

**Corpo do POST `/TipoExame`:**

```json
{
  "nome": "Tomografia de Crânio",
  "descricao": "Exame de imagem detalhado da região craniana."
}
```

O campo `criado_em` é preenchido automaticamente com a data atual (`YYYY-MM-DD`).

### Solicitação de Exame

| Método | Rota | Descrição |
|--------|------|-----------|
| POST | `/SolicitacaoExame` | Cria uma solicitação (valida o UUID e a consulta no G4). |
| GET | `/SolicitacaoExame` | Lista todas as solicitações. |
| GET | `/SolicitacaoExame/:id` | Busca uma solicitação por ID. |
| PATCH | `/SolicitacaoExame/Status/:id` | Atualiza apenas o status da solicitação. |

**Corpo do POST `/SolicitacaoExame`:**

```json
{
  "consulta_id": "3f8a1c2e-0b4d-4e6f-8a90-1b2c3d4e5f60",
  "tipo_exame_id": 1,
  "justificativa": "Paciente com quadro de anemia para investigação.",
  "status": "PENDENTE",
  "solicitado_em": "2026-06-16"
}
```

**Corpo do PATCH `/SolicitacaoExame/Status/:id`:**

```json
{
  "status": "EM_ANDAMENTO"
}
```

Valores aceitos para `status`: `PENDENTE`, `EM_ANDAMENTO`, `CONCLUIDO`, `CANCELADO`. Um valor fora dessa lista retorna **422**.

### Resultado de Exame

| Método | Rota | Descrição |
|--------|------|-----------|
| POST | `/ResultadoExame/:solicitacao_exame_id` | Registra o resultado de uma solicitação. |
| GET | `/ResultadoExame` | Lista todos os resultados. |
| GET | `/ResultadoExame/:id` | Busca um resultado por ID. |

**Corpo do POST `/ResultadoExame/:solicitacao_exame_id`:**

```json
{
  "dados_resultado": "Hemoglobina 13,5 g/dL; Hematócrito 41%; sem alterações relevantes.",
  "observacoes": "Coleta realizada em jejum."
}
```

O `solicitacao_exame_id` vem pela URL e `registrado_em` é preenchido com a data atual. Como a relação é 1:1, cada solicitação aceita apenas um resultado.

### Códigos de resposta utilizados

- **200** — Sucesso (a API responde com `res.json(...)`).
- **401** — Token ausente, inválido ou expirado.
- **422** — Dados inválidos (UUID malformado, consulta inexistente no G4, status inválido).
- **500** — Erro interno; o corpo inclui `mensagem`, `detalhe` (detalhe do erro do banco) e `sql`.

---

## Modelo de dados

### Diagrama de relacionamento

```
tipo_exame (1) ────< (N) solicitacao_exame (1) ──── (1) resultado_exame
```

- Uma **solicitação** referencia um **tipo de exame** (`tipo_exame_id`).
- Um **resultado** referencia uma **solicitação** (`solicitacao_exame_id`, único).

### tipo_exame

| Coluna | Tipo | Restrições |
|--------|------|-----------|
| id | SERIAL | PK |
| nome | VARCHAR(150) | NOT NULL |
| descricao | TEXT | NOT NULL |
| criado_em | TIMESTAMP | NOT NULL, default `NOW()` |

### solicitacao_exame

| Coluna | Tipo | Restrições |
|--------|------|-----------|
| id | SERIAL | PK |
| consulta_id | UUID | NOT NULL |
| tipo_exame_id | INTEGER | NOT NULL, FK → `tipo_exame(id)` (ON UPDATE CASCADE, ON DELETE RESTRICT) |
| justificativa | TEXT | NOT NULL |
| status | VARCHAR(20) | NOT NULL, default `PENDENTE`, CHECK na lista de status válidos |
| solicitado_em | TIMESTAMP | NOT NULL, default `NOW()` |

### resultado_exame

| Coluna | Tipo | Restrições |
|--------|------|-----------|
| id | SERIAL | PK |
| solicitacao_exame_id | INTEGER | NOT NULL, **UNIQUE**, FK → `solicitacao_exame(id)` (ON UPDATE CASCADE, ON DELETE CASCADE) |
| dados_resultado | TEXT | NOT NULL |
| observacoes | TEXT | (opcional) |
| registrado_em | TIMESTAMP | NOT NULL, default `NOW()` |

### Índices

Criados para acelerar buscas frequentes: `consulta_id`, `tipo_exame_id` e `status` em `solicitacao_exame`, e `solicitacao_exame_id` em `resultado_exame`.

### Dados iniciais (seed)

O script insere 5 tipos de exame: Hemograma Completo, Glicemia em Jejum, Raio-X de Tórax, Eletrocardiograma e Urina Tipo I.

---

## Recursos do banco (view, procedures, triggers)

Os scripts SQL incluem objetos avançados de banco, além das tabelas:

### Views

- **`vw_solicitacoes_com_resultado`** — junta solicitação + tipo de exame + resultado (LEFT JOIN), expondo um registro consolidado por solicitação.
- **`vw_exames_por_paciente`** — histórico de exames agrupado por consulta (paciente), incluindo um campo booleano `possui_resultado`. Documentada para uso pelo serviço de exames no `GET /exam-requests` e na tela de histórico do frontend.

### Stored Procedures

- **`sp_registrar_resultado(p_solicitacao_id, p_dados_resultado, p_observacoes)`** — registra um resultado validando que a solicitação existe e que ainda não possui resultado; ao final, atualiza o status da solicitação para `CONCLUIDO`.
- **`sp_solicitar_exame(p_consulta_id, p_tipo_exame_id, p_justificativa)`** — cria uma solicitação validando a existência do tipo de exame, a obrigatoriedade da justificativa e a ausência de solicitação ativa duplicada (mesmo tipo + mesma consulta em status `PENDENTE`/`EM_ANDAMENTO`).

### Triggers

- **`trg_impedir_resultado_cancelado`** — impede registrar resultado para uma solicitação `CANCELADO`.
- **`trg_bloquear_resultado_duplicado`** — bloqueia resultado duplicado para a mesma solicitação, emitindo `unique_violation` para que a aplicação possa retornar **HTTP 409 Conflict**.

> Esses objetos parecem fazer parte de uma evolução do projeto (há referências a chamadas `[EE-20]`, `[EE-21]`, `[EE-22]` e a endpoints no padrão `/exam-requests`, `/exam-types`). Veja a seção abaixo.

---

## Observações e melhorias sugeridas

Pontos que valem atenção para quem for manter ou evoluir o projeto:

- **Procedures não são chamadas pela API.** Os controllers usam diretamente os métodos do Sequelize (`Model.create`, `Model.update`), e não as procedures `sp_solicitar_exame` / `sp_registrar_resultado`. Assim, validações importantes que existem só no banco (justificativa não-vazia, bloqueio de solicitação ativa duplicada, mudança automática de status para `CONCLUIDO`) **não são acionadas** pelo fluxo atual da aplicação. Considere alinhar a aplicação às procedures (ou replicar as regras na camada de aplicação).

- **Credenciais do banco fixas no código.** Usuário, senha e host estão escritos em `banco.js`. O ideal é movê-los para variáveis de ambiente (`.env`), junto às demais.

- **Inconsistência de nomenclatura de rotas.** A API expõe rotas em PascalCase (`/SolicitacaoExame`, `/ResultadoExame`, `/TipoExame`), enquanto os comentários do SQL mencionam rotas REST mais convencionais (`/exam-requests`, `/exam-types`). Convém padronizar.

- **Sem endpoint de atualização de resultado.** O comentário da trigger menciona "utilize o endpoint de atualização" para corrigir laudos, mas esse endpoint não existe no roteamento atual.

- **Respostas sem padronização de status HTTP.** Operações de leitura/criação retornam sempre `res.json(...)` (200), mesmo em criações (onde 201 seria mais apropriado) ou quando um recurso não é encontrado (onde 404 seria adequado — hoje `findByPk` sem resultado retorna `null` com 200).

- **Detalhes de erro expostos.** As respostas 500 retornam `mensagem`, `detalhe` e `sql` do banco, o que é útil em desenvolvimento, mas pode vazar informações sensíveis em produção.

- **Ausência de testes.** O `package.json` não define testes; o script `test` é o placeholder padrão.

- **README mínimo.** O `README.md` original contém apenas o título do projeto — este documento pode substituí-lo ou complementá-lo.
