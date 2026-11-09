# Roadmap: plataforma de pagamentos em Go + GCP

Fonte de verdade do plano. Cada épico agrupa tarefas de **Estudar** e **Implementar**. Atualize o campo **Status** conforme avançar: `backlog`, `em andamento`, `em revisão` ou `concluído`.

Tarefas de implementação detalhadas (conceitos novos, passo a passo, código base e como validar): E1-4, E2-2, E2-3, E2-5, E3-2, E3-3, E3-4 e E3-5. As demais ainda estão resumidas e devem ser detalhadas no mesmo formato antes de começarem.

Observação: os trechos de código são esqueletos para guiar o aprendizado e não foram executados. Se algo não compilar, leia a mensagem de erro com o Claude e corrija junto. Troque `SEU_USUARIO` pelo usuário do GitHub.

## E1 Go e concorrência (P0, Semanas 1-3)

Base da linguagem. Aplica no app: worker pool e context usados no export e nos consumidores.

### E1-1 [Estudar] Sintaxe e tipos do Go

- **Status:** em andamento
- **O que fazer:** Tipos, structs, slices, maps, ponteiros e funções. Faça os exercícios do tour e reescreva três deles sem olhar.
- **Aprendeu quando:** Consigo escrever um programa com structs e maps sem consultar.
- **Material:** https://go.dev/tour/

### E1-2 [Estudar] Interfaces, erros e generics

- **Status:** backlog
- **O que fazer:** Interfaces implícitas, errors.Is e errors.As, wrapping com %w e tipos genéricos. É a base da arquitetura hexagonal.
- **Aprendeu quando:** Sei explicar por que interface pequena é melhor e quando usar generics.
- **Material:** https://go.dev/doc/effective_go

### E1-3 [Estudar] Concorrência: goroutines, channels e sync

- **Status:** backlog
- **O que fazer:** Goroutines, canais com e sem buffer, select, WaitGroup, Mutex e vazamento de goroutine.
- **Aprendeu quando:** Sei apontar uma race condition e corrigi-la com go test -race.
- **Material:** https://gobyexample.com/

### E1-4 [Implementar] Pacote internal/pool: worker pool com context

- **Status:** backlog
- **O que fazer:** Crie um pool com N workers que recebe jobs por canal, respeita context.Done e devolve erros agregados. Será usado no export e nos consumidores.
- **Pronto quando:** Teste com -race passando e o cancelamento encerra todas as goroutines.
- **Conceitos novos:** goroutine, channel, sync.WaitGroup, select, context, errors.Join

**Passo a passo**

- [ ] No terminal: mkdir payments && cd payments && go mod init github.com/SEU_USUARIO/payments
- [ ] Criar internal/pool/pool.go com o código ao lado. Leia linha por linha e escreva os comentários com suas palavras
- [ ] Entender o fluxo: o laço principal envia jobs no canal ch e cada worker lê de ch até ele ser fechado
- [ ] Entender o buffer de errCh: com tamanho len(jobs), nenhum worker trava ao reportar um erro
- [ ] Criar internal/pool/pool_test.go com três casos: 100 jobs com sucesso, um job que falha e context cancelado no meio (esqueleto ao lado)
- [ ] Rodar go test -race ./internal/pool/ e corrigir qualquer aviso
- [ ] Experimento: remova o WaitGroup, rode o teste e observe o que quebra. Depois desfaça

**Código base**

```
// internal/pool/pool.go
package pool

import (
	"context"
	"errors"
	"sync"
)

type Job func(ctx context.Context) error

// Run executa os jobs com n workers e devolve todos os erros juntos.
func Run(ctx context.Context, n int, jobs []Job) error {
	ch := make(chan Job)
	errCh := make(chan error, len(jobs))
	var wg sync.WaitGroup

	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range ch {
				if err := job(ctx); err != nil {
					errCh <- err
				}
			}
		}()
	}

loop:
	for _, j := range jobs {
		select {
		case ch <- j:
		case <-ctx.Done():
			break loop
		}
	}
	close(ch)
	wg.Wait()
	close(errCh)

	var errs []error
	for err := range errCh {
		errs = append(errs, err)
	}
	if err := ctx.Err(); err != nil {
		errs = append(errs, err)
	}
	return errors.Join(errs...)
}

// internal/pool/pool_test.go (esqueleto do caso de cancelamento)
func TestRun_cancelamento(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	jobs := make([]Job, 100)
	for i := range jobs {
		jobs[i] = func(ctx context.Context) error {
			cancel() // cancela no primeiro job
			return nil
		}
	}
	err := Run(ctx, 4, jobs)
	if !errors.Is(err, context.Canceled) {
		t.Fatalf("esperava context.Canceled, veio %v", err)
	}
}
```

**Como validar:** `go test -race -count=1 ./internal/pool/`

## E2 Núcleo hexagonal e API REST (P0, Semanas 1-3)

