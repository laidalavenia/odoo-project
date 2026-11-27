# F9. Machine Data Gateway (Go + Fiber) + Simulator

| | |
|---|---|
| **Requirement FSD** | FR-GW-01…14, FR-SIM-01…05, FR-API-07/10 (end-to-end), FR-TG-04 dipindah ke Go |
| **Minggu** | 8 (23–29 Nov 2026) |
| **Hasil** | Gateway menerima data, menyimpan di `sensor_db`, memetakan WO, mengirim ringkasan & QC gagal ke Odoo, Telegram anomali, bot `/start` & `/produksi`, simulator semua skenario |

> 🧭 **Pembagian kode:** `go.mod`, Dockerfile, migrasi SQL, config, tipe domain, wiring `main.go`, dan **simulator** = **lengkap**. Store, ingest, detector, outbox, klien Odoo, Telegram, dan handler = **konsep + kerangka + petunjuk**.

---

## 1. Konsep

### 1.1 Pipeline

```mermaid
flowchart LR
    SIM[Simulator] -->|POST /readings| H[Handler<br/>validasi + token]
    H -->|non-blocking send| CH((ingestCh))
    H -->|503 bila penuh| SIM
    CH --> FAN[Dispatcher goroutine]
    FAN --> BW[Batch writer<br/>CopyFrom tiap 500 / 1 dtk]
    FAN --> DET[Detector<br/>state per mesin]
    FAN --> HUB[SSE hub]
    BW --> DB[(sensor_db)]
    DET -->|Anomaly| TX[Tx: anomalies + outbox]
    TX --> DB
    DET -->|alert| TG[Telegram sender]
    OB[Outbox worker] -->|SKIP LOCKED| DB
    OB -->|REST + retry| ODOO[Odoo]
    ODOO -->|POST /events/workorder| EV[Event handler] --> CACHE[Interval cache] --> H
    EV -->|finished| SUM[Summary SQL] --> DB
    BOT[Bot poller] <--> TAPI[Telegram API]
    BOT --> ODOO
```

### 1.2 Kenapa Dispatcher, bukan langsung tulis DB di handler?
- Handler tidak menunggu DB → latensi rendah (202 Accepted).
- Batch insert jauh lebih efisien daripada insert per baris.
- **Backpressure**: channel penuh → 503, simulator retry. Tidak ada goroutine handler yang menumpuk tanpa batas.
- Trade-off yang harus kamu sebut: data di buffer memori bisa hilang bila proses crash (bukan shutdown normal). Mitigasi: buffer kecil + flush cepat + graceful shutdown; untuk jaminan penuh pakai MQTT dengan QoS 1/persistent broker atau tulis WAL lokal.

### 1.3 Idempotensi & outbox (ulang dari F5, sisi Go)
- Setiap anomali punya `ref` UUID → dikirim sebagai `external_ref` → Odoo menolak duplikat.
- Outbox ditulis dalam **transaksi yang sama** dengan `anomalies`, jadi tidak mungkin ada anomali tanpa event keluar (atau sebaliknya).
- Worker memakai `SELECT ... FOR UPDATE SKIP LOCKED` sehingga aman bila suatu saat ada dua replika gateway.

---

## 2. Langkah — Kerangka Project (lengkap)

### Langkah 1 — Init

```bash
cd ~/Documents/odoo-project/gateway
go mod init github.com/<your-github>/odoo-project/gateway
go get github.com/gofiber/fiber/v3        # ⚠️ if v3 is not stable yet: github.com/gofiber/fiber/v2
go get github.com/jackc/pgx/v5
go get github.com/google/uuid
mkdir -p cmd/gateway cmd/simulator migrations \
  internal/{config,domain,store,ingest,detector,stream,odoo,outbox,telegram,httpapi,migrate}
```

### Langkah 2 — Migrasi SQL

`gateway/migrations/0001_init.sql`
```sql
-- Machines known by the gateway
CREATE TABLE IF NOT EXISTS machines (
    code                TEXT PRIMARY KEY,
    name                TEXT NOT NULL,
    kind                TEXT NOT NULL CHECK (kind IN ('fryer', 'scale')),
    odoo_workcenter_id  INTEGER,
    temp_limit          NUMERIC(5,2) NOT NULL DEFAULT 180.0,
    temp_limit_seconds  INTEGER      NOT NULL DEFAULT 60,
    weight_min          NUMERIC(6,1) NOT NULL DEFAULT 195.0,
    weight_max          NUMERIC(6,1) NOT NULL DEFAULT 210.0,
    active              BOOLEAN      NOT NULL DEFAULT TRUE
);

-- Active/closed workorder intervals per machine (from Odoo events)
CREATE TABLE IF NOT EXISTS wo_intervals (
    id                  BIGSERIAL PRIMARY KEY,
    event_ref           TEXT UNIQUE,
    odoo_workorder_id   INTEGER NOT NULL,
    odoo_production_id  INTEGER,
    machine_code        TEXT NOT NULL REFERENCES machines(code),
    started_at          TIMESTAMPTZ NOT NULL,
    ended_at            TIMESTAMPTZ,
    end_reason          TEXT CHECK (end_reason IN ('paused', 'finished'))
);
CREATE INDEX IF NOT EXISTS wo_intervals_open_idx ON wo_intervals (machine_code) WHERE ended_at IS NULL;
CREATE INDEX IF NOT EXISTS wo_intervals_wo_idx ON wo_intervals (odoo_workorder_id);

CREATE TABLE IF NOT EXISTS temperature_readings (
    id                  BIGSERIAL PRIMARY KEY,
    machine_code        TEXT NOT NULL REFERENCES machines(code),
    ts                  TIMESTAMPTZ NOT NULL,
    celsius             NUMERIC(5,2) NOT NULL,
    odoo_workorder_id   INTEGER
);
CREATE INDEX IF NOT EXISTS temperature_machine_ts_idx ON temperature_readings (machine_code, ts DESC);
CREATE INDEX IF NOT EXISTS temperature_wo_idx ON temperature_readings (odoo_workorder_id) WHERE odoo_workorder_id IS NOT NULL;

CREATE TABLE IF NOT EXISTS weight_readings (
    id                  BIGSERIAL PRIMARY KEY,
    machine_code        TEXT NOT NULL REFERENCES machines(code),
    ts                  TIMESTAMPTZ NOT NULL,
    grams               NUMERIC(6,1) NOT NULL,
    in_tolerance        BOOLEAN NOT NULL,
    odoo_workorder_id   INTEGER
);
CREATE INDEX IF NOT EXISTS weight_machine_ts_idx ON weight_readings (machine_code, ts DESC);
CREATE INDEX IF NOT EXISTS weight_wo_idx ON weight_readings (odoo_workorder_id) WHERE odoo_workorder_id IS NOT NULL;

CREATE TABLE IF NOT EXISTS anomalies (
    id                  BIGSERIAL PRIMARY KEY,
    ref                 UUID NOT NULL UNIQUE,
    kind                TEXT NOT NULL CHECK (kind IN ('temp_over_limit', 'weight_out_of_tolerance')),
    machine_code        TEXT NOT NULL REFERENCES machines(code),
    odoo_workorder_id   INTEGER,
    started_at          TIMESTAMPTZ NOT NULL,
    ended_at            TIMESTAMPTZ,
    peak_value          NUMERIC(6,2) NOT NULL,
    status              TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'closed')),
    odoo_qc_id          INTEGER,
    telegram_sent_at    TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS anomalies_open_idx ON anomalies (machine_code, kind) WHERE status = 'open';

CREATE TABLE IF NOT EXISTS outbox (
    id                  BIGSERIAL PRIMARY KEY,
    kind                TEXT NOT NULL CHECK (kind IN ('qc_auto_fail', 'sensor_summary')),
    payload             JSONB NOT NULL,
    attempts            INTEGER NOT NULL DEFAULT 0,
    next_attempt_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_error          TEXT,
    sent_at             TIMESTAMPTZ,
    dead                BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS outbox_pending_idx ON outbox (next_attempt_at) WHERE sent_at IS NULL AND NOT dead;

CREATE TABLE IF NOT EXISTS bot_state (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- Seed machines (match machine_code on Odoo work centers)
INSERT INTO machines (code, name, kind) VALUES
    ('FRY-01', 'Penggoreng 1', 'fryer'),
    ('FRY-02', 'Penggoreng 2', 'fryer'),
    ('SCL-01', 'Timbangan Kemas', 'scale')
ON CONFLICT (code) DO NOTHING;
```

