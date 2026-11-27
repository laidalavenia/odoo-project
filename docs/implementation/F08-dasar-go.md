# F8. Dasar Go Secukupnya untuk Gateway

| | |
|---|---|
| **Requirement FSD** | Persiapan FR-GW, FR-SIM |
| **Waktu** | Paralel, ±2 jam/minggu, 2–22 Nov 2026 |
| **Hasil** | Paham syntax, struct, pointer, interface, error handling, goroutine, channel, select, context, testing; latihan kecil yang langsung dipakai di F9 |

> Fokus di sini adalah **perbedaan dengan Python/JS**. Semua contoh boleh disalin dan dijalankan, karena ini materi belajar, bukan kode inti proyek.

---

## 0. Setup

```bash
brew install go
go version                          # >= 1.25 recommended
mkdir -p ~/go-playground/basics && cd ~/go-playground/basics
go mod init example.com/basics      # like `npm init` / pyproject
```

Editor: VS Code + ekstensi **Go** (gopls), atau GoLand. Aktifkan format-on-save (`gofmt`). Di Go, format kode tidak bisa diperdebatkan: semua orang memakai `gofmt`.

Materi resmi: <https://go.dev/tour> (Basics, Methods & Interfaces, Concurrency) + <https://gobyexample.com>.

---

## 1. Syntax & Tipe (vs Python/JS)

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	// Static typing with inference (like TS, unlike Python)
	var name string = "FRY-01"
	temp := 172.4            // inferred float64
	count := 0               // int
	const limit = 180.0      // untyped constant

	// Zero values: no undefined/None. int=0, string="", bool=false, pointer/map/slice=nil
	var n int
	fmt.Println(n) // 0

	// Only ONE loop keyword: for
	for i := 0; i < 3; i++ {
		count += i
	}
	machines := []string{"FRY-01", "FRY-02"} // slice (dynamic array)
	for idx, m := range machines {
		fmt.Println(idx, strings.ToLower(m))
	}

	// if with short statement
	if over := temp - limit; over > 0 {
		fmt.Println("over by", over)
	}

	// map (like dict / object)
	lastTemp := map[string]float64{"FRY-01": 171.2}
	v, ok := lastTemp["FRY-02"] // "comma ok": ok=false when missing (no KeyError)
	fmt.Println(v, ok, name, count)
}
```

| Hal | Python / JS | Go |
|---|---|---|
| Tipe | Dinamis / TS opsional | Statis, dikompilasi |
| `None` / `undefined` | Ada | **Zero value**; `nil` hanya untuk pointer, slice, map, channel, interface, func |
| Class | `class` | `struct` + method; **tanpa inheritance** (pakai composition/embedding) |
| Exception | `try/except` | **Nilai `error` dikembalikan**, dicek eksplisit |
| Async | `async/await`, event loop | **goroutine** + channel, tanpa function coloring |
| Visibility | `_private` (konvensi) | Huruf **Besar** = exported (publik dari package), kecil = private |
| Package manager | pip / npm | `go mod` (bawaan) |
| Hasil build | Interpreter | Satu **binary statis** |

---

## 2. Struct, Method, Pointer

```go
package main

import (
	"fmt"
	"time"
)

// Struct with JSON tags (like a Pydantic model / TS interface + serializer)
type Reading struct {
	MachineCode string    `json:"machine_code"`
	TS          time.Time `json:"ts"`
	Celsius     float64   `json:"celsius"`
}

type Detector struct {
	Limit      float64
	overSince  *time.Time // pointer: nil means "not over limit"
	alertCount int
}

// Value receiver: gets a COPY (cannot modify the original)
func (d Detector) IsOver(r Reading) bool {
	return r.Celsius > d.Limit
}

// Pointer receiver: modifies the original (like `self` in Python)
func (d *Detector) Observe(r Reading) {
	if d.IsOver(r) {
		if d.overSince == nil {
			t := r.TS
			d.overSince = &t // & = address of
		}
		return
	}
	d.overSince = nil
}

func main() {
	d := Detector{Limit: 180}           // value
	d.Observe(Reading{"FRY-01", time.Now(), 185})
	fmt.Println(d.overSince != nil)     // true; Go auto-takes &d for pointer methods

	p := &Detector{Limit: 180}          // pointer to a new Detector
	p.Observe(Reading{"FRY-01", time.Now(), 150})
	fmt.Println(p.overSince == nil)     // true
}
```

**Pointer, tanpa ribet:**
- Di Python/JS, objek selalu dioper "by reference"; di Go semuanya **dioper by value (disalin)**, kecuali kamu mengoper pointer (`*T`).
- Pakai pointer receiver (`func (d *Detector)`) jika method **mengubah state** atau struct-nya besar.
- Konsisten: kalau satu method pakai pointer receiver, biasanya semua method tipe itu juga pakai pointer receiver.
- Tidak ada aritmatika pointer seperti C. GC tetap ada.
- `nil` pointer dereference = panic (mirip `AttributeError: 'NoneType'`), jadi cek `if p == nil`.

---

## 3. Interface (implisit!)

```go
package main

