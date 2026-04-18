"# 02 — SQLite — Nasıl Çalışıyor, Nerede Duruyor, Nasıl Yönetilir

PocketBase'in en önemli özelliklerinden biri: **içinde SQLite var ve dışarıdan hiçbir DB kurulumu gerekmez**. Bu dokümanda:

1. Hangi SQLite driver'ı kullanılıyor
2. DB dosyaları nerede duruyor, kaç tane var
3. Hangi PRAGMA'lar açık ve neden
4. Concurrent / Nonconcurrent DB nedir
5. `_collections` ve diğer sistem tabloları
6. Migration sistemi (sistem migration'ları vs. kullanıcı migration'ları)
7. Transaction (Tx) & lock retry
8. Backup / restore
9. Performans, tuning, common pitfalls

---

## 1. Hangi driver? — `modernc.org/sqlite`

`go.mod`:

```
modernc.org/sqlite v1.48.2
```

Bu, **saf-Go (CGO'suz) SQLite** driver'ı. C library'sini Go'ya transpile ederek (`modernc.org/libc`) çalışıyor. Faydası:

- ✅ **CGO gerekmiyor** → `CGO_ENABLED=0 go build` ile statik binary çıkıyor, her platform için kolay cross-compile
- ✅ **Saf Go** → garbage collector uyumlu, goroutine-safe handle yönetimi
- ⚠️ **Performans** → `mattn/go-sqlite3` (CGO) ile karşılaştırıldığında biraz daha yavaş olabilir; pratikte çoğu workload için fark yok

Alternatif driver istersen `no_default_driver` build tag'i var:

```bash
go build -tags no_default_driver
```

Bu flag açıkken `core/db_connect.go` devreye girmez, `core/db_connect_nodefaultdriver.go` devreye girer → kendi `DBConnect` fonksiyonunu `PocketBase.New(Config{ DBConnect: ... })` ile geçebilirsin. Örneğin `mattn/go-sqlite3` ya da `libSQL` / `rqlite` bile bağlayabilirsin.

---

## 2. DB dosyaları — `pb_data/` klasörü

PocketBase çalıştığında **`pb_data/`** klasörünü (flag: `--dir`) oluşturur. İçinde:

```
pb_data/
├── data.db                        ← Asıl veritabanı (koleksiyonlar + record'lar)
├── data.db-shm                    ← SQLite WAL shared memory
├── data.db-wal                    ← SQLite Write-Ahead Log
├── auxiliary.db                   ← İkincil DB (log, cache, kısa ömürlü tablolar)
├── auxiliary.db-shm
├── auxiliary.db-wal
├── storage/                       ← Upload edilen dosyalar (local mode)
│   └── <collectionId>/<recordId>/<filename>
├── backups/                       ← `app.CreateBackup()` çıktıları (zip)
├── .autocert_cache/               ← Let's Encrypt sertifika cache (HTTPS kullanıyorsan)
└── types.d.ts                     ← pb_hooks için TypeScript type'ları (jsvm tarafından üretilir)
```

### `data.db` vs `auxiliary.db` — Neden iki tane?

**İki ayrı SQLite dosyası** var ve birbirinden bağımsız çalışıyorlar. Sebep: **yazma yükünü bölmek ve büyük DB'de bile loglama/metering'in asıl veriyi yavaşlatmaması**.

| DB | İçinde ne var | Kim yazar |
|---|---|---|
| **`data.db`** | `_collections`, `_superusers`, `users` (ve senin koleksiyonların), `_externalAuths`, `_authOrigins`, `_mfas`, `_otps`, `_params` | Uygulama (API, admin, kodun) |
| **`auxiliary.db`** | `_logs` (slog handler), `_backups` metadata, `_migrations_aux` | Logger goroutine, backup, rate-limit sayaçları |

Kodda kullanım:

```go
app.DB()                 // data.db (regular, safer)
app.ConcurrentDB()       // data.db (çoklu okuma bağlantısı)
app.NonconcurrentDB()    // data.db (tek yazma bağlantısı)

app.AuxDB()              // auxiliary.db
app.AuxConcurrentDB()
app.AuxNonconcurrentDB()
```

> Ayrıntı: `modernc.org/sqlite` çoklu concurrent writer desteklemez. PocketBase çift bağlantı açar — biri **read-only** gibi kullanılır (çok sayıda goroutine paralel okuyabilir), diğeri **tek yazıcı**. `baseLockRetry(...)` sarmalı SQLITE_BUSY hatalarını (`database is locked`) yeniden deneyerek maskelemeye çalışır.

---

## 3. Hangi PRAGMA'lar açık? — `core/db_connect.go`

```go
pragmas := \"?_pragma=\" + strings.Join([]string{
    \"busy_timeout(10000)\",         // SQLITE_BUSY olursa 10 saniye bekle
    \"journal_mode(WAL)\",           // Write-Ahead Log — paralel okuma için kritik
    \"journal_size_limit(200000000)\", // WAL dosyası max 200 MB, sonra truncate
    \"synchronous(NORMAL)\",         // FULL değil NORMAL — disk fsync'i azalt, WAL ile güvenli
    \"foreign_keys(ON)\",            // FK constraint'leri uygula
    \"temp_store(MEMORY)\",          // Geçici tablolar RAM'de
    \"cache_size(-32000)\",          // 32 MB page cache (- = KB cinsinden negatif)
}, \"&_pragma=\")
```

### Neden bu ayarlar?

| PRAGMA | Değer | Neden |
|---|---|---|
| `busy_timeout` | 10000 ms | Tek writer olduğu için lock çakışmasını kendi retry'ımızla + SQLite retry'ıyla yumuşatıyoruz |
| `journal_mode` | **WAL** | Okuyucular yazıcıyı bloklamaz. PocketBase'in paralel read performansının kilit taşı. |
| `synchronous` | **NORMAL** | WAL'da NORMAL = yazma sonunda fsync yapmaz (checkpoint'te yapar). FULL'dan 2-3× hızlı, WAL sayesinde crash-safe. |
| `foreign_keys` | ON | PocketBase relation field'ları için FK oluşturuyor; referential integrity şart. |
| `temp_store` | MEMORY | Geçici tablolar (sort, subquery materialization) RAM'de → hızlı. |
| `cache_size` | -32000 (32 MB) | Büyük DB'de sık erişilen sayfalar RAM'de kalsın. |