### Langkah 3 — Migrator sederhana (embed)

`gateway/migrations/embed.go`
```go
package migrations

import "embed"

// FS contains all *.sql migration files, applied in lexical order.
//
//go:embed *.sql
var FS embed.FS
```

`gateway/internal/migrate/migrate.go`
```go
package migrate

import (
	"context"
	"fmt"
	"io/fs"
	"sort"

	"github.com/jackc/pgx/v5/pgxpool"
)

// Up applies every *.sql file that has not been applied yet.
func Up(ctx context.Context, pool *pgxpool.Pool, files fs.FS) error {
	if _, err := pool.Exec(ctx, `CREATE TABLE IF NOT EXISTS schema_migrations (
		name TEXT PRIMARY KEY, applied_at TIMESTAMPTZ NOT NULL DEFAULT now())`); err != nil {
		return fmt.Errorf("create schema_migrations: %w", err)
	}

	names, err := fs.Glob(files, "*.sql")
	if err != nil {
		return err
	}
	sort.Strings(names)

	for _, name := range names {
		var exists bool
		if err := pool.QueryRow(ctx,
			`SELECT EXISTS (SELECT 1 FROM schema_migrations WHERE name = $1)`, name).Scan(&exists); err != nil {
			return err
		}
		if exists {
			continue
		}
		sql, err := fs.ReadFile(files, name)
		if err != nil {
			return err
		}
		tx, err := pool.Begin(ctx)
		if err != nil {
			return err
		}
		if _, err := tx.Exec(ctx, string(sql)); err != nil {
			_ = tx.Rollback(ctx)
			return fmt.Errorf("apply %s: %w", name, err)
		}
		if _, err := tx.Exec(ctx, `INSERT INTO schema_migrations (name) VALUES ($1)`, name); err != nil {
			_ = tx.Rollback(ctx)
			return err
		}
		if err := tx.Commit(ctx); err != nil {
			return err
		}
	}
	return nil
}
```

### Langkah 4 — Config

`gateway/internal/config/config.go`
```go
package config

import (
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"
)

type Config struct {
	Addr                string
	SensorDBURL         string
	OdooBaseURL         string
	OdooAPIToken        string
	InboundToken        string // Odoo -> gateway
	MonitorToken        string // Next.js -> gateway
	MachineToken        string // simulator/machines -> gateway
	TelegramBotToken    string
	TelegramPoller      bool
	TelegramAlertChatIDs []string
	IngestBuffer        int
	BatchSize           int
	BatchInterval       time.Duration
	RetentionDays       int
}

func Load() (Config, error) {
	c := Config{
		Addr:                 env("GATEWAY_ADDR", ":8080"),
		SensorDBURL:          os.Getenv("SENSOR_DB_URL"),
		OdooBaseURL:          strings.TrimRight(os.Getenv("ODOO_BASE_URL"), "/"),
		OdooAPIToken:         os.Getenv("ODOO_API_TOKEN"),
		InboundToken:         os.Getenv("GATEWAY_INBOUND_TOKEN"),
		MonitorToken:         os.Getenv("MONITOR_TOKEN"),
		MachineToken:         os.Getenv("MACHINE_TOKEN"),
		TelegramBotToken:     os.Getenv("TELEGRAM_BOT_TOKEN"),
		TelegramPoller:       env("TELEGRAM_POLLER", "off") == "gateway",
		TelegramAlertChatIDs: splitCSV(os.Getenv("TELEGRAM_ALERT_CHAT_IDS")),
		IngestBuffer:         envInt("INGEST_BUFFER", 10000),
		BatchSize:            envInt("BATCH_SIZE", 500),
		BatchInterval:        envDuration("BATCH_INTERVAL", time.Second),
		RetentionDays:        envInt("RETENTION_DAYS", 90),
	}

	var missing []string
	for name, v := range map[string]string{
		"SENSOR_DB_URL":         c.SensorDBURL,
		"ODOO_BASE_URL":         c.OdooBaseURL,
		"ODOO_API_TOKEN":        c.OdooAPIToken,
		"GATEWAY_INBOUND_TOKEN": c.InboundToken,
		"MONITOR_TOKEN":         c.MonitorToken,
		"MACHINE_TOKEN":         c.MachineToken,
	} {
		if v == "" {
			missing = append(missing, name)
		}
	}
	if len(missing) > 0 {
		return c, fmt.Errorf("missing required env: %s", strings.Join(missing, ", "))
	}
	return c, nil
}

func env(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}

func envInt(key string, def int) int {
	if v, err := strconv.Atoi(os.Getenv(key)); err == nil {
		return v
	}
	return def
}

func envDuration(key string, def time.Duration) time.Duration {
	if v, err := time.ParseDuration(os.Getenv(key)); err == nil {
		return v
	}
	return def
}

func splitCSV(s string) []string {
	var out []string
	for _, p := range strings.Split(s, ",") {
		if p = strings.TrimSpace(p); p != "" {
			out = append(out, p)
		}
	}
	return out
}
```

