# Carbon-OMS Patterns (Fiber, flat layout)

`Carbon-OMS/` — Go 1.18, Fiber v2, raw `database/sql`, port 3000. Package still named `OMSPrivateBondApi` (rename pending). Layout is flat: `main.go`, `router.go`, `conn-database.go`, `handler-*.go`, `datastruct/`, `migrations/`, `infra/http/`.

When adding to OMS, **match this style** — do not introduce clean-arch layering here.

## Bootstrap & global state (`main.go`)

Init-time loads all globals; `checkErr` / `log.Fatal` panics on failure.

```go
var dbpg *sql.DB           // Postgres
var dbG3SB *sql.DB         // MSSQL G3 (read-only)
var rdb *redis.Client
var memcache *cache.Cache  // patrickmn/go-cache
var netClient *http.Client
var validate *validator.Validate
var masterFee map[string]float64
var ctx = context.Background()

func main() {
    godotenv.Load(".env")
    g3sbApiUrl = strings.Trim(os.Getenv("G3SBApi_Url"), "\"' \t\r\n") // env may have stray quotes
    connectDatabase()
    ctab := crontab.New(); ctab.MustAddJob("07:30 *", getFeeMaster); getFeeMaster()
    app := fiber.New()
    app.Use(recover.New()); app.Use(logger.New())
    SetupRoutes(app)
    app.Listen(":3000")
}
```

New globals are tolerated by the existing style, but **guard cron-mutated maps with a mutex** and **only register a route if its dependency initialised** (see `anti-patterns.md` A5, B1).

## Database (`conn-database.go`)

Raw `database/sql`, no GORM, no query builder (note: `goqu` is in `go.mod` but unused — do not start using it).

```go
psqlInfo := fmt.Sprintf("host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
    os.Getenv("PG_HOST"), os.Getenv("PG_PORT"), os.Getenv("PG_USERNAME"),
    os.Getenv("PG_PASSWORD"), os.Getenv("PG_DATABASE"))
dbpg, err = sql.Open("postgres", psqlInfo); checkErr(err); checkErr(dbpg.Ping())
```

When adding queries: set pool limits (`dbpg.SetMaxOpenConns`, `SetMaxIdleConns`) if you touch connection setup.

## SQL query pattern — parameterized

The existing code string-concatenates (a known SQLi bug). **New code MUST parameterize:**

```go
// ✅ correct
const q = `SELECT col1, col2 FROM "order" WHERE account_id = $1 AND product_type = $2`
rows, err := dbpg.Query(q, accountID, productType)
if err != nil {
    log.Printf("query order: %v", err)
    return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{"error": "db error"})
}
defer rows.Close()
for rows.Next() {
    var o datastruct.Order
    if err := rows.Scan(&o.Col1, &o.Col2); err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{"error": "scan error"})
    }
}
```

MSSQL G3 driver uses `@p1` or named params — still never concatenate.

## Fiber handler pattern

```go
func NewOrderHandler(c *fiber.Ctx) error {
    var req datastruct.NewOrderRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "bad request"})
    }
    if err := validate.Struct(req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
    }
    // ... business logic
    return c.Status(fiber.StatusOK).JSON(res)
}
```

- Signature `(c *fiber.Ctx) error`. Path params `c.Params()`, body `c.BodyParser()`.
- DTOs go in `datastruct/` with `json` tags.
- **Return a real status code on failure** — the legacy habit of `c.Status(200).JSON(emptyRes)` is wrong (see `anti-patterns.md` A6).
- Routes wired in `router.go` `SetupRoutes(app)`, grouped by `app.Group("/equity")` etc.

## G3 SOAP client (`handler-core-api.go`)

SOAP POST to `g3sbApiUrl`, `Content-Type: text/xml`, `SOAPAction: urn:messageTransfer`. Body is a `<soapenv:Envelope>` with `CompanyID=HSC`, credentials from env, `RequestXML` CDATA carrying `<REQUEST Type="CreateCashHold">`. Amount = `fmt.Sprintf("%.0f", amount)` (integer VND). Tx ref = `yyyyMMddHHmmss` + random.

When touching this:
- Build the request with a timeout context, not just the global `netClient` 60s.
- Set headers **before** `Do()`, not after (existing code adds `SOAPAction` post-`Post()` — a bug).
- **Never call it inside a Postgres transaction** (see `anti-patterns.md` A4).
- Prefer returning `(result, error)` over a bare `bool`.

## Validation

`validator` singleton (`validate.Struct(req)`) for DTOs; some flows use manual functions returning a string error-code (e.g. `validateProductType` → `"productTypeQuotaCodeLenMismatch"`). Match whichever the surrounding handler uses.

## Outstanding repo TODOs (don't reintroduce)

Package rename `OMSPrivateBondApi`→`CarbonOMS`; VCB→VTB (`infra/http/vtb/`); `DefaultAccountClassId="BOND_PLUS"` leftover in `datastruct/constant.go`.

See `anti-patterns.md` for what to actively avoid, `migrations.md` for OMS migration handling, `hscgomodules.md` for shared code OMS should adopt.