### WAL checkpoint

PocketBase arka planda periyodik olarak `PRAGMA wal_checkpoint(PASSIVE)` çalıştırır (hem data.db hem auxiliary.db için — `core/base.go:1367` civarı log var). Bu, WAL dosyasının büyümesini engeller.

---

## 4. `dbx` — Query Builder

SQLite driver'ının üstünde **[`github.com/pocketbase/dbx`](https://pkg.go.dev/github.com/pocketbase/dbx)** kullanılıyor — PocketBase ekibinin maintain ettiği hafif query builder.

```go
// SELECT örneği
var records []*core.Record
err := app.DB().
    NewQuery(\"SELECT * FROM users WHERE active = {:active}\").
    Bind(dbx.Params{\"active\": true}).
    All(&records)

// Daha high-level: FindRecordsByFilter
records, err := app.FindRecordsByFilter(
    \"posts\",
    \"status = 'published' && created > @today\",
    \"-created\",          // sort
    50, 0,               // limit, offset
)
```

`core/record_query.go`, `core/collection_query.go` bu builder'ı sarmalayan yüksek-seviye API'lar sağlıyor.

---

## 5. Sistem Tabloları — `data.db` içinde

PocketBase çalıştığında otomatik oluşan tablolar:

| Tablo | İş |
|---|---|
| `_collections` | Koleksiyon tanımları (şema) |
| `_superusers` | Admin/superuser hesapları (önceden ayrı \"_admins\" idi) |
| `_params` | App settings (SMTP, meta, S3 vs. JSON olarak) |
| `_externalAuths` | OAuth2 ile bağlı hesaplar (Google, GitHub, …) |
| `_authOrigins` | Hangi cihaz/IP ile login var |
| `_mfas` | Multi-factor auth kayıtları |
| `_otps` | OTP code'ları (TTL'lı) |
| `_migrations` | Çalıştırılmış migration'lar (sistem) |

Senin koleksiyonların ise **`_collections` tablosuna kayıt** olarak düşer **ve aynı zamanda ayrı birer SQL tablosu olarak** oluşturulur.

### Örnek: \"posts\" koleksiyonu oluşturunca

1. `_collections` tablosuna bir satır eklenir (JSON olarak şema)
2. `posts` adında gerçek bir SQL tablosu oluşturulur:
   ```sql
   CREATE TABLE posts (
       id       TEXT PRIMARY KEY,
       title    TEXT,
       body     TEXT,
       author   TEXT,                    -- relation field → users(id) FK
       created  TEXT,
       updated  TEXT
   );
   CREATE INDEX `idx_posts_author` ON posts(author);
   ```
3. Koleksiyonun `indexes` ayarına göre ek CREATE INDEX statement'ları çalıştırılır

Bu senkronizasyonu yapan dosya: **`core/collection_record_table_sync.go`**. Koleksiyon şeması değişince otomatik `ALTER TABLE` üretir.

---

## 6. Migration Sistemi — İki Tür

PocketBase'de iki ayrı migration zinciri var:

### 6.1 Sistem migration'ları (`/app/migrations/`)