### Langkah 5 — Tipe domain

`gateway/internal/domain/types.go`
```go
package domain

import "time"

type ReadingKind string

const (
	KindTemperature ReadingKind = "temperature"
	KindWeight      ReadingKind = "weight"
)

// Reading is the unit flowing through the ingest pipeline.
type Reading struct {
	Kind        ReadingKind `json:"kind"`
	MachineCode string      `json:"machine_code"`
	TS          time.Time   `json:"ts"`
	Value       float64     `json:"value"` // celsius or grams
	WorkorderID *int        `json:"workorder_id,omitempty"` // nil when no active WO
}

type Machine struct {
	Code             string
	Name             string
	Kind             string // fryer | scale
	TempLimit        float64
	TempLimitSeconds int
	WeightMin        float64
	WeightMax        float64
}

type AnomalyKind string

const (
	AnomalyTempOverLimit AnomalyKind = "temp_over_limit"
	AnomalyWeight        AnomalyKind = "weight_out_of_tolerance"
)

// Anomaly is emitted by the detector (pure logic) and persisted by the service layer.
type Anomaly struct {
	Ref         string
	Kind        AnomalyKind
	MachineCode string
	WorkorderID *int
	StartedAt   time.Time
	PeakValue   float64
	Message     string
}

// AnomalyClosed is emitted when the condition returns to normal.
type AnomalyClosed struct {
	MachineCode string
	Kind        AnomalyKind
	EndedAt     time.Time
}

// WorkorderEvent is sent by Odoo.
type WorkorderEvent struct {
	Ref          string    `json:"ref"`
	Event        string    `json:"event"` // started | paused | finished
	WorkorderID  int       `json:"workorder_id"`
	ProductionID int       `json:"production_id"`
	MachineCode  string    `json:"machine_code"`
	At           time.Time `json:"at"`
}

type SensorSummary struct {
	ExternalRef     string    `json:"external_ref"`
	MachineCode     string    `json:"machine_code"`
	PeriodStart     time.Time `json:"period_start"`
	PeriodEnd       time.Time `json:"period_end"`
	Temperature     *TempAgg  `json:"temperature"`
	Weight          *WeightAgg `json:"weight"`
}

type TempAgg struct {
	Avg              float64 `json:"avg"`
	Min              float64 `json:"min"`
	Max              float64 `json:"max"`
	Samples          int     `json:"samples"`
	SecondsOverLimit int     `json:"seconds_over_limit"`
}

type WeightAgg struct {
	Count          int     `json:"count"`
	Total          float64 `json:"total"`
	Avg            float64 `json:"avg"`
	OutOfTolerance int     `json:"out_of_tolerance"`
}
```

### Langkah 6 — `main.go` (wiring; bagian service berupa TODO)

`gateway/cmd/gateway/main.go`
```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<your-github>/odoo-project/gateway/internal/config"
	"github.com/<your-github>/odoo-project/gateway/internal/migrate"
	"github.com/<your-github>/odoo-project/gateway/migrations"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
	slog.SetDefault(logger)

	if err := run(); err != nil {
		slog.Error("gateway stopped with error", "err", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	// Root context: cancelled on SIGINT/SIGTERM (docker stop)
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	pool, err := connectWithRetry(ctx, cfg.SensorDBURL)
	if err != nil {
		return err
	}
	defer pool.Close()

	if err := migrate.Up(ctx, pool, migrations.FS); err != nil {
		return err
	}
	slog.Info("migrations applied")

	// ---------------------------------------------------------------
	// TODO: build dependencies (manual DI):
	//   st      := store.New(pool)
	//   odooCli := odoo.NewClient(cfg.OdooBaseURL, cfg.OdooAPIToken)
	//   tg      := telegram.NewClient(cfg.TelegramBotToken)   // no-op when token empty
	//   hub     := stream.NewHub()
	//   cache   := ingest.NewIntervalCache(); load open intervals from st
	//   det     := detector.New(machines loaded from st)
	//   ingestCh := make(chan domain.Reading, cfg.IngestBuffer)
	//   svc     := ingest.NewService(...)   // dispatcher + batch writer + detector + hub
	//   ob      := outbox.NewWorker(st, odooCli)
	//   app     := httpapi.New(cfg, ingestCh, cache, st, hub, ...)
	// ---------------------------------------------------------------

	var wg sync.WaitGroup
	start := func(name string, fn func(context.Context) error) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := fn(ctx); err != nil && !errors.Is(err, context.Canceled) {
				slog.Error("component stopped", "component", name, "err", err)
				stop() // one component failing stops the whole service
			}
		}()
	}
	_ = start

	// TODO: start("ingest", svc.Run)
	// TODO: start("outbox", ob.Run)
	// TODO: start("retention", st.RunRetention(cfg.RetentionDays))
	// TODO: if cfg.TelegramPoller { start("bot", bot.Run) }

	// TODO: HTTP server
	// go func() {
	//     if err := app.Listen(cfg.Addr); err != nil { slog.Error("http", "err", err); stop() }
	// }()

	<-ctx.Done()
	slog.Info("shutting down")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	_ = shutdownCtx
	// TODO: app.ShutdownWithContext(shutdownCtx)  -> stop accepting requests first
	// TODO: close(ingestCh) so the batch writer flushes the rest (or let it react to ctx)

	wg.Wait()
	slog.Info("bye")
	return nil
}

func connectWithRetry(ctx context.Context, url string) (*pgxpool.Pool, error) {
	var lastErr error
	for attempt := 1; attempt <= 30; attempt++ {
		pool, err := pgxpool.New(ctx, url)
		if err == nil {
			if err = pool.Ping(ctx); err == nil {
				return pool, nil
			}
			pool.Close()
		}
		lastErr = err
		slog.Warn("waiting for sensor_db", "attempt", attempt, "err", err)
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(2 * time.Second):
		}
	}
	return nil, lastErr
}
```

### Langkah 7 — Dockerfile

`gateway/Dockerfile`
```dockerfile
FROM golang:1.27-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/gateway ./cmd/gateway \
 && CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/simulator ./cmd/simulator

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/gateway /usr/local/bin/gateway
COPY --from=build /out/simulator /usr/local/bin/simulator
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/usr/local/bin/gateway"]
```