import "fmt"

// Interface = set of methods. No "implements" keyword.
type Notifier interface {
	Notify(chatID, text string) error
}

type TelegramNotifier struct{ token string }

func (t TelegramNotifier) Notify(chatID, text string) error {
	fmt.Printf("[telegram %s] %s\n", chatID, text)
	return nil
}

// Fake for tests: also satisfies Notifier automatically
type FakeNotifier struct{ Sent []string }

func (f *FakeNotifier) Notify(chatID, text string) error {
	f.Sent = append(f.Sent, text)
	return nil
}

func alert(n Notifier, msg string) error {
	return n.Notify("123", msg)
}

func main() {
	_ = alert(TelegramNotifier{token: "x"}, "Oil too hot")
	fake := &FakeNotifier{}
	_ = alert(fake, "test")
	fmt.Println(len(fake.Sent)) // 1
}
```

| Python / TS | Go |
|---|---|
| ABC / `implements Interface` eksplisit | **Implisit**: punya method yang sama = memenuhi interface (≈ duck typing, tapi dicek compiler) |
| Interface besar | Konvensi: interface **kecil** (1–3 method), didefinisikan **di sisi pemakai** |
| Mock library | Cukup buat struct fake yang memenuhi interface |

Di gateway: `OdooClient`, `Notifier`, `Store` dibuat sebagai interface agar detector & outbox bisa diuji tanpa jaringan/DB.

---

## 4. Error Handling

```go
package main

import (
	"errors"
	"fmt"
	"strconv"
)

var ErrUnknownMachine = errors.New("unknown machine") // sentinel error (like a custom exception class)

type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string { return e.Field + ": " + e.Msg } // satisfies `error` interface

func parseCelsius(s string) (float64, error) { // multiple return values
	v, err := strconv.ParseFloat(s, 64)
	if err != nil {
		// wrap with context using %w (like `raise X from err`)
		return 0, fmt.Errorf("parse celsius %q: %w", s, err)
	}
	if v < -20 || v > 400 {
		return 0, &ValidationError{Field: "celsius", Msg: "out of range"}
	}
	return v, nil
}

func lookup(code string) error {
	if code != "FRY-01" {
		return fmt.Errorf("lookup %s: %w", code, ErrUnknownMachine)
	}
	return nil
}

func main() {
	if _, err := parseCelsius("abc"); err != nil {
		fmt.Println("error:", err)
	}

	_, err := parseCelsius("999")
	var ve *ValidationError
	if errors.As(err, &ve) { // like `except ValidationError as ve`
		fmt.Println("invalid field:", ve.Field)
	}

	if err := lookup("X"); errors.Is(err, ErrUnknownMachine) { // like isinstance check through the chain
		fmt.Println("404 machine")
	}
}
```

Aturan praktis:
- **Selalu cek `err`** segera setelah pemanggilan. Pola `if err != nil { return ..., fmt.Errorf("context: %w", err) }` akan kamu tulis ratusan kali, dan itu normal.
- Jangan abaikan error dengan `_` kecuali benar-benar sengaja.
- `panic` hanya untuk bug yang tidak bisa dipulihkan (≈ `assert`), bukan alur bisnis.
- Tambahkan konteks saat naik ke atas, lalu putuskan HTTP status di lapisan handler (sama seperti pola controller tipis di Odoo).

---

## 5. `defer`

```go
func readFile(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close() // runs when the function returns (like `with open(...)` / finally)
	// ... use f
	return nil
}
```

`defer` dieksekusi LIFO. Sering dipakai untuk `rows.Close()`, `tx.Rollback()`, `cancel()`, `mu.Unlock()`.

---

## 6. Goroutine & Channel

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	readings := make(chan float64, 100) // buffered channel (queue with capacity 100)
	var wg sync.WaitGroup

	// consumer goroutine (like a background worker)
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range readings { // loop until channel is closed
			fmt.Println("got", v)
		}
	}()

	// producer
	for i := 0; i < 3; i++ {
		readings <- 170 + float64(i) // send (blocks if buffer full)
	}
	close(readings) // signal "no more data"
	wg.Wait()       // like Promise.all / thread.join

	time.Sleep(10 * time.Millisecond)
}
```

