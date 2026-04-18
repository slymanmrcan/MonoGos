"# PocketBase Fork — Türkçe Dokümantasyon

Bu klasör, elindeki PocketBase fork'unun **iç yapısını**, **hangi dosyaların \"core\" hangilerinin \"eklenti noktası\"** olduğunu ve **kendi feature'larını / kendi UI'ını nasıl entegre edebileceğini** hiç bilmeyen birinin bile anlayabileceği şekilde anlatır.

PocketBase; Go ile yazılmış, içinde **gömülü SQLite veritabanı**, **REST API**, **realtime**, **dosya yönetimi**, **auth** ve **Svelte ile yazılmış admin paneli** olan tek-binary bir backend framework'üdür. \"Firebase'in self-hosted / single-file hali\" diye düşünebilirsin.

---

## İçindekiler

| # | Doküman | Ne bulacaksın |
|---|---|---|
| 01 | [`01-klasor-yapisi.md`](./01-klasor-yapisi.md) | Tüm klasörlerin rolü, hangisi core hangisi eklenti, bağımlılık akışı |
| 02 | [`02-sqlite.md`](./02-sqlite.md) | SQLite nasıl embed ediliyor, `data.db` vs `auxiliary.db`, WAL mod, pragma'lar, migration sistemi, performans ipuçları |
| 03 | [`03-ozellik-ekleme.md`](./03-ozellik-ekleme.md) | Kendi feature'ını nereye ekleyeceksin, hangi hook'u kullanacaksın, var olan dosyalara dokunmadan nasıl iş görürsün |
| 04 | [`04-ui-degistirme-nextjs.md`](./04-ui-degistirme-nextjs.md) | Mevcut Svelte admin panelini **Next.js projenle** nasıl değiştirirsin (adım adım, 3 farklı strateji) |

---

## Hızlı başlangıç (5 dakikada PocketBase çalıştır)

```bash
# 1. Go 1.25+ gerekiyor (go.mod'da belirtilmiş)
cd /app/examples/base

# 2. Build
go build

# 3. Çalıştır
./base serve
# Output:
# ├─ REST API:  http://127.0.0.1:8090/api/
# └─ Dashboard: http://127.0.0.1:8090/_/
```

İlk çalıştırmada console'a bir `installer URL` basar (`/_/#/pbinstal/...`) — tarayıcıda açıp superuser (admin) hesabını oluşturursun. Bundan sonra `pb_data/` klasöründe `data.db`, `auxiliary.db` ve upload edilen dosyalar saklanır.

---

## Dosya yapısına hızlı bakış

```
/app
├── pocketbase.go           ← PocketBase struct + New() facade
├── examples/base/main.go   ← Binary giriş noktası (plugin register et)
├── core/                   ← Domain model, DB, event bus, 83+ hook
├── apis/                   ← HTTP handler'lar (REST API)
├── forms/                  ← Request DTO + validation
├── mails/                  ← Mail template'leri
├── migrations/             ← Sistem tablolarının migration'ları
├── cmd/                    ← CLI komutları (serve, superuser)
├── plugins/                ← Opsiyonel eklentiler (jsvm, migratecmd, ghupdate)
├── tools/                  ← Altyapı kütüphaneleri (hook, router, mailer, cron, …)
├── tests/                  ← Test utilities + fixture data
├── ui/                     ← Svelte admin paneli (go:embed ile binary'e gömülür)
└── docs/                   ← ← BURASI (Türkçe dokümantasyon)
```

Detay için → [`01-klasor-yapisi.md`](./01-klasor-yapisi.md)

---

## \"Ben ne yapmak istiyorum?\" → Hangi dokümanı oku?

| İhtiyacın | Oku |
|---|---|
| Klasörleri ve ilişkileri anlamak | 01 |
| SQLite'ı merak ediyorum / migration yazacağım | 02 |
| Yeni endpoint / business logic / CLI komutu eklemek | 03 |
| Mevcut admin UI'yi Next.js ile değiştirmek | 04 |
| Yeni 3rd-party servis entegrasyonu (Stripe, Resend, …) | 03 (hook pattern kısmı) |

---

## Önemli notlar

- **Fork'u güncel tutmak istiyorsan** → mümkün olduğunca `core/`, `apis/`, `forms/`, `mails/`, `migrations/`, `tools/` **dizinlerine DOKUNMA**. Tüm feature'larını `examples/base/main.go` + yeni bir `plugins/<adın>/` paketinde yaz. Böylece `git pull upstream master` yaptığında conflict yemezsin.
- **SDK uyumluluğu** → `/api/collections/...` gibi endpoint'lerin davranışını değiştirme. Bunlar js-sdk ve dart-sdk ile konuşuyor; kırdığında tüm SDK'lar bozulur.
- **UI** → `ui/dist/` Go `//go:embed` ile binary'e gömülüyor. Yeni UI istiyorsan ya `ui/dist/`'i kendi build'inle değiştir ya da `/_/` route'unu kendi UI'ına yönlendir. Detay → 04.
"