> Tag `golang:` harus ≥ versi `go` di `go.mod` (laptopmu memakai Go 1.27.1, jadi `go mod init` menulis `go 1.27`). Image distroless tidak punya shell; untuk debugging sementara pakai `alpine`.

---

## 3. Langkah — Komponen Inti (kerangka + petunjuk)

### Langkah 8 — Store (`internal/store`)

```go
package store

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/<your-github>/odoo-project/gateway/internal/domain"
)

type Store struct{ pool *pgxpool.Pool }

func New(pool *pgxpool.Pool) *Store { return &Store{pool: pool} }

// InsertReadings writes a batch using COPY (fast path).
func (s *Store) InsertReadings(ctx context.Context, batch []domain.Reading) error {
	// TODO: split batch by Kind
	// temperature: s.pool.CopyFrom(ctx, pgx.Identifier{"temperature_readings"},
	//     []string{"machine_code", "ts", "celsius", "odoo_workorder_id"},
	//     pgx.CopyFromSlice(len(temps), func(i int) ([]any, error) { ... }))
	// weight: same + in_tolerance (computed by caller or here)
	_ = pgx.Identifier{}
	return nil
}

func (s *Store) Machines(ctx context.Context) ([]domain.Machine, error) { /* TODO */ return nil, nil }

// OpenIntervals returns machine_code -> workorder id for intervals with ended_at IS NULL.
func (s *Store) OpenIntervals(ctx context.Context) (map[string]int, error) { /* TODO */ return nil, nil }

// ApplyWorkorderEvent opens/closes intervals idempotently (event_ref UNIQUE).
func (s *Store) ApplyWorkorderEvent(ctx context.Context, ev domain.WorkorderEvent) (applied bool, err error) {
	// TODO:
	// started  -> close any open interval of that machine (end_reason 'paused'), insert new interval
	// paused   -> UPDATE ... SET ended_at = ev.At, end_reason = 'paused' WHERE machine & wo & ended_at IS NULL
	// finished -> same with 'finished'
	// use INSERT ... ON CONFLICT (event_ref) DO NOTHING on a marker row, or check existence first
	return false, nil
}

// BuildSummary aggregates readings of a workorder (FR-GW-05).
func (s *Store) BuildSummary(ctx context.Context, workorderID int) (domain.SensorSummary, error) {
	// HINT SQL (temperature):
	// SELECT avg(celsius), min(celsius), max(celsius), count(*),
	//        min(ts), max(ts)
	// FROM temperature_readings WHERE odoo_workorder_id = $1
	// seconds_over_limit: count(*) FILTER (WHERE celsius > m.temp_limit) * <interval seconds>
	//   (approximation; exact: sum of ts deltas using window function LEAD(ts))
	// weight: count, sum, avg, count(*) FILTER (WHERE NOT in_tolerance)
	return domain.SensorSummary{}, nil
}

// CreateAnomalyWithOutbox inserts anomaly + outbox row in ONE transaction.
func (s *Store) CreateAnomalyWithOutbox(ctx context.Context, a domain.Anomaly, payload []byte) error {
	// TODO: tx := s.pool.Begin; defer tx.Rollback (no-op after commit); insert both; commit
	return nil
}

func (s *Store) CloseAnomaly(ctx context.Context, c domain.AnomalyClosed) error { /* TODO */ return nil }

// ClaimOutbox locks due rows so concurrent workers never send the same row.
func (s *Store) ClaimOutbox(ctx context.Context, limit int) (pgx.Tx, []OutboxRow, error) {
	// HINT:
	// SELECT id, kind, payload, attempts FROM outbox
	// WHERE sent_at IS NULL AND NOT dead AND next_attempt_at <= now()
	// ORDER BY id LIMIT $1 FOR UPDATE SKIP LOCKED
	return nil, nil, nil
}

type OutboxRow struct {
	ID       int64
	Kind     string
	Payload  []byte
	Attempts int
}

// RunRetention deletes old readings once a day (FR-GW-14).
func (s *Store) RunRetention(days int) func(context.Context) error {
	return func(ctx context.Context) error {
		t := time.NewTicker(24 * time.Hour)
		defer t.Stop()
		for {
			// TODO: DELETE FROM temperature_readings WHERE ts < now() - make_interval(days => $1)
			// TODO: same for weight_readings; log rows affected
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-t.C:
			}
		}
	}
}
```

### Langkah 9 — Ingest service (`internal/ingest`)

```go
package ingest

// IntervalCache: machine_code -> active workorder id (sync.RWMutex, see F8 §6)

// Service owns the pipeline goroutine.
//
// Run(ctx):
//   batch := make([]domain.Reading, 0, cfg.BatchSize)
//   ticker := time.NewTicker(cfg.BatchInterval)
//   for {
//     select {
//     case r := <-in:
//         batch = append(batch, r)
//         s.hub.Publish(r)                    // non-blocking
//         events := s.detector.Observe(r)     // pure, returns []any (Anomaly / AnomalyClosed)
//         s.handleDetectorEvents(ctx, events) // persist + telegram
//         if len(batch) >= cfg.BatchSize { s.flush(ctx, &batch) }
//     case <-ticker.C:
//         s.flush(ctx, &batch)
//     case <-ctx.Done():
//         // drain remaining items in channel without blocking, then final flush
//         // use a fresh context with timeout for the final DB write (ctx is already cancelled!)
//         return ctx.Err()
//     }
//   }
//
// flush: on DB error keep the batch (retry next tick) but cap its size to avoid unbounded memory;
//        log dropped count if capped.
```

Petunjuk: detector dipanggil **sinkron** di goroutine yang sama, jadi state detector tidak perlu mutex. Itu keuntungan desain "satu pemilik state".

### Langkah 10 — Detector (`internal/detector`) — logika murni