| Python asyncio / JS | Go |
|---|---|
| `asyncio.create_task(f())` / panggil fungsi async tanpa await | `go f()` |
| `asyncio.Queue` | `chan T` |
| `await queue.get()` | `v := <-ch` |
| `asyncio.gather` | `sync.WaitGroup` / `errgroup` |
| Satu thread event loop | Goroutine dijadwalkan di beberapa thread OS (paralel sungguhan) |
| `async` menular (function coloring) | Tidak ada; fungsi biasa bisa dijalankan sebagai goroutine |

> Karena goroutine benar-benar paralel, **data race** bisa terjadi. Aturan Go: *"Don't communicate by sharing memory; share memory by communicating."* Gunakan channel, atau `sync.Mutex` bila memang berbagi map. Jalankan test dengan `go test -race`.

### `select` + timeout + ticker

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
batch := make([]float64, 0, 500)

for {
	select {
	case v, ok := <-readings:
		if !ok { // channel closed -> flush and exit
			flush(batch)
			return
		}
		batch = append(batch, v)
		if len(batch) >= 500 {
			flush(batch)
			batch = batch[:0]
		}
	case <-ticker.C: // flush every second even if batch not full
		if len(batch) > 0 {
			flush(batch)
			batch = batch[:0]
		}
	}
}
```

Ini **persis** pola batch writer di F9.

### Mutex untuk state bersama

```go
type IntervalCache struct {
	mu      sync.RWMutex
	byMachine map[string]int // machine_code -> active workorder id
}

func (c *IntervalCache) Get(code string) (int, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	id, ok := c.byMachine[code]
	return id, ok
}

func (c *IntervalCache) Set(code string, wo int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.byMachine[code] = wo
}
```

---

## 7. `context.Context`: pembatalan & timeout

```go
package main

import (
	"context"
	"fmt"
	"os/signal"
	"syscall"
	"time"
)

func worker(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done(): // cancelled (SIGTERM) or timeout
			fmt.Println(name, "stopping:", ctx.Err())
			return
		case <-time.After(500 * time.Millisecond):
			fmt.Println(name, "tick")
		}
	}
}

func main() {
	// Root context cancelled on Ctrl+C / docker stop (SIGTERM)
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	go worker(ctx, "outbox")

	// Per-request timeout (like AbortSignal.timeout in JS / requests timeout=3)
	reqCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	_ = reqCtx // pass to http.NewRequestWithContext / pgx Query

	<-ctx.Done()
	time.Sleep(100 * time.Millisecond) // let worker print
}
```

Aturan:
- `ctx` selalu **parameter pertama** fungsi yang melakukan I/O.
- Jangan simpan ctx di struct (kecuali alasan kuat).
- Setiap `WithTimeout`/`WithCancel` wajib `defer cancel()`.

---

## 8. HTTP & JSON (Fiber v3 dan `net/http` client)

Server kecil (Fiber v3, ⚠️ cek API di docs; v2 memakai `*fiber.Ctx` dan `c.BodyParser`):
```go
package main

import (
	"log"

	"github.com/gofiber/fiber/v3"
)

type tempIn struct {
	MachineCode string  `json:"machine_code"`
	Celsius     float64 `json:"celsius"`
}

func main() {
	app := fiber.New()

	app.Get("/healthz", func(c fiber.Ctx) error {
		return c.JSON(fiber.Map{"status": "ok"})
	})

	app.Post("/readings", func(c fiber.Ctx) error {
		var in tempIn
		if err := c.Bind().Body(&in); err != nil {
			return c.Status(fiber.StatusUnprocessableEntity).JSON(fiber.Map{"error": err.Error()})
		}
		return c.Status(fiber.StatusAccepted).JSON(fiber.Map{"accepted": 1})
	})

	log.Fatal(app.Listen(":8080"))
}
```
```bash
go get github.com/gofiber/fiber/v3
go run .
curl -s -X POST localhost:8080/readings -H 'Content-Type: application/json' -d '{"machine_code":"FRY-01","celsius":172}'
```

HTTP client dengan timeout & context:
```go
func postJSON(ctx context.Context, client *http.Client, url, token string, body any) error {
	buf, err := json.Marshal(body)
	if err != nil {
		return fmt.Errorf("marshal: %w", err)
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(buf))
	if err != nil {
		return err
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+token)

	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("post %s: %w", url, err) // network error -> retryable
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 300 {
		b, _ := io.ReadAll(io.LimitReader(resp.Body, 2048))
		return fmt.Errorf("post %s: status %d: %s", url, resp.StatusCode, b)
	}
	return nil
}
```

> Selalu buat `&http.Client{Timeout: 5 * time.Second}`. Default client **tanpa timeout** bisa menggantung selamanya.

---

## 9. Testing (table-driven)

`detector.go`
```go
package detector

