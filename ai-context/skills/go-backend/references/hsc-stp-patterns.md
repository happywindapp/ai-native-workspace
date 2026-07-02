# HSC_STP Patterns (clean architecture, Gin)

`HSC_STP/` — Go 1.19, Gin + ginwrapper, GORM, zap, YAML config, port 3006. This is the cleaner of the two Go services — match its layering when adding to STP.

## Layout

```
cmd/{service,migrate}/main.go
config/config.go
server/server.go
handler/            admin, auth, common, configure, stp_handler, stp_allocation, ...
internal/
  common/error.go
  domain/hsc_stp/{constant,model,repo,usecase}
  infra/{http/itrade, http/smartNotify, repo}
migration/sql/
```

Flow: **handler → usecase (domain) → repo (infra)**. Handlers never touch the DB directly.

## Bootstrap (`cmd/service/main.go`)

```go
func main() {
    var configFile string
    flag.StringVar(&configFile, "config-file", "", "config file path")
    flag.Parse()

    cfg, err := config.Load(configFile)
    if err != nil { zap.S().Errorf("load config fail: %v", err); panic(err) }

    s, err := server.NewServer(cfg)
    if err != nil { zap.S().Errorf("create server fail: %v", err); panic(err) }
    s.Init()
    if err := s.ListenHTTP(); err != nil { panic(err) }
}
```

## Config (`config/config.go`) — YAML, not `.env`

```go
type AppConfig struct {
    ServiceName string
    HscStp      *HscStpConfig    `yaml:"hsc_stp"`
    DB          *PostgresConfig  `yaml:"db"`
    Redis       *RedisConfig     `yaml:"redis"`
    G3SBDB      *SQLServerConfig `yaml:"g3sb_db"`
    RateLimit   *RateLimitConfig `yaml:"rate_limit"`
}

func Load(filePath string) (*AppConfig, error) {
    if len(filePath) == 0 { filePath = os.Getenv("CONFIG_FILE") }
    b, err := os.ReadFile(filePath)
    if err != nil { return nil, err }
    b = []byte(os.ExpandEnv(string(b)))   // ${VAR} expansion inside YAML
    cfg := &AppConfig{}
    return cfg, yaml.Unmarshal(b, cfg)
}
```

Add new config as a struct field with a `yaml:` tag; secrets stay as `${ENV_VAR}` placeholders in the YAML file.

## Server init (`server/server.go`)

Two-phase: `NewServer()` builds router + middleware; `Init()` allocates resources and wires layers.

```go
func NewServer(cfg *AppConfig) (*Server, error) {
    router := gin.New()
    router.Use(middleware.SetRequestID())   // HscGoModules
    router.Use(contxt.SetupMyContext())     // HscGoModules
    router.Use(middleware.SetupLog())
    router.Use(gzip.Gzip(gzip.DefaultCompression))
    router.Use(gin.Recovery())
    return &Server{router: router, cfg: cfg}, nil
}

func (s *Server) Init() {
    redisClient, _ := redis_wrapper.InitRedis(s.cfg.Redis)
    db, _ := postgres_wrapper.InitPostgres(s.cfg.DB)          // *gorm.DB
    sqlServer, _ := sql_server_wrapper.InitSqlServer(s.cfg.G3SBDB)
    sqlRepo := repo.NewSQLRepo(db, sqlServer)
    csvRepo := repo.NewCSVRepo(s.cfg.HscStp.ReportFolder)
    domains := s.initDomains(sqlRepo, csvRepo)               // build usecases
    s.initRouters(domains)                                   // wire handlers
}
```

Resource init panics on failure (acceptable at boot). New dependency → add to `Init()`, inject down through `initDomains` → usecase constructor → repo.

## Handler pattern (`handler/*.go`) — ginwrapper

```go
func (h *Handler) listStpOutputByType() gin.HandlerFunc {
    return ginwrapper.WithContext(func(ctx *ginwrapper.Context) {
        log := contxt.NewContext(ctx).GetLoggerWithPrefix("handler-listStpOutputByType")
        AllowHost(ctx, h.cfg.AllowHost)

        req := &model.ListStpOutputByTypeRequest{}
        if err := ctx.ShouldBind(req); err != nil {
            log.Warnf("parse request fail: %v", err)
            ctx.BadRequest(err); return
        }
        items, err := h.hscStp.ListStpOutputByType(ctx, req.Type, req.Ref)
        if err != nil { ctx.BadRequest(err); return }

        var res *model.ListStpOutputByTypeResponse
        ctx.PureJSON(http.StatusOK, res.ToResponse(req.Type, items))
    })
}
```

- Handler returns `gin.HandlerFunc`, wrapped by `ginwrapper.WithContext`.
- Logger from `contxt.NewContext(ctx).GetLoggerWithPrefix(...)` — carries request ID.
- Bind with `ctx.ShouldBind`; respond with `ctx.BadRequest` / `ctx.PureJSON`.
- Request/response DTOs in `handler/model/` (or `internal/domain/hsc_stp/model`).

## Repo pattern (`internal/infra/repo`)

GORM-based; `base_sql` holds the shared `*gorm.DB` handles, per-entity files (`stps_o518.go`, `send_stp.go`, …) hold queries. `NewSQLRepo(db, sqlServer)` is the constructor injected into usecases. Add a new entity = new repo file + method on the repo interface in `internal/domain/hsc_stp/repo`.

## Error handling

`internal/common/error.go` holds sentinel error vars (`ErrNotFound = errors.New("not found")`). Propagate `error` up through usecase → handler; the handler maps it to `ctx.BadRequest` / status. Wrap with `fmt.Errorf("...: %w", err)` so callers can `errors.Is`.

See `hscgomodules.md` for the shared packages STP relies on, `migrations.md` for `cmd/migrate`, `anti-patterns.md` B5 (avoid string-enum `switch` on `StpType`).