```go
package detector

import (
	"time"

	"github.com/google/uuid"

	"github.com/<your-github>/odoo-project/gateway/internal/domain"
)

const closeAfter = 10 * time.Second // hysteresis (FR-GW-06)

type fryerState struct {
	overSince  *time.Time // first reading above limit in the current streak
	normalSince *time.Time // first normal reading after an open anomaly
	open       bool
	peak       float64
}

type scaleState struct {
	outOfTol []time.Time // timestamps of recent out-of-tolerance packs (sliding window 5 min)
	open     bool
	lastOut  time.Time
}

type Detector struct {
	machines map[string]domain.Machine
	fryers   map[string]*fryerState
	scales   map[string]*scaleState
	newRef   func() string // injectable for tests
}

func New(machines []domain.Machine) *Detector {
	d := &Detector{
		machines: map[string]domain.Machine{},
		fryers:   map[string]*fryerState{},
		scales:   map[string]*scaleState{},
		newRef:   func() string { return uuid.NewString() },
	}
	for _, m := range machines {
		d.machines[m.Code] = m
	}
	return d
}

// Observe returns zero or more events (domain.Anomaly or domain.AnomalyClosed).
func (d *Detector) Observe(r domain.Reading) []any {
	m, ok := d.machines[r.MachineCode]
	if !ok {
		return nil
	}
	switch r.Kind {
	case domain.KindTemperature:
		return d.observeTemp(m, r)
	case domain.KindWeight:
		return d.observeWeight(m, r)
	}
	return nil
}

func (d *Detector) observeTemp(m domain.Machine, r domain.Reading) []any {
	// TODO (FR-GW-06):
	// st := d.fryers[m.Code] (create if nil)
	// if r.Value > m.TempLimit:
	//     st.normalSince = nil
	//     if st.overSince == nil { t := r.TS; st.overSince = &t }
	//     st.peak = max(st.peak, r.Value)
	//     if !st.open && r.TS.Sub(*st.overSince) >= time.Duration(m.TempLimitSeconds)*time.Second:
	//         st.open = true -> return Anomaly{Ref: d.newRef(), ...}
	// else:
	//     st.overSince = nil
	//     if st.open: start/continue normalSince; if >= closeAfter -> st.open=false, peak=0 -> AnomalyClosed
	// Use reading timestamps (r.TS), NOT time.Now(): deterministic tests + correct with delayed batches.
	return nil
}

func (d *Detector) observeWeight(m domain.Machine, r domain.Reading) []any {
	// TODO (FR-GW-07): out of [WeightMin, WeightMax] -> append ts; drop entries older than 5 min;
	// if len >= 3 and !open -> open + Anomaly; close when no out-of-tolerance for 5 min
	return nil
}
```

Test (wajib), `detector_test.go`, table-driven:
| Kasus | Input | Harapan |
|---|---|---|
| normal | 170 °C tiap 2 dtk selama 5 menit | 0 event |
| overheat 75 dtk | 185 °C selama 75 dtk | tepat 1 Anomaly, di detik ke-60 |
| spike 40 dtk | 185 °C selama 40 dtk | 0 event |
| close | overheat lalu 172 °C 12 dtk | Anomaly lalu AnomalyClosed |
| re-open | overheat → close → overheat lagi | 2 Anomaly |
| weight 2 out | 2 kemasan 190 g | 0 |
| weight 3 out | 3 kemasan 190 g dalam 5 menit | 1 Anomaly |

### Langkah 11 — Klien Odoo (`internal/odoo`)

```go
package odoo

// Client wraps the Odoo REST API (D6 §6.2).
//
// type Client struct { base, token string; http *http.Client }
// func NewClient(base, token string) *Client  // http.Client{Timeout: 5 * time.Second}
//
// func (c *Client) AutoFailQC(ctx, payload AutoFailRequest) (qcID int, err error)
// func (c *Client) PostSensorSummary(ctx, workorderID int, s domain.SensorSummary) error
// func (c *Client) LinkTelegram(ctx, code, chatID, username string) (employeeName string, err error)
// func (c *Client) ProductionToday(ctx) (ProductionToday, error)
// func (c *Client) Recipients(ctx, event string) ([]Recipient, error)
//
// Error design:
//   type APIError struct { Status int; Code, Message string }
//   func (e *APIError) Error() string
//   func IsRetryable(err error) bool   // network errors, 5xx, 429 -> true; 4xx -> false
// Outbox worker uses IsRetryable to decide backoff vs dead-letter.
```

### Langkah 12 — Outbox worker (`internal/outbox`)

```go
package outbox

// Worker.Run(ctx):
//   ticker 2s
//   loop: tx, rows := store.ClaimOutbox(ctx, 20)
//         for each row:
//             switch row.Kind {
//             case "qc_auto_fail":   err = odoo.AutoFailQC(...)     -> on success also UPDATE anomalies SET odoo_qc_id
//             case "sensor_summary": err = odoo.PostSensorSummary(...)
//             }
//             success          -> UPDATE outbox SET sent_at = now()
//             retryable error  -> attempts+1, next_attempt_at = now() + NextDelay(attempts), last_error
//             permanent error  -> dead = true, last_error  (log ERROR)
//             attempts >= 15   -> dead = true
//         tx.Commit
//
// NextDelay(attempt) = min(10m, 2s * 2^attempt) +/- 20% jitter   (F8 exercise L4)
//
// NOTE: HTTP calls while holding row locks: OK for low volume (20 rows, 5s timeout).
// Alternative for scale: claim with UPDATE ... SET locked_until, commit, then send.
```

### Langkah 13 — Telegram (`internal/telegram`)

```go
package telegram

// Client.Send(ctx, chatID, text string) error
//   POST https://api.telegram.org/bot<token>/sendMessage  {chat_id, text, parse_mode:"HTML"}
//   429 -> read parameters.retry_after, sleep (respect ctx), retry once
//   rate limit: golang.org/x/time/rate limiter (e.g. 20 msg/s global) or a simple ticker
//
// Bot (long polling) FR-GW-12:
//   Run(ctx):
//     offset := load from bot_state
//     for {
//       updates, err := getUpdates(ctx, offset, timeout=30)   // HTTP client timeout must be > 30s (e.g. 40s)
//       if err -> log, sleep 3s (respect ctx), continue
//       for u := range updates {
//           offset = u.UpdateID + 1
//           handle(ctx, u)      // /start <code>, /produksi, /help
//       }
//       save offset to bot_state
//     }
//
//   /start <code>  -> odoo.LinkTelegram(code, chatID, username) -> reply name / error message
//   /produksi      -> allowed only if chatID in recipients (cached) -> odoo.ProductionToday -> format
//   other          -> help text
//
// Before running in gateway mode: set Odoo param mrp_keripik.telegram_poller = gateway
// (otherwise Telegram returns 409 Conflict: terminated by other getUpdates request)
```

Format `/produksi`:
```
📊 Produksi hari ini (12/10/2026)
✅ MO selesai: 3
⚙️ MO berjalan: 2
⏰ MO terlambat: 1
📦 Keripik 200 g selesai: 90 bungkus
📈 Yield rata-rata: 96,4 %
❌ QC gagal: 1
```

### Langkah 14 — HTTP API (`internal/httpapi`)

