# İlerleme Kaydı — Faz 1 (Pilot MVP)

Durum değerleri: `bekliyor` / `sürüyor` / `tamamlandı`. Her görev kapandığında: ne yapıldı, eklenen testler, açık kalan `MINOR` bulgular yazılır.

Son güncelleme: 2026-09-25 — ADR-0001 kabul edildi (K22–K27); Redis → Valkey; `packages/server` eklendi; rapor kuralı çalışma döngüsüne eklendi; M0.2 başlıyor.

---

## M0 — İskelet ve altyapı
**Taş durumu:** sürüyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M0.1 | ADR-0001: yığın kararı (Node/pnpm sürümleri, modül sistemi, lint yapılandırması, sürüm sabitleme) | architect | tamamlandı | `docs/adr/0001-stack.md` kabul edildi (2026-09-25). [D] değerleri ve TS/Prisma/Valkey doğrulama sonuçları M0.2/M0.3'te devops tarafından ADR'ye yazılır |
| M0.2 | Monorepo iskeleti: pnpm workspaces, Turborepo, ortak tsconfig (strict), ESLint, Prettier | devops | bekliyor | |
| M0.0 | `git init`, `.gitignore`, uzak depo, `main` dalı | orkestratör | tamamlandı | `origin` = https://github.com/gurcaneker/club.git (HTTPS; SSH kurum ağında engelli), `main` push edildi, upstream bağlı |
| M0.3 | Uygulama/paket iskeletleri: `apps/web`, `apps/api`, `apps/worker`, `packages/db\|shared\|templates\|server` | devops | bekliyor | M0 iskelet istisnası (K3); CJS→ESM ve @club/db import smoke testleri (K22) |
| M0.4 | `packages/shared`: env zod şema yardımcıları; `packages/db`: boş Prisma şeması + client | architect | bekliyor | |
| M0.5 | Her uygulamada env doğrulama (zod), `.env.example` (açıklamalı) | devops | bekliyor | |
| M0.6 | `docker-compose.dev.yml`: PostgreSQL, Valkey, MinIO + bucket init | devops | bekliyor | İmajlar digest ile sabit (K24, K26) |
| M0.7 | CI: lint + typecheck + unit test + build (GitHub Actions) + `renovate.json` | devops | bekliyor | Renovate gruplaması K25 |
| M0.8 | Test kapısı: iskelet testleri, `pnpm lint/typecheck/test/build`, `pnpm dev` dumanı | test-engineer | bekliyor | |
| M0.9 | İnceleme kapısı | code-reviewer (+ security-auditor: env/sır) | bekliyor | |

## M1 — Veri modeli, auth, tenant izolasyonu
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M1.1 | Prisma şeması (18 model; `trainingDays`, `websiteAllowed`, `timezone`, saklama süreleri dahil), API sözleşmesi `docs/api.md` | architect | bekliyor | |
| M1.1a | ADR-0002 tenant çözümleme, ADR-0003 cookie + CSRF | architect | bekliyor | |
| M1.1b | `packages/shared/src/state/` public API | architect | bekliyor | |
| M1.1c | Durum geçiş fonksiyonu uygulaması | backend-dev | bekliyor | |
| M1.2 | Tenant bağlamı: istek başına çözümleme + Prisma extension | backend-dev | bekliyor | |
| M1.3 | Auth: e-posta+şifre (argon2), JWT cookie (httpOnly, Secure, SameSite=Lax, Domain yok), double-submit CSRF, rol guard'ları, davet linki, giriş hız sınırı | backend-dev | bekliyor | |
| M1.3a | `EmailSender` arayüzü + `MockEmailSender` | backend-dev | bekliyor | |
| M1.3b | PLATFORM_ADMIN tenant açma CLI/seed betiği | backend-dev | bekliyor | |
| M1.4 | Seed: pilot tenant, 1 admin, 2 antrenör, 3 grup, sentetik öğrenciler | architect / backend-dev | bekliyor | |
| M1.5 | Çapraz tenant ve rol guard testleri | test-engineer | bekliyor | |
| M1.6 | İnceleme + güvenlik denetimi | code-reviewer, security-auditor | bekliyor | |

