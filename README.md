# Report Service Swagger

Repositório de documentação do **Report Service** — sistema Chave (PUCRS/UFRGS).

## Conteúdo

- `swagger.yaml` — especificação OpenAPI 3.0.3 da API.
- `report.md` — resumo funcional do serviço: endpoints, entidades e regras de negócio.
- `report_service.dbml` — modelo de dados em DBML (visualize em [dbdiagram.io](https://dbdiagram.io)).

## Resumo da API

A API documenta um serviço de relatórios e painel administrativo com:

- geração assíncrona de relatórios por usuário, grupo, competência ou avaliação;
- painel administrativo com métricas globais, estatísticas de avaliações e participação;
- exportação de relatórios em PDF, CSV e XLSX com URLs pré-assinadas;
- progresso individual de competências ao longo do tempo;
- autenticação via JWT em todos os endpoints.

## Como rodar o Swagger UI

Este repositório não contém backend — apenas a documentação OpenAPI.

Para visualizar o Swagger UI com Docker:

```bash
docker run -p 8080:8080 \
  -e SWAGGER_JSON=/swagger.yaml \
  -v "$PWD/swagger.yaml:/swagger.yaml" \
  swaggerapi/swagger-ui
```

Abra no navegador:

```
http://localhost:8080
```

## Relacionamento com outros serviços

O Report Service consome dados de:

| Serviço               | Dados lidos                                          |
|-----------------------|------------------------------------------------------|
| Assessment Service    | Respostas submetidas, avaliações, categorias, níveis |
| Competency Service    | Competências, subcompetências e seus níveis          |
| User & Group Service  | Usuários e grupos                                    |

Veja o diagrama de arquitetura do sistema em `assessment_diagram.png` do repositório Assessment Service.

## Observação

O botão **Try it out** do Swagger UI só funcionará se houver uma API real rodando compatível
com o `servers.url` definido no `swagger.yaml`.