```go
package httpapi

// New(cfg, deps) *fiber.App
//   app := fiber.New(fiber.Config{ BodyLimit: 1 << 20, ReadTimeout: 10 * time.Second })
//   ⚠️ SSE (F10) needs no WriteTimeout or a long one.
//
//   app.Get("/api/v1/healthz", ...)
//   app.Get("/api/v1/readyz", ...)             // pool.Ping
//
//   machine := app.Group("/api/v1/readings", requireHeaderToken("X-Machine-Token", cfg.MachineToken))
//   machine.Post("/temperature", h.postTemperature)
//   machine.Post("/weight", h.postWeight)
//
//   inbound := app.Group("/api/v1/events", requireBearer(cfg.InboundToken))
//   inbound.Post("/workorder", h.postWorkorderEvent)
//
//   mon := app.Group("/api/v1", requireBearer(cfg.MonitorToken))
//   mon.Get("/machines", ...)
//   mon.Get("/machines/:code/temperature", ...)
//   mon.Get("/anomalies", ...)
//   mon.Get("/stream", h.stream)              // F10
//
// requireBearer: compare with crypto/subtle.ConstantTimeCompare
//
// postTemperature:
//   1. bind body {machine_code, readings: [{ts, celsius}]}; 1..500 readings
//   2. validate machine exists (cache), celsius range, ts not > 1 min in future (E2)
//   3. wo := cache.Get(machine)
//   4. for each reading: select { case ch <- r: default: return 503 BACKPRESSURE }
//   5. return 202 {"accepted": n}
//
// postWorkorderEvent:
//   1. bind + validate
//   2. applied := store.ApplyWorkorderEvent(ev); update cache (started -> Set, paused/finished -> Delete)
//   3. if finished && applied: summary := store.BuildSummary(wo); insert outbox sensor_summary
//   4. 200 {"ok": true, "duplicate": !applied}
```

---

## 4. Simulator (lengkap)

`gateway/cmd/simulator/main.go`
```go
// Command simulator generates fryer temperatures and pack weights for the gateway.
//
// Scenarios:
//   normal       stable temperatures, normal pack weights
//   overheat     after -overheat-after, fryers target 187 C for 90 s (must raise ONE anomaly)
//   spike        after -overheat-after, fryers target 186 C for 40 s (must NOT raise an anomaly)
//   underweight  after -overheat-after, 5 consecutive packs around 190 g (ONE weight anomaly)
//   burst        load test: 20 fryers LOAD-01..LOAD-20 every 100 ms
//
// -speed only accelerates the heating phase and the wait before the scenario event.
// The event itself and all timestamps are real time, so the gateway's 60-second rule is exercised.
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"math"
	"math/rand/v2"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

type tempReading struct {
	TS      time.Time `json:"ts"`
	Celsius float64   `json:"celsius"`
}

type weightReading struct {
	TS    time.Time `json:"ts"`
	Grams float64   `json:"grams"`
}

type batch[T any] struct {
	MachineCode string `json:"machine_code"`
	Readings    []T    `json:"readings"`
}

type config struct {
	url           string
	token         string
	fryers        []string
	scale         string
	interval      time.Duration
	packInterval  time.Duration
	scenario      string
	overheatAfter time.Duration
	speed         float64
	duration      time.Duration
}

type stats struct {
	sent   atomic.Int64
	failed atomic.Int64
}

const (
	ambient = 25.0
	tau     = 20.0 // seconds; first-order lag toward the target temperature
)

func main() {
	slog.SetDefault(slog.New(slog.NewTextHandler(os.Stdout, nil)))
	cfg := parseFlags()

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	if cfg.duration > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, cfg.duration)
		defer cancel()
	}

	client := &http.Client{Timeout: 5 * time.Second}
	st := &stats{}
	start := time.Now()
	eventStart := start.Add(time.Duration(float64(cfg.overheatAfter) / cfg.speed))

	slog.Info("simulator started",
		"url", cfg.url, "scenario", cfg.scenario, "fryers", strings.Join(cfg.fryers, ","),
		"scale", cfg.scale, "event_at", eventStart.Format(time.TimeOnly))

	var wg sync.WaitGroup
	for _, code := range cfg.fryers {
		wg.Add(1)
		go func(code string) {
			defer wg.Done()
			runFryer(ctx, client, cfg, st, code, start, eventStart)
		}(code)
	}
	if cfg.scale != "" {
		wg.Add(1)
		go func() {
			defer wg.Done()
			runScale(ctx, client, cfg, st, eventStart)
		}()
	}
	wg.Add(1)
	go func() {
		defer wg.Done()
		reportStats(ctx, st)
	}()

	wg.Wait()
	slog.Info("simulator stopped", "sent", st.sent.Load(), "failed", st.failed.Load())
}

func parseFlags() config {
	var cfg config
	var fryers string
	flag.StringVar(&cfg.url, "url", envOr("GATEWAY_URL", "http://localhost:8080"), "gateway base URL")
	flag.StringVar(&cfg.token, "token", os.Getenv("MACHINE_TOKEN"), "machine token (X-Machine-Token)")
	flag.StringVar(&fryers, "fryers", "FRY-01,FRY-02", "comma-separated fryer codes")
	flag.StringVar(&cfg.scale, "scale", "SCL-01", "scale code (empty = no scale)")
	flag.DurationVar(&cfg.interval, "interval", 2*time.Second, "temperature sampling interval")
	flag.DurationVar(&cfg.packInterval, "pack-interval", 3*time.Second, "time between packs")
	flag.StringVar(&cfg.scenario, "scenario", "normal", "normal|overheat|spike|underweight|burst")
	flag.DurationVar(&cfg.overheatAfter, "overheat-after", 2*time.Minute, "delay before the scenario event")
	flag.Float64Var(&cfg.speed, "speed", 1, "acceleration of heating phase and pre-event wait")
	flag.DurationVar(&cfg.duration, "duration", 0, "stop after this duration (0 = until Ctrl+C)")
	flag.Parse()

	switch cfg.scenario {
	case "normal", "overheat", "spike", "underweight", "burst":
	default:
		fmt.Fprintf(os.Stderr, "invalid -scenario %q\n\n", cfg.scenario)
		flag.Usage()
		os.Exit(2)
	}
	if cfg.speed <= 0 {
		fmt.Fprintln(os.Stderr, "-speed must be > 0")
		os.Exit(2)
	}
	if cfg.token == "" {
		fmt.Fprintln(os.Stderr, "missing -token or MACHINE_TOKEN")
		os.Exit(2)
	}
	cfg.url = strings.TrimRight(cfg.url, "/")
	cfg.fryers = splitCodes(fryers)

	if cfg.scenario == "burst" {
		cfg.fryers = cfg.fryers[:0]
		for i := 1; i <= 20; i++ {
			cfg.fryers = append(cfg.fryers, fmt.Sprintf("LOAD-%02d", i))
		}
		cfg.interval = 100 * time.Millisecond
		cfg.scale = ""
	}
	return cfg
}

// targetTemp returns the set point for a fryer at time now.
func targetTemp(cfg config, base float64, now, eventStart time.Time) float64 {
	since := now.Sub(eventStart)
	switch cfg.scenario {
	case "overheat":
		if since >= 0 && since < 90*time.Second {
			return 187
		}
	case "spike":
		if since >= 0 && since < 40*time.Second {
			return 186
		}
	}
	return base
}

func runFryer(ctx context.Context, client *http.Client, cfg config, st *stats, code string, start, eventStart time.Time) {
	base := 170 + rand.Float64()*4 - 2 // each fryer has its own set point (168..172)
	cur := ambient + rand.Float64()*3
	last := start

	ticker := time.NewTicker(cfg.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case now := <-ticker.C:
			dt := now.Sub(last).Seconds()
			last = now
			if cur < 150 { // accelerate only the heating phase
				dt *= cfg.speed
			}
			target := targetTemp(cfg, base, now, eventStart)
			cur += (target - cur) * (1 - math.Exp(-dt/tau))
			value := math.Round((cur+rand.NormFloat64()*1.2)*10) / 10

			body := batch[tempReading]{
				MachineCode: code,
				Readings:    []tempReading{{TS: now.UTC(), Celsius: value}},
			}
			post(ctx, client, cfg, "/api/v1/readings/temperature", body, st)
		}
	}
}

func runScale(ctx context.Context, client *http.Client, cfg config, st *stats, eventStart time.Time) {
	ticker := time.NewTicker(cfg.packInterval)
	defer ticker.Stop()
	underweightLeft := 5

	for {
		select {
		case <-ctx.Done():
			return
		case now := <-ticker.C:
			grams := 201 + rand.NormFloat64()*2
			if cfg.scenario == "underweight" && !now.Before(eventStart) && underweightLeft > 0 {
				grams = 190 + rand.NormFloat64()*0.5
				underweightLeft--
			}
			grams = math.Round(grams*10) / 10

			body := batch[weightReading]{
				MachineCode: cfg.scale,
				Readings:    []weightReading{{TS: now.UTC(), Grams: grams}},
			}
			post(ctx, client, cfg, "/api/v1/readings/weight", body, st)
		}
	}
}

// post sends JSON with retry on network errors, 5xx and 503 backpressure (FR-SIM-05).
func post(ctx context.Context, client *http.Client, cfg config, path string, body any, st *stats) {
	buf, err := json.Marshal(body)
	if err != nil {
		slog.Error("marshal", "err", err)
		st.failed.Add(1)
		return
	}

	backoff := 200 * time.Millisecond
	for attempt := 1; attempt <= 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, cfg.url+path, bytes.NewReader(buf))
		if err != nil {
			st.failed.Add(1)
			return
		}
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("X-Machine-Token", cfg.token)

		resp, err := client.Do(req)
		if err == nil {
			_, _ = io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			switch {
			case resp.StatusCode < 300:
				st.sent.Add(1)
				return
			case resp.StatusCode >= 500:
				// retry below (includes 503 BACKPRESSURE)
			default:
				slog.Warn("reading rejected", "path", path, "status", resp.StatusCode)
				st.failed.Add(1)
				return
			}
		} else if ctx.Err() != nil {
			return
		}

		select {
		case <-ctx.Done():
			return
		case <-time.After(backoff):
		}
		backoff *= 2
	}
	st.failed.Add(1)
}

func reportStats(ctx context.Context, st *stats) {
	t := time.NewTicker(10 * time.Second)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			slog.Info("stats", "sent", st.sent.Load(), "failed", st.failed.Load())
		}
	}
}

func splitCodes(s string) []string {
	var out []string
	for _, p := range strings.Split(s, ",") {
		if p = strings.TrimSpace(p); p != "" {
			out = append(out, p)
		}
	}
	return out
}

func envOr(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}
```

