# Plataforma de pagamentos em Go + GCP

## Objetivo do projeto
Construir uma plataforma de pagamentos simplificada em Go, rodando no Google Cloud (GCP). O projeto tem dois objetivos ao mesmo tempo: aprender Go e GCP na prática e montar um projeto de portfólio que cubra o que vagas de backend em Go costumam pedir (microsserviços, arquitetura hexagonal, PostgreSQL, mensageria, Kubernetes, observabilidade, testes e CI/CD). Como bônus, a prática serve de preparação para a certificação GCP Associate Cloud Engineer (ACE).

O plano completo está em `docs/ROADMAP.md`: 10 épicos e 52 tarefas, com passo a passo, código base e critério de pronto. Esse arquivo é a fonte de verdade do que fazer e em que ordem.

## Como o Claude deve trabalhar comigo
Eu estou começando em Go neste projeto e muita coisa é nova. Por isso:
- Explique cada conceito novo de Go ou de GCP antes de usá-lo, em poucas frases e em português.
- Trabalhe em passos pequenos, uma tarefa do ROADMAP por vez. Não gere o projeto inteiro de uma vez.
- Prefira me guiar para eu escrever o código principal. Quando você escrever, explique o que cada parte faz e por que foi feita assim.
- Sempre termine cada passo com o comando exato para validar (`go test`, `go run`, `curl`, etc.) e o resultado esperado.
- Quando um comando falhar, me ajude a ler a mensagem de erro antes de propor a correção.
- Marque o status das tarefas no `docs/ROADMAP.md` conforme avançamos (backlog, em andamento, em revisão, concluído).
- Se uma decisão tiver trade-offs (por exemplo, devolver `nil, nil` ou um erro `ErrNotFound`), apresente as opções e me deixe decidir. Registre decisões importantes em `docs/adr/`.
- Antes de instalar ferramentas ou rodar algo que altere minha máquina (Docker, CLIs, `go install`), me diga o que vai fazer.

## Arquitetura: hexagonal (portas e adaptadores)
Regra de dependência: `adapters` dependem de `ports` e `domain`; `ports` depende de `domain`; `domain` não importa nada do projeto. O `main` monta tudo.

```
cmd/payments-api/      ponto de entrada, monta as dependências
internal/domain/       entidades e regras de negócio (Payment, Account, erros)
internal/ports/        interfaces (PaymentRepository, EventBus, ...)
internal/app/          casos de uso (service)
internal/adapters/
  httpapi/             handlers HTTP (REST)
  postgres/            repositórios com pgx
  memory/              repositórios em memória (testes e início)
migrations/            SQL versionado (golang-migrate)
docs/                  ROADMAP.md e ADRs
```

## Stack
- Go 1.22 ou mais recente (o roteamento `POST /payments` do `net/http` exige 1.22+)
- PostgreSQL 16 (Docker Compose localmente, Cloud SQL no GCP) com `pgx/v5`
- golang-migrate para migrations
- Mensageria atrás de uma porta `EventBus`: NATS localmente e Pub/Sub no GCP
- Docker, Kubernetes (kind localmente, GKE Autopilot no GCP), Terraform, CI/CD
- Observabilidade: slog, Prometheus, OpenTelemetry, Grafana
- Testes: biblioteca padrão, testify, testcontainers; lint com golangci-lint

## Convenções de código
- Identificadores em inglês; comentários e documentação em português.
- Dinheiro sempre em centavos (`int64`), nunca `float`.
- Erros: `fmt.Errorf("contexto: %w", err)`, comparação com `errors.Is` e `errors.As`. Erros de domínio são variáveis (`ErrInvalidAmount`).
- `context.Context` é sempre o primeiro parâmetro de funções que fazem I/O.
- Testes table-driven com `t.Run`. Em goroutines de teste, usar `t.Errorf` (nunca `t.Fatal`).
- Meta: zero `//nolint`, cobertura do domínio igual ou acima de 80%, `go test -race` sempre verde.
- O domínio não importa pacotes de `adapters` nem de `ports` (a regra será reforçada com o linter `depguard`).
- Commits pequenos, um por passo do ROADMAP, com mensagem clara.

## Comandos úteis
```
go run ./cmd/payments-api
go test ./...
go test -race -count=1 ./...
go test -cover ./internal/domain/
golangci-lint run
docker compose up -d
migrate -path migrations -database "$DATABASE_URL" up
```
`DATABASE_URL=postgres://app:app@localhost:5432/payments?sslmode=disable`

## Estado atual
Nada foi implementado ainda. Trocar `SEU_USUARIO` pelo meu usuário do GitHub nos caminhos de módulo (`github.com/SEU_USUARIO/payments`).
- Em andamento: E1-1 (estudar a sintaxe do Go no Tour of Go).
- Próximo passo prático: preparar o ambiente (Go, Docker, Git, VS Code), criar o repositório e o módulo, e seguir o ROADMAP a partir do épico E1.
- Épicos E1 a E3 estão detalhados com passo a passo e código base. Os épicos E4 a E10 estão resumidos e devem ser detalhados no mesmo formato quando chegarmos neles (peça isso ao Claude no momento certo).

## Resumo dos épicos
1. E1 Go e concorrência (P0)
2. E2 Núcleo hexagonal e API REST (P0)
3. E3 Ledger e PostgreSQL (P0)
4. E4 Eventos e mensageria: outbox, EventBus, consumidor idempotente (P0)
5. E5 Testes e CI/CD (P0)
6. E6 Docker, Kubernetes e Terraform (P0)
7. E7 Observabilidade e confiabilidade (P0)
8. E8 Export de volume e PII (P1)
9. E9 GCP e certificação ACE (P1)
10. E10 Extras: gRPC, MCP, painel React e ADRs (P2)
