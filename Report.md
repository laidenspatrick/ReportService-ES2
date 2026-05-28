# Report Service — Resumo Funcional

## Visão Geral

O **Report Service** é o serviço responsável por consolidar e expor dados analíticos do sistema **Chave — Autoavaliação de Competências para Pessoas Idosas**. Ele é um consumidor de dados: não persiste avaliações, competências ou usuários diretamente, mas lê esses dados dos serviços especializados (Assessment Service, Competency Service e User & Group Service) para gerar relatórios e alimentar o painel administrativo.

---

## Responsabilidades

- Gerar relatórios consolidados por usuário, grupo, competência ou avaliação
- Expor painel administrativo com métricas globais do sistema
- Exportar relatórios nos formatos PDF, CSV e XLSX
- Rastrear o progresso individual de competências ao longo do tempo

---

## Dependências Externas

| Serviço               | Tipo de dependência | Dados consumidos                                      |
|-----------------------|---------------------|-------------------------------------------------------|
| Assessment Service    | Leitura             | Respostas submetidas, avaliações, categorias, níveis  |
| Competency Service    | Leitura             | Competências, subcompetências, níveis e suas definições |
| User & Group Service  | Leitura             | Usuários, grupos, membros                             |

---

## Endpoints

### Reports — `tag: Reports`

| Método   | Rota                             | Descrição                                             | Roles       |
|----------|----------------------------------|-------------------------------------------------------|-------------|
| `POST`   | `/reports`                       | Solicitar geração assíncrona de relatório             | USER, ADMIN |
| `GET`    | `/reports`                       | Listar relatórios (admin: todos; user: seus)          | USER, ADMIN |
| `GET`    | `/reports/me`                    | Listar apenas meus relatórios pessoais                | USER        |
| `GET`    | `/reports/me/progress`           | Meu progresso cronológico por competência             | USER        |
| `GET`    | `/reports/{reportId}`            | Buscar relatório com dados consolidados               | USER, ADMIN |
| `DELETE` | `/reports/{reportId}`            | Excluir relatório                                     | USER, ADMIN |

### Dashboard — `tag: Dashboard`

| Método | Rota                           | Descrição                                       | Roles |
|--------|--------------------------------|-------------------------------------------------|-------|
| `GET`  | `/dashboard/summary`           | Resumo geral (totais, tendências, top)          | ADMIN |
| `GET`  | `/dashboard/assessments`       | Estatísticas por avaliação publicada            | ADMIN |
| `GET`  | `/dashboard/users`             | Participação por usuário e por grupo            | ADMIN |
| `GET`  | `/dashboard/competencies`      | Cobertura e distribuição de níveis              | ADMIN |

### Exports — `tag: Exports`

| Método | Rota                                          | Descrição                              | Roles       |
|--------|-----------------------------------------------|----------------------------------------|-------------|
| `POST` | `/reports/{reportId}/exports`                 | Solicitar exportação (PDF/CSV/XLSX)    | USER, ADMIN |
| `GET`  | `/reports/{reportId}/exports`                 | Listar exportações de um relatório     | USER, ADMIN |
| `GET`  | `/reports/{reportId}/exports/{exportId}`      | Status e link de download da exportação| USER, ADMIN |

---

## Entidades

### Report
Metadados do relatório gerado. O campo `data` só é populado quando `status = READY`.

| Campo              | Tipo        | Descrição                                              |
|--------------------|-------------|--------------------------------------------------------|
| `id`               | integer     | PK                                                     |
| `type`             | enum        | `INDIVIDUAL` `GROUP` `COMPETENCY` `ASSESSMENT`         |
| `status`           | enum        | `PENDING` `PROCESSING` `READY` `FAILED`                |
| `requestedByUserId`| integer     | ID do usuário solicitante (extraído do JWT)             |
| `filters`          | object      | Filtros aplicados (período, usuário, grupo, competência)|
| `createdAt`        | datetime    |                                                        |
| `readyAt`          | datetime?   | Momento em que ficou disponível                        |
| `expiresAt`        | datetime?   | Expiração automática (padrão: 30 dias)                 |