func Classify(celsius, min, max float64) string {
	switch {
	case celsius < min:
		return "low"
	case celsius > max:
		return "high"
	default:
		return "ok"
	}
}
```

`detector_test.go`
```go
package detector

import "testing"

func TestClassify(t *testing.T) {
	tests := []struct {
		name string
		in   float64
		want string
	}{
		{"below", 150, "low"},
		{"inside", 170, "ok"},
		{"edge max", 180, "ok"},
		{"above", 181, "high"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Classify(tt.in, 160, 180); got != tt.want {
				t.Errorf("Classify(%v) = %q, want %q", tt.in, got, tt.want)
			}
		})
	}
}
```
```bash
go test ./... -race -v
```

Mirip `pytest.mark.parametrize`. Tidak perlu library assert.

---

## 10. Struktur project & module

```
gateway/
├── go.mod                 # module github.com/<you>/odoo-project/gateway
├── cmd/gateway/main.go    # package main (entrypoint). Wiring only.
├── cmd/simulator/main.go
└── internal/...           # packages not importable from outside this module
```

- `internal/` dijaga compiler: package di bawahnya tidak bisa di-import module lain.
- Satu folder = satu package.
- Hindari package bernama `utils`/`common`; beri nama sesuai tanggung jawab (`detector`, `outbox`).
- Dependency injection manual di `main.go` (buat store, client, lalu oper ke service), tanpa framework DI.

---

## 11. Latihan (wajib sebelum F9)

| # | Latihan | Konsep | Dipakai di |
|---|---|---|---|
| L1 | Struct `Reading` + validasi → kembalikan `*ValidationError` | struct, error, errors.As | Handler ingest |
| L2 | Batch writer: channel + ticker + flush ke `fmt.Println`, berhenti saat ctx cancel & flush sisa | goroutine, select, context | `internal/ingest` |
| L3 | Detector suhu: state per mesin (`map[string]*state`), event "anomaly" bila > 180 selama ≥ 60 dtk (pakai timestamp reading, bukan jam sistem) + table test | struct, pointer, testing | `internal/detector` |
| L4 | Retry dengan exponential backoff + jitter, fungsi `NextDelay(attempt int) time.Duration` + test | time, math/rand | `internal/outbox` |
| L5 | Fiber server `/healthz` + `POST /readings` yang memasukkan ke channel L2; return 503 jika channel penuh (`select` dengan `default`) | Fiber, backpressure | `internal/httpapi` |
| L6 | SSE hub: `map[chan []byte]struct{}`, register/unregister, broadcast non-blocking | channel, mutex | `internal/stream` |

Petunjuk L5 (backpressure):
```go
select {
case ingestCh <- r:
	// accepted
default:
	return c.Status(fiber.StatusServiceUnavailable).JSON(fiber.Map{"error": "BACKPRESSURE"})
}
```

Petunjuk L3: buat detector **murni** (tanpa I/O). Input reading → output `[]Event`. I/O (DB, Telegram, Odoo) di lapisan lain, jadi mudah diuji.

---

## 12. Pertanyaan Interview (Go)

1. **Bagaimana Go menangani error dibanding exception?** Error adalah nilai; dikembalikan dan dicek eksplisit; dibungkus dengan `%w`; diperiksa dengan `errors.Is/As`. Keuntungannya, alur error terlihat jelas di kode; kekurangannya, kode lebih verbose.
2. **Goroutine vs thread?** Goroutine sangat ringan (stack kecil, tumbuh dinamis), dijadwalkan runtime Go di atas sejumlah thread OS (M:N).
3. **Kapan pakai channel vs mutex?** Channel untuk aliran data/kepemilikan berpindah (pipeline ingest), mutex untuk state bersama kecil (cache interval).
4. **Apa fungsi `context`?** Propagasi pembatalan, deadline, dan nilai request-scoped ke seluruh rantai panggilan; dasar graceful shutdown.
5. **Interface implisit, apa untungnya?** Decoupling: package pemakai mendefinisikan interface kecil; implementasi tidak perlu tahu. Mudah membuat fake untuk test.
6. **Value vs pointer receiver?** Pointer bila mengubah state atau struct besar; value untuk tipe kecil immutable.