Domínio isolado e API com contrato. Aplica no app: payments-api com Idempotency-Key.

### E2-1 [Estudar] Arquitetura hexagonal: portas e adaptadores

- **Status:** backlog
- **O que fazer:** Domínio no centro, portas (interfaces) e adaptadores nas bordas. Diferença entre porta de entrada e de saída e a regra de dependência.
- **Aprendeu quando:** Consigo desenhar o fluxo de um pagamento marcando domínio, portas e adaptadores.
- **Material:** https://alistair.cockburn.us/hexagonal-architecture/

### E2-2 [Implementar] Esqueleto do módulo hexagonal

- **Status:** backlog
- **O que fazer:** Criar domain/, ports/, adapters/ e cmd/ com go.mod. O pacote domain não pode importar nada de adapters.
- **Pronto quando:** Um lint (depguard) falha se o domínio importar infraestrutura.
- **Conceitos novos:** módulo (go.mod), pacote internal/, regra de dependência entre camadas, depguard

**Passo a passo**

- [ ] Na raiz do módulo: mkdir -p cmd/payments-api internal/domain internal/ports internal/adapters/memory internal/adapters/httpapi internal/app
- [ ] Criar cmd/payments-api/main.go com o código ao lado e rodar go run ./cmd/payments-api
- [ ] Memorizar a regra: adapters dependem de ports e domain; ports depende de domain; domain não importa nada do projeto
- [ ] Instalar o golangci-lint (link do card de lint) e criar .golangci.yml com a regra ao lado. A sintaxe mudou na v2, então confira na documentação da sua versão
- [ ] Provar que a regra funciona: em um arquivo de internal/domain, importe temporariamente internal/adapters/memory, rode golangci-lint run e veja o erro. Depois remova o import
- [ ] Rodar git init, criar .gitignore com bin/ e fazer o primeiro commit

**Código base**

```
// cmd/payments-api/main.go
package main

import "fmt"

func main() {
	fmt.Println("ok")
}

# .golangci.yml
version: "2"
linters:
  default: none
  enable: [depguard]
  settings:
    depguard:
      rules:
        domain:
          list-mode: lax
          files: ["**/internal/domain/**"]
          deny:
            - pkg: github.com/SEU_USUARIO/payments/internal/adapters
              desc: domínio não pode depender de adapters
            - pkg: github.com/SEU_USUARIO/payments/internal/ports
              desc: domínio não pode depender de ports
```

**Como validar:** `go run ./cmd/payments-api && golangci-lint run`

### E2-3 [Implementar] Entidades Payment e Account no domínio

- **Status:** backlog
- **O que fazer:** Estados do pagamento (criado, autorizado, capturado, estornado), valores em centavos (int64) e regras de transição inválidas.
- **Pronto quando:** Testes table-driven cobrem todas as transições permitidas e proibidas.
- **Conceitos novos:** structs, métodos com ponteiro (*Payment), errors.New, errors.Is, testes table-driven e t.Run

**Passo a passo**

- [ ] Criar internal/domain/payment.go com o código ao lado
- [ ] Notar: dinheiro em centavos (int64), nunca float
- [ ] Notar: o método move concentra a regra de transição, então Authorize, Capture e Refund ficam com uma linha cada
- [ ] Criar internal/domain/payment_test.go começando pelo caso de Capture (ao lado). Depois copie o padrão para Authorize e Refund
- [ ] Adicionar casos de NewPayment com valor 0 e negativo, esperando ErrInvalidAmount
- [ ] Rodar go test -cover ./internal/domain/ e chegar a 80% ou mais

**Código base**

```
// internal/domain/payment.go
package domain

import "errors"

type Status string

const (
	StatusCreated    Status = "created"
	StatusAuthorized Status = "authorized"
	StatusCaptured   Status = "captured"
	StatusRefunded   Status = "refunded"
)

var (
	ErrInvalidAmount     = errors.New("valor deve ser maior que zero")
	ErrInvalidTransition = errors.New("transição de status inválida")
)

type Payment struct {
	ID          string
	AmountCents int64
	Status      Status
}

func NewPayment(id string, amountCents int64) (*Payment, error) {
	if amountCents <= 0 {
		return nil, ErrInvalidAmount
	}
	return &Payment{ID: id, AmountCents: amountCents, Status: StatusCreated}, nil
}

func (p *Payment) Authorize() error { return p.move(StatusCreated, StatusAuthorized) }
func (p *Payment) Capture() error   { return p.move(StatusAuthorized, StatusCaptured) }
func (p *Payment) Refund() error    { return p.move(StatusCaptured, StatusRefunded) }

func (p *Payment) move(from, to Status) error {
	if p.Status != from {
		return ErrInvalidTransition
	}
	p.Status = to
	return nil
}

// internal/domain/payment_test.go (padrão table-driven)
func TestCapture(t *testing.T) {
	tests := []struct {
		name    string
		from    Status
		wantErr error
	}{
		{"autorizado captura", StatusAuthorized, nil},
		{"criado não captura", StatusCreated, ErrInvalidTransition},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			p := &Payment{Status: tc.from}
			if err := p.Capture(); !errors.Is(err, tc.wantErr) {
				t.Fatalf("got %v, want %v", err, tc.wantErr)
			}
		})
	}
}
```