Menjalankan:
```bash
cd gateway
MACHINE_TOKEN=dev-machine-token go run ./cmd/simulator -scenario normal
go run ./cmd/simulator -token dev-machine-token -scenario overheat -overheat-after 30s -speed 5
go run ./cmd/simulator -token dev-machine-token -scenario underweight -fryers "" -overheat-after 20s
```

Untuk skenario `burst`, daftarkan dulu mesinnya:
```sql
INSERT INTO machines (code, name, kind)
SELECT format('LOAD-%s', lpad(i::text, 2, '0')), format('Load %s', i), 'fryer'
FROM generate_series(1, 20) i ON CONFLICT DO NOTHING;
```

---

## 5. Integrasi dengan Docker Compose & Odoo (dev)

Tambahkan ke `docker-compose.yml`:
```yaml
  sensor-db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: sensor
      POSTGRES_PASSWORD: ${SENSOR_DB_PASSWORD}
      POSTGRES_DB: sensor_db
    volumes:
      - sensor-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sensor -d sensor_db"]
      interval: 5s
      retries: 10

  gateway:
    build: ./gateway
    restart: unless-stopped
    depends_on:
      sensor-db:
        condition: service_healthy
    environment:
      SENSOR_DB_URL: postgres://sensor:${SENSOR_DB_PASSWORD}@sensor-db:5432/sensor_db?sslmode=disable
      ODOO_BASE_URL: http://odoo:8069
      ODOO_API_TOKEN: ${ODOO_GATEWAY_TOKEN}
      GATEWAY_INBOUND_TOKEN: ${GATEWAY_INBOUND_TOKEN}
      MONITOR_TOKEN: ${GATEWAY_MONITOR_TOKEN}
      MACHINE_TOKEN: ${MACHINE_TOKEN}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      TELEGRAM_POLLER: gateway
    ports:
      - "8080:8080"   # dev only

  simulator:
    build: ./gateway
    entrypoint: ["/usr/local/bin/simulator"]
    command: ["-url", "http://gateway:8080", "-scenario", "normal"]
    environment:
      MACHINE_TOKEN: ${MACHINE_TOKEN}
    depends_on:
      - gateway
    profiles: ["demo"]
```
dan volume `sensor-db-data:`.

Generate token acak:
```bash
for v in GATEWAY_INBOUND_TOKEN GATEWAY_MONITOR_TOKEN MACHINE_TOKEN; do echo "$v=$(openssl rand -hex 24)"; done
```