### ExportJob
Job de exportação de um relatório em formato de arquivo.

| Campo         | Tipo     | Descrição                                              |
|---------------|----------|--------------------------------------------------------|
| `id`          | integer  | PK                                                     |
| `reportId`    | integer  | FK → Report                                            |
| `format`      | enum     | `PDF` `CSV` `XLSX`                                     |
| `status`      | enum     | `PENDING` `PROCESSING` `DONE` `FAILED`                 |
| `downloadUrl` | string?  | URL pré-assinada (validade 1h, presente quando DONE)   |
| `expiresAt`   | datetime?| Expiração da URL                                       |
| `createdAt`   | datetime |                                                        |
| `readyAt`     | datetime?|                                                        |

---

## Tipos de Relatório

### `INDIVIDUAL`
Consolida o desempenho de um único usuário em todas as avaliações submetidas no período.
- Disponível para qualquer usuário autenticado (para si mesmo)
- Admin pode gerar para qualquer usuário informando `filters.userId`
- Retorna: `IndividualReportData` com histórico de respostas e resumo por competência

### `GROUP`
Consolida o desempenho de todos os membros de um grupo.
- Exclusivo para ADMIN
- Requer `filters.groupId`
- Retorna: `GroupReportData` com taxa de participação, médias por competência e breakdown por membro

### `COMPETENCY`
Analisa a distribuição de níveis atingidos por competência.
- Disponível para qualquer usuário autenticado (verá apenas seus dados)
- Admin vê dados de todos os usuários; pode filtrar por `filters.competencyId`
- Retorna: `CompetencyReportData` com distribuição percentual por nível

### `ASSESSMENT`
Análise detalhada de uma avaliação específica: participação, conclusão e distribuição de respostas por categoria.
- Exclusivo para ADMIN
- Requer `filters.assessmentId`
- Retorna: `AssessmentReportData` com taxa de conclusão e breakdown por categoria

---

## Regras de Negócio

1. **userId extraído do JWT** — nunca enviado no body da requisição. Endpoints que aceitam `filters.userId` ignoram o campo para usuários comuns e usam sempre o ID do token.
2. **Geração assíncrona** — `POST /reports` e `POST /reports/{id}/exports` retornam status `202 Accepted` com status `PENDING`. O cliente deve fazer polling em `GET /reports/{id}`.
3. **Expiração de relatórios** — relatórios são excluídos automaticamente 30 dias após a geração.
4. **URLs de download** — pré-assinadas e válidas por 1 hora a partir da geração do arquivo.
5. **Acesso ao relatório** — usuários comuns só acessam relatórios que solicitaram. Admins acessam qualquer relatório.
6. **Dashboard em tempo real** — endpoints do dashboard não são cacheados por padrão e consultam os serviços externos diretamente.
7. **Filtros de data** — `dateFrom` e `dateTo` são inclusivos e no formato `YYYY-MM-DD`. Se omitidos, considera todo o histórico disponível.

---

## Fluxo de Geração de Relatório

```
1. POST /reports          → status: PENDING    (202 Accepted)
2. GET  /reports/{id}     → status: PROCESSING (aguarda)
3. GET  /reports/{id}     → status: READY      (dados disponíveis em data)

4. POST /reports/{id}/exports    → status: PENDING (202 Accepted)
5. GET  /reports/{id}/exports/{exportId} → status: DONE + downloadUrl
```

---

## Fluxo de Progresso Individual

```
GET /reports/me/progress
  → Consulta Assessment Service: respostas SUBMITTED do userId
  → Para cada resposta: lê categoryResults do Assessment Service
  → Consulta Competency Service: nome do nível de cada categoryResult
  → Agrupa por competencyId e ordena por submittedAt
  → Retorna array de CompetencyProgress com entradas cronológicas
```