**Como validar:** `go test -cover ./internal/domain/`

### E2-4 [Estudar] Contrato de API: OpenAPI e net/http

- **Status:** backlog
- **O que fazer:** Escrever spec OpenAPI 3, rotas, status codes, erros padronizados e idempotência em POST.
- **Aprendeu quando:** Sei por que o POST de pagamento exige Idempotency-Key e como reaproveitar a resposta.
- **Material:** https://swagger.io/specification/

### E2-5 [Implementar] POST /payments com Idempotency-Key

- **Status:** backlog
- **O que fazer:** Handler HTTP que valida, chama o caso de uso e devolve a mesma resposta quando a chave se repete. Spec openapi.yaml versionada.
- **Pronto quando:** Duas chamadas com a mesma chave criam um único pagamento, comprovado em teste.
- **Conceitos novos:** interface como porta, http.Handler e ServeMux, encoding/json, httptest, status codes HTTP

**Passo a passo**

- [ ] Em internal/ports/payment.go, declarar a interface PaymentRepository (primeiro bloco ao lado)
- [ ] Adicionar tags json em Payment (por exemplo `json:"id"`) para a resposta sair em JSON
- [ ] Criar internal/adapters/memory/payment_repo.go: um map[string]*domain.Payment protegido por sync.Mutex e indexado pela chave de idempotência
- [ ] Criar internal/app/service.go com Create(ctx, key, amountCents): consulta o repositório pela chave; se existir, devolve o existente; senão cria com domain.NewPayment, salva e devolve
- [ ] Criar internal/adapters/httpapi/handler.go com o handler ao lado e registrar a rota com mux.HandleFunc("POST /payments", h.create), que exige Go 1.22 ou mais
- [ ] Montar tudo no main.go: repositório em memória, serviço, handler e http.ListenAndServe(":8080", mux)
- [ ] Escrever o teste com httptest.NewRecorder enviando a mesma chave duas vezes e comparando os ids
- [ ] Escrever openapi.yaml mínimo descrevendo POST /payments, o header Idempotency-Key e as respostas 201, 400 e 422

**Código base**

```
// internal/ports/payment.go
type PaymentRepository interface {
	Save(ctx context.Context, key string, p *domain.Payment) error
	FindByKey(ctx context.Context, key string) (*domain.Payment, error)
}

// internal/adapters/httpapi/handler.go
func (h *Handler) create(w http.ResponseWriter, r *http.Request) {
	key := r.Header.Get("Idempotency-Key")
	if key == "" {
		http.Error(w, "Idempotency-Key obrigatório", http.StatusBadRequest)
		return
	}
	var in struct {
		AmountCents int64 `json:"amount_cents"`
	}
	if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
		http.Error(w, "json inválido", http.StatusBadRequest)
		return
	}
	p, err := h.svc.Create(r.Context(), key, in.AmountCents)
	if errors.Is(err, domain.ErrInvalidAmount) {
		http.Error(w, err.Error(), http.StatusUnprocessableEntity)
		return
	}
	if err != nil {
		http.Error(w, "erro interno", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	_ = json.NewEncoder(w).Encode(p)
}
```

**Como validar:** `go run ./cmd/payments-api e, em outro terminal, repetir duas vezes: curl -i -X POST localhost:8080/payments -H 'Idempotency-Key: abc' -d '{"amount_cents":1000}'. O id da resposta deve ser o mesmo`

## E3 Ledger e PostgreSQL (P0, Semanas 2-4)

Dinheiro com consistência. Aplica no app: partidas dobradas e proteção contra concorrência.

### E3-1 [Estudar] Transações e isolamento no PostgreSQL

- **Status:** backlog
- **O que fazer:** Read committed vs serializable, locks de linha, deadlock e quando usar SELECT FOR UPDATE.
- **Aprendeu quando:** Sei explicar o que acontece com dois débitos simultâneos na mesma conta.
- **Material:** https://www.postgresql.org/docs/current/transaction-iso.html

### E3-2 [Implementar] Migrations do esquema financeiro

- **Status:** backlog
- **O que fazer:** Tabelas accounts, payments e ledger_entries com golang-migrate, chaves, índices e constraint de valor positivo.
- **Pronto quando:** Subir e descer as migrations em banco limpo sem erro.
- **Conceitos novos:** migration up/down, constraint CHECK, chave estrangeira, índice, docker compose

**Passo a passo**