## M2 — Rıza modülü
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M2.0 | Giriş ekranı, oturum yönetimi, panel iskeleti | frontend-dev | bekliyor | İlk görev |
| M2.1 | Öğrenci CRUD + CSV içe aktarma (API) | backend-dev | bekliyor | |
| M2.2 | Öğrenci CRUD + CSV ekranı (panel) | frontend-dev | bekliyor | |
| M2.3 | Veli rıza linki (tek kullanımlık, süreli, imzalı) | backend-dev | bekliyor | |
| M2.4 | Veli formu: dört onay (`websiteAllowed` dahil), metin versiyonu, zaman, IP | frontend-dev | bekliyor | |
| M2.4a | Rıza formu hız sınırı | backend-dev | bekliyor | |
| M2.5 | Rıza geri alma (taslakları geri çekme, yayınlanmışları "kontrol edilmeli" işaretleme) + grup rıza özeti | backend-dev | bekliyor | |
| M2.6 | KVKK dışa aktarma / silme | backend-dev | bekliyor | |
| M2.7 | Testler | test-engineer | bekliyor | |
| M2.8 | İnceleme + güvenlik denetimi | code-reviewer, security-auditor | bekliyor | |

## M3 — Antrenör yükleme PWA'sı
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M3.1 | PWA manifest + service worker | frontend-dev | bekliyor | |
| M3.2 | Antrenör girişi (davet linki), yalnızca kendi grupları | frontend-dev, backend-dev | bekliyor | |
| M3.3 | Presigned S3 yükleme, çoklu dosya, ilerleme, yeniden deneme | backend-dev, frontend-dev | bekliyor | |
| M3.4 | İdman notu, rızasız çocuk uyarısı | frontend-dev | bekliyor | |
| M3.5 | `Upload` kaydı + kuyruk olayı; sunucu tarafı tip/boyut doğrulaması | backend-dev | bekliyor | |
| M3.6 | 20 fotoğraflık mobil e2e, yetki testleri | test-engineer | bekliyor | |
| M3.7 | İnceleme + güvenlik denetimi | code-reviewer, security-auditor | bekliyor | |

## M4 — İşleme hattı ve şablonlar
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M4.1 | Toplama penceresi zamanlayıcısı | pipeline-dev | bekliyor | |
| M4.2 | Kalite puanı + perceptual hash kopya eleme | pipeline-dev | bekliyor | |
| M4.3 | En iyi N kare seçimi, 4:5 kırpma | pipeline-dev | bekliyor | |
| M4.4 | Gruplama önerisi (DESIGN §6) | pipeline-dev | bekliyor | |
| M4.4a | EXIF/konum temizleme | pipeline-dev | bekliyor | |
| M4.5 | Satori şablonları (kapak, ara kapak, alt bant) | pipeline-dev | bekliyor | |
| M4.5a | Saklama işi (90 gün / 30 gün, tenant ayarı) | pipeline-dev | bekliyor | |
| M4.6 | `Draft` + `DraftItem` üretimi, `PENDING_REVIEW`, mock metin | pipeline-dev | bekliyor | |
| M4.7 | Sentetik fikstürler + testler, iki temayla snapshot | test-engineer | bekliyor | |
| M4.8 | İnceleme | code-reviewer (+ security-auditor: EXIF, S3 anahtarları) | bekliyor | |

## M5 — AI metin motoru
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M5.1 | `CaptionProvider` arayüzü, Anthropic + Mock uygulamaları | ai-caption-dev | bekliyor | |
| M5.2 | Prompt oluşturucu (ton profili, son 30 metin) | ai-caption-dev | bekliyor | |
| M5.3 | Korkuluklar: isim, emoji, açılış benzerliği, uzunluk | ai-caption-dev | bekliyor | |
| M5.4 | Yeniden üretme politikası (≤2) + uyarı bayrağı | ai-caption-dev | bekliyor | |
| M5.5 | Eval seti (20+ senaryo) + puanlama betiği | ai-caption-dev | bekliyor | |
| M5.6 | Yönetici düzeltmelerini örnek havuzuna ekleme | ai-caption-dev | bekliyor | |
| M5.7 | Korkuluk testleri (isim sızıntısı %100) | test-engineer | bekliyor | |
| M5.8 | İnceleme + güvenlik denetimi (dış API, PII) | code-reviewer, security-auditor | bekliyor | |

## M6 — Onay paneli
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M6.1 | Taslak listesi + rozet; e-posta bildirimi; `SmtpEmailSender` | frontend-dev, backend-dev | bekliyor | |
| M6.2 | Instagram görünümlü önizleme | frontend-dev | bekliyor | |
| M6.3 | Düzenle: metin, kare çıkar/sırala, kapak, yeniden üret | frontend-dev, backend-dev | bekliyor | |
| M6.4 | Birleştir/Ayır (worker yeniden düzenler) | backend-dev, pipeline-dev, frontend-dev | bekliyor | |
| M6.5 | Onayla → zamanlama; Reddet → not | backend-dev, frontend-dev | bekliyor | |
| M6.6 | Rıza ve isim-kontrol uyarıları; rızasız grupta zorunlu onay kutusu | frontend-dev, backend-dev | bekliyor | |
| M6.6a | "Kontrol edilmeli" listesi; galeri için yönetici onayı | frontend-dev, backend-dev | bekliyor | |
| M6.7 | Onaysız taslak hatırlatması; idman günü yükleme hatırlatması | pipeline-dev / backend-dev | bekliyor | |
| M6.8 | Onaysız yayın engeli testi, Birleştir/Ayır e2e, mobil Playwright | test-engineer | bekliyor | |
| M6.9 | İnceleme + güvenlik denetimi | code-reviewer, security-auditor | bekliyor | |

## M7 — Instagram entegrasyonu
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M7.1 | Meta API değerlerinin doğrulanması → `meta.config.ts` | instagram-dev | bekliyor | |
| M7.2 | `InstagramPublisher`: gerçek + mock | instagram-dev | bekliyor | |
| M7.3 | OAuth akışı, token şifreli saklama, yenileme işi | instagram-dev (+ frontend-dev: panel butonu) | bekliyor | |
| M7.4 | Tekli/carousel yayın, durum sorgulama | instagram-dev | bekliyor | |
| M7.5 | Zamanlayıcı + günlük sınır + idempotency | instagram-dev | bekliyor | |
| M7.6 | Yeniden deneme, `FAILED`, bildirim, `Post` + `AuditLog` | instagram-dev | bekliyor | |
| M7.7 | Mock modda tam akış e2e | test-engineer | bekliyor | |
| M7.7a | cloudflared tüneli (medya URL + OAuth callback) | devops | bekliyor | |
| M7.7b | API ile post silme desteğinin doğrulanması | instagram-dev | bekliyor | |
| M7.8 | Gerçek mod duman testi (manuel, kullanıcı onayıyla, tünel üzerinden) | kullanıcı + instagram-dev | bekliyor | |
| M7.9 | İnceleme + güvenlik denetimi | code-reviewer, security-auditor | bekliyor | |

## M8 — Tema sistemi ve vitrin sayfası
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M8.0 | `packages/shared/src/theme/` public API | architect | bekliyor | |
| M8.1 | Token türetme + WCAG kontrast düzeltme | frontend-dev | bekliyor | K4 kararı |
| M8.2 | Tema ayarları ekranı: logo, renk önerisi, renk seçimi | frontend-dev | bekliyor | |
| M8.3 | Vitrin: hero, tanıtım, iletişim, otomatik galeri | frontend-dev | bekliyor | |
| M8.4 | Şablonların yeni token'larla uyumu | pipeline-dev | bekliyor | |
| M8.5 | Alt alan adına göre tenant çözümleme (web) | frontend-dev | bekliyor | |
| M8.6 | Kontrast unit testleri, iki temayla vitrin snapshot | test-engineer | bekliyor | |
| M8.7 | İnceleme | code-reviewer | bekliyor | |

## M9 — Sertleştirme ve dağıtım
**Taş durumu:** bekliyor

| # | Görev | Ajan | Durum | Not |
| --- | --- | --- | --- | --- |
| M9.1 | Tam e2e senaryo | test-engineer | bekliyor | |
| M9.2 | Prod `docker-compose.yml`, Nginx, wildcard sertifika, presigned proxy | devops | bekliyor | |
| M9.3 | Yedekleme + geri yükleme betikleri | devops | bekliyor | |
| M9.4 | PII'siz loglama, sağlık kontrolleri | devops, backend-dev, pipeline-dev | bekliyor | |
| M9.5 | `docs/RUNBOOK.md` | devops | bekliyor | |
| M9.6 | Son tam inceleme + güvenlik denetimi, açık bulguların kapatılması | code-reviewer, security-auditor | bekliyor | |

---

## Açık bulgular

İnceleme/test kapılarından gelen açık bulgular. Format: `| # | Önem | Kaynak | Dosya:satır | Sorun | Sorumlu | Taş |`

| # | Önem | Kaynak | Dosya:satır | Sorun | Sorumlu | Taş |
| --- | --- | --- | --- | --- | --- | --- |
| — | — | — | — | Henüz bulgu yok | — | — |

## Kararlar

Plan aşamasında tespit edilen konular. Tüm kararlar 2026-09-25'te kullanıcı tarafından verildi; ayrıntılar DESIGN.md ve PLAN.md'ye işlendi.

| # | Konu | Karar | Durum |
| --- | --- | --- | --- |
| K1 | Alt ajanlar oturumda yüklü değil | Oturum yeniden başlatıldı; 10 alt ajanın tamamı yüklü olarak doğrulandı (2026-09-25). general-purpose yedek yolu kullanılmaz; ajan eksikse durulur ve bildirilir | kapalı |
| K2 | Git deposu / CI (C6) | Özel GitHub deposu, HTTPS remote (https://github.com/gurcaneker/club.git); SSH kullanılmaz. `main` push edildi, upstream bağlı; CI kriteri aynen kalır. Push başarısız olursa kimlik bilgisi sorunu çözülmeye çalışılmaz, durulur ve bildirilir | kapalı |
| K3 | M0 iskelet istisnası (A1) | Onay: devops yalnızca M0'da iskelet dosyaları yazar | kapalı |
| K4 | `packages/shared` mantık kodu (A2) | `state/`: API architect, uygulama backend-dev. `theme/`: API architect, uygulama frontend-dev, pipeline-dev tüketir | kapalı |
| K5 | Fikstür sahipliği (A3) | test-engineer tek sahip; pipeline-dev kullanır | kapalı |
| K6 | Rızasız çocuklu grupta yayın (C1) | Açık onay kutusu + `AuditLog`; geri almada `APPROVED`/`SCHEDULED` → `PENDING_REVIEW`; yayınlanmış postlar "kontrol edilmeli" listesine | kapalı |
| K7 | Vitrin rızası (C2) | Ayrı `websiteAllowed` alanı; rızasız çocuklu post galeriye otomatik düşmez, yönetici onaylar; rıza metni versiyonu güncellenir | kapalı |
| K8 | M7 gerçek mod testi (A4) | cloudflared tüneli; duman testi M7'de kalır | kapalı |
| K9 | E-posta (B1) | `EmailSender`, sahibi backend-dev; mock M1, SMTP M6 | kapalı |
| K10 | Docker erişimi | Erişim var | kapalı |
| K11 | Tenant çözümleme (A5) | API M1'de (ADR-0002), web M8'de | kapalı |
| K12 | Panel iskeleti (B2) | M2'nin ilk görevi, frontend-dev | kapalı |
| K13 | PLATFORM_ADMIN (B3) | MVP'de CLI/seed betiği, arayüz Faz 3 | kapalı |
| K14 | İdman günü hatırlatması (B4) | `Group.trainingDays` M1 şemasında; hatırlatma M6'da | kapalı |
| K15 | Güvenlik görevleri (B5) | EXIF M4; hız sınırı M1 (giriş) + M2 (rıza formu); httpOnly+Secure+SameSite=Lax cookie, Domain yok, double-submit CSRF (ADR-0003) | kapalı |
| K16 | Saklama süreleri (B6) | Orijinaller yayından 90 gün, reddedilenler 30 gün sonra; tenant ayarı | kapalı |
| K17 | Instagram'dan kaldırma (B7) | M7'de doğrulanır; gerekirse elle silme runbook'a | kapalı |
| K18 | Carousel sayımı (C3) | Kapak ve ara kapaklar 10 öğe sınırına dahil | kapalı |
| K19 | Durum geçişleri (C4) | DESIGN §4 geçiş tablosu | kapalı |
| K20 | Saat dilimi / toplama penceresi (C5) | Tenant saat dilimi (varsayılan Europe/Istanbul); pencere tenant başına | kapalı |
| K21 | Ek durum geçişleri ve Birleştir/Ayır düzenleme kaybı | `FAILED → PROCESSING` (işleme hatası sonrası yeniden deneme) ve `PENDING_REVIEW → PROCESSING` (Birleştir/Ayır) onaylandı. Yönetici düzenlemeleri kaybolacaksa önce uyarı + teyit; düzenlenmiş metin mümkünse korunur (DESIGN §8, PLAN M6 kabul). Saklama işi M4'te (M4.5a) | kapalı |
| K22 | ADR-0001 onayı ve doğrulama kuralları | ADR "Kabul edildi". TS: Next.js, Nest CLI/SWC ve typescript-eslint TS 6'yı resmi olarak destekliyorsa 6.x, desteklemiyorsa 5.9.x; sonuç ve kaynak ADR'ye yazılır. M0 kabulüne CJS→ESM smoke testi eklendi: api `@club/shared`'den gerçek bir fonksiyonu çağırır; `tsc --noEmit`, SWC build ve `node dist/main.js` üçü de geçmeli. Prisma ana sürümü, üretici, modül biçimi ve driver adapter doğrulanır; @club/db için CJS geri dönüşü bu sonuca göre değerlendirilir; api ve worker'dan import smoke testi yapılır. [D] farkı ya da TS/Prisma/Valkey kontrollerinden biri başarısız olursa devops durur ve bildirir | kapalı |
| K23 | S4: paylaşılan sunucu kodu | Tenant Prisma extension `packages/db/src/` altında (`prisma/` → architect, `src/` → backend-dev). Yeni paket `packages/server` (`@club/server`): S3, `EmailSender` ve diğer adaptörler; public API architect, uygulama backend-dev, worker yalnızca tüketir. web'in `@club/server` ve `@club/db` import etmesi lint ile yasak. CLAUDE.md ve backend-dev.md güncellendi | kapalı |
| K24 | S1: MinIO | Geliştirmede doğrulanmış son imaj digest ile sabitlenir. Prod depolama M9'da ayrı ADR ile belirlenir; adaylar arasında Contabo Object Storage var | kapalı |
| K25 | S2: Renovate | GitHub App'ini kullanıcı kuracak. Gruplama: pnpm catalog tek PR, GitHub Actions tek PR, Docker digest'leri tek PR; haftalık program | kapalı |
| K26 | S3: Redis → Valkey | Lisans sorusu kapandı. BullMQ uyumluluğunu devops doğrular; imaj digest ile sabitlenir; env adı `REDIS_URL` kalır | kapalı |
| K27 | S5 ve rapor kuralı | `tooling/tsconfig/` devops'ta; paketlerdeki `vitest.config.ts` M0 sonrası test-engineer'da. Rapor vermeden biten ajan görevi tamamlanmış sayılmaz; aynı ajana raporu tamamlatmak için takip görevi verilir (CLAUDE.md) | kapalı |
