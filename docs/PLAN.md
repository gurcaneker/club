# Uygulama Planı — Faz 1 (Pilot MVP)

Her kilometre taşı (M) bittiğinde: tüm testler yeşil, `code-reviewer` onayı, gerekiyorsa `security-auditor` onayı, `docs/PROGRESS.md` güncel, kullanıcı onayı. Sonra bir sonraki taş.

Varsayılan: Instagram, Anthropic ve e-posta çağrıları mock modda (`INSTAGRAM_MODE=mock`, `CAPTION_MODE=mock`, `EMAIL_MODE=mock`). Gerçek modlar env ile açılır: caption M5'in son görevinde, SMTP M6'da, Instagram M7'de.

---

## M0 — İskelet ve altyapı
**Sahipler:** devops, architect

- Monorepo (pnpm + Turborepo), `apps/web`, `apps/api`, `apps/worker`, `packages/db|shared|templates` (devops, M0 iskelet istisnasıyla; bkz. CLAUDE.md)
- TypeScript strict, ESLint, Prettier, ortak tsconfig
- `docker-compose.dev.yml`: PostgreSQL, Redis, MinIO (+ bucket oluşturma)
- `.env.example` (tüm değişkenler açıklamalı), env doğrulama (zod) her uygulamada
- CI: lint + typecheck + unit test + build
- `docs/adr/0001-stack.md`
- Git deposu, GitHub uzak deposu, `main` dalı

**Kabul:** `pnpm dev` ile üç uygulama ayağa kalkar; `pnpm test` ve `pnpm lint` geçer; CI yeşil.

## M1 — Veri modeli, auth, tenant izolasyonu
**Sahipler:** architect (şema, ADR), backend-dev, test-engineer

- Prisma şeması: `Tenant`, `User` (rol: PLATFORM_ADMIN, CLUB_ADMIN, COACH), `Group` (`trainingDays` dahil), `CoachGroup`, `Student`, `Consent` (`photoAllowed`, `socialMediaAllowed`, `websiteAllowed`, `faceVisibleAllowed`, metin versiyonu, zaman, IP), `ConsentToken`, `Upload`, `Photo`, `Draft` (`approvedAt`, `approvedBy`, `consentAcknowledged`), `DraftItem`, `Post` (galeri durumu, "kontrol edilmeli" işareti), `ToneProfile`, `ThemeSettings`, `TenantSettings` (`timezone`, saklama süreleri), `InstagramAccount`, `CaptionHistory`, `AuditLog`
- `packages/shared/src/state/`: durum geçiş public API'si (architect) + DESIGN §4 tablosunun uygulaması (backend-dev)
- ADR-0002 (tenant çözümleme): API tarafında alt alan adı + JWT. PLATFORM_ADMIN için açıkça tanımlanmış tenant'sız bağlam. Veli rıza linki için tenant'ın token'dan çözülmesi. Web tarafı M8'de.
- ADR-0003 (auth oturumu): access ve refresh token `httpOnly` + `Secure` + `SameSite=Lax` cookie'lerde; **`Domain` niteliği yok** (cookie host'a özel). Tenant alt alan adları birbirine göre same-site sayıldığı için değişiklik yapan her istekte (POST/PUT/PATCH/DELETE) ayrıca double-submit CSRF token'ı zorunlu.
- Tenant bağlamı: istek başına tenant çözümleme, Prisma extension ile otomatik `tenantId` filtresi
- Auth: e-posta + şifre (argon2), JWT access + refresh (ADR-0003'e göre cookie), CSRF koruması, rol guard'ları; antrenör için davet linki
- Giriş endpoint'inde hız sınırlama (IP ve hesap bazlı)
- `EmailSender` arayüzü + `MockEmailSender` (backend-dev)
- PLATFORM_ADMIN için tenant açma CLI/seed betiği (arayüz Faz 3)
- Seed: pilot tenant, 1 admin, 2 antrenör, 3 grup (idman günleriyle), sentetik öğrenciler

**Kabul:** Çapraz tenant okuma/yazma testleri başarısız olur (beklenen). Rol guard testleri geçer. Durum geçiş fonksiyonunun izinli ve izinsiz her geçişi testle doğrulanır. CSRF token'ı olmayan değiştirici istek reddedilir. Cookie'lerde `Domain` niteliği olmadığı testle doğrulanır. Giriş hız sınırı testle doğrulanır. `security-auditor` onayı.

## M2 — Rıza modülü
**Sahipler:** backend-dev, frontend-dev

- **İlk görev (frontend-dev):** giriş ekranı, oturum yönetimi (cookie + CSRF), panel iskeleti ve navigasyon
- Öğrenci CRUD (panel), CSV içe aktarma
- Veli rıza linki üretimi (tek kullanımlık, süreli, imzalı token)
- Veli formu (giriş yok): dört ayrı onay (`photoAllowed`, `socialMediaAllowed`, `websiteAllowed`, `faceVisibleAllowed`), rıza metni versiyonu, zaman, IP
- Rıza formu endpoint'lerinde hız sınırlama
- Rıza geri alma; grup bazlı rıza özeti endpoint'i (Instagram ve vitrin için ayrı sayımlar)
- Rıza geri alındığında: grubun `APPROVED`/`SCHEDULED` taslakları `PENDING_REVIEW`'e geri çekilir; grubu içeren yayınlanmış postlar "kontrol edilmeli" olarak işaretlenir (DESIGN §10)
- Öğrenci verisi dışa aktarma ve silme (KVKK talepleri)

**Kabul:** Süresi dolmuş/kullanılmış token reddedilir. Grup rıza özeti doğru. Silme işlemi bağlı kayıtları temizler. Rıza formu hız sınırı testle doğrulanır. Rıza geri alındığında zamanlanmış taslak `PENDING_REVIEW`'e döner ve yayınlanmış post "kontrol edilmeli" olarak işaretlenir (testle). `security-auditor` onayı.

## M3 — Antrenör yükleme PWA'sı
**Sahipler:** frontend-dev, backend-dev

- PWA manifest + service worker, ana ekrana ekleme
- Antrenör girişi (davet linki), yalnızca kendi grupları
- Presigned S3 yükleme, çoklu dosya, ilerleme, hata ve yeniden deneme
- İsteğe bağlı idman notu
- Rızasız çocuk uyarısı
- Yükleme tamamlanınca `Upload` kaydı + kuyruk olayı

**Kabul:** Antrenör başka grubun yüklemesini yapamaz. 20 fotoğraflık yükleme mobil görünümde e2e testle tamamlanır. Dosya tipi/boyut doğrulaması sunucu tarafında.

## M4 — İşleme hattı ve şablonlar
**Sahipler:** pipeline-dev, test-engineer

- Toplama penceresi zamanlayıcısı (tenant ayarına göre, tenant başına, tenant saat diliminde)
- Kalite puanı (netlik, parlaklık), perceptual hash ile kopya eleme
- Grup başına en iyi N kare seçimi
- 4:5 akıllı kırpma
- Türev görsellerde EXIF ve konum verisinin temizlenmesi
- Gruplama önerisi kuralları (DESIGN §6; kapak ve ara kapaklar 10 öğe sınırına dahil)
- Satori şablonları: kapak, grup ara kapağı, alt bant (logo, tarih, grup); tenant token'larıyla (`packages/shared/src/theme/` tüketilir)
- Saklama işi: yayından 90 gün sonra orijinaller, retten 30 gün sonra reddedilen taslaklar silinir (tenant ayarından)
- Çıktı: `Draft` + `DraftItem`, durum `PENDING_REVIEW` (metin M5'te eklenir; o zamana kadar mock metin)

**Kabul:** Sentetik fikstürlerle: bulanık görsel elenir, kopyalar birleşir, kapak ve ara kapaklar dahil 10 öğe sınırı aşılmaz, gruplama tablosundaki üç durum testle doğrulanır. Türev görsellerde EXIF bulunmadığı testle doğrulanır. Saklama işi süresi dolan kayıtları siler, dolmayanları korur (testle). Şablon render snapshot testleri iki farklı tenant temasıyla.

## M5 — AI metin motoru
**Sahipler:** ai-caption-dev, test-engineer

- `CaptionProvider` arayüzü: `AnthropicCaptionProvider` + `MockCaptionProvider`
- Prompt oluşturucu: ton profili, girdiler, son 30 metin
- Programatik korkuluklar: isim kontrolü (öğrenci listesi, Türkçe karakter/büyük-küçük harf duyarsız), emoji sınırı, açılış benzerliği, uzunluk
- Yeniden üretme politikası (en fazla 2), başarısızlıkta uyarı bayrağı
- Eval seti: `evals/caption/` altında 20+ senaryo, otomatik puanlama betiği
- Yönetici düzeltmelerini örnek havuzuna ekleme

**Kabul:** İsim sızıntısı testleri %100 yakalanır. Eval betiği gerçek modda çalıştırılıp rapor üretir (kullanıcı API anahtarını verdiğinde). Model adı env'den.

## M6 — Onay paneli
**Sahipler:** frontend-dev, backend-dev

- Taslak listesi + bildirim rozeti; e-posta bildirimi
- `SmtpEmailSender` (backend-dev), `EMAIL_MODE=smtp` ile açılır
- Instagram görünümlü önizleme (carousel, metin)
- Düzenle: metin, kare çıkar/sırala, kapak seç, metni yeniden üret (onaylı veya zamanlanmış taslak düzenlenirse `PENDING_REVIEW`'e döner)
- Birleştir/Ayır (worker yeniden düzenler)
- Onayla → zamanlama; Reddet → not
- Rızasız çocuklu grupta onay için açık onay kutusu zorunlu; `AuditLog`'a yazılır
- Rıza uyarısı, isim-kontrol uyarısı
- "Kontrol edilmeli" post listesi (rıza geri alma sonrası) ve inceleme kapatma eylemi
- Vitrin galerisi için yönetici onayı (`websiteAllowed` açısından rızasız çocuk içeren postlar)
- Onaysız taslak hatırlatması
- İdman günü yükleme hatırlatması (`Group.trainingDays`, tenant saat dilimi, e-posta)

**Kabul:** Onaylanmamış taslağın yayına gitmesini engelleyen test. Onay kutusu işaretlenmeden rızasız çocuklu grubun taslağı onaylanamaz (testle). Onaylı taslak düzenlenince `PENDING_REVIEW`'e döner (testle). İdman günü hatırlatması yalnızca yükleme yapılmamış idman günlerinde gönderilir (testle). Birleştir/Ayır e2e testi. Yönetici düzenlemesi olan taslakta Birleştir/Ayır önce kaybolacak düzenlemeleri listeleyen uyarı gösterir; yönetici teyit etmeden `PENDING_REVIEW → PROCESSING` geçişi yapılmaz (testle). Birleştir/Ayır sonrası, düzenlenmiş metin DESIGN §8'deki eşleme koşulunu sağlıyorsa hedef taslağa taşınır ve yeniden üretilmez; sağlamıyorsa yeniden üretilir (her iki dal testle). Mobil görünüm Playwright testleri.

## M7 — Instagram entegrasyonu
**Sahipler:** instagram-dev, security-auditor

- `InstagramPublisher` arayüzü: gerçek + mock
- OAuth bağlantı akışı (panelden), token şifreli saklama, yenileme işi
- Tekli görsel ve carousel yayın akışı, durum sorgulama
- Zamanlayıcı: `SCHEDULED` taslakları zamanı gelince yayınlar; günlük post sınırı
- Yeniden deneme + `FAILED` + bildirim
- Yayın sonrası `Post` kaydı, `AuditLog`; vitrin galerisi kuralının (`websiteAllowed`) uygulanması
- API ile post silmenin desteklenip desteklenmediği Meta dokümanından doğrulanır; desteklenmiyorsa elle silme adımı runbook taslağına not düşülür
- Geliştirme ortamında **cloudflared tüneli**: medya URL'leri ve OAuth callback için herkese açık HTTPS
- Pilot hesabıyla gerçek mod dumanı testi (kullanıcı onayıyla, manuel, cloudflared üzerinden)

**Kabul:** Mock modda tüm akış e2e yeşil. Token loglara düşmez. Gerçek mod duman testi tünel üzerinden tekli ve carousel yayınla tamamlanır. `security-auditor` onayı.

## M8 — Tema sistemi ve vitrin sayfası
**Sahipler:** frontend-dev, pipeline-dev (şablon tarafı, tüketici)

- Tema ayarları ekranı: logo yükleme, logodan renk önerisi, ana/ikincil renk seçimi
- Token türetme + WCAG kontrast düzeltme (`packages/shared/src/theme/`; public API architect, uygulama frontend-dev)
- Vitrin sayfası: hero, tanıtım, iletişim, otomatik galeri (yalnızca galeri kuralını geçen yayınlanmış postlar)
- Alt alan adına göre tenant çözümleme (web tarafı)

**Kabul:** Kontrast fonksiyonu unit testleri (bilinen kötü kombinasyonlar düzeltilir). İki farklı tema ile vitrin görsel snapshot testleri. Galeri kuralı testle doğrulanır.

## M9 — Sertleştirme ve dağıtım
**Sahipler:** devops, test-engineer, code-reviewer, security-auditor

- Tam e2e senaryo: antrenör yükler → worker işler → taslak → yönetici onaylar → mock yayın → galeri
- Prod `docker-compose.yml`, Nginx (alt alan adları, wildcard sertifika, medya için presigned proxy), yedekleme betiği (Postgres + S3)
- Loglama (PII'siz), sağlık kontrolleri
- Son tam inceleme ve güvenlik denetimi, açık bulguların kapatılması
- `docs/RUNBOOK.md`: kurulum, yedek, geri yükleme, token yenileme sorunları, Instagram'dan elle post silme (gerekirse)

**Kabul:** Temiz makinede runbook'la kurulum yapılır; tam e2e yeşil; `BLOCKER`/`MAJOR` bulgu yok.