- [ ] Criar docker-compose.yml na raiz com o Postgres (bloco ao lado) e subir com docker compose up -d
- [ ] Instalar a CLI: go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
- [ ] Exportar a variável: export DATABASE_URL='postgres://app:app@localhost:5432/payments?sslmode=disable'
- [ ] Gerar os arquivos: migrate create -ext sql -dir migrations -seq create_ledger. Isso cria 000001_create_ledger.up.sql e .down.sql
- [ ] Escrever o up.sql e o down.sql com o conteúdo ao lado. Regra do ledger: débito é valor negativo, crédito é positivo, e a soma por pagamento é zero
- [ ] Rodar migrate -path migrations -database "$DATABASE_URL" up, conferir as tabelas com \dt, rodar down e up de novo
- [ ] Testar a constraint: tente inserir um pagamento com amount_cents = -5 no psql e leia a mensagem de erro

**Código base**

```
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: payments
    ports: ["5432:5432"]

-- migrations/000001_create_ledger.up.sql
CREATE TABLE accounts (
    id         TEXT PRIMARY KEY,
    balance    BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id              TEXT PRIMARY KEY,
    idempotency_key TEXT NOT NULL UNIQUE,
    amount_cents    BIGINT NOT NULL CHECK (amount_cents > 0),
    status          TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ledger_entries (
    id           BIGSERIAL PRIMARY KEY,
    payment_id   TEXT NOT NULL REFERENCES payments(id),
    account_id   TEXT NOT NULL REFERENCES accounts(id),
    amount_cents BIGINT NOT NULL CHECK (amount_cents <> 0),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON ledger_entries (account_id);

-- migrations/000001_create_ledger.down.sql
DROP TABLE ledger_entries;
DROP TABLE payments;
DROP TABLE accounts;
```

**Como validar:** `migrate -path migrations -database "$DATABASE_URL" up && docker compose exec db psql -U app -d payments -c "\dt"`

### E3-3 [Implementar] Lançamento em partidas dobradas

- **Status:** backlog
- **O que fazer:** Cada pagamento gera um débito e um crédito de mesmo valor na mesma transação; a soma das entradas é sempre zero.
- **Pronto quando:** Teste prova que falha no meio da transação não deixa lançamento pela metade.
- **Conceitos novos:** pgxpool, transação (Begin, Commit, Rollback), defer, parâmetros $1 e $2 nas queries, fmt.Errorf com %w

**Passo a passo**

- [ ] Instalar o driver: go get github.com/jackc/pgx/v5 (o pgxpool vem junto)
- [ ] Criar internal/adapters/postgres/ledger.go com a função RecordTransfer do código ao lado
- [ ] Entender o defer tx.Rollback(ctx): se algo falhar no meio ele desfaz tudo; depois do Commit ele não faz nada
- [ ] Entender os valores: o débito entra como -amount na conta de origem e o crédito como +amount na de destino, então a soma é zero
- [ ] Criar internal/adapters/postgres/ledger_test.go: no setup, insira duas contas e um pagamento (a chave estrangeira exige que existam), depois chame RecordTransfer
- [ ] Teste de falha: chame RecordTransfer com uma conta de destino que não existe e confirme que nenhuma linha de ledger_entries ficou para aquele pagamento
- [ ] Conferir no banco: a soma de amount_cents do pagamento deve ser 0

**Código base**

```
// internal/adapters/postgres/ledger.go
package postgres

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

func RecordTransfer(ctx context.Context, pool *pgxpool.Pool, paymentID, from, to string, amount int64) error {
	tx, err := pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin: %w", err)
	}
	defer tx.Rollback(ctx) // sem efeito depois do Commit

	const insert = `INSERT INTO ledger_entries (payment_id, account_id, amount_cents) VALUES ($1, $2, $3)`
	if _, err := tx.Exec(ctx, insert, paymentID, from, -amount); err != nil {
		return fmt.Errorf("débito: %w", err)
	}
	if _, err := tx.Exec(ctx, insert, paymentID, to, amount); err != nil {
		return fmt.Errorf("crédito: %w", err)
	}

	const upd = `UPDATE accounts SET balance = balance + $1 WHERE id = $2`
	if _, err := tx.Exec(ctx, upd, -amount, from); err != nil {
		return fmt.Errorf("saldo origem: %w", err)
	}
	if _, err := tx.Exec(ctx, upd, amount, to); err != nil {
		return fmt.Errorf("saldo destino: %w", err)
	}
	return tx.Commit(ctx)
}
```

**Como validar:** `go test -count=1 ./internal/adapters/postgres/ (com o compose no ar). No psql: SELECT SUM(amount_cents) FROM ledger_entries WHERE payment_id = 'p1'; deve dar 0`

### E3-4 [Implementar] Proteção contra concorrência no saldo

- **Status:** backlog
- **O que fazer:** Usar SELECT FOR UPDATE (ou versão otimista) ao debitar. Escrever teste com 50 goroutines debitando a mesma conta.
- **Pronto quando:** Saldo final correto e go test -race sem alertas.
- **Conceitos novos:** SELECT ... FOR UPDATE, lost update, sync/atomic, t.Errorf dentro de goroutine (nunca t.Fatal), limite de conexões do pool