Odoo (*System Parameters*):
| Key | Value |
|---|---|
| `mrp_keripik.gateway_url` | `http://gateway:8080` |
| `mrp_keripik.gateway_inbound_token` | sama dengan `GATEWAY_INBOUND_TOKEN` |
| `mrp_keripik.telegram_poller` | `gateway` |

Work center Penggoreng 1/2: isi `machine_code` FRY-01/FRY-02, `is_sensor_enabled` ✓. Mesin Kemas: SCL-01.

Jalankan selama develop (lebih cepat dari rebuild image):
```bash
docker compose up -d db odoo mailpit sensor-db
cd gateway && set -a && source ../.env && set +a
SENSOR_DB_URL="postgres://sensor:$SENSOR_DB_PASSWORD@localhost:5434/sensor_db?sslmode=disable" \
ODOO_BASE_URL=http://localhost:8069 ODOO_API_TOKEN=$ODOO_GATEWAY_TOKEN \
MONITOR_TOKEN=$GATEWAY_MONITOR_TOKEN go run ./cmd/gateway
# (expose sensor-db port 5434:5432 in docker-compose.override.yml for this)
```
Catatan: bila gateway jalan di host, Odoo (di container) harus memanggil `http://host.docker.internal:8080` → set `mrp_keripik.gateway_url` sementara ke URL itu.

---

## 6. Testing (berdasarkan AC FSD)

| AC | Cara |
|---|---|
| AC-GW-01 | `curl -X POST localhost:8080/api/v1/readings/temperature -H "X-Machine-Token: $MACHINE_TOKEN" -d '{"machine_code":"FRY-01","readings":[{"ts":"…","celsius":171}]}'` → 202; `psql` cek baris |
| AC-GW-02 | Simulator normal 10 menit (2 fryer) → `SELECT count(*) FROM temperature_readings WHERE ts > now()-interval '10 min'` ≈ 600 |
| AC-GW-03 | Mulai WO Penggorengan di layar operator → reading baru punya `odoo_workorder_id` |
| AC-GW-04 | WO berjalan + `-scenario overheat` → 1 baris `anomalies`, 1 Telegram, 1 QC gagal (sumber gateway) di Odoo |
| AC-GW-05 | `-scenario spike` → 0 anomali |
| AC-GW-06 | `docker compose stop odoo` → overheat → `outbox` pending → start odoo → terkirim, tidak dobel |
| AC-GW-07 | Selesai WO → `keripik.sensor.summary` muncul ≤ 5 dtk; bandingkan dengan SQL agregat manual |
| AC-GW-09 | Telegram `/produksi` dari akun terhubung vs akun lain |
| AC-GW-10 | `docker compose stop gateway` → log "bye", exit code 0 (`docker compose ps -a`) |
| AC-SIM-01…05 | Jalankan tiap skenario |

Unit test:
```bash
cd gateway
go vet ./...
go test ./... -race -count=1
```
Minimal: `detector` (tabel di Langkah 10), `outbox.NextDelay`, handler validasi (pakai `app.Test(req)` Fiber).

---

## 7. Error yang Sering Terjadi

| Gejala | Penyebab | Solusi |
|---|---|---|
| `missing required env` | `.env` tidak dimuat | `set -a; source ../.env` atau compose |
| `dial tcp: lookup sensor-db: no such host` | Gateway jalan di host, URL pakai nama service | Pakai `localhost:5434` |
| 401 dari Odoo | `ODOO_API_TOKEN` salah / scope monitor | Buat token scope gateway |
| Odoo tidak bisa kirim event ke Go | `gateway_url` salah (host vs container) | `http://gateway:8080` di compose, `host.docker.internal` jika Go di host |
| `409 Conflict ... getUpdates` | Poller Odoo masih aktif | `telegram_poller = gateway` |
| Anomali tidak muncul padahal suhu tinggi | Detector pakai `time.Now()` vs ts, atau `-speed` membuat event pendek | Pakai `r.TS`; ingat event selalu real time |
| Anomali dobel | State detector hilang saat restart | Saat start, muat anomali `open` dari DB ke state detector |
| `CopyFrom` error tipe | `numeric` vs `float64` | Kirim `float64` (pgx mengonversi) atau cast di SQL |
| Goroutine leak / tidak berhenti | Loop tanpa `case <-ctx.Done()` | Setiap loop wajib memeriksa ctx |
| Data race (`-race`) | Map diakses dari handler & dispatcher | Mutex atau satu pemilik state |
| Simulator `connection refused` | Gateway belum siap | Simulator retry; tunggu |

---

## 8. Pertanyaan Interview

1. **Kenapa data sensor tidak ke Odoo?** (Lihat D1 §4.) Volume & frekuensi, biaya ORM per write, worker terbatas, DB bisnis gemuk, kebutuhan time-series & retensi, isolasi kegagalan.
2. **Bagaimana kamu menghubungkan data mesin ke WO, padahal mesin tidak tahu WO?** Odoo mengirim event start/pause/finish; gateway menyimpan interval per mesin dan menandai setiap reading dengan WO aktif saat ingest.
3. **Bagaimana menjamin tidak ada QC gagal ganda?** Dedup di detector (satu anomali open per mesin+jenis) + `ref` UUID sebagai `external_ref` unik di Odoo + outbox idempoten.
4. **Kenapa outbox, bukan langsung panggil Odoo?** Odoo bisa down/lambat; outbox dalam transaksi yang sama dengan anomali menjamin event tidak hilang dan bisa di-retry dengan backoff.
5. **Apa yang terjadi saat beban melonjak?** Channel buffer menyerap; bila penuh → 503 backpressure; simulator/mesin retry. Batch writer memakai `COPY`. Skala lanjut: partisi tabel, TimescaleDB, MQTT, beberapa replika dengan `SKIP LOCKED`.
6. **Kenapa detector logika murni?** Deterministik (pakai timestamp reading), mudah diuji table-driven, dan tidak butuh mutex karena satu goroutine pemilik.
7. **Graceful shutdown?** `signal.NotifyContext` → stop HTTP → drain channel → flush dengan context baru ber-timeout → tutup pool.
8. **Kenapa Telegram anomali dikirim dari Go?** Peringatan keselamatan harus cepat dan tidak bergantung Odoo; penerima di-cache dari Odoo.
9. **Kenapa Fiber?** Familiar (mirip Express), cepat. Trade-off: berbasis fasthttp, tidak 100% kompatibel `net/http`, jadi sebagian middleware ekosistem tidak bisa dipakai langsung.
