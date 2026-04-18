"# 03 — Özellik Ekleme Rehberi

> \"Ben X feature'ını eklemek istiyorum, nereye, nasıl yazmalıyım ki var olan dosyalara dokunmayayım?\"

Kısa cevap: **%90 durumda sadece `examples/base/main.go` + (opsiyonel) yeni bir `plugins/<adın>/` paketi yeterli**. Core'a hiç dokunmadan her işi görebilirsin.

Bu dokümanda en yaygın senaryolarla, adım adım örneklerle, hangi hook'u / hangi helper'ı kullanacağını anlatıyorum.

---

## İçindekiler

1. [Yeni REST endpoint ekleme](#1-yeni-rest-endpoint-ekleme)
2. [Record oluşturma/güncellenme sırasında iş mantığı (hook)](#2-record-hookları--business-logic)
3. [Auth akışına hook takma (2FA, custom validation)](#3-auth-akışına-hook-takma)
4. [E-posta template'ini özelleştirme](#4-e-posta-templateini-özelleştirme)
5. [Scheduled task / Cron job](#5-scheduled-task--cron-job)
6. [Custom CLI komutu](#6-custom-cli-komutu)
7. [3rd-party entegrasyon (Stripe, Resend, OpenAI…)](#7-3rd-party-entegrasyon)
8. [Yeniden kullanılabilir plugin yazma](#8-yeniden-kullanılabilir-plugin-yazma)
9. [Middleware ekleme (route'a özel veya global)](#9-middleware-ekleme)
10. [Realtime (SSE) kendi kanalın](#10-realtime-sse-kendi-kanalın)
11. [Tam bir \"blog + yorum\" senaryosu örneği](#11-tam-örnek--blog--yorum-senaryosu)
12. [JS ile yapma (pb_hooks) — Go yazmak istemiyorsan](#12-js-ile-yapma-pb_hooks)

---

## 0. Altın Kural: Tüm Hook'ların Listesi

`core/app.go`'da 83 tane `OnXxx` method'u var. En sık kullanılanları:

### Request lifecycle (HTTP ile gelen istek)
- `OnRecordsListRequest` — `/api/collections/:name/records` GET
- `OnRecordViewRequest` — tek record GET
- `OnRecordCreateRequest` — record POST
- `OnRecordUpdateRequest` — record PATCH
- `OnRecordDeleteRequest` — record DELETE
- `OnRecordAuthRequest` / `OnRecordAuthWithPasswordRequest` / `OnRecordAuthWithOAuth2Request` / `OnRecordAuthWithOTPRequest` — login akışları
- `OnRecordAuthRefreshRequest` — token yenileme
- `OnCollectionCreateRequest` / `OnCollectionUpdateRequest` / `OnCollectionDeleteRequest`
- `OnFileDownloadRequest` / `OnFileTokenRequest`
- `OnBatchRequest` — batch endpoint
- `OnRealtimeConnectRequest` / `OnRealtimeSubscribeRequest` / `OnRealtimeMessageSend`

### Model lifecycle (DB'ye yazılma — request'siz de çağrılır, örn. migration'dan)
- `OnModelCreate` / `OnModelCreateExecute` / `OnModelAfterCreateSuccess` / `OnModelAfterCreateError`
- `OnModelUpdate` / `OnModelUpdateExecute` / `OnModelAfterUpdateSuccess` / `OnModelAfterUpdateError`
- `OnModelDelete` / `OnModelDeleteExecute` / `OnModelAfterDeleteSuccess` / `OnModelAfterDeleteError`
- `OnModelValidate`
- `OnRecord*` muadilleri (sadece Record'lar için)
- `OnCollection*` muadilleri (sadece Collection'lar için)

### Mailer
- `OnMailerSend` (her mail gönderimi)
- `OnMailerRecordPasswordResetSend`
- `OnMailerRecordVerificationSend`
- `OnMailerRecordEmailChangeSend`
- `OnMailerRecordOTPSend`
- `OnMailerRecordAuthAlertSend`

### Lifecycle
- `OnBootstrap` — uygulama başlarken (DB open'dan sonra)
- `OnServe` — HTTP server başlamadan önce (burada route tanımla!)
- `OnTerminate` — shutdown

### Backup
- `OnBackupCreate` / `OnBackupRestore`

---

### Hook çağırma şablonu

```go
app.OnXxx().BindFunc(func(e *core.XxxEvent) error {
    // işini yap
    return e.Next()   // <-- MUTLAKA. Next'i çağırmazsan sonraki handler çalışmaz.
})
```

Veya priority'li + ID'li versiyon:

```go
app.OnServe().Bind(&hook.Handler[*core.ServeEvent]{
    Id:       \"my-plugin-static-route\",
    Priority: 999,   // yüksek priority = sonra çalışır
    Func: func(e *core.ServeEvent) error {
        e.Router.GET(\"/...\", ...)
        return e.Next()
    },
})
```

---

## 1. Yeni REST endpoint ekleme

**Senaryo:** `/api/hello?name=kanka` → `Hello, kanka!`

**Nereye yaz:** `examples/base/main.go`

```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    se.Router.GET(\"/api/hello\", func(re *core.RequestEvent) error {
        name := re.Request.URL.Query().Get(\"name\")
        if name == \"\" {
            name = \"dünya\"
        }
        return re.JSON(200, map[string]string{
            \"message\": \"Hello, \" + name + \"!\",
        })
    })
    return se.Next()
})
```

### Route group + middleware

```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    // auth gerektiren grup
    g := se.Router.Group(\"/api/admin-only\")
    g.Bind(apis.RequireSuperuserAuth())

    g.GET(\"/stats\", func(re *core.RequestEvent) error {
        count, _ := re.App.CountRecords(\"users\")
        return re.JSON(200, map[string]any{\"users\": count})
    })

    return se.Next()
})
```

Kullanabileceğin hazır middleware'ler (`apis/` paketi içinde):
- `apis.RequireAuth(\"users\")` — belirtilen koleksiyonun auth'u
- `apis.RequireSuperuserAuth()` — admin
- `apis.RequireSuperuserOrOwnerAuth(...)` — admin veya kaynak sahibi
- `apis.RequireGuestOnly()` — sadece giriş yapmamış
- `apis.Gzip()`, `apis.BodyLimit(...)`, vs.

### Request body parse + validation

```go
se.Router.POST(\"/api/my-action\", func(re *core.RequestEvent) error {
    var payload struct {
        Title string `json:\"title\" validate:\"required\"`
        Body  string `json:\"body\"`
    }
    if err := re.BindBody(&payload); err != nil {
        return apis.NewBadRequestError(\"Invalid payload\", err)
    }
    // ... iş mantığı
    return re.JSON(200, payload)
})
```

---

## 2. Record hook'ları — business logic

**Senaryo 1:** Yeni `posts` record'u oluşturulmadan önce `slug` alanını otomatik doldur.

```go
app.OnRecordCreate(\"posts\").BindFunc(func(e *core.RecordEvent) error {
    if e.Record.GetString(\"slug\") == \"\" {
        title := e.Record.GetString(\"title\")
        e.Record.Set(\"slug\", inflector.Snakecase(title))
    }
    return e.Next()
})
```

> `OnRecordCreate(\"posts\")` → **sadece `posts` koleksiyonu için filtreli**. Boş bırakırsan tüm koleksiyonlar için tetiklenir: `app.OnRecordCreate()`.

**Senaryo 2:** Record update'inde \"başlık değişemez\" kuralı.

```go
app.OnRecordUpdateRequest(\"posts\").BindFunc(func(e *core.RecordRequestEvent) error {
    // Original değer DB'den okunur
    original, _ := e.App.FindRecordById(\"posts\", e.Record.Id)
    if original.GetString(\"title\") != e.Record.GetString(\"title\") {
        return apis.NewBadRequestError(\"Title cannot be changed\", nil)
    }
    return e.Next()
})
```

**Senaryo 3:** Record silindikten sonra Slack'e mesaj gönder.

```go
app.OnRecordAfterDeleteSuccess(\"orders\").BindFunc(func(e *core.RecordEvent) error {
    go notifySlack(\"Order deleted: \" + e.Record.Id)
    return e.Next()
})
```

### Hook'lar arası fark:

| Hook | Ne zaman |
|---|---|
| `OnRecordCreate` | Oluşturma **başlamak üzere** (validasyon öncesi ve DB yazımı dahil kapsar) |
| `OnRecordCreateExecute` | Sadece DB insert anı — validasyon geçti, yazım yapılacak |
| `OnRecordAfterCreateSuccess` | DB insert **başarılı** oldu (transaction commit ettiyse) |
| `OnRecordAfterCreateError` | Insert başarısız oldu |
| `OnRecordCreateRequest` | **HTTP request** ile create geldiğinde (API dışı insert'lerde tetiklenmez) |

Business kuralı: genelde **`OnRecordCreate`** yeterli.
\"Event dışarı göndermek (webhook, queue, mail)\" için: **`OnRecordAfterCreateSuccess`** tercih et ki başarısız insert'te gönderim olmasın.

---

## 3. Auth akışına hook takma

**Senaryo 1:** Login sonrası son login zamanını kaydet.

```go
app.OnRecordAuthRequest(\"users\").BindFunc(func(e *core.RecordAuthRequestEvent) error {
    if err := e.Next(); err != nil {
        return err
    }
    // auth başarılı olduysa buraya geldik
    e.Record.Set(\"lastLogin\", time.Now())
    return e.App.SaveNoValidate(e.Record)
})
```

**Senaryo 2:** Belirli domain'den email ile login yasak.

```go
app.OnRecordAuthWithPasswordRequest(\"users\").BindFunc(func(e *core.RecordAuthWithPasswordRequestEvent) error {
    if strings.HasSuffix(e.Identity, \"@blocked.com\") {
        return apis.NewBadRequestError(\"This email domain is not allowed\", nil)
    }
    return e.Next()
})
```

**Senaryo 3:** Kaydolunca otomatik bir \"welcome\" record oluştur.

```go
app.OnRecordAfterCreateSuccess(\"users\").BindFunc(func(e *core.RecordEvent) error {
    settings, _ := e.App.FindCollectionByNameOrId(\"user_settings\")
    row := core.NewRecord(settings)
    row.Set(\"user\", e.Record.Id)
    row.Set(\"theme\", \"light\")
    return e.App.Save(row)
})
```

---

## 4. E-posta template'ini özelleştirme

`mails/templates/` dosyalarını değiştirmek upstream conflict yapar. Yerine:

```go
app.OnMailerRecordPasswordResetSend().BindFunc(func(e *core.MailerRecordEvent) error {
    // e.Meta içinde token var:
    token := e.Meta[\"token\"].(string)
    resetURL := fmt.Sprintf(\"https://myapp.com/reset?token=%s\", token)

    e.Message.Subject = \"Şifreni sıfırla\"
    e.Message.HTML = fmt.Sprintf(`
        <h1>Merhaba %s</h1>
        <p>Şifreni sıfırlamak için <a href=\"%s\">tıkla</a>.</p>
    `, e.Record.GetString(\"name\"), resetURL)
    e.Message.Text = \"Şifre sıfırlama: \" + resetURL

    return e.Next()
})
```

Tüm mail'lere ortak müdahale için `OnMailerSend`.

---

## 5. Scheduled task / Cron job

PocketBase'in dahili cron'u var (`tools/cron`):

```go
app.OnBootstrap().BindFunc(func(e *core.BootstrapEvent) error {
    if err := e.Next(); err != nil {
        return err
    }
    e.App.Cron().MustAdd(\"daily-cleanup\", \"0 3 * * *\", func() {
        e.App.Logger().Info(\"Running daily cleanup\")
        // eski log'ları sil, cache temizle, vs.
    })
    return nil
})
```

Cron expression formatı standart (dakika saat gün ay gün-of-week). `@every 1h`, `@daily` kısayolları da var.

---

## 6. Custom CLI komutu

`cmd/` klasörüne DOKUNMA. `main.go`'ya ekle:

```go
import \"github.com/spf13/cobra\"

app.RootCmd.AddCommand(&cobra.Command{
    Use:   \"seed\",
    Short: \"Seed demo data\",
    Run: func(cmd *cobra.Command, args []string) {
        if err := app.Bootstrap(); err != nil {
            log.Fatal(err)
        }
        col, _ := app.FindCollectionByNameOrId(\"posts\")
        for i := 0; i < 10; i++ {
            r := core.NewRecord(col)
            r.Set(\"title\", fmt.Sprintf(\"Post #%d\", i))
            app.Save(r)
        }
    },
})
```

Çalıştır: `./base seed`

---

## 7. 3rd-party entegrasyon

**Senaryo:** Kullanıcı kaydolunca Resend üzerinden welcome mail gönder.

```go
import \"github.com/resend/resend-go/v2\"

client := resend.NewClient(os.Getenv(\"RESEND_API_KEY\"))

app.OnRecordAfterCreateSuccess(\"users\").BindFunc(func(e *core.RecordEvent) error {
    go func() {
        _, err := client.Emails.Send(&resend.SendEmailRequest{
            From:    \"hello@myapp.com\",
            To:      []string{e.Record.GetString(\"email\")},
            Subject: \"Hoş geldin!\",
            Html:    \"<h1>Teşekkürler kaydolduğun için</h1>\",
        })
        if err != nil {
            e.App.Logger().Error(\"resend failed\", \"error\", err)
        }
    }()
    return e.Next()
})
```

Genel pattern:
1. Gerekli SDK'yı `go get` ile ekle (go.mod güncellenir)
2. Secret'ı env'den oku (`os.Getenv(...)`)
3. İlgili hook'tan tetikle
4. Uzun süren işleri `go func(){ ... }()` ile goroutine'e at — hook'u bloklama

---

## 8. Yeniden kullanılabilir plugin yazma

Eğer bir feature'ı ileride başka PocketBase projende de kullanacaksan **plugin yap**. Pattern şu:

```
/app/plugins/myplugin/
├── myplugin.go
├── myplugin_test.go
└── README.md
```

**`plugins/myplugin/myplugin.go`**

```go
package myplugin

import (
    \"github.com/pocketbase/pocketbase/core\"
)

type Config struct {
    APIKey string
    Enable bool
}

func MustRegister(app core.App, config Config) {
    if err := Register(app, config); err != nil {
        panic(err)
    }
}

func Register(app core.App, config Config) error {
    if !config.Enable {
        return nil
    }

    p := &plugin{app: app, config: config}

    app.OnBootstrap().BindFunc(p.onBootstrap)
    app.OnServe().BindFunc(p.onServe)
    app.OnRecordAfterCreateSuccess(\"users\").BindFunc(p.onUserCreated)

    return nil
}

type plugin struct {
    app    core.App
    config Config
}

func (p *plugin) onBootstrap(e *core.BootstrapEvent) error {
    p.app.Logger().Info(\"myplugin ready\", \"enabled\", p.config.Enable)
    return e.Next()
}

func (p *plugin) onServe(e *core.ServeEvent) error {
    e.Router.GET(\"/api/myplugin/status\", func(re *core.RequestEvent) error {
        return re.JSON(200, map[string]any{\"ok\": true})
    })
    return e.Next()
}

func (p *plugin) onUserCreated(e *core.RecordEvent) error {
    // …
    return e.Next()
}
```

**`examples/base/main.go`'ya ekle:**

```go
import \"github.com/pocketbase/pocketbase/plugins/myplugin\"

// main içinde
myplugin.MustRegister(app, myplugin.Config{
    APIKey: os.Getenv(\"MYPLUGIN_KEY\"),
    Enable: true,
})
```

---

## 9. Middleware ekleme

**Global:**
```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    se.Router.Bind(func(next hook.Handler[*core.RequestEvent]) hook.Handler[*core.RequestEvent] {
        return &hook.Handler[*core.RequestEvent]{
            Func: func(re *core.RequestEvent) error {
                re.Response.Header().Set(\"X-My-Header\", \"yes\")
                return next.Func(re)
            },
        }
    })
    return se.Next()
})
```

Pratikte **`apis.CORS`, `apis.Gzip`, `apis.BodyLimit`** gibi hazır middleware'lerin pattern'ini `apis/middlewares*.go` dosyalarından bakıp benzer yapı kurabilirsin.

---

## 10. Realtime (SSE) kendi kanalın

PocketBase'in kendi realtime subscription'ı var (`/api/realtime`). Broadcast için:

```go
app.OnRecordAfterUpdateSuccess(\"orders\").BindFunc(func(e *core.RecordEvent) error {
    // kendi mesajını subscribers'a yolla
    message := subscriptions.Message{
        Name: \"order-updated\",
        Data: []byte(fmt.Sprintf(`{\"id\":\"%s\",\"status\":\"%s\"}`, e.Record.Id, e.Record.GetString(\"status\"))),
    }
    for _, client := range e.App.SubscriptionsBroker().Clients() {
        client.Send(message)
    }
    return e.Next()
})
```

Frontend'de (PocketBase js-sdk):
```javascript
pb.collection(\"orders\").subscribe(\"*\", (e) => {
    console.log(e.action, e.record)
})
```

---

## 11. Tam örnek — \"Blog + yorum\" senaryosu

**Gereksinimler:**
1. `posts` koleksiyonunda `slug` otomatik üretilsin
2. `comments` koleksiyonunda spam kelimesi varsa reddedilsin
3. Yorum eklenince post sahibine mail git
4. Admin panelde görünmese bile API'den \"popular posts\" endpoint'i olsun

**`examples/base/main.go`:**

```go
// --- 1. slug ---
app.OnRecordCreate(\"posts\").BindFunc(func(e *core.RecordEvent) error {
    if e.Record.GetString(\"slug\") == \"\" {
        e.Record.Set(\"slug\", slugify(e.Record.GetString(\"title\")))
    }
    return e.Next()
})

// --- 2. spam filter ---
bannedWords := []string{\"viagra\", \"casino\"}
app.OnRecordCreate(\"comments\").BindFunc(func(e *core.RecordEvent) error {
    body := strings.ToLower(e.Record.GetString(\"body\"))
    for _, w := range bannedWords {
        if strings.Contains(body, w) {
            return apis.NewBadRequestError(\"Spam detected\", nil)
        }
    }
    return e.Next()
})

// --- 3. comment notify mail ---
app.OnRecordAfterCreateSuccess(\"comments\").BindFunc(func(e *core.RecordEvent) error {
    postID := e.Record.GetString(\"post\")
    post, err := e.App.FindRecordById(\"posts\", postID)
    if err != nil { return e.Next() }

    authorID := post.GetString(\"author\")
    author, err := e.App.FindRecordById(\"users\", authorID)
    if err != nil { return e.Next() }

    go e.App.NewMailClient().Send(&mailer.Message{
        From:    mail.Address{Name: \"Blog\", Address: \"noreply@myblog.com\"},
        To:      []mail.Address{{Address: author.GetString(\"email\")}},
        Subject: \"Yeni yorum: \" + post.GetString(\"title\"),
        HTML:    fmt.Sprintf(\"<p>Biri yorum yaptı: %s</p>\", e.Record.GetString(\"body\")),
    })
    return e.Next()
})

// --- 4. popular posts endpoint ---
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    se.Router.GET(\"/api/popular-posts\", func(re *core.RequestEvent) error {
        records, err := re.App.FindRecordsByFilter(
            \"posts\",
            \"views > 100\",
            \"-views\",
            10, 0,
        )
        if err != nil {
            return apis.NewInternalError(\"Query failed\", err)
        }
        return re.JSON(200, records)
    })
    return se.Next()
})
```

Bu **core'a sıfır dokunarak** yazıldı. Upstream güncellemesi geldiğinde `main.go` dışı her şey değişmemiş olur.

---

## 12. JS ile yapma (pb_hooks)

Go yazmak istemiyorsan — `plugins/jsvm` zaten aktif. `pb_hooks/` klasörüne `.js` dosyası at:

```
pb_data/../pb_hooks/
└── my_hooks.pb.js
```

```javascript
// pb_hooks/my_hooks.pb.js
onRecordCreate((e) => {
    if (!e.record.get(\"slug\")) {
        e.record.set(\"slug\", slugify(e.record.get(\"title\")))
    }
    e.next()
}, \"posts\")

routerAdd(\"GET\", \"/api/popular-posts\", (e) => {
    const records = e.app.findRecordsByFilter(\"posts\", \"views > 100\", \"-views\", 10, 0)
    return e.json(200, records)
})

function slugify(s) {
    return s.toLowerCase().replace(/[^a-z0-9]+/g, \"-\").replace(/^-|-$/g, \"\")
}
```

Dosya değişikliği otomatik hot-reload olur (`--hooksWatch=true` default). Type tamamlama için `types.d.ts` otomatik üretiliyor.

> **Trade-off:** JS hook'ları `goja` runtime'ında çalışır — Go kadar hızlı değil (~20× yavaş). Hot-path (ör. /api/list-all her request'te) için Go tercih et. Rare-event (kullanıcı kaydı sonrası welcome mail) için JS hiç sorun değil.

---

## 13. Karar özeti

| Sorum | Cevap |
|---|---|
| Core/apis/forms/mails/migrations/tools'a dokunmak gerekir mi? | **Neredeyse hiç** — %99 durum için hook + plugin yeter |
| Yeni feature nereye? | `examples/base/main.go` (küçük) veya `plugins/<adın>/` (büyük/reusable) |
| Hook mu, endpoint mi? | Business kuralı → hook; yeni URL → `OnServe` içinde router |
| Go mu, JS mi? | Performans kritik ise Go, rare-event ise JS |
| Hook zinciri nasıl çalışır? | `e.Next()` ile sonraki handler'a devret; çağırmazsan zincir kesilir |
| `e.App` ile `app` (dış scope) farkı? | Transaction içinde `e.App` → tx-scoped. Dış `app` direk DB'ye yazar → tx'e dahil olmaz |

Sırada → [`04-ui-degistirme-nextjs.md`](./04-ui-degistirme-nextjs.md)
"