**Passo a passo**

- [ ] Entender o problema (lost update): duas transações leem saldo 100, as duas debitam 100, e o saldo final fica errado. Escreva com suas palavras no README como o FOR UPDATE evita isso
- [ ] Adicionar ErrInsufficientFunds ao bloco var de internal/domain/payment.go
- [ ] Criar internal/adapters/postgres/debit.go com a função Debit do código ao lado
- [ ] Escrever o teste concorrente: conta com saldo 10000 e 200 goroutines debitando 100 cada. Esperado: 100 sucessos, 100 ErrInsufficientFunds e saldo final 0
- [ ] Usar t.Errorf (e não t.Fatal) dentro das goroutines: t.Fatal só pode ser chamado pela goroutine do teste
- [ ] Experimento: remova o FOR UPDATE da query, rode o teste de novo e veja os números errados. Depois desfaça
- [ ] Rodar go test -race -count=1 e anotar no README o que observou

**Código base**

```
// internal/adapters/postgres/debit.go
func Debit(ctx context.Context, pool *pgxpool.Pool, accountID string, amount int64) error {
	tx, err := pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	var balance int64
	err = tx.QueryRow(ctx,
		`SELECT balance FROM accounts WHERE id = $1 FOR UPDATE`, accountID,
	).Scan(&balance)
	if err != nil {
		return fmt.Errorf("ler saldo: %w", err)
	}
	if balance < amount {
		return domain.ErrInsufficientFunds
	}
	if _, err := tx.Exec(ctx,
		`UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amount, accountID,
	); err != nil {
		return fmt.Errorf("debitar: %w", err)
	}
	return tx.Commit(ctx)
}

// esqueleto do teste
func TestDebit_concorrente(t *testing.T) {
	// preparar: conta "acc1" com saldo 10000
	var wg sync.WaitGroup
	var ok, insufficient atomic.Int64
	for i := 0; i < 200; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			err := Debit(ctx, pool, "acc1", 100)
			switch {
			case err == nil:
				ok.Add(1)
			case errors.Is(err, domain.ErrInsufficientFunds):
				insufficient.Add(1)
			default:
				t.Errorf("erro inesperado: %v", err)
			}
		}()
	}
	wg.Wait()
	// conferir: ok == 100, insufficient == 100 e saldo final 0
}
```

**Como validar:** `go test -race -count=1 -run TestDebit_concorrente ./internal/adapters/postgres/`

### E3-5 [Implementar] Adapter pgx para o repositório

- **Status:** backlog
- **O que fazer:** Implementar a porta AccountRepository com pgxpool, mapeando erros do banco para erros de domínio.
- **Pronto quando:** Os testes de contrato passam com o adapter real e com um fake em memória.
- **Conceitos novos:** pgxpool, Exec/QueryRow/Scan, pgx.ErrNoRows, erro 23505 do Postgres (unique violada), teste de contrato, variável de ambiente

**Passo a passo**

- [ ] Adicionar ErrDuplicateKey ao bloco var de internal/domain/payment.go
- [ ] Criar internal/adapters/postgres/payment_repo.go implementando a interface ports.PaymentRepository do card E2-5, com o código ao lado. Imports: pgx/v5, pgx/v5/pgconn e pgx/v5/pgxpool
- [ ] Decidir e documentar o contrato: FindByKey devolve nil, nil quando não existe. A alternativa é um erro ErrNotFound. O adapter em memória precisa seguir a mesma escolha
- [ ] Escrever um teste de contrato testPaymentRepo(t, repo ports.PaymentRepository) com os casos: salvar e achar pela chave, chave inexistente e chave duplicada. Chamar o teste com o adapter em memória e com o Postgres
- [ ] No main.go: ler DATABASE_URL com os.Getenv, criar o pool com pgxpool.New e usar NewPaymentRepo no lugar do adapter em memória
- [ ] Rodar a API, repetir duas vezes o curl do card E2-5 e conferir que existe uma única linha em payments

**Código base**

```
// internal/adapters/postgres/payment_repo.go
type PaymentRepo struct{ pool *pgxpool.Pool }

func NewPaymentRepo(pool *pgxpool.Pool) *PaymentRepo { return &PaymentRepo{pool: pool} }

func (r *PaymentRepo) Save(ctx context.Context, key string, p *domain.Payment) error {
	_, err := r.pool.Exec(ctx,
		`INSERT INTO payments (id, idempotency_key, amount_cents, status) VALUES ($1, $2, $3, $4)`,
		p.ID, key, p.AmountCents, string(p.Status))
	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) && pgErr.Code == "23505" {
		return domain.ErrDuplicateKey
	}
	return err
}

func (r *PaymentRepo) FindByKey(ctx context.Context, key string) (*domain.Payment, error) {
	var p domain.Payment
	var status string
	err := r.pool.QueryRow(ctx,
		`SELECT id, amount_cents, status FROM payments WHERE idempotency_key = $1`, key,
	).Scan(&p.ID, &p.AmountCents, &status)
	if errors.Is(err, pgx.ErrNoRows) {
		return nil, nil
	}
	if err != nil {
		return nil, err
	}
	p.Status = domain.Status(status)
	return &p, nil
}
```

**Como validar:** `DATABASE_URL="postgres://app:app@localhost:5432/payments?sslmode=disable" go test -count=1 ./internal/adapters/... e depois: docker compose exec db psql -U app -d payments -c "SELECT count(*) FROM payments"`

## E4 Eventos e mensageria (P0, Semanas 4-5)

Arquitetura orientada a eventos. Aplica no app: pedido.criado e pagamento.aprovado via outbox.

### E4-1 [Estudar] Outbox, entrega at-least-once e idempotência

- **Status:** backlog
- **O que fazer:** Por que gravar o evento no banco na mesma transação, duplicidade de mensagens e chaves de deduplicação.
- **Aprendeu quando:** Sei explicar como o outbox evita perder ou duplicar um evento.
- **Material:** https://microservices.io/patterns/data/transactional-outbox.html

### E4-2 [Estudar] Pub/Sub: tópicos, assinaturas e dead-letter

- **Status:** backlog
- **O que fazer:** Modelo de tópico e assinatura, ack e retry, ordering e política de dead-letter.
- **Aprendeu quando:** Sei quando uma mensagem vai para a dead-letter e como reprocessá-la.
- **Material:** https://cloud.google.com/pubsub/docs

### E4-3 [Implementar] Tabela outbox e relay de eventos

- **Status:** backlog
- **O que fazer:** Gravar o evento na tabela outbox junto com o pagamento; um relay lê os pendentes e publica, marcando como enviado.
- **Pronto quando:** Matar o relay no meio e reiniciar não perde nem duplica eventos.

### E4-4 [Implementar] Porta EventBus com NATS e Pub/Sub

- **Status:** backlog
- **O que fazer:** Interface Publish/Subscribe no domínio; adaptador NATS para dev local e Pub/Sub para o GCP, escolhidos por configuração.
- **Pronto quando:** Trocar o adaptador por variável de ambiente sem alterar o domínio.

### E4-5 [Implementar] Consumidor idempotente com retry e DLQ

- **Status:** backlog
- **O que fazer:** Consumidor de pagamento.aprovado que grava o id processado, aplica backoff exponencial e envia ao dead-letter após N falhas.
- **Pronto quando:** Mensagem duplicada é ignorada e mensagem venenosa vai para a DLQ.

## E5 Testes e CI/CD (P0, Semanas 1-7)

Qualidade automatizada. Aplica no app: cobertura medida, zero nolint e deploy por pipeline.

### E5-1 [Estudar] Testes em Go: table-driven, testify e testcontainers

- **Status:** backlog
- **O que fazer:** Subtests, mocks vs fakes, testify/assert e subir Postgres real em container nos testes.
- **Aprendeu quando:** Sei decidir quando usar fake em memória e quando usar banco real.
- **Material:** https://golang.testcontainers.org/

### E5-2 [Implementar] Testes do domínio com 80% de cobertura

- **Status:** backlog
- **O que fazer:** Table-driven para as regras de Payment e Account; go test -cover no Makefile.
- **Pronto quando:** Cobertura do pacote domain igual ou acima de 80%.

### E5-3 [Implementar] Testes de integração com Postgres em container

- **Status:** backlog
- **O que fazer:** Suíte que sobe Postgres via testcontainers, roda as migrations e valida o adapter pgx.
- **Pronto quando:** go test ./... passa em máquina limpa só com Docker.

### E5-4 [Implementar] Lint com golangci-lint e zero nolint

- **Status:** backlog
- **O que fazer:** Configurar .golangci.yml (govet, errcheck, staticcheck, gosec) e corrigir tudo sem diretivas nolint.
- **Pronto quando:** O CI falha se aparecer qualquer //nolint no código.
- **Material:** https://golangci-lint.run/

### E5-5 [Implementar] Pipeline de CI: lint, teste, build e imagem

- **Status:** backlog
- **O que fazer:** Workflow que roda lint, testes com cobertura, build da imagem e push no Artifact Registry a cada merge.
- **Pronto quando:** Um PR com teste quebrado é bloqueado pelo pipeline.
- **Material:** https://docs.github.com/actions

## E6 Docker, Kubernetes e Terraform (P0, Semanas 6-7)

Plataforma como código. Aplica no app: GKE com infra reproduzível.

### E6-1 [Estudar] Kubernetes: Pods, Deployments, Services e probes

- **Status:** backlog
- **O que fazer:** Objetos básicos, readiness vs liveness, requests e limits e como o HPA escala.
- **Aprendeu quando:** Sei explicar por que um pod reinicia e como o tráfego só chega em pods prontos.
- **Material:** https://kubernetes.io/docs/tutorials/kubernetes-basics/

### E6-2 [Implementar] Dockerfile multi-stage e docker-compose

- **Status:** backlog
- **O que fazer:** Imagem final enxuta (distroless), usuário não root e compose com Postgres e NATS para desenvolvimento.
- **Pronto quando:** docker compose up sobe o ambiente completo com um comando.

### E6-3 [Implementar] Manifests e HPA no kind

- **Status:** backlog
- **O que fazer:** Deployment, Service, ConfigMap, Secret, probes e HPA por CPU para cada serviço, testados em cluster kind.
- **Pronto quando:** Aumentar a carga escala os pods e o rolling update não derruba requisições.

### E6-4 [Estudar] Terraform no GCP: providers, state e módulos

- **Status:** backlog
- **O que fazer:** Provider google, backend remoto do state no GCS, variáveis, outputs e módulos.
- **Aprendeu quando:** Sei explicar por que o state fica remoto e como evitar divergência.
- **Material:** https://developer.hashicorp.com/terraform/tutorials/gcp-get-started

### E6-5 [Implementar] Infra como código: VPC, GKE, Cloud SQL e Pub/Sub

- **Status:** backlog
- **O que fazer:** Módulos Terraform para rede privada, cluster Autopilot, instância Cloud SQL e tópicos com assinaturas.
- **Pronto quando:** terraform apply cria tudo e destroy remove tudo sem sobras.

### E6-6 [Implementar] Deploy automático no GKE pelo pipeline

- **Status:** backlog
- **O que fazer:** Estender o CI para aplicar os manifests no GKE com Workload Identity, sem chave JSON.
- **Pronto quando:** Merge na main publica a nova versão e o rollback sai com um comando.

## E7 Observabilidade e confiabilidade (P0, Semanas 8-9)

Operar em produção. Aplica no app: métricas, tracing, alertas e incidentes simulados.

### E7-1 [Estudar] Observabilidade: métricas, logs e traces

- **Status:** backlog
- **O que fazer:** Os três pilares, RED e USE, tipos de métrica do Prometheus e propagação de contexto no OpenTelemetry.
- **Aprendeu quando:** Sei escolher entre contador, gauge e histograma para uma métrica.
- **Material:** https://opentelemetry.io/docs/languages/go/

### E7-2 [Implementar] Logs estruturados com slog e correlation id

- **Status:** backlog
- **O que fazer:** JSON em stdout, nível por ambiente, id da requisição propagado por contexto e nenhum dado sensível.
- **Pronto quando:** Dado um id, encontro todos os logs de um pagamento nos três serviços.

### E7-3 [Implementar] Métricas Prometheus por rota

- **Status:** backlog
- **O que fazer:** Histograma de latência, contador por status e gauge de mensagens pendentes expostos em /metrics.
- **Pronto quando:** As métricas aparecem no Prometheus e batem com o tráfego gerado.

### E7-4 [Implementar] Tracing distribuído com OpenTelemetry

- **Status:** backlog
- **O que fazer:** Spans na API, no banco, na publicação e no consumidor, com contexto propagado pelos headers da mensagem.
- **Pronto quando:** Um pagamento aparece como um único trace atravessando a fila.

### E7-5 [Implementar] Dashboards e alertas no Grafana

- **Status:** backlog
- **O que fazer:** Painéis de latência p95, taxa de erro e backlog da fila; alertas para erro acima de 2% e fila crescendo.
- **Pronto quando:** Derrubar o consumidor dispara o alerta em poucos minutos.

### E7-6 [Implementar] Carga com k6 e três incidentes simulados

- **Status:** backlog
- **O que fazer:** Cenário k6 de pagamentos; derrubar um pod, travar o consumidor e simular queda do banco, registrando um postmortem curto de cada.
- **Pronto quando:** Três postmortems no repositório com causa, impacto e ação preventiva.
- **Material:** https://grafana.com/docs/k6/latest/

## E8 Export de volume e PII (P1, Semanas 10-11)

Grande volume com segurança. Aplica no app: export-service com snapshot, retomada e mascaramento.

### E8-1 [Estudar] Paginação por cursor e consistência de leitura

- **Status:** backlog
- **O que fazer:** Offset vs keyset, ordenação estável, snapshot e custo de leituras longas.
- **Aprendeu quando:** Sei explicar por que offset degrada em tabelas grandes.
- **Material:** https://use-the-index-luke.com/no-offset

### E8-2 [Implementar] export-service com paginação por cursor

- **Status:** backlog
- **O que fazer:** Endpoint que exporta lançamentos em páginas por chave (created_at, id) com tamanho máximo configurável.
- **Pronto quando:** Exportar 1 milhão de linhas com memória constante.

### E8-3 [Implementar] Leitura paralela com snapshot consistente

- **Status:** backlog
- **O que fazer:** Usar pg_export_snapshot para que workers paralelos enxerguem o mesmo instante dos dados.
- **Pronto quando:** Escritas durante o export não aparecem nem duplicam linhas no resultado.

### E8-4 [Implementar] Checkpoint e retomada após falha

- **Status:** backlog
- **O que fazer:** Gravar o último cursor processado por job e retomar a partir dele ao reiniciar.
- **Pronto quando:** Matar o processo no meio e retomar gera arquivo final sem lacunas.

### E8-5 [Implementar] Policies de acesso e mascaramento de PII

- **Status:** backlog
- **O que fazer:** Policy por campo, canal e tamanho de página; CPF e e-mail mascarados por padrão, completos só com permissão.
- **Pronto quando:** Teste confirma que perfil sem permissão nunca recebe PII em claro, nem nos logs.
- **Material:** https://www.gov.br/anpd/pt-br

## E9 GCP e certificação ACE (P1, Semanas 6-10)

Cloud na prática. Aplica no app: IAM mínimo, Workload Identity e rede privada.

### E9-1 [Estudar] Mapa do exame ACE e plano de estudo

- **Status:** backlog
- **O que fazer:** Ler o exam guide, listar os domínios e marcar o que já domina e o que falta (IAM, redes, operações).
- **Aprendeu quando:** Tenho uma lista priorizada de lacunas por domínio do exame.
- **Material:** https://cloud.google.com/learn/certification/cloud-engineer

### E9-2 [Implementar] Service account por serviço com menor privilégio

- **Status:** backlog
- **O que fazer:** Criar uma service account por serviço com roles mínimas e abandonar as contas padrão.
- **Pronto quando:** Nenhum serviço usa a conta padrão do Compute Engine.
- **Material:** https://cloud.google.com/iam/docs

### E9-3 [Implementar] Workload Identity no GKE

- **Status:** backlog
- **O que fazer:** Vincular service accounts do Kubernetes às do GCP, sem chaves JSON nos pods.
- **Pronto quando:** Pod acessa Pub/Sub e Cloud SQL sem nenhum arquivo de credencial.

### E9-4 [Implementar] Cloud SQL em rede privada

- **Status:** backlog
- **O que fazer:** IP privado via VPC, Cloud SQL Auth Proxy ou conector, sem IP público e com backups ativos.
- **Pronto quando:** A instância não responde pela internet e o app conecta normalmente.

### E9-5 [Estudar] Simulados do ACE e revisão de erros

- **Status:** backlog
- **O que fazer:** Fazer simulados cronometrados, anotar cada erro com o conceito por trás e refazer o lab correspondente.
- **Aprendeu quando:** Acerto consistente acima de 80% em dois simulados seguidos.

### E9-6 [Implementar] Agendar a prova do ACE

- **Status:** backlog
- **O que fazer:** Escolher a data, confirmar o formato (online ou presencial) e o valor atual no site oficial.
- **Pronto quando:** Prova agendada com data definida no calendário.

## E10 Extras (P2, Semana 12)

Diferenciais das vagas. Aplica no app: gRPC interno, MCP, painel e documentação.

### E10-1 [Estudar] gRPC e Protocol Buffers

- **Status:** backlog
- **O que fazer:** Arquivos .proto, geração de código, streaming e interceptors.
- **Aprendeu quando:** Sei quando gRPC faz mais sentido que REST.
- **Material:** https://grpc.io/docs/languages/go/

### E10-2 [Implementar] gRPC entre payments-api e ledger

- **Status:** backlog
- **O que fazer:** Definir o .proto do ledger, gerar o código e trocar a chamada direta por gRPC com interceptors de log e métricas.
- **Pronto quando:** payments-api conversa com o ledger por gRPC e as métricas aparecem no Grafana.

### E10-3 [Implementar] Servidor MCP somente leitura do ledger

- **Status:** backlog
- **O que fazer:** Expor consultas de saldo e extrato como ferramentas MCP, respeitando as mesmas policies de PII.
- **Pronto quando:** Um agente de IA consulta o extrato mascarado sem acesso de escrita.
- **Material:** https://modelcontextprotocol.io/

### E10-4 [Implementar] Painel em React + TypeScript

- **Status:** backlog
- **O que fazer:** Tela com pagamentos recentes, status e eventos, consumindo a payments-api com tipos gerados do OpenAPI.
- **Pronto quando:** O painel mostra um pagamento novo em até 5 segundos.
- **Material:** https://react.dev/learn

### E10-5 [Implementar] ADRs e runbooks

- **Status:** backlog
- **O que fazer:** Um ADR por decisão (outbox, dois adaptadores de fila, snapshot) e um runbook por alerta do Grafana.
- **Pronto quando:** Cada alerta tem um runbook com diagnóstico e ação.
- **Material:** https://adr.github.io/
