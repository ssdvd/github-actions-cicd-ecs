# github-actions-cicd-ecs

Pipeline de CI/CD com GitHub Actions que testa uma API em Go, publica a imagem no Docker Hub e atualiza o serviço no Amazon ECS a cada push.

Projeto do curso **Integração Contínua: automatize o deploy no Amazon ECS**, da Alura. Continuação de [github-actions-cicd-ec2](https://github.com/ssdvd/github-actions-cicd-ec2).

## O pipeline

```
push / pull request
        │
      test ──► build ──► docker ──► deploy_ecs
```

| Job | Workflow | O que faz |
| --- | --- | --- |
| `test` | [`go.yml`](.github/workflows/go.yml) | Matriz com Go `1.19`, `1.20` e `>=1.20`; sobe o PostgreSQL com `docker-compose` |
| `build` | [`go.yml`](.github/workflows/go.yml) | Publica o binário `main` como artefato (`app_go`) |
| `docker` | [`docker.yml`](.github/workflows/docker.yml) | Builda e envia a imagem `ssdvd/go_ci:<número da execução>` |
| `deploy_ecs` | [`ecs.yml`](.github/workflows/ecs.yml) | Baixa a task definition atual, troca a imagem e as variáveis de ambiente e publica a nova revisão no serviço |

O deploy no ECS segue três passos:

1. `aws ecs describe-task-definition` exporta a definição da task `task-api-go`.
2. `amazon-ecs-render-task-definition` insere a nova imagem e as variáveis do banco no container `go`.
3. `amazon-ecs-deploy-task-definition` aplica a revisão no serviço `service-api-go` do cluster `api-go`.

O workflow [`ec2.yml`](.github/workflows/ec2.yml) do projeto anterior continua no repositório, mas a chamada dele está comentada no `go.yml`.

> Os passos `go test` e `go build` estão comentados no `go.yml`; o artefato publicado é o binário `main` versionado no repositório.

### Secrets necessários

| Secret | Uso |
| --- | --- |
| `USER_DOCKERHUB`, `PW_DOCKERHUB` | Login no Docker Hub |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credenciais da AWS (região `us-east-2`) |
| `DBHOST`, `DBPORT`, `DBUSER`, `DBPASSWORD`, `DBNAME` | Conexão com o PostgreSQL |

### Infraestrutura esperada

O pipeline não cria a infraestrutura; ele assume que já existem na conta:

- cluster ECS `api-go` com o serviço `service-api-go`;
- task definition `task-api-go` com um container chamado `go`;
- um PostgreSQL acessível pelas tasks.

## A aplicação

API REST de cadastro de alunos escrita em Go com [Gin](https://gin-gonic.com/) e [GORM](https://gorm.io/), usando PostgreSQL. É a aplicação de exemplo dos cursos da Alura (`guilhermeonrails/api-go-gin`); o foco deste repositório é o pipeline, não a API.

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/:nome` | Saudação em JSON |
| `GET` | `/alunos` | Lista todos os alunos |
| `GET` | `/alunos/:id` | Busca um aluno pelo ID |
| `GET` | `/alunos/cpf/:cpf` | Busca um aluno pelo CPF |
| `POST` | `/alunos` | Cria um aluno (`nome`, `cpf` com 11 dígitos, `rg` com 9 dígitos) |
| `PATCH` | `/alunos/:id` | Edita um aluno |
| `DELETE` | `/alunos/:id` | Remove um aluno |
| `GET` | `/index` | Página HTML com a lista de alunos |

## Rodando localmente

Pré-requisitos: Go 1.19 ou superior, Docker e Docker Compose.

```bash
# sobe o PostgreSQL e o pgAdmin
docker-compose up -d

# variáveis lidas pela aplicação para conectar no banco
export HOST=localhost USER=root PASSWORD=root DBNAME=root DBPORT=5432

go run main.go              # API em http://localhost:8080
go test -v main_test.go     # testes de integração (precisam do banco no ar)
```

O pgAdmin fica em <http://localhost:54321>.

## Série de CI/CD com GitHub Actions

Este repositório faz parte de uma sequência em que o mesmo pipeline vai ganhando etapas:

| # | Repositório | O que acrescenta |
| --- | --- | --- |
| 1 | [github-actions-ci](https://github.com/ssdvd/github-actions-ci) | Testes automatizados e matriz de versões do Go |
| 2 | [github-actions-ci-docker](https://github.com/ssdvd/github-actions-ci-docker) | Build da imagem e push para o Docker Hub |
| 3 | [github-actions-cicd-ec2](https://github.com/ssdvd/github-actions-cicd-ec2) | Deploy contínuo em uma instância EC2 via SSH |
| 4 | [github-actions-cicd-ecs](https://github.com/ssdvd/github-actions-cicd-ecs) | Deploy contínuo no Amazon ECS |
| 5 | [github-actions-cicd-rollback-tests](https://github.com/ssdvd/github-actions-cicd-rollback-tests) | Rollback automático e teste de carga |
| 6 | [github-actions-cicd-kubernetes](https://github.com/ssdvd/github-actions-cicd-kubernetes) | Deploy contínuo no Kubernetes (EKS) |

As anotações de cada aula estão na pasta [`notes/`](notes).
