"# 01 — Klasör Yapısı, Core ve Eklenti Noktaları

Bu dokümanda PocketBase fork'undaki **her klasörün ne işe yaradığını**, **hangisi \"core\" (el sürme) hangisi \"eklenti noktası\" (güvenle kullan)** olduğunu, ve **dosyalar arası bağımlılık yönünü** anlatıyorum.

> TL;DR — Feature eklerken sadece `examples/base/main.go` + yeni `plugins/<adın>/` paketi yeterli. `core/`, `apis/`, `forms/`, `mails/`, `migrations/`, `tools/` upstream ile senkron kalsın.

---

## 1. Proje kökü — dosyalar

```
/app
├── pocketbase.go              ← PocketBase facade, New(), default config
├── pocketbase_test.go
├── modernc_versions_check.go  ← build-time uyarı (modernc/sqlite + libc versiyon kontrolü)
├── go.mod, go.sum             ← Go dependency manifest
├── Makefile                   ← Geliştirici komutları
├── CHANGELOG.md, README.md, LICENSE.md, CONTRIBUTING.md
├── .goreleaser.yaml           ← Release build config
└── .github/workflows/         ← CI/CD pipeline'ları
```

### `pocketbase.go` — Facade

PocketBase'in ana giriş yapısı. `pocketbase.New()` fonksiyonu bunu döndürür. İçinde:

```go
type PocketBase struct {
    core.App                    // <- BaseApp embed'li (core/base.go)
    RootCmd *cobra.Command      // <- cobra CLI kökü
}
```

Burada **zaten hazır** olan şeyler:
- `RootCmd`'ye `serve` ve `superuser` komutları eklenmiş (cmd/ klasöründen)
- `--dir`, `--encryptionEnv`, `--dev`, `--automigrate` gibi global flag'ler tanımlı
- Bootstrap & signal handling (graceful shutdown)

Bu dosyaya **normal şartlarda dokunma**. Dokunursan `pocketbase.New()` davranışı değişir — bunun yerine `examples/base/main.go`'da flag ekle veya `app.RootCmd.AddCommand(...)` ile CLI komutu ekle.

---

## 2. `core/` — Domain Katmanı (CORE — DOKUNMA)

**En kritik klasör.** 80+ Go dosyası içeriyor. Tüm iş mantığı, veri modeli, event bus ve DB erişimi burada.

### Alt kategorileri

```
core/
│
├── app.go                          ← App interface (1541 satır!) — tüm 83 hook buradan exposed
├── base.go                         ← BaseApp struct — App interface'inin default implementation'ı
├── base_backup.go                  ← Backup/restore motoru
│
├── events.go (595 satır)           ← Event type'ları (ServeEvent, RecordRequestEvent, ModelEvent…)
├── event_request*.go               ← HTTP request context sarmalayıcı
│
├── db.go, db_*.go                  ← DB soyutlaması (dbx üstünde)
│   ├── db_connect.go               ← SQLite bağlantı (modernc.org/sqlite) + pragma'lar
│   ├── db_connect_nodefaultdriver.go ← `no_default_driver` build tag için alt. yol
│   ├── db_builder.go               ← Query builder sarmalayıcı
│   ├── db_tx.go                    ← Transaction yönetimi
│   ├── db_retry.go                 ← SQLITE_BUSY retry mantığı
│   └── db_model.go, db_table.go
│
├── collection_model.go             ← Collection (tablo tanımı) modeli
├── collection_model_auth_options.go   ← Auth collection ayarları
├── collection_model_view_options.go   ← View collection ayarları
├── collection_model_base_options.go   ← Base collection ayarları
├── collection_query.go             ← Collection bulma/listeleme
├── collection_validate.go          ← Collection şema doğrulaması
├── collection_record_table_sync.go ← Şema değişince DB tablosunu otomatik eşitle (ALTER TABLE)
├── collection_import.go
│
├── record_model.go                 ← Record (satır) modeli
├── record_query.go                 ← Record arama/filtreleme/sayfalama
├── record_*.go                     ← Auth, OTP, MFA, email-change, password-reset, impersonation, proxy
│
├── field.go + field_*.go           ← Field tipleri (text, number, bool, relation, file, date, json, url, select, email, editor, password, geopoint, autodate)
│
├── auth_origin_*.go                ← Auth origin (oturum cihaz/IP kaydı)
├── external_auth_*.go              ← OAuth2 provider kayıtları (Google, GitHub vs.)
├── mfa_*.go, otp_*.go              ← 2FA, OTP akışları
├── log_*.go                        ← Request/error loglama modeli
├── settings_model.go               ← App settings (SMTP, meta, S3, backup)
│
├── logger_*.go                     ← slog tabanlı logger (DB'ye yazan handler)
├── mailer_*.go                     ← Mail gönderim (Sendmail / SMTP)
├── filesystem.go                   ← S3 / local file storage facade
├── cron.go                         ← Arka plan cron scheduler
├── geo.go                          ← Geopoint hesapları
├── test_app.go                     ← TestApp (unit test fixture)
│
└── validators/                     ← Ortak validation helper'ları (alt paket)
```

### Neden dokunma?

Bu dosyalar PocketBase'in \"kalbi\". İçlerinde:
- **Tüm hook'lar tanımlı** (`OnRecordCreate`, `OnCollectionUpdate`, `OnServe`, vs. — 83 adet)
- **Tüm default doğrulamalar**
- **DB şema senkronizasyonu** — koleksiyon tanımı değişince tablonun `ALTER TABLE`'ını bu katman yazıyor

Değiştirirsen:
1. `upstream`'den `git pull` yaptığında merge conflict yersin
2. SDK'lar (js-sdk, dart-sdk) bekledikleri davranışı bulamaz
3. Admin UI (ui/) bazı field'leri doğru göstermeyebilir

**İstisna:** Yeni bir `field tipi` eklemek, yeni bir sistem event'i eklemek zorundaysan — bunlar **core'u genişletmeyi gerektirir**. O zaman dokunduğun satırları minimal tut; ideal olarak ayrı bir commit'te hallet, upstream'e PR açmayı düşün.

---

## 3. `apis/` — HTTP Katmanı (CORE — DOKUNMA)

```
apis/
├── base.go                  ← Router kurulum + default middleware stack
├── serve.go                 ← HTTP server başlatma, TLS, autocert, graceful shutdown
├── installer.go             ← İlk superuser oluşturma akışı (/_/#/pbinstal/...)
├── middlewares*.go          ← CORS, Gzip, Rate limit, Body limit
├── realtime*.go             ← SSE subscriptions (/api/realtime)
│
├── collection*.go           ← /api/collections endpoint'leri
├── record_crud*.go          ← /api/collections/:col/records endpoint'leri
├── record_auth*.go          ← Auth endpoint'leri (login, refresh, OAuth2, OTP, MFA, email-change, password-reset, verification, impersonate)
├── file.go                  ← /api/files dosya servisi (thumbnail, protected token)
├── batch.go                 ← /api/batch transactional çoklu istek
├── backup*.go               ← /api/backups (list, create, upload, download, restore)
├── settings*.go, logs*.go, health*.go, cron.go
│
├── api_error*.go            ← Standart API hata yapısı (ApiError)
└── static.go                ← Static dosya servisi (ui/dist'i de bu servisi kullanıyor)
```

### Route haritası (özet)

| Path | İş |
|---|---|
| `/api/collections/:name/records/...` | CRUD + listeleme |
| `/api/collections/:name/auth-with-password` | Login |
| `/api/collections/:name/auth-refresh` | Token yenile |
| `/api/collections/:name/auth-with-oauth2` | OAuth2 |
| `/api/collections/:name/request-password-reset` | Şifre sıfırlama |
| `/api/files/:col/:id/:filename` | Dosya servisi |
| `/api/realtime` | SSE |
| `/api/batch` | Transactional batch |
| `/api/backups/...` | Backup yönetimi |
| `/api/health` | Health check |
| `/api/settings` | App settings |
| `/_/...` | Admin UI (embed edilmiş Svelte) |

### Neden dokunma?

Bu endpoint'lerin URL'i, request/response şekli **SDK sözleşmesi**. Kırdığında js-sdk ve dart-sdk çalışmaz. Yeni endpoint ekleyeceksen `OnServe` içinde `se.Router.GET(...)` kullan (Bkz. 03-ozellik-ekleme.md).

---

## 4. `forms/` — Request Validation (CORE)

```
forms/
├── base.go
├── record_upsert.go           ← Record create/update için form DTO
├── record_password_reset_*.go
├── record_email_change_*.go
├── record_verification_*.go
├── record_otp_request.go
├── test_email_send.go         ← SMTP test email
└── …
```

Form'lar `ozzo-validation` kullanan tipik DTO'lar. `apis/` katmanı bunları handler'larda kullanıyor. Sen de kendi endpoint'inde istersen kullanabilirsin ama genellikle gerek olmaz — kendi struct'ını kendin validate edebilirsin.

---

## 5. `mails/` — E-posta Template'leri (CORE)

```
mails/
├── mailer.go                  ← Mailer sarmalayıcı
├── record_password_reset.go
├── record_email_change.go
├── record_verification.go
├── record_otp.go
├── record_auth_alert.go
└── templates/                 ← HTML + text template dosyaları
    ├── layout.html
    ├── password_reset.html
    ├── verification.html
    ├── email_change.html
    ├── otp.html
    └── auth_alert.html
```

Bu template'leri **doğrudan değiştirmek** yerine, `OnMailerRecordPasswordResetSend`, `OnMailerRecordVerificationSend`, `OnMailerRecordOTPSend` hook'larını dinleyerek `e.Message.HTML`, `e.Message.Text`, `e.Message.Subject` alanlarını override et. Upstream'e bağımlılığı kırmaz.

---

## 6. `migrations/` — Sistem Migration'ları (CORE — DOKUNMA)

```
migrations/
├── 1640988000_init.go                   ← İlk kurulum: _collections, _users, vs.
├── 1673167670_multi_match_expr.go
├── 1689579878_renamed_ext_auths.go
├── 1718894037_delete_external_auths.go
├── 1727864177_superusers_collection.go  ← Superuser (admin) modelini record'a dönüştürür
├── 1745248340_reset_rate_limit_rules.go
└── ...
```

Bunlar **sistem tablolarının** (koleksiyonların kendisi, _superusers_ collection'ı, auth_origins vs.) şemasını evrimleştiren Go migration'ları. Her biri `core/base.go`'nun `RunAllMigrations()` tarafından `_migrations` tablosuna bakılarak bir kez çalıştırılır.

> **Kendi uygulaman için migration yazacaksan** → buraya değil! `pb_migrations/` klasörü (JS) ya da `plugins/migratecmd/` üzerinden oluşturulan kullanıcı migration klasörüne. Detay → [02-sqlite.md § Migration bölümü](./02-sqlite.md).

---

## 7. `cmd/` — CLI Komutları (CORE — HAFİF DOKUNULABİLİR)

```
cmd/
├── serve.go                   ← `pocketbase serve` komutu
└── superuser.go               ← `pocketbase superuser upsert|create|update|delete|otp` komutları
```

Yeni CLI komutu eklemek için **bu klasöre DOSYA EKLEME**. Bunun yerine:

```go
// examples/base/main.go içine
app.RootCmd.AddCommand(&cobra.Command{
    Use:   \"mycmd\",
    Short: \"…\",
    Run: func(cmd *cobra.Command, args []string) { … },
})
```

---

## 8. `plugins/` — Opsiyonel Eklentiler (EKLENTİ NOKTASI — ÖRNEK AL)

Burası **\"kendi eklentini yazmak için şablon bakılacak\"** klasör.

```
plugins/
├── jsvm/                      ← pb_hooks/ ve pb_migrations/ JS dosyalarını goja ile çalıştırır
│   ├── jsvm.go                ← MustRegister(app, Config) → goja runtime havuzu kurar
│   ├── binds.go               ← Go fonksiyonlarını JS'e bind eder
│   ├── mapper.go              ← Go ↔ JS tip dönüşümü
│   ├── pool.go                ← Prewarm goja runtime havuzu
│   ├── form_data.go
│   └── internal/              ← Generated JS type tanımları
│
├── migratecmd/                ← `pocketbase migrate create|up|down|...` komutunu ekler
│   └── ...                    ← Go ve JS template üretimi
│
└── ghupdate/                  ← `pocketbase update` komutu (GitHub Releases'tan selfupdate)
    └── ghupdate.go
```

### Plugin yazmanın standart şablonu

```go
package myplugin

import \"github.com/pocketbase/pocketbase/core\"

type Config struct { /* … */ }

func MustRegister(app core.App, config Config) {
    if err := Register(app, config); err != nil {
        panic(err)
    }
}

func Register(app core.App, config Config) error {
    app.OnBootstrap().BindFunc(func(e *core.BootstrapEvent) error {
        // init work
        return e.Next()
    })
    app.OnServe().BindFunc(func(e *core.ServeEvent) error {
        e.Router.GET(\"/api/myplugin/hello\", func(re *core.RequestEvent) error {
            return re.String(200, \"hi\")
        })
        return e.Next()
    })
    return nil
}
```

Sonra `examples/base/main.go`'da:

```go
myplugin.MustRegister(app, myplugin.Config{ /* … */ })
```

---

## 9. `tools/` — Altyapı Kütüphaneleri (CORE — KULLAN, DOKUNMA)

Bunlar \"PocketBase'in iç Standard Library'si\". Alt paketler bağımsız modüller.

```
tools/
├── hook/            ← Hook<Event> generic event bus (OnXxx sistemlerinin temeli)
├── router/          ← HTTP router (net/http + ServeMux wrapper) — se.Router bunu kullanır
├── mailer/          ← SMTP + sendmail gönderim
├── cron/            ← Cron scheduler (5/6 alan, timezone'lu)
├── filesystem/      ← S3 + local storage (io/fs facade)
├── logger/          ← slog handler'ları (DB'ye yazan, batch'li)
├── security/        ← JWT, token, encrypt/decrypt (AES-GCM), hashing, random string
├── subscriptions/   ← Realtime broker (SSE client registry + broadcast)
├── search/          ← List+filter+sort parser + dbx query generator
├── tokenizer/       ← PocketBase filter-expression scanner
├── picker/          ← Response field picker (`?fields=...`)
├── types/           ← JsonArray, JsonMap, DateTime, Pointer, Downloader
├── store/           ← Concurrent map + LRU cache
├── dbutils/         ← DB helper'lar (version, schema parse)
├── archive/         ← ZIP create/extract
├── auth/            ← OAuth2 provider'lar (Google, GitHub, Discord, Apple, …)
├── inflector/       ← singularize/pluralize/camel/snake
├── list/            ← slice helper'lar
├── osutils/         ← FS helper'lar (IsProbablyGoRun vs.)
├── template/        ← html/text template wrapper
└── routine/         ← FireAndForget, RunWithRecover (panic-safe goroutine)
```

Sen bunları **kendi plugin kodunda da import edip kullanabilirsin.** Örnek:

```go
import \"github.com/pocketbase/pocketbase/tools/security\"

token, _ := security.NewJWT(payload, secret, 24*time.Hour)
```

---

## 10. `ui/` — Admin Dashboard (EKLENTİ NOKTASI — DEĞİŞTİRİLEBİLİR)

```
ui/
├── embed.go                   ← //go:embed all:dist  → binary'e gömer
├── README.md
├── package.json               ← Svelte + Vite + CodeMirror + Chart.js + Leaflet
├── vite.config.js
├── index.html
├── public/                    ← Static asset'ler
├── src/                       ← Svelte kaynak kodu
│   ├── App.svelte
│   ├── main.js
│   ├── routes.js              ← SPA route tanımları (svelte-spa-router)
│   ├── components/
│   ├── scss/
│   ├── stores/                ← Svelte store'lar
│   ├── utils/
│   ├── actions/
│   ├── providers.js
│   ├── mimes.js
│   └── autocomplete.worker.js
└── dist/                      ← Prod build (vite build) — embed buradan alıyor
```

### Admin UI nasıl serve ediliyor?

`apis/serve.go:80`:

```go
pbRouter.GET(\"/_/{path...}\", Static(ui.DistDirFS, false))
```

`ui.DistDirFS` → `//go:embed all:dist` sayesinde **binary'nin içinde** taşınan gömülü dosya sistemi. Yani PocketBase derlendiği anda `ui/dist/` içeriği `.exe`/`base` dosyasının içine kopyalanıyor.

### Değiştirme yolları (3 strateji):

1. **Svelte'i modifiye et** — `ui/src/` içinde ne değiştirirsen `npm run build` ile `ui/dist/` oluşur, sonra `go build` ile tekrar Go binary'sine gömülür.
2. **`ui/dist/`'i tamamen Next.js export'u ile değiştir** — aynı embed mekanizmasından yararlanıp kendi UI'ını binary'e göm.
3. **`/_/` route'unu devre dışı bırakıp kendi UI'ını ayrı host'la** — PocketBase sadece API sunsun, UI ayrı port/domain'de koşsun.

Detaylı adım adım rehber → [**04-ui-degistirme-nextjs.md**](./04-ui-degistirme-nextjs.md)

---

## 11. `examples/base/main.go` — Binary Giriş Noktası (SENİN ALANIN)

Prebuilt PocketBase executable'ı **bu dosyadan** derleniyor. Yani `github.com/pocketbase/pocketbase/releases` sayfasından indirdiğin `.exe` = `examples/base/main.go`'nun build'i.

```go
// tam içeriği özetle:
func main() {
    app := pocketbase.New()

    // CLI flag'leri
    app.RootCmd.PersistentFlags().StringVar(&hooksDir, \"hooksDir\", \"\", \"…\")
    // … diğer flag'ler

    // Eklentileri register et
    jsvm.MustRegister(app, jsvm.Config{…})
    migratecmd.MustRegister(app, app.RootCmd, migratecmd.Config{…})
    ghupdate.MustRegister(app, app.RootCmd, ghupdate.Config{})

    // OnServe hook: static pb_public klasörü
    app.OnServe().Bind(&hook.Handler[*core.ServeEvent]{…})

    if err := app.Start(); err != nil {
        log.Fatal(err)
    }
}
```

**Senin feature'larını tipik olarak buraya ekliyoruz.** Yeni plugin yazdıysan bir `MustRegister` çağrısı, basit bir logic ise `app.OnXxx().BindFunc(...)` kullanıyorsun.

---

## 12. `tests/` — Test Utilities (CORE)

```
tests/
├── app.go                     ← NewTestApp() — unit test'ler için izole PocketBase instance
├── request.go                 ← ApiScenario (tipik API endpoint testi DSL'i)
├── http_test_util.go
├── event_calls.go             ← Hook çağrılarını assert eden yardımcı
└── data/                      ← Fixture data: test collections, files
    ├── data.db                ← Önceden hazırlanmış test DB
    ├── auxiliary.db
    └── storage/
```

Kendi plugin'ini test ederken `tests.NewTestApp(...)` kullan.

---

## 13. Bağımlılık yönü (import graph)

Aşağıdaki yön doğal, aksi yön genelde yanlış:

```
tools/*           ← hiçbir PB paketine bağımlı değil (leaf)
  ↑
core/             ← tools/* kullanır, DB+model+event kurar
  ↑
forms/            ← core kullanır (record_upsert, vs.)
mails/            ← core + tools/mailer
  ↑
apis/             ← core + forms + mails + tools + ui (embed)
  ↑
plugins/*         ← core + apis + tools (hook üzerinden davranış ekler)
  ↑
pocketbase.go     ← core + cmd + ui + tools
  ↑
examples/base/main.go   ← her şeyi orkestre eder
```

> \"Core'a dokunma\" demenin teknik karşılığı: alt katmanların, üst katmanların davranışına bağımlı olmadığı bir grafiği bozma.

---

## 14. Karar Tablosu — \"Ne istiyorsam nereye eklemeliyim?\"

| İhtiyaç | Yer | Core'a dokunur mu? |
|---|---|---|
| Yeni REST endpoint | `main.go` → `OnServe` → `se.Router.GET(...)` | ❌ Hayır |
| Record create/update sırasında business logic | `main.go` → `OnRecordCreate/Update` hook | ❌ Hayır |
| Yeni CLI komutu | `main.go` → `app.RootCmd.AddCommand(...)` | ❌ Hayır |
| Yeniden kullanılabilir bileşen | Yeni `plugins/<adım>/` paketi | ❌ Hayır |
| E-posta template'ini özelleştir | `OnMailerRecordXxxSend` hook | ❌ Hayır |
| Auth akışına ek doğrulama | `OnRecordAuthWithPasswordRequest` hook | ❌ Hayır |
| Scheduled task | `app.Cron().Add(...)` (içeride `tools/cron`) | ❌ Hayır |
| Kendi admin UI'ı | `ui/dist/`'i değiştir veya `/_/` yerine custom | ❌ (dist değiştirilebilir) |
| Custom koleksiyon alan tipi | `core/field_*.go` + `core/field.go` | ✅ Evet |
| Yeni OAuth2 provider | `tools/auth/` — upstream PR öner | ✅ Evet |
| Yeni built-in hook event | `core/events.go` + `core/app.go` | ✅ Evet |

---

## 15. Özet

- **core/**, **apis/**, **forms/**, **mails/**, **migrations/**, **tools/** → upstream'e ait, dokunma
- **plugins/** → örnek al, kendi pluginini yan klasörde yaz
- **examples/base/main.go** → build hedefi, feature register etme yeri
- **ui/** → admin paneli, değiştirilebilir (embed.go sayesinde kendi UI'ını gömebilirsin)

Devamında:
- SQLite detayları → [`02-sqlite.md`](./02-sqlite.md)
- Feature ekleme rehberi → [`03-ozellik-ekleme.md`](./03-ozellik-ekleme.md)
- UI değiştirme (Next.js) → [`04-ui-degistirme-nextjs.md`](./04-ui-degistirme-nextjs.md)
"