- Go ile yazılmış
- `migrations/*.go` dosyaları
- Binary'ye **derleme zamanında gömülüdür**
- Sistem tablolarını (örn. `_superusers`'ın 2024'te `_admins`'ten ayrı collection'a geçişi) evrimleştirir
- Her migration unix-timestamp isimli: `1727864177_superusers_collection.go`
- Uygulanma durumu `_migrations` tablosunda
- `app.RunAllMigrations()` (apis/serve.go server start'ta çağırır) idempotent olarak çalıştırır

**Sen buraya dosya EKLEME.** Upstream pull'da karışıklık yaratır.

### 6.2 Kullanıcı migration'ları (`pb_migrations/`)

`plugins/migratecmd/` plugin'i **runtime CLI komutları** ekler:

```bash
./base migrate create add_posts_column     # yeni .js/.go dosyası üretir
./base migrate up                          # uygulanmamışları çalıştırır
./base migrate down 1                      # son 1 migration'ı geri al
./base migrate history-sync                # _migrations tablosunu klasörle eşitle
```

Migration dili seçilebilir (`examples/base/main.go`'da):

```go
migratecmd.MustRegister(app, app.RootCmd, migratecmd.Config{
    TemplateLang: migratecmd.TemplateLangJS,   // veya TemplateLangGo
    Automigrate:  true,                         // admin panelden değişiklik yapınca otomatik .js üret
    Dir:          migrationsDir,                // default: pb_migrations/
})
```

#### Örnek JS migration

```javascript
// pb_migrations/1700000000_add_posts_index.js
migrate((app) => {
    // up
    const collection = app.findCollectionByNameOrId(\"posts\")
    collection.indexes = [...collection.indexes, \"CREATE INDEX `idx_posts_status` ON `posts` (`status`)\"]
    app.save(collection)
}, (app) => {
    // down
    const collection = app.findCollectionByNameOrId(\"posts\")
    collection.indexes = collection.indexes.filter(i => !i.includes(\"idx_posts_status\"))
    app.save(collection)
})
```

#### Örnek Go migration

```go
// migrations/1700000000_add_posts_index.go (senin klasöründe, /app/migrations/ DEĞİL!)
package migrations

import (
    \"github.com/pocketbase/pocketbase/core\"
    m \"github.com/pocketbase/pocketbase/migrations\"
)

func init() {
    m.Register(func(app core.App) error {
        // up
        collection, err := app.FindCollectionByNameOrId(\"posts\")
        if err != nil { return err }
        collection.Indexes = append(collection.Indexes, \"CREATE INDEX idx_posts_status ON posts (status)\")
        return app.Save(collection)
    }, func(app core.App) error {
        // down
        collection, err := app.FindCollectionByNameOrId(\"posts\")
        if err != nil { return err }
        // filtrele ve kaydet
        return app.Save(collection)
    })
}
```

> Dikkat: Go migration'larda `package migrations` adı ve `m.Register` import'u **upstream migrations paketi** `github.com/pocketbase/pocketbase/migrations`. Kendi migrations klasörün `myapp/migrations/` gibi ayrı bir yerde olabilir ve `migratecmd.Config.Dir` ile belirtilir.

### Automigrate nedir?

Admin panelden \"yeni alan ekle / koleksiyon oluştur\" yaptığında, eğer `Automigrate: true` ise PocketBase **o değişikliği yeniden üretecek bir `.js` migration** otomatik oluşturur. Böylece dev'de yaptığın değişiklik prod'a deploy'da aynen uygulanır.

---

## 7. Transaction (Tx) & Lock Retry

### `app.RunInTransaction(...)`

```go
err := app.RunInTransaction(func(txApp core.App) error {
    collection, _ := txApp.FindCollectionByNameOrId(\"posts\")
    record := core.NewRecord(collection)
    record.Set(\"title\", \"Hello\")
    if err := txApp.Save(record); err != nil {
        return err // rollback
    }
    // başka işler…
    return nil // commit
})
```

`txApp` — transaction-scoped app. Bütün hook'lar dahil **atomic** çalışır. `return error` → rollback. Core/db_tx.go implementasyonuna bakabilirsin.

### SQLITE_BUSY retry

`core/db_retry.go` — yazma işlemi lock çakışırsa expontential backoff ile yeniden dener. Varsayılan 8 retry. Bu genelde **multiple goroutine aynı anda yazıyor** senaryosunda devreye girer.

---

## 8. Backup / Restore

`data.db` + `auxiliary.db` + `storage/` + `.autocert_cache/` → hepsi bir `zip` dosyasına toplanır.

Kod:
```go
name := fmt.Sprintf(\"pb_backup_%s.zip\", time.Now().Format(\"20060102150405\"))
err := app.CreateBackup(context.Background(), name)  // pb_data/backups/ içine yazar

// Restore — uyarı: uygulamayı restart eder
err := app.RestoreBackup(context.Background(), name)
```

**S3 entegrasyonu:** Admin settings'te \"Backups storage\" → S3 yapılandırırsan ZIP'ler oraya da yüklenir. Otomatik cron ile günlük/haftalık backup scheduling var (admin UI'dan).

REST endpoint'leri: `GET/POST /api/backups`, `POST /api/backups/:key/restore`, `GET /api/backups/:key` (download).

---

## 9. Performans Tuning & Tavsiyeler

### Yapmalı

- **WAL mode zaten açık** — müdahale etme
- **Indexlerini ekle** → Koleksiyon ayarında \"indexes\" alanından, yahut migration'dan:
  ```sql
  CREATE INDEX idx_posts_status_created ON posts(status, created)
  ```
- **Relation field'lar otomatik indexleniyor** ama kompozit index gerekirse manuel ekle
- **List endpoint'te `?filter=...&sort=...&page=...`** kullan; PocketBase server-side LIMIT/OFFSET atıyor
- Büyük `?perPage` değerlerinden kaçın (max 500 zaten)
- `VACUUM` periyodik çalışıyor admin → Settings → Logs/Backups kısmında

### Yapmamalı

- **Birden çok uygulama aynı `data.db`'ye yazmasın** — SQLite tek-writer'dır, WAL bile olsa cross-process write overlap'i problemli olur
- **Network file system üstünde `pb_data` tutma (NFS, SMB)** — SQLite lock'u bozulur. Local disk veya block storage kullan.
- **`PRAGMA synchronous=OFF`** yapma — PocketBase NORMAL'de bırakmış, WAL ile güvenli. OFF crash'te veri kaybı yapar.

### İzleme

```go
// app'te slog handler var, log'lar auxiliary.db/_logs tablosuna yazılır
// Admin UI → \"Logs\" sekmesinden query atarak görebilirsin

// kendi metriklerin için
app.Logger().Info(\"slow query\", \"duration_ms\", elapsed, \"sql\", query)
```

---

## 10. Custom DB driver kullanmak (ileri düzey)

Uzaktan SQLite-compatible (örn. `libSQL`, `rqlite`) veya farklı bir driver kullanmak istersen:

```bash
go build -tags no_default_driver
```

Sonra `main.go`'da:

```go
app := pocketbase.NewWithConfig(pocketbase.Config{
    DBConnect: func(dbPath string) (*dbx.DB, error) {
        // kendi bağlantını aç
        return dbx.Open(\"libsql\", \"file:\"+dbPath+\"?...\")
    },
})
```

**Uyarı**: PocketBase migration'ları ve `collection_record_table_sync.go` SQLite'a özgü SQL syntax kullanır (örn. `sqlite_master`, `ALTER TABLE RENAME COLUMN`). Başka DB'ye geçmek çoğu yerde yeniden yazım gerektirir. Çoğu durumda gereksizdir — önce SQLite'ı yormayı gerçekten başardığından emin ol.

---

## 11. DB dosyalarını direkt açmak

İstediğin zaman herhangi bir SQLite client ile inceleyebilirsin:

```bash
# CLI
sqlite3 pb_data/data.db

.tables                           # tabloları listele
SELECT name FROM _collections;    # koleksiyon isimleri
SELECT * FROM users LIMIT 5;
.schema posts                     # bir tablonun şeması

# GUI
# - DB Browser for SQLite (https://sqlitebrowser.org)
# - TablePlus
# - Beekeeper Studio
```

**Ama çalışan PocketBase varken** `.db` dosyasını başka process'ten **yazma** yapma. Okumak sorun değil (WAL sayesinde read-only tutuyor).

---

## 12. Özet

- **Driver:** `modernc.org/sqlite` (saf Go, CGO'suz)
- **Dosyalar:** `pb_data/data.db` + `pb_data/auxiliary.db` (+ `-wal`, `-shm`)
- **Modlar:** WAL + NORMAL synchronous — hızlı ve güvenli
- **Bağlantılar:** Concurrent (read) + Nonconcurrent (write) çift bağlantı
- **Şema:** `_collections` tablosu + ayna gerçek tablolar + otomatik `ALTER TABLE`
- **Migration:** Sistem (Go, gömülü, dokunma) + Kullanıcı (pb_migrations/, JS veya Go)
- **Tx:** `app.RunInTransaction(...)` — hook'lar dahil atomic
- **Backup:** `app.CreateBackup()` → ZIP (DB + storage + autocert)

Sırada → [`03-ozellik-ekleme.md`](./03-ozellik-ekleme.md) (feature'ını nereye koyacaksın)
"