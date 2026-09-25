# ADR-0001: Yığın kararı — sürümler, modül sistemi, monorepo yapılandırması

- **Durum:** Kabul edildi (2026-09-25), Revizyon 2 (2026-09-25)
- **Tarih:** 2026-09-25
- **Yazan:** architect
- **Onaylayan:** kullanıcı (gurcaneker), 2026-09-25. Onayla birlikte gelen kararlar bu metne işlendi (TS seçim kuralı, duman testleri, S1–S5). Revizyon 2'deki kararlar kullanıcının 2026-09-25 kararlarıdır; `module`/`moduleResolution` seçimi kullanıcının verdiği yetkiyle orkestratörün kararıdır.
- **İlgili:** CLAUDE.md "Teknoloji yığını", `docs/PLAN.md` M0, `docs/PROGRESS.md` M0.1–M0.7
- **Etkilediği görevler:** M0.2, M0.3 (devops), M0.4 (architect), M0.5–M0.7 (devops), M0.8 (test-engineer), M1.x (backend-dev: `packages/db/src/`, `packages/server`)

## Revizyon 2 (2026-09-25)

Nedeni: devops doğrulamaları (D-TS, D-PRISMA, D-GENEL, D-NESTREF) ilk metnin varsayımlarını geçersiz kıldı ve kullanıcı yeni kararlar verdi.

1. **Tüm Node paketleri ESM.** NestJS 12 yalnızca ESM yayımlanıyor (D-GENEL 6), api de ESM oldu. `require(esm)` köprüsü, @club/db CJS geri dönüşü ve bunlara bağlı kısıtlar kaldırıldı (Karar 3).
2. **TypeScript 6.0.3, tam sürüm.** Karar 4 kontrol listesi Next.js, Nest 12 CLI/schematics, typescript-eslint ve Prisma 7.10.0 olarak yenilendi (SWC çıkarıldı). TS 6.1'e geçişi typescript-eslint'in peer aralığı engelliyor; Renovate kuralı eklendi (Karar 7).
3. **`module: "node20"` + `moduleResolution: "nodenext"`.** TS 6.0.3 `moduleResolution: node20`'yi kabul etmiyor (TS6046, D-NESTREF).
4. **SWC tamamen kalktı.** api `nest build` (`tsc` builder) ile derlenir. Vitest 4.1.11 decorator metadata'yı Vite 8'in Oxc dönüştürücüsüyle üretir. DI metadata duman testi (S-3) eklendi. `typecheck` görevi `build`'den ayrı ve zorunlu (Karar 8).
5. **Araç kararları:** ESLint 9 + `eslint-config-next`, `packages/*` için lint zorunlu top-level await yasağı, Prisma 7.10.0, BullMQ 6.x, pnpm `allowBuilds` ve 1 günlük `minimumReleaseAge`.

## Bu belgeyi kim değiştirebilir

- Kararlar, gerekçeler ve kontrol listeleri **architect**'e aittir.
- **devops bu ADR'de yalnızca "Doğrulama sonucu" alanlarını doldurabilir** (bölüm "Doğrulama sonuçları" altındaki tablolar ve `Doğrulama sonucu:` ile başlayan satırlar). Başka bir satırı değiştiremez. Bir kararın değişmesi gerekiyorsa aşağıdaki durma kuralı uygulanır.
- **Durma kuralı (devops):** devops **[D]** işaretli bir değeri doğrularken ADR'deki karardan farklı bir sonuç bulursa ya da TypeScript, Prisma, Valkey kontrollerinden, lint kural testlerinden veya duman testlerinden biri başarısız olursa **durur**, sonucu ilgili "Doğrulama sonucu" alanına yazar ve raporunda bildirir. Karar architect tarafından ADR'ye işlenmeden uygulanmaz. Bu ADR'de açıkça "deterministik" diye yazılan kurallar istisnadır; devops onları uygular ve sonucu yazar. Şu an tek deterministik kural `@nestjs/cli` sürüm kuralıdır (Karar 3.4).
- **Sürüm yaşı kuralı:** Bu ADR'de tam sürümüyle verilen bir paket kurulum anında pnpm'in `minimumReleaseAge` sınırına (1 gün, Karar 2) takılırsa devops durur ve bildirir. `minimumReleaseAgeExclude` kullanımına orkestratör karar verir. `@nestjs/cli` için kural önceden yazılmıştır (Karar 3.4).

## Bağlam

Yığının bileşenleri CLAUDE.md'de sabittir (pnpm workspaces + Turborepo, TypeScript strict, Next.js App Router, NestJS, BullMQ, Prisma + PostgreSQL, Satori, sharp, kuyruk sunucusu, S3/MinIO, Anthropic SDK, Vitest/Testcontainers/Playwright, Docker Compose + Nginx, tek VPS). Bu ADR yığını yeniden seçmiyor; tek istisna kuyruk sunucusunun Redis yerine **Valkey** olmasıdır (Karar 10). Yaptığı iş, bu bileşenleri devops'un M0 iskeletini tahmin yürütmeden kurabileceği kadar somut hale getirmek: sürümler, sabitleme yöntemi, modül sistemi, tsconfig hiyerarşisi, lint, test, Turborepo görev grafiği ve env doğrulama deseni.

Geliştirme makinesi: Node v24.18.0, pnpm 11.18.0, corepack 0.35.0, Docker 29.8.1. Karar tarihi: 2026-09-25.

**Sürüm bilgisi uyarısı:** Bu ADR'de "doğrulanacak" (kısaca **[D]**) diye işaretlenen her bilgi, M0.2/M0.3 kurulumu sırasında devops tarafından resmi kaynaktan (sürüm notları, `npm view <paket> version`, `engines`/`peerDependencies`) teyit edilir ve kaynağıyla birlikte "Doğrulama sonuçları" bölümüne yazılır.

## Paket adları ve dizinler

| Dizin | Paket adı | `type` | Rol |
| --- | --- | --- | --- |
| `apps/web` | `@club/web` | `module` | Next.js (App Router) |
| `apps/api` | `@club/api` | `module` | NestJS 12 REST API |
| `apps/worker` | `@club/worker` | `module` | BullMQ worker (düz Node, Nest yok) |
| `packages/db` | `@club/db` | `module` | Prisma şeması, üretilmiş client, tenant extension |
| `packages/shared` | `@club/shared` | `module` | zod şemaları, tipler, `state/`, `theme/`, `env/` (tarayıcıda da çalışabilir kod) |
| `packages/server` | `@club/server` | `module` | Sunucuya özel adaptörler: S3 istemcisi, `EmailSender` vb. (Karar 3.8) |
| `packages/templates` | `@club/templates` | `module` | Satori şablonları (TSX) |
| `tooling/tsconfig/` | (paket değil, düz JSON dosyaları) | — | Ortak tsconfig ön ayarları |

Tüm paketler `"private": true`. `@club` kapsamı npm'e yayımlanmaz, yalnızca iç ad alanıdır.

### Sahiplik (S4 ve S5 kararları)

| Yol | Sahip | Not |
| --- | --- | --- |
| `packages/db/prisma/` (şema, migration'lar), `packages/db/prisma.config.ts` | architect | |
| `packages/db/src/` (client fabrikası, tenant kapsamlı Prisma extension) | backend-dev | Üretilmiş kod (`src/generated/`) elle düzenlenmez |
| `packages/server/src/*/contract.ts`, `packages/server/src/*/index.ts` | architect | Public API (Karar 3.8) |
| `packages/server/src/*/` altındaki diğer dosyalar | backend-dev | Uygulama |
| `tooling/tsconfig/`, kök yapılandırma dosyaları | devops | |
| Paketlerdeki `vitest.config.ts` / `vitest.int.config.ts` | M0'da devops, M0 sonrasında test-engineer | |
| `apps/api/src/smoke.ts`, `apps/api/src/di-smoke/*.service.ts` | M0'da devops, M0 sonrasında backend-dev | Duman testleri S-1, S-2, S-3 |
| `apps/worker/src/smoke.ts` | M0'da devops, M0 sonrasında pipeline-dev | Duman testi S-2 |
| `apps/api/src/di-smoke/*.test.ts` | test-engineer | S-3 test dosyası (CLAUDE.md: `**/*.test.ts`) |

---

## Karar 1 — Node.js sürümü

**Karar:** Node.js **24 (LTS, "Krypton")**. Tam sürüm **24.18.0** olarak sabitlenir. Yükseltmeler tek bir PR'da tüm sabitleme noktalarını birlikte günceller.

| Sabitleme noktası | Değer |
| --- | --- |
| `.nvmrc` (kök) | `24.18.0` (tek doğruluk kaynağı; `.node-version` oluşturulmaz) |
| Kök `package.json` → `engines.node` | `">=24.11.0 <25"` |
| `pnpm-workspace.yaml` | `engineStrict: true` (yanlış Node ile kurulum başarısız olur) **[D: pnpm 11 ayar adı]** |
| CI | `actions/setup-node` → `node-version-file: .nvmrc` |
| Docker (M9) | `node:24.18.0-trixie-slim`; prod'da ayrıca digest ile sabitlenir **[D: etiketin varlığı]** |
| `@types/node` | `24.x` (catalog'da tam sürüm) |

**Gerekçe:**
- 2026-09-25 itibarıyla Node 24 Active LTS durumunda. Ekim 2026'da Maintenance'a geçer, EOL tarihi 2028-04-30 **[D: tarihler]**. Pilot ve ilk satış dönemini rahatça kapsar.
- Geliştirme makinesinde zaten 24.18.0 kurulu.
- NestJS 12, Vitest 4 ve pnpm 11 Node 24'ü destekliyor (D-NESTREF ortamı: Node 24.18.0).
- `process.loadEnvFile()` yerleşik olduğu için dotenv bağımlılığı gerekmiyor.

**Alternatifler:**
- *Node 22 (Maintenance LTS):* EOL 2027-04-30 (D-GENEL 1), pilot ömrü içinde bitiyor. Reddedildi.
- *Node 26:* 2026-09-25'te henüz LTS değil (LTS geçişi 2026-10-28, D-GENEL 1). Yerel eklentilerin (argon2, sharp) prebuilt ikili dosyaları gecikebilir. Reddedildi. 2027'de yeniden değerlendirilir.
- *`engines`'i yalnızca major ile sabitlemek (`24.x`), `.nvmrc`'yi `24` bırakmak:* Geliştirici, CI ve Docker arasında yama sürümü sapması olur. Reddedildi. `engines` bilinçli olarak aralık tutulur ki yama farkı kurulumu kırmasın; tam eşitliği `.nvmrc`, CI ve Docker sağlar.
- *pnpm'in Node çalışma zamanı yönetimi (`devEngines.runtime`) **[D]**:* Umut verici, ama tek araç kilidini artırıyor ve Docker/CI akışında karşılığı yok. Reddedildi, ileride yeniden bakılabilir.

**Sonuçlar/riskler:**
- Alpine (musl) imaj kullanılmaz. sharp, argon2 ve Prisma için glibc (Debian trixie) daha az sürpriz çıkarıyor.
- Node 26 LTS'e geçiş ayrı bir ADR ile yapılır.

## Karar 2 — pnpm sürümü, sabitleme ve lockfile politikası

**Karar:** pnpm **11.18.0**.
- Kök `package.json` → `"packageManager": "pnpm@11.18.0+sha512.<hash>"`. Alan elle yazılmaz, `corepack use pnpm@11.18.0` ile üretilir (hash'i corepack ekler).
- Yerel geliştirme: `corepack enable`. pnpm'in kendi sürüm yönetimi (`packageManager` alanını okuyup doğru sürüme geçmesi) de aynı alanı okuduğu için iki yol çelişmiyor **[D: pnpm 11 varsayılan davranışı]**.
- CI: `pnpm/action-setup` sürüm girdisi **verilmeden** kullanılır, sürümü `packageManager` alanından okur. CI corepack'e bağlı değil.
- Docker (M9): `corepack enable` (node:24 imajında corepack var) ya da `npm i -g pnpm@11.18.0`. Seçim M9'da devops'a bırakılıyor.
- Tüm pnpm ayarları `pnpm-workspace.yaml` içinde. `.npmrc` yalnızca registry/auth için gerekirse oluşturulur **[D: pnpm 11'in `.npmrc` ayar desteği]**.

**Lockfile politikası:**
- `pnpm-lock.yaml` repoya girer ve elle düzenlenmez.
- CI ve Docker: `pnpm install --frozen-lockfile`. Lockfile ile `package.json` uyuşmazsa build kırılır.
- Bağımlılık değişikliği içeren her PR lockfile değişikliğini de içerir. Lockfile'ı değiştiren bir PR `package.json` ya da `pnpm-workspace.yaml` değişikliği olmadan kabul edilmez.
- **Kurulum betikleri izin listesi:** pnpm 10'dan beri bağımlılıkların `postinstall` betikleri varsayılan olarak çalışmıyor. İzin listesi `pnpm-workspace.yaml` içinde **`allowBuilds`** ayarıyla açıkça tutulur (D-GENEL 4; `onlyBuiltDependencies` pnpm 11'de yok). Ayarın değer biçimi pnpm 11.18.0 dokümanından/`pnpm help` çıktısından alınır ve D-GENEL 4'e yazılır; biçim kararı değiştirmez. İlk liste: `prisma`, `@prisma/engines` (hâlâ betiği varsa), `esbuild` (tsx bağımlılığı), `sharp`, `argon2` (M1'de eklenir). Listeye ekleme PR açıklamasında gerekçelendirilir. `@nestjs/core` gibi yalnızca reklam amaçlı betikler izinli değil. `dangerouslyAllowAllBuilds` kullanılmaz.
- **Tedarik zinciri gecikmesi:** `minimumReleaseAge` pnpm 11.18.0'da varsayılan olarak **1440 dakika (1 gün)** (D-GENEL 4). Bu varsayılan bilinçli olarak korunur ve görünür olsun diye `pnpm-workspace.yaml`'a aynı değerle (`minimumReleaseAge: 1440`) açıkça yazılır. 24 saatten genç bir sürüm gerektiğinde (ör. güvenlik yaması) paket `minimumReleaseAgeExclude` listesine eklenir; yanına gerekçe ve kaldırma koşulu yorum olarak yazılır. Sürüm 1 günü geçince giriş kaldırılır.

**Alternatifler:**
- *Yalnızca corepack:* Node 25'ten itibaren corepack Node dağıtımıyla birlikte gelmiyor **[D]**. Uzun vadede tek dayanak olamaz. Reddedildi. Yerelde kolaylık olarak kalıyor.
- *pnpm 10.x:* Makinede 11 var. 10'u sabitlemek geriye gitmek olur. Reddedildi.
- *Hash'siz `packageManager`:* Hash, indirilen pnpm ikilisinin bütünlüğünü doğruluyor. Hash'li hali seçildi.

**Sonuçlar/riskler:** Ayar adları D-GENEL 4'te teyit edildi (`engineStrict`, `savePrefix`, `allowBuilds`, `minimumReleaseAge`, `minimumReleaseAgeExclude`). 1 günlük gecikme, yeni çıkan bir sürümün hemen kurulamaması demek (D-NESTREF'te `@nestjs/cli` 12.0.7 yerine 12.0.6 kuruldu). Bu beklenen davranıştır.

## Karar 3 — Modül sistemi ve paket tüketim biçimi

### 3.1 Paket bazında modül sistemi

| Paket | Çıktı | `module` / `moduleResolution` | Derleyici (build) | Geliştirme | Not |
| --- | --- | --- | --- | --- | --- |
| `apps/api` (NestJS 12) | **ESM** | `node20` / `nodenext` | `nest build` (Nest CLI varsayılanı `tsc` builder) | `nest start --watch` | `experimentalDecorators` + `emitDecoratorMetadata`. Metadata'yı `tsc` üretir (build), testte Vite 8'in Oxc dönüştürücüsü (Karar 6). Çıktı `dist/main.js` (D-NESTREF). |
| `apps/worker` | **ESM** | `node20` / `nodenext` | `tsc -p tsconfig.build.json` | `tsx watch src/main.ts` | Decorator yok. Nest kullanılmaz (Karar 3.4). |
| `apps/web` (Next.js) | Next yönetir (ESM) | `ESNext` / `Bundler` | `next build` (Turbopack) | `next dev` | `noEmit`. Typecheck `tsc --noEmit` ile. |
| `packages/shared` | **ESM** | `node20` / `nodenext` | `tsc -p tsconfig.build.json` | `tsc -p tsconfig.build.json --watch` | `sideEffects: false` |
| `packages/db` | **ESM** | `node20` / `nodenext` | `prisma generate` + `tsc -p tsconfig.build.json` | `tsc -p tsconfig.build.json --watch` | Prisma 7.10.0, `prisma-client` üreteci (Karar 3.5) |
| `packages/server` | **ESM** | `node20` / `nodenext` | `tsc -p tsconfig.build.json` | `tsc -p tsconfig.build.json --watch` | Karar 3.8 |
| `packages/templates` | **ESM** | `node20` / `nodenext` | `tsc -p tsconfig.build.json` (`jsx: react-jsx`) | `tsc -p tsconfig.build.json --watch` | Satori; **yalnızca worker tüketir** (Karar 3.3) |

**Karar: tüm Node paketleri ESM.** Her `package.json`'da `"type": "module"` açıkça yazılır. Node'un sözdizimi sezme davranışına güvenilmez. Repoda `"type": "commonjs"` paket yoktur; CJS'e geçiş ayrı bir ADR gerektirir.

**`module: "node20"` + `moduleResolution: "nodenext"` (api, worker ve tüm `packages/*`; iki alan da açıkça yazılır):**
- **`module: "node20"`:** TypeScript 5.9'da eklenen **sabit** moddur, Node 20.19+ modül davranışını modeller ve TS sürümüyle anlam değiştirmez. İlk karardaki `node20` gerekçesinin bu kısmı geçerli kalıyor. `nodenext` ise TS sürümüyle birlikte kayan bir moddur; `module` alanında kullanılmaz.
- **`moduleResolution: "nodenext"`:** TS 6.0.3 `moduleResolution` için yalnızca `node16`, `nodenext` ve `bundler` kabul ediyor; `node20` yazmak TS6046 hatası veriyor (D-NESTREF). `node20` modunun eşleştiği tek tam uyumlu çözümleme modu `nodenext`'tir (`module: node20`'nin örtük seçtiği `node16` çözümleme de testlerden geçti, ama örtük değere güvenilmez). `nodenext`'in TS sürümüyle anlam değiştirme riski, TypeScript tam sürümle sabitlendiği için (Karar 4) kontrol altında: her TS yükseltmesi ayrı PR'dır ve typecheck, build ve duman testlerinden geçer.
- D-NESTREF'te bu kombinasyon Nest 12 iskeletinde `tsc --noEmit`, `nest build`, çalışma zamanı, unit ve e2e testlerinin hepsinden geçti. Nest araçları `nodenext`/`nodenext` gerektirmiyor.
- `target`/`lib` ön ayarlarda açıkça yazıldığı için (Karar 3.6) `module: node20`'nin ima ettiği `target` varsayılanı devreye girmez.
- Göreli içe aktarmalarda `.js` uzantısı ve `exports` haritası kuralları geçerlidir (Karar 3.2 "İçe aktarma yazım kuralları").
- `apps/web` bundler kullandığı için `module: ESNext` + `moduleResolution: Bundler` ile kalır.

### 3.2 İç paketlerin tüketimi: önceden derlenmiş paketler

**Karar:** `packages/*` önceden derlenir (`dist/`). Tüketiciler `exports` haritası üzerinden **yalnızca `dist`**'i görür. Kaynak TS'i doğrudan transpile etme ("just-in-time internal packages") kullanılmaz.

`exports` biçimi (örnek, `@club/shared`):

```json
{
  "name": "@club/shared",
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".":        { "types": "./dist/index.d.ts",        "default": "./dist/index.js" },
    "./env":    { "types": "./dist/env/index.d.ts",    "default": "./dist/env/index.js" },
    "./state":  { "types": "./dist/state/index.d.ts",  "default": "./dist/state/index.js" },
    "./theme":  { "types": "./dist/theme/index.d.ts",  "default": "./dist/theme/index.js" },
    "./package.json": "./package.json"
  }
}
```

- Koşul olarak yalnızca `types` ve `default` kullanılır; `import`/`require` ayrımı yapılmaz. Tüm tüketiciler ESM olduğu için CJS girişi yayımlanmaz; her tüketici aynı ESM dosyasına çözülür (tek modül örneği).
- Alt yol listesi `@club/shared` public API'sinin parçasıdır ve architect tarafından yönetilir. M0.3'te devops yalnızca `.` ve `./env` girişlerini oluşturur. `./state` ve `./theme` ilgili public API yazıldığında (M1.1b, M8.0) eklenir.
- `exports` dışındaki derin içe aktarmalar (`@club/shared/src/...`, `@club/shared/dist/...`) yasaktır ve lint ile engellenir.
- `declarationMap: true` sayesinde editörde "tanıma git" kaynak `.ts` dosyasına gider.

**İçe aktarma yazım kuralları (Node paketleri: api, worker, `packages/*`):**
1. Göreli içe aktarmalar **`.js` uzantısıyla** yazılır (`import { x } from './x.js'`), kaynak dosya `.ts` olsa bile.
2. `exports` alanı **olmayan** dış paketlerin alt yolları **`.js` uzantısıyla** yazılır (ör. `import type { ... } from 'supertest/types.js'`). `nodenext` çözümlemesinde uzantısız alt yol TS2307 verir (D-NESTREF).
3. `exports` alanı **olan** paketlerde yalnızca haritada tanımlı alt yollar, haritadaki yazımıyla kullanılır (ör. `@club/shared/env`, uzantısız).
4. Yalnızca tip olarak kullanılan içe aktarmalar `import type` ile yazılır; `verbatimModuleSyntax` bunu derleyicide zorunlu kılar (Karar 3.6). Nest'te DI ile enjekte edilen sınıflar **değer** olarak içe aktarılır, `import type` ile değil (decorator metadata, S-3).

**Gerekçe:**
- api (`nest build`, `tsc`) ve worker (`tsc`) bundler kullanmıyor, çalışma zamanında Node'un kendisi çözümlüyor. Kaynak TS'i doğrudan tüketmek bu iki uygulamada ya bundler ya da çalışma zamanı transpile gerektirir. Nest'te decorator metadata yüzünden esbuild tabanlı araçlar da elenir.
- Tek bir çözümleme yolu (dist) Next, Nest, worker ve Vitest'te aynı davranışı veriyor. Turborepo cache'iyle derleme maliyeti düşük.

### 3.3 Paketler arası kurallar

Tüm tüketiciler ESM olduğu için modül köprüsü yoktur. Aşağıdaki kurallar köprüden bağımsız gerekçelerle geçerlidir ve lint ile uygulanır (Karar 5).

1. **`packages/*` içinde top-level await yasak; uygulama giriş noktalarında serbest.** `apps/*/src/main.ts` ve `apps/*/src/smoke.ts` gibi giriş noktalarında top-level await kullanılabilir (D-NESTREF: Nest 12 `main.ts` sorunsuz). Paketlerde yasaktır. Gerekçe: (a) top-level await bir kütüphane modülünü içe aktaran her modülün değerlendirmesini bekletir ve içe aktarma grafiğini asenkron yapar; `@club/shared` web istemci paketine de girdiği için bu, bundler'da asenkron chunk demektir. (b) Kütüphaneler içe aktarma anında G/Ç ya da yan etki yapmamalı (`sideEffects: false`, Karar 9'daki "modül üst düzeyinde parse yapılmaz" ilkesi). (c) Kütüphaneleri bir CJS araç (ör. bir yapılandırma yükleyicisi) `require(esm)` ile yüklemek zorunda kalırsa top-level await bunu kırar (`ERR_REQUIRE_ASYNC_MODULE`). Mekanizma: Karar 5 "Top-level await kuralı". Kural yalnızca repodaki kaynak kodu kapsar; üretilmiş kod (`packages/db/src/generated/`) ve üçüncü taraf bağımlılıklar kapsam dışıdır.
2. **api, `@club/templates` paketini içe aktaramaz.** Yeni gerekçe (mimari): şablon render (Satori, font yükleme, sharp) bir işleme hattı işidir ve yalnızca worker'da, kuyruk üzerinden çalışır. api istek yolunda CPU yoğun render yapmaz, render bağımlılıklarını belleğine yüklemez; render davranışının tek sahibi worker'dır (CLAUDE.md: pipeline-dev). api'nin önizleme ihtiyacı worker'ın ürettiği türev görsellerden karşılanır. Kural `no-restricted-imports` ile uygulanır.
3. **web, `@club/db` ve `@club/server` paketlerini içe aktaramaz.** Veri erişimi ve sunucu adaptörleri API üzerinden kullanılır, tenant izolasyonu tek noktada (api) kalır.
4. **ESM içe aktarma duman testleri** (M0 kabul kriteri, ayrıntı: "Duman testleri" bölümü): S-1 (api → `@club/shared/env`), S-2 (api ve worker → `@club/db`), S-3 (api DI metadata).

### 3.4 Neden api ESM, worker neden Nest değil?

- **NestJS 12 yalnızca ESM yayımlanıyor.** `@nestjs/core` ve `@nestjs/common` 12.1.0 `"type": "module"` ve `exports` yalnızca ESM dosyalarına çözülüyor (D-GENEL 6). api'yi CJS tutmak her Nest içe aktarmasını `require(esm)`'e bağlardı. Nest 12 iskeleti (`ts-esm` şablonu) ESM + `tsc` builder ile build, çalıştırma, test ve DI metadata kontrollerinden geçti (D-NESTREF). Bu yüzden api de ESM.
- **Nest paket sürümleri (tam sürüm):** `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`, `@nestjs/testing` **12.1.0**; `@nestjs/schematics` **12.0.5**; `@nestjs/cli` **12.0.7**. **Deterministik kural:** `@nestjs/cli` 12.0.7 kurulum anında pnpm'in 1 günlük `minimumReleaseAge` sınırını geçmemişse **12.0.6** kullanılır (D-NESTREF'te olan budur). devops seçimi D-GENEL 6'ya yazar. Ek bağımlılıklar: `reflect-metadata` 0.2.2, `rxjs` 7.8.2, `@types/express` 5.0.6 (dev). `supertest` ve `@types/supertest` yalnızca e2e ihtiyacı doğduğunda eklenir.
- **Nest iskeletinden alınmayanlar:** `oxlint`, `oxlint-tsgolint`, `@nestjs/mau`, `vite-tsconfig-paths`, `source-map-support`, `@swc/core`, `unplugin-swc`, `*.spec.ts` unit test kalıbı, `vitest/globals` tipleri. İskelet repoya kopyalanmaz; api iskeleti bu ADR'ye göre yazılır.
- **ESM'de decorator metadata riski:** ESM'de `emitDecoratorMetadata`, Nest provider dosyaları arasındaki döngüsel içe aktarmalarda TDZ kaynaklı `ReferenceError` üretebilir; `forwardRef` bunu her zaman çözmez. Kural: provider ve modül dosyaları arasında döngüsel içe aktarma kurulmaz; ihtiyaç doğarsa ortak parça ayrı bir dosyaya/provider'a çıkarılır. code-reviewer bunu inceleme kontrol listesinde tutar. Döngü tespiti için lint eklentisi M0'da eklenmez (yeniden değerlendirme tetikleyicisi).
- **Worker neden Nest değil:** BullMQ'nun kendi API'si yeterli; DI konteynerine ihtiyaç yok. Nest eklemek worker'a decorator metadata zorunluluğu getirir: worker'ın geliştirme döngüsü `tsx watch` (esbuild tabanlı) ile çalışıyor ve esbuild `emitDecoratorMetadata` üretmiyor, yani worker'ın dev ve build zinciri değişmek zorunda kalırdı. Bedeli: api ve worker arasında paylaşılan sunucu kodu Nest modülü olarak değil, `packages/*` içinde düz TS olarak yazılıyor: tenant kapsamlı Prisma extension `packages/db/src/`'de, S3 istemcisi ve `EmailSender` `packages/server`'da (Karar 3.8). api bunları Nest provider'larına kendisi sarar.

### 3.5 Prisma client

**Karar:**
- Prisma **7.10.0, tam sürüm**: `prisma`, `@prisma/client` ve `@prisma/adapter-pg` üçü de `7.10.0`. npm `latest` etiketi `8.0.0-rc.17`'yi gösteriyor (D-PRISMA); sürüm etiketsiz ya da aralıkla yazılırsa 8 RC kurulur. Catalog'a açıkça `7.10.0` yazılır.
- Üretici **`prisma-client`** (Rust'sız, ESM). `prisma-client-js` kullanılmaz. Çıktı `packages/db/src/generated/` altında (gitignore'da), `@club/db`'nin `tsc` derlemesine dahil. Modül biçimi ESM, göreli import uzantısı `.js`. Üretici seçenek adları (`moduleFormat`, `importFileExtension`, `runtime` vb.) **[D]** kurulumdan sonra D-PRISMA'ya yazılır; seçenek adları kararı değiştirmez.
- PostgreSQL sürücü adaptörü **`@prisma/adapter-pg` + `pg` zorunlu** (Prisma 7 tüm veritabanları için sürücü adaptörü istiyor, D-PRISMA). `pg` kurulum günündeki en güncel kararlı sürümle tam sürüm yazılır; `@types/pg` gerekiyorsa aynı şekilde.
- **`.env` otomatik yüklenmez** (D-PRISMA). `packages/db/prisma.config.ts` kök `.env`'i kendisi yükler: dosya varsa ve `NODE_ENV !== 'production'` ise `process.loadEnvFile(<repo kökü>/.env)`. `dotenv` kullanılmaz (Karar 9). Yazımı M0.4'te architect yapar.
- `prisma generate` Turborepo'da `db:generate` görevi olarak çalışır (Karar 8).
- `@club/db` ESM'dir; CJS geri dönüşü yoktur. Üretilmiş istemcide ya da Prisma runtime'ında top-level await bulunması tüketicileri (api ve worker, ikisi de ESM) kırmaz; bulunup bulunmadığı bilgi olarak D-PRISMA'ya yazılır, durma sebebi değildir. Top-level await lint kuralı üretilmiş kodu kapsamaz (Karar 3.3).
- Üretilmiş kod `library.json`'daki `verbatimModuleSyntax` veya `erasableSyntaxOnly` bayrağıyla derlenemezse devops **durur ve bildirir**; architect yalnızca `packages/db/tsconfig*.json` için hangi bayrağın kapatılacağını buraya yazar.
- `prisma generate` ya da `prisma-client` üreteci 7.10.0'da beklendiği gibi çalışmazsa devops durur ve bildirir.

**@club/db dışa aktarımları:** M0'da yalnızca üretilmiş `PrismaClient` sınıfı, üretilmiş tipler ve (gerekiyorsa) sürücü adaptörünün yeniden dışa aktarımı. Client fabrikası ve tenant kapsamlı extension M1'de backend-dev tarafından `packages/db/src/` altında yazılır; tenant çözümleme sözleşmesi ADR-0002'de tanımlanır.

**Modelsiz şemayla (M0) duman testi:** M0.4'teki şema yalnızca `generator` + `datasource` içerir, model yoktur. Duman testi için geçici ya da sahte bir model **eklenmez**. Beklenen **[D]**: `prisma generate` modelsiz şemada da (uyarıyla) bir client üretir. Bu client'ta model erişimcileri olmaz, ama sınıf ve `$connect`/`$disconnect`/`$extends` gibi yerleşik üyeler bulunur. Duman testi bu yüzden yalnızca şunları kapsar: `@club/db`'yi içe aktarmak, `PrismaClient`'ı (gerekiyorsa adaptörle) örneklemek, `typeof client.$extends === 'function'` kontrolü ve `await client.$disconnect()`. `$connect` ya da sorgu çağrılmaz; bağlantı URL'si olarak ulaşılamaz sahte bir değer verilir (`postgresql://smoke:smoke@127.0.0.1:1/smoke`). Prisma istemcisi ve pg adaptörü bağlantıyı ilk sorguda ya da `$connect`'te açar **[D]**, bu yüzden veritabanı gerekmez. Modelsiz şemada `prisma generate` client üretmiyorsa devops durur ve bildirir.

### 3.6 tsconfig hiyerarşisi

`tooling/tsconfig/` altında düz JSON dosyaları bulunur (sahibi devops). Paketler bunları göreli yolla `extends` eder (`"extends": "../../tooling/tsconfig/library.json"`).

```
tooling/tsconfig/
  base.json          # strict + ek bayraklar (Karar 4), target/lib ES2024, isolatedModules, skipLibCheck
  node-esm.json      # extends base: module "node20", moduleResolution "nodenext", verbatimModuleSyntax, erasableSyntaxOnly, types ["node"]
  library.json       # extends node-esm: declaration, declarationMap, sourceMap, outDir dist, rootDir src   (shared, db, server, templates)
  worker.json        # extends node-esm: outDir dist, rootDir src, sourceMap
  nest.json          # extends node-esm: Nest ayarları (aşağıdaki tablo)
  nextjs.json        # extends base: module ESNext, moduleResolution Bundler, jsx preserve, lib DOM + DOM.Iterable, noEmit, plugins [next]
```

**`nest.json` (api) ayarları.** Değerler D-NESTREF'te sınanmış Nest 12 iskeletinden türetildi; Karar 4'teki strict ve ek bayraklar `base.json`'dan gelir ve iskeletin gevşek ayarları (ör. `noImplicitAny: false`, `strictBindCallApply: false`) **alınmaz**.

| Ayar | Değer | Kaynak / gerekçe |
| --- | --- | --- |
| `extends` | `./node-esm.json` | `module: "node20"`, `moduleResolution: "nodenext"`, `verbatimModuleSyntax: true`, `types: ["node"]` buradan gelir |
| `target` | `ES2023` | Nest 12 iskeletinin değeri (D-NESTREF'te sınandı). `lib` `base.json`'daki `ES2024` olarak kalır (Node 24 çalışma zamanı API'leri). |
| `experimentalDecorators` | `true` | Nest decorator'ları (legacy) |
| `emitDecoratorMetadata` | `true` | DI; Vitest de bu değeri tsconfig'den okur (Karar 6, D-NESTREF negatif kontrol) |
| `erasableSyntaxOnly` | `false` | Nest'in constructor parameter property kullanımı (`constructor(private readonly x: X)`) |
| `strictPropertyInitialization` | `false` | Nest DTO ve decorator'lı sınıf alanları başlatıcısız tanımlanır. `strict: true` açık kalır, yalnızca bu alt bayrak kapanır |
| `isolatedModules` | `true` | `base.json`'dan (açıkça tekrar yazılmaz) |
| `outDir` / `rootDir` | `dist` / `src` (`tsconfig.build.json`'da) | Çıktı `dist/main.js` (D-NESTREF) |
| `sourceMap` | `true` | |
| `declaration` | `false` | Uygulama, kütüphane değil |

`verbatimModuleSyntax` api'de de **açık**. Gerekçe: D-NESTREF'te DI, `design:paramtypes` ve build bu bayrakla etkilenmedi; derleyici decorator metadata'da kullanılan sınıf içe aktarmalarını değer kullanımı sayıyor, yalnızca gerçekten tip olan içe aktarmalar için `import type` istiyor (TS1484). Bu kontrol lint kuralından daha güvenilir olduğu için api'de `consistent-type-imports` kapalı kalır (Karar 5).

Her pakette iki dosya bulunur:
- `tsconfig.json`: editör ve `typecheck` için. `src` + testler + yapılandırma dosyaları, `noEmit: true`.
- `tsconfig.build.json` (web hariç): yalnızca `src`, testler hariç, emit açık.

**Kurallar:**
- `baseUrl` kullanılmaz (TS 6'da kullanımdan kalkıyor **[D]**). Paketler arası içe aktarma yalnızca paket adıyla (`@club/shared`). `paths` yalnızca `apps/web` içinde `@/*` → `./src/*` için kullanılabilir.
- Node ön ayarlarında `module: "node20"` ve `moduleResolution: "nodenext"` **ikisi de açıkça** yazılır. `moduleResolution: "node20"` yazılmaz (TS 6.0.3'te geçersiz, TS6046). `module: "nodenext"`, `moduleResolution: "node"`/`"node10"`/`"node16"` kullanılmaz (Karar 3.1).
- TS project references (`composite`, `tsc -b`) kullanılmaz. Derleme sırasını Turborepo `^build` bağımlılığıyla yönetir.
- `types` her ön ayarda açıkça yazılır (`["node"]`). Vitest `globals: false` olduğu için test tipleri (`vitest/globals`) eklenmez; testler `vitest`'ten açıkça içe aktarır.
- `target`/`lib`: `ES2024` (Node 24 tamamını destekliyor). İstisna: `nest.json` `target: ES2023` (yukarıdaki tablo). `nextjs.json` ayrıca `DOM`, `DOM.Iterable` ekler.
- `verbatimModuleSyntax` tüm Node ön ayarlarında (api dahil) açık. `erasableSyntaxOnly` `node-esm.json`'da açık, yalnızca `nest.json`'da kapalı (parameter property).
- `erasableSyntaxOnly` sonucu olarak api dışındaki Node paketlerinde TS `enum` ve `namespace` kullanılmaz. api'de de `enum` kullanılmaz (tutarlılık; kural code-review ile). Durum/rol değerleri `as const` nesne + birleşim tipi + `z.enum(...)` ile ifade edilir. Veritabanı enum'ları Prisma şemasındadır.
- `nest build` geçersiz tsconfig seçeneklerini sessizce yutuyor (D-NESTREF). tsconfig değişikliği her zaman `tsc --noEmit` (typecheck) ile doğrulanır (Karar 8).

### 3.7 Satori / sharp / ESM-only bağımlılıkların etkisi

- Satori ve JSX yalnızca `packages/templates` ve worker'da. Worker ESM olduğu için Satori modül formatı **[D]** ne olursa olsun sorun çıkmaz.
- SVG→PNG/JPEG dönüşümü için yeni bir yerel bağımlılık (`@resvg/resvg-js`) eklemeden sharp kullanılabilir. Seçim M4'te pipeline-dev'in. Yeni bir yerel bağımlılık eklenirse pnpm build izin listesine girer.
- sharp CJS/ESM ikisini de destekliyor, prebuilt ikili dosyalar `@img/sharp-*` opsiyonel bağımlılıklarından geliyor.
- api ve worker ESM olduğu için hem ESM hem CJS dış paketleri (Nest 12, BullMQ, AWS SDK v3, argon2, zod) doğrudan içe aktarabilir. CJS paketlerden adlandırılmış içe aktarma Node'un CJS adlandırılmış dışa aktarma sezgisine bağlıdır; çalışmayan bir pakette varsayılan içe aktarma (`import pkg from 'x'`) kullanılır.

### 3.8 `packages/server` (`@club/server`)

**Amaç:** api ve worker'ın ortak kullandığı, sunucuya özel (tarayıcıda çalışmayan, sır ya da ağ istemcisi taşıyan) adaptörler: S3 istemcisi (`storage`), `EmailSender` (`email`), ileride benzer adaptörler. Instagram ve caption adaptörleri **buraya girmez**; onlar kendi sahiplerinin dizinlerinde kalır (`apps/worker/src/publish/`, `apps/worker/src/caption/`).

**Modül biçimi:** ESM, önceden derlenmiş (Karar 3.2), `sideEffects: false`, top-level await yasak (Karar 3.3 kural 1, Karar 5).

**Dizin düzeni ve sahiplik sınırı (dosya yolundan okunur):**

```
packages/server/src/
  index.ts                  # M0: yalnızca `export {};` (tsc girdisi). exports'ta yer almaz.
  <modül>/                  # ör. storage/, email/
    contract.ts             # architect: arayüzler, tipler, config tipleri, fabrika imzaları (yalnızca tip ve sabit; çalışan kod yok)
    index.ts                # architect: modülün public yüzü; contract'tan tipleri ve uygulama dosyalarından fabrikayı yeniden dışa aktarır
    *.ts (diğer her dosya)  # backend-dev: gerçek ve mock uygulamalar (ör. s3-storage.ts, mock-email-sender.ts, smtp-email-sender.ts, factory.ts)
```

- Tüketiciler yalnızca `@club/server/<modül>` alt yolunu kullanır. Kök (`.`) giriş yoktur; böylece bir modülü içe aktarmak diğer modüllerin bağımlılıklarını (ör. AWS SDK) yüklemez.
- `index.ts` içindeki yeniden dışa aktarılan uygulama adlarını architect belirler; bu adları taşıyan dosyaları backend-dev yazar. Public API değişikliği gerekiyorsa backend-dev raporunda ister, architect `contract.ts`/`index.ts`'i günceller (CLAUDE.md'deki `state/` modeliyle aynı).
- Her modülde gerçek + mock uygulama bulunur, mod env'den seçilir (ör. `EMAIL_MODE`). Mock varsayılandır.

**`exports`:**

```json
{
  "name": "@club/server",
  "type": "module",
  "sideEffects": false,
  "exports": {
    "./storage": { "types": "./dist/storage/index.d.ts", "default": "./dist/storage/index.js" },
    "./email":   { "types": "./dist/email/index.d.ts",   "default": "./dist/email/index.js" },
    "./package.json": "./package.json"
  }
}
```

M0'da yalnızca `"./package.json"` girişi bulunur; `./storage` ve `./email` girişlerini architect ilgili public API'yi yazdığında (M1) ekler.

**Bağımlılık yönü:** `@club/server` → `@club/shared` (env şemaları, tipler) ve gerekirse `@club/db`. Ters yön yasak: `@club/shared` ve `@club/db`, `@club/server`'ı içe aktaramaz. `@club/templates` `@club/server`'ı içe aktarmaz.

**tsconfig:** `library.json` (Karar 3.6).

**Turborepo:** standart görevler: `build`, `dev`, `lint`, `typecheck`, `test`, `clean`; S3 entegrasyon testleri eklendiğinde (Testcontainers + MinIO) `test:int`. Ek görev tanımı gerekmez.

**Lint yasakları:** Karar 5'teki `no-restricted-imports` listesine bakın (web → server/db, shared/db → server, derin içe aktarma).

---

## Karar 4 — TypeScript sürümü ve strict ayarları

**Karar:** TypeScript **6.0.3**, catalog'da tam sürüm (`typescript: 6.0.3`). npm `latest` etiketi `7.0.2`'yi gösteriyor (D-TS); sürüm etiketsiz yazılırsa TS 7 kurulur.

**Seçim kuralı (Revizyon 2, kullanıcı düzeltmesi).** TS sürümü şu **dört aracın** resmi desteğine göre seçilir:

1. **Next.js** (kurulacak 16.x sürümü)
2. **Nest 12 CLI/schematics** (`@nestjs/cli`, `@nestjs/schematics`)
3. **typescript-eslint**
4. **Prisma 7.10.0** (`prisma`, `@prisma/client`)

- **Dördü de destekliyorsa:** TypeScript **6.x**, en güncel kararlı 6 minor'ın dört aracın aralığına uyan en güncel yaması, tam sürümle.
- **Biri desteklemiyorsa:** kullanıcıya sorulur. **5.9'a otomatik dönüş yoktur**: Nest 12 CLI `typescript ~6.0.2`'ye bağımlı, `@nestjs/schematics` 12.0.5 peer'ı `>=6.0.0` (D-TS), yani 5.9 Nest 12 ile çelişir.
- SWC kuraldan çıkarıldı: TypeScript derleyicisini kullanmıyor, ayrıca projede artık SWC yok (Karar 6).

**"Resmi destek" tanımı:** aracın kurulacak sürümünde (a) `peerDependencies` içindeki `typescript` aralığı hedef sürümü kapsıyor, ya da (b) aracın `typescript`'e peer bağımlılığı yoksa resmi sürüm notu, changelog ya da dokümanı TS 6 desteğini açıkça belirtiyor. Yalnızca "uyarıyla çalışıyor" ya da topluluk issue'sunda "çalışıyor" demek destek sayılmaz. Belirsiz kalan araç **desteklemiyor** sayılır. Aracın kendi `dependencies`'inde sabit bir `typescript` sürümü taşıması (ör. Nest CLI) tek başına destek yok anlamına gelmez; bu durumda (b) ölçütü uygulanır.

**Sonuç (D-TS, 2026-09-25):** dördü de destekliyor; aralıkların kesişimi 6.0.2 ve 6.0.3. Seçilen: **6.0.3**. D-NESTREF'te Nest 12 iskeleti 6.0.3 ile `nest build`, `tsc --noEmit`, Vitest ve decorator metadata kontrollerinden geçti.

**TS 6.1 kısıtı:** typescript-eslint 8.70.1'in `typescript` peer aralığı `>=4.8.4 <6.1.0`. TS 6.1 yayımlansa bile typescript-eslint'in peer aralığı genişleyene kadar yükseltilmez. Bu kısıt Renovate kuralıyla uygulanır (Karar 7). Her TS yükseltmesi (yama dahil) bu seçim kuralını yeni sürüm için yeniden işletir.

TypeScript 7 (Go ile yazılmış yerel derleyici, npm `latest` 7.0.2) ayrı bir ADR ile, dört aracın desteği teyit edilince değerlendirilir.

`tooling/tsconfig/base.json` bayrakları:

| Bayrak | Değer | Not |
| --- | --- | --- |
| `strict` | `true` | Açıkça yazılır (TS 6 varsayılanına güvenilmez) |
| `noUncheckedIndexedAccess` | `true` | Dizi/kayıt erişiminde `undefined` zorunlu ele alınır |
| `noImplicitOverride` | `true` | |
| `noImplicitReturns` | `true` | |
| `noFallthroughCasesInSwitch` | `true` | |
| `useUnknownInCatchVariables` | `true` | (`strict` içinde, açıkça yazılır) |
| `isolatedModules` | `true` | tsx (esbuild) ve Vite (Oxc) dosya bazlı transpile ediyor |
| `forceConsistentCasingInFileNames` | `true` | |
| `skipLibCheck` | `true` | |
| `resolveJsonModule` | `true` | |
| `esModuleInterop` | `true` | |
| `exactOptionalPropertyTypes` | `false` | Bkz. alternatifler |
| `noPropertyAccessFromIndexSignature` | `false` | Bkz. alternatifler |

`nest.json` bu tabloya ek olarak `strictPropertyInitialization: false` ile ezer (Karar 3.6); diğer bayraklar api'de de aynıdır.

**Gerekçe:** Tenant ve rıza verisinin işlendiği bir kod tabanında `noUncheckedIndexedAccess` ve `switch-exhaustiveness-check` (lint) en yüksek getirili ek güvenceler. Seçim kuralı, "hangi TS" sorusunu yoruma bırakmadan ekosistem durumuna bağlıyor; tam sürüm sabitleme `nodenext` gibi sürümle kayan modların anlamını da sabitliyor.

**Alternatifler:**
- *`exactOptionalPropertyTypes: true`:* Prisma ve zod'un ürettiği tiplerle sürekli sürtünme yaratıyor. Reddedildi.
- *`noPropertyAccessFromIndexSignature: true`:* `process.env` erişimi zaten `env.ts` ile tek noktada toplandığı için getirisi düşük, gürültüsü yüksek. Reddedildi.
- *TS 7 (yerel) şimdi:* typescript-eslint peer'ı `<6.1.0`, Nest CLI `~6.0.2`'ye bağımlı. Reddedildi. Hızlı typecheck için isteğe bağlı olarak denenebilir, CI kapısı olmaz.
- *TS 5.9:* Nest 12 CLI/schematics ile çelişiyor (yukarıda). Reddedildi.
- *TS'i aralıkla (`~6.0.3`) yazmak:* Karar 7'nin tam sürüm politikasına aykırı; `nodenext`'in anlamını sürüm dışı bir değişkene bağlar. Reddedildi.

**Sonuçlar/riskler:** TS 6'nın varsayılan değişiklikleri (`types`, `rootDir`, `strict` vb. **[D]**) nedeniyle tüm bu ayarlar ön ayarlarda açıkça yazılır. Hiçbir davranış varsayılana bırakılmaz.

## Karar 5 — Lint ve format

**Karar:**
- **ESLint flat config**, kökte **tek dosya**: `eslint.config.mjs`. Paket bazında ESLint yapılandırma dosyası yok. Paylaşılan config paketi de yok. Alanlara özgü kurallar aynı dosyada `files` glob'larıyla tanımlanır.
- **ESLint 9** (9.x hattının kurulum günündeki en güncel yaması, tam sürüm) + **`eslint-config-next`** (sürümü `next` ile aynı, ör. 16.3.6). ESLint 10 kullanılmaz: `eslint-config-next`'in bağımlılıkları (`eslint-plugin-react`, `eslint-plugin-import`, `eslint-plugin-jsx-a11y`) ESLint'i en fazla `^9`'a kadar destekliyor (D-GENEL 9). `@eslint/js` de 9.x hattından, `eslint` ile aynı major'da seçilir (npm `latest` 10.x'i gösteriyor; etiketsiz kurulmaz).
- **typescript-eslint 8.70.1** (tam sürüm; `typescript-eslint` meta paketi), tip farkındalıklı lint: `parserOptions.projectService: true`, `tsconfigRootDir: import.meta.dirname`. Peer aralığı `typescript <6.1.0` (Karar 4, Karar 7).
- **Prettier 3.x** biçimlendirici olarak kullanılır. `eslint-config-prettier` biçim kurallarını kapatır. Prettier ESLint içinden çalıştırılmaz.
- Her pakette `lint` betiği: `eslint . --max-warnings=0`. Kökte `format` (`prettier --write .`) ve `format:check` (`prettier --check .`) betikleri bulunur. Format Turborepo görevi değildir.

**Kural seti (asgari):**
- Taban: `@eslint/js` recommended + `tseslint.configs.recommendedTypeChecked`.
- Ek kurallar (error): `@typescript-eslint/switch-exhaustiveness-check`, `@typescript-eslint/no-floating-promises`, `@typescript-eslint/no-misused-promises`, `@typescript-eslint/consistent-type-imports` (api hariç, aşağıya bakın), `no-console` (apps altında; betikler, CLI ve duman testi giriş noktaları hariç), `eqeqeq`.
- `no-restricted-imports` ile şu içe aktarmalar engellenir:
  - Her yerde: `@club/*/src/*`, `@club/*/dist/*` (derin içe aktarma)
  - `apps/api/**` içinden `@club/templates` (Karar 3.3)
  - `apps/web/**` içinden `@club/db` ve `@club/server` (Karar 3.3)
  - `packages/shared/**` ve `packages/db/**` içinden `@club/server` (Karar 3.8 bağımlılık yönü)
  - `packages/shared/**` içinden `@club/db`
  - `apps/*` arasında birbirine içe aktarma
- `apps/api/**` override'ları: `consistent-type-imports` kapalı. Bu kural, Nest DI'nin ihtiyaç duyduğu sınıf importlarını `import type`'a çevirerek decorator metadata'yı bozabiliyor. api'de tip içe aktarmalarını `verbatimModuleSyntax` derleyicide, metadata'yı hesaba katarak zorunlu kılıyor (Karar 3.6). `@typescript-eslint/no-extraneous-class` kapalı (Nest modülleri boş sınıf).
- `apps/web/**`: `eslint-config-next`'in flat config'i (`core-web-vitals`). Next 16 ile `next lint` kaldırıldı, lint doğrudan `eslint` ile çalışır **[D]**.
- Test dosyaları (`**/*.test.ts(x)`): `@typescript-eslint/unbound-method` kapalı.

**Top-level await kuralı (Karar 3.3 kural 1).** Mekanizma: ESLint çekirdek kuralı `no-restricted-syntax`, esquery seçicileriyle. Ek eklenti yok.

- **Kapsam (`files`):** `packages/*/src/**/*.{ts,tsx}`. **Hariç (`ignores`):** `**/*.test.ts`, `**/*.test.tsx`, `**/*.int.test.ts` (testler `dist`'e girmez, tüketici içe aktarmaz) ve `packages/db/src/generated/**` (zaten global ignore'da). `apps/*` bu bloğun dışında kaldığı için giriş noktalarında top-level await serbesttir. Paket kökündeki yapılandırma dosyaları (`vitest.config.ts`, `prisma.config.ts`) `src/` dışında olduğu için kapsam dışıdır.
- **Seçiciler** (üçü tek kural girdisinde):

  ```js
  'no-restricted-syntax': ['error',
    { selector: 'AwaitExpression:not(:function AwaitExpression)',
      message: 'packages/* içinde top-level await yasak (ADR-0001 Karar 3.3).' },
    { selector: 'ForOfStatement[await=true]:not(:function ForOfStatement)',
      message: 'packages/* içinde top-level for-await yasak (ADR-0001 Karar 3.3).' },
    { selector: 'VariableDeclaration[kind="await using"]:not(:function VariableDeclaration)',
      message: 'packages/* içinde top-level await using yasak (ADR-0001 Karar 3.3).' },
  ]
  ```

- **Seçicinin anlamı:** `:function` esquery'de `FunctionDeclaration`, `FunctionExpression` ve `ArrowFunctionExpression`'ı (sınıf ve nesne metotları dahil, çünkü metot gövdesi bir `FunctionExpression`'dır) kapsar. `X:not(:function X)` "atalarından hiçbiri fonksiyon olmayan X" demektir. JavaScript'te `await` yalnızca async fonksiyon gövdesinde ya da modül üst düzeyinde geçerli olduğu için (sınıf alanı başlatıcıları ve `static` bloklar `await`'e izin vermez) fonksiyon atası olmayan her `await`, üst düzeydeki bir blok (`if`, `try`, `for` vb.) içinde olsa bile, top-level await'tir. Fonksiyon içindeki hiçbir `await` eşleşmez.
- **Flat config birleştirme uyarısı:** Flat config'de aynı kural birden çok blokta tanımlanırsa sonraki blok seçenekleri **birleştirmez, tamamen değiştirir**. Bu yüzden `packages/*/src/**` dosyalarına uygulanan `no-restricted-syntax` seçenekleri tek blokta, tam liste olarak yazılır. Aynı durum `no-restricted-imports` için de geçerlidir: bir dosya birden çok bloğun kapsamına giriyorsa, o dosyaya son uygulanan blok o dosya için gereken **tüm** yasakları (derin içe aktarma dahil) içermelidir. devops lint kural testlerinde her yasağı ayrı dosyayla doğrular (M0.2/M0.3 kontrol listeleri).
- **Doğrulama:** devops M0.2'de negatif ve pozitif test dosyasıyla sınar (M0.2 kontrol listesi). Seçiciler beklenen sonucu vermezse devops **durur ve bildirir**; architect mekanizmayı değiştirir (aday: `eslint-plugin-es-x` `no-top-level-await` kuralı).

**Prettier ayarı** (`.prettierrc.json`): `{ "singleQuote": true, "trailingComma": "all", "printWidth": 100 }`. `.prettierignore`: `dist`, `.next`, `coverage`, `.turbo`, `pnpm-lock.yaml`, `**/generated/**`.

**Gerekçe:** Flat config dosyası kökten aşağıya doğru bulunduğu için tek dosya bütün monorepoyu kapsıyor. Tek kural kaynağı, tek bağımlılık listesi. Paylaşılan config paketi yalnızca birden çok repo arasında paylaşımda anlamlı olurdu.

**Alternatifler:**
- *`packages/eslint-config` paylaşılan paketi:* Fazladan paket, sürümleme ve sahiplik yükü getiriyor. Reddedildi.
- *Biome (lint + format):* Hızlı ama tip farkındalıklı kuralları (`no-floating-promises`, exhaustiveness) typescript-eslint kadar olgun değil. Next eklentisi de yok **[D]**. Reddedildi.
- *oxlint + oxlint-tsgolint (Nest 12 iskeletinin varsayılanı):* Hızlı, ama repoda ikinci bir lint aracı demek; `eslint-config-next` ve `no-restricted-syntax` seçicileri gibi mevcut kurallar ESLint'e bağlı, tip farkındalıklı kuralları typescript-eslint'e göre daha yeni. Tek lint aracı ilkesi (tek kök config) korunur. Reddedildi. `@nestjs/mau` (Nest'in bulut dağıtım platformu aracı; dağıtım Docker Compose + VPS) de alınmadı.
- *ESLint 10:* `eslint-config-next`'in bağımlılıkları desteklemiyor; yalnızca `@next/eslint-plugin-next` ile mümkün olurdu, bu da Next'in React/a11y/import kurallarını elle kurmak demek. Reddedildi.
- *Top-level await için eklenti (`eslint-plugin-es-x`):* Çalışır, ama çekirdek kuralla ifade edilebilen bir yasak için yeni bağımlılık. Seçiciler başarısız olursa geri dönüş adayı.
- *Git hook'ları (husky/lefthook + lint-staged):* M0'da eklenmez. Kalite kapısı CI. İstenirse ayrı karar.

**Sonuçlar/riskler:**
- ESLint eklentileri (`@eslint/js`, `typescript-eslint`, `eslint-config-next`, `eslint-config-prettier`, `globals`) kök `devDependencies`'de bulunur (config kökten çözümlüyor). `eslint` ikilisi ise her paketin `devDependencies`'inde `catalog:` ile yer alır ki `pnpm --filter <paket> lint` çalışsın.
- Tip farkındalıklı lint, bağımlı paketlerin `dist/*.d.ts`'ine ihtiyaç duyduğu için `lint` görevi `^build`'e bağlı.

## Karar 6 — Test (Vitest)

**Karar:**
- **Vitest 4.1.11** ve `@vitest/coverage-v8` **4.1.11**, tam sürüm (Nest 12 iskeletinin ana sürümü; D-NESTREF'te Vite 8.3.0 ile sınandı). Vitest 5 kullanılmaz (npm `latest` 5.x'i gösteriyor, etiketsiz kurulmaz). `test.projects` API'si **[D]**.
- **SWC yok.** `unplugin-swc` ve `@swc/core` kurulmaz. Nest decorator metadata'sını (`design:paramtypes`) testte Vite 8'in varsayılan dönüştürücüsü **Oxc** üretir; `experimentalDecorators`/`emitDecoratorMetadata` değerlerini ilgili paketin `tsconfig.json`'ından okur (D-NESTREF: `@Inject()` olmadan DI testi geçti; `emitDecoratorMetadata: false` ile düştü, yani Vitest tsconfig'i izliyor). Bu davranışın bekçisi S-3 duman testidir.
- **Dar geri dönüş:** S-3 ileride (ör. Vitest/Vite yükseltmesinden sonra) başarısız olursa build `tsc`'de kalır, **yalnızca** `apps/api/vitest.config.ts`'e `unplugin-swc` (+ `@swc/core` dev bağımlılığı, pnpm `allowBuilds` girişi) eklenir. Tam SWC yoluna (SWC builder, SWC ile dev) dönülmez. Geri dönüş bu ADR'ye işlenerek yapılır.
- **Yol eşleme:** `vite-tsconfig-paths` kullanılmaz (bağımlılığı `tsconfck` TS 6 ile peer uyumsuz, D-NESTREF). tsconfig `paths` çözümü gerekirse (yalnızca `apps/web` `@/*`) Vite 8'in yerleşik `resolve: { tsconfigPaths: true }` ayarı kullanılır.
- **Paket bazında yapılandırma:** her pakette `vitest.config.ts` (`defineProject`), `environment: 'node'`, `globals: false` (testler `import { describe, it, expect } from 'vitest'` kullanır; tsconfig `types: ["node"]`, `vitest/globals` yok). Turborepo her paketin `test` görevini ayrı çalıştırır ve cache'ler. Bu dosyaların sahibi M0'da devops, M0 sonrasında test-engineer'dır.
- **Kök `vitest.config.ts`:** `test.projects: ['apps/*', 'packages/*']`. Yalnızca IDE ve tek komutla tüm testleri çalıştırmak için. CI Turborepo üzerinden çalışır. `vitest.workspace.ts` kullanılmaz (Vitest 3.2'de kullanımdan kalktı).
- **Dosya adlandırma:**
  - Unit: `src/**/*.test.ts(x)` → `test` görevi. M0 CI'da çalışan tek test katmanı. Nest iskeletinin `*.spec.ts` unit kalıbı alınmaz; `.spec.ts` yalnızca Playwright e2e içindir.
  - Entegrasyon (Testcontainers): `src/**/*.int.test.ts` → ayrı `vitest.int.config.ts` ve `test:int` görevi. Docker gerektirir, M1'de CI'a ayrı job olarak eklenir. Unit config bu dosyaları `exclude` eder.
  - E2E (Playwright): `e2e/**/*.spec.ts` → `e2e` görevi. Playwright kurulumu ilk e2e ihtiyacında (M3) test-engineer tarafından yapılır. `turbo.json`'da görev tanımı M0'da bulunur.
- **Testlerde env:** testler geliştiricinin `.env` dosyasını okumaz. Gerekli env değerleri `vitest.config.ts` içinde `test.env` ile ya da test içinde açıkça verilir.
- **M0 geçici ayarı:** M0'da test dosyası olmayan paketlerde `vitest run --passWithNoTests`. Bayrak M0.8'de test-engineer her pakete duman testi ekledikten sonra kaldırılır.
- Kapsam raporu: `@vitest/coverage-v8`. M0'da eşik tanımlanmaz.

**Alternatifler:**
- *Jest (Nest'in varsayılanı):* İkinci test koşucusu, ESM desteği zayıf. Reddedildi.
- *Yalnızca kökte `projects` ile tek Vitest çağrısı:* Turborepo paket bazlı cache'i kaybolur. Reddedildi.
- *Vitest 5:* Güncel major, ama Nest 12 iskeleti ve D-NESTREF doğrulaması 4.1.11 üzerinde. Reddedildi; geçiş ayrı PR'da S-3 ile doğrulanarak yapılır.
- *`unplugin-swc` ile SWC dönüşümü (ilk karar):* Oxc metadata'yı ürettiği için gereksiz bağımlılık ve yerel ikili. Reddedildi; yalnızca dar geri dönüş olarak kalır.

**Sonuçlar/riskler:** Decorator metadata'nın Oxc tarafından üretilmesi Vite'ın bir iç davranışına bağlı. Vitest/Vite yükseltmesinde S-3 kırılırsa dar geri dönüş uygulanır. Renovate'in Vitest PR'larında S-3'ün geçmesi birleştirme koşuludur (CI zaten `test` görevini çalıştırır).

## Karar 7 — Bağımlılık sürüm sabitleme politikası

**Karar:**
- **Tam sürüm:** tüm doğrudan bağımlılıklar tam sürümle yazılır (`^`/`~` yok). `pnpm-workspace.yaml` → `savePrefix: ''` **[D: pnpm 11 ayar adı]**.
- **pnpm catalog:** birden fazla pakette kullanılan her bağımlılık `pnpm-workspace.yaml` → `catalog:` altında tam sürümle tanımlanır, paketlerde `"catalog:"` ile referans verilir. Asgari catalog: `typescript` (`6.0.3`), `@types/node` (`24.13.6`), `zod`, `vitest` (`4.1.11`), `@vitest/coverage-v8` (`4.1.11`), `eslint` (9.x), `prettier`, `tsx`, `prisma` (`7.10.0`), `@prisma/client` (`7.10.0`), `react`, `react-dom`, `@types/react`. Tek pakette kullanılan bağımlılıklar (ör. `@nestjs/*`, `next`, `bullmq`, `satori`, `@prisma/adapter-pg`, `pg`) o paketin `package.json`'ında tam sürümle durur. Not: `pnpm add -E` paket zaten kuruluysa ("Already up to date") `package.json`'ı yeniden yazmıyor (D-NESTREF); tam sürüm gerekirse elle yazılır.
- **zod tek sürüm:** zod **4.x** catalog'da. `@club/shared` zod'u `dependency` olarak alır. Uygulamalar aynı catalog girdisini kullanır. Böylece şemalar tek zod örneği üzerinde paylaşılır.
- **Overrides:** yalnızca güvenlik yaması için, `pnpm-workspace.yaml` → `overrides` altında, yanında gerekçe yorumu ve kaldırma koşuluyla.
- **Docker imajları** (dev compose ve M9 prod) tam etiket + **digest** ile sabitlenir (`image: valkey/valkey:<sürüm>@sha256:<digest>`).
- **GitHub Actions** commit SHA ile sabitlenir (sürüm yorumuyla).

**Güncelleme botu: Renovate (S2 kararı).** Kullanıcı Renovate GitHub App'ini depoya kurar. `renovate.json`'u devops M0.7'de aşağıdaki politikaya göre yazar:

| Kural | Değer |
| --- | --- |
| Program | Haftalık, pazartesi sabahı (`timezone: "Europe/Istanbul"`, ör. `"before 9am on monday"`) |
| Grup 1 — pnpm catalog | `pnpm-workspace.yaml` catalog'undaki tüm bağımlılıkların minor/patch güncellemeleri **tek PR** |
| Grup 2 — GitHub Actions | Tüm action SHA/sürüm güncellemeleri **tek PR** |
| Grup 3 — Docker digest'leri | Tüm imajların digest (ve aynı etiket içindeki) güncellemeleri **tek PR** |
| Node çalışma zamanı | `.nvmrc` ve `node` Docker imaj etiketi birlikte, ayrı PR. `@types/node` major'ı Node major'ına sabitlenir (`allowedVersions: "<25"`); Node major geçişi ADR ile yapılır. |
| TypeScript (Revizyon 2) | `typescript` paketi için `allowedVersions: "6.0.x"`. `typescript`, `typescript-eslint` ve `@typescript-eslint/*` güncellemeleri **aynı grupta, tek PR** gelir (`groupName: "typescript"`). Bu kural `packageRules` içinde catalog grubundan **sonra** yazılır ki `typescript` catalog grubuna değil bu gruba düşsün. **Genişletme/kaldırma koşulu:** typescript-eslint'in yayımlanmış bir sürümünün `typescript` peer aralığı yeni minor'ı (ör. 6.1) kapsadığında ve Karar 4 seçim kuralı o minor için dört araçta da olumlu sonuç verdiğinde, `allowedVersions` o minor'a (`"6.1.x"`) güncellenir; güncelleme ADR'nin Karar 4 notuyla aynı PR'da yapılır. Kural tamamen kaldırılmaz; TS 7 geçişi ayrı ADR'dir. |
| Vitest | `vitest` ve `@vitest/coverage-v8` aynı grupta; major (5.x) Dashboard onaylı ayrı PR. S-3 (DI metadata) geçmeden birleştirilmez (Karar 6). |
| Prisma | `prisma`, `@prisma/client`, `@prisma/adapter-pg` aynı grupta ve aynı sürümde. Ön sürümler (`8.0.0-rc.*`) önerilmez (`ignoreUnstable: true`, Renovate varsayılanı, açıkça yazılır). |
| Major güncellemeler | Gruplanmaz; her biri ayrı PR, Dependency Dashboard onayıyla açılır |
| Catalog dışı bağımlılıklar | Renovate'in varsayılan monorepo gruplaması (ör. `@nestjs/*`) dışında paket başına PR |
| `minimumReleaseAge` | `"3 days"` |
| Otomatik birleştirme | Yok |
| Kilit dosyası bakımı | `lockFileMaintenance` açık, aynı haftalık programda |

**Alternatifler:**
- *Caret (`^`) aralıkları + lockfile:* Lockfile zaten sabitliyor ama `package.json` okunarak hangi sürümün çalıştığı anlaşılamıyor, catalog hizası bozuluyor. Reddedildi.
- *Dependabot:* Yerleşik, kurulum istemiyor. Gruplama ve monorepo desteği Renovate'ten zayıf. Kullanıcı Renovate'i kurmayı kabul ettiği için kullanılmıyor.

**Sonuçlar/riskler:** Tam sürüm + bot, düzenli PR akışı demek. Tek geliştirici bağlamında haftalık takvim ve üç ana grup bu yükü sınırlı tutuyor.

## Karar 8 — Turborepo görev grafiği ve cache kuralları

**Karar:** Turborepo **2.x** **[D: güncel minor]**. Remote cache yok (Vercel kullanılmıyor). Yerel cache var. CI'da `.turbo/cache` `actions/cache` ile saklanır.

`turbo.json` görevleri:

| Görev | `dependsOn` | `outputs` | Cache | Not |
| --- | --- | --- | --- | --- |
| `db:generate` | — | `src/generated/**` | evet | Yalnızca `@club/db`. `inputs`: `prisma/**`, `prisma.config.ts` |
| `build` | `^build`, `db:generate` | `dist/**`, `.next/**`, `!.next/cache/**` | evet | `db:generate` yalnızca betiği olan pakette çalışır |
| `typecheck` | `^build`, `db:generate` | — | evet | Web: `next typegen && tsc --noEmit` **[D: `next typegen`]**. Diğerleri (api dahil): `tsc --noEmit -p tsconfig.json` |
| `lint` | `^build`, `db:generate` | — | evet | `eslint . --max-warnings=0` |
| `test` | `^build`, `db:generate` | `coverage/**` | evet | Yalnızca unit |
| `smoke` | `build` | — | **hayır** | Yalnızca api ve worker: `node dist/smoke.js` (Duman testleri bölümü) |
| `test:int` | `^build`, `db:generate` | — | **hayır** | Docker gerektirir |
| `e2e` | `build` | `playwright-report/**`, `test-results/**` | **hayır** | M3'ten itibaren |
| `dev` | `^build`, `db:generate` | — | **hayır**, `persistent: true` | Paketler `tsc --watch`, uygulamalar dev sunucusu |
| `clean` | — | — | **hayır** | |

**`typecheck` ayrı ve zorunlu (Revizyon 2):** `typecheck` görevi her pakette `tsc --noEmit`'tir; `build` onun yerine geçmez. `nest build` geçersiz tsconfig seçeneklerini (ör. TS6046) sessizce yutup çıkış kodu 0 veriyor ve `tsconfig.build.json` testleri dışladığı için test dosyalarını denetlemiyor (D-NESTREF). CI'da `typecheck` ve `build` ikisi de çalışır.

**Cache kuralları:**
- `globalDependencies`: `tooling/tsconfig/**`, `eslint.config.mjs`, `.prettierrc.json`, `.nvmrc`, `vitest.config.ts` (kök). Lockfile'ı Turborepo zaten izliyor.
- **Env modu `strict`** (Turborepo 2 varsayılanı). Build çıktısını etkileyen değişkenler görev bazında `env` ile listelenir: `apps/web` `build` için `NEXT_PUBLIC_*`. Uygulamalar `.env` dosyasını kendileri okuduğu için (Karar 9) dev görevine env geçirmeye gerek yok.
- `.env` dosyaları cache girdisi **değil**. `build` ve `test` çıktısı `.env`'e bağlı olmamalı. Bağlıysa bu bir hatadır.
- Her görev paket betiğine karşılık gelir. Betiği olmayan pakette görev atlanır.

**Kök `package.json` betikleri:** `dev`, `build`, `lint`, `typecheck`, `test`, `smoke`, `test:int`, `e2e`, `clean` (hepsi `turbo run <görev>`), `format`, `format:check`, `dev:infra` (`docker compose -f docker-compose.dev.yml up -d`), `dev:infra:down`.

**Portlar (dev):** web `3000`, api `3001`. Worker M0'da HTTP dinlemez (sağlık kontrolü M9). Dev compose servisleri yalnızca `127.0.0.1`'e bağlanır.

**Alternatifler:**
- *`typecheck`/`lint`'i `^build`'den bağımsız tutmak (kaynak koşulu ile):* Karar 3.2'deki alternatif. M0'da reddedildi.
- *Vercel remote cache:* Dış servis bağımlılığı, gereksiz. Reddedildi.

## Karar 9 — Env doğrulama deseni

**Karar:**
- **Ortak yardımcı (M0.4, architect):** `@club/shared/env` alt yolu (`packages/shared/src/env/`). Public API taslağı:

  ```ts
  // packages/shared/src/env/index.ts (taslak imzalar; kesin hali M0.4'te)
  export function parseEnv<T extends z.ZodType>(schema: T, source?: Record<string, string | undefined>): z.infer<T>;
  export class EnvValidationError extends Error { readonly issues: ReadonlyArray<{ key: string; message: string }> }
  export function loadEnvFileIfPresent(path: string): void; // yalnızca NODE_ENV !== 'production' iken process.loadEnvFile
  // Ortak parçalar:
  export const envSchemas: {
    nodeEnv; logLevel; port; url; databaseUrl; redisUrl; s3; // z şemaları (redisUrl: Valkey bağlantısı, Karar 10)
    captionMode;    // z.enum(['mock', 'anthropic']), varsayılan 'mock'
    emailMode;      // z.enum(['mock', 'smtp']),      varsayılan 'mock'
    instagramMode;  // z.enum(['mock', 'meta']),      varsayılan 'mock'
  };
  ```

  - `parseEnv` hata durumunda **yalnızca değişken adlarını ve hata nedenini** raporlar. Değerleri asla içermez, böylece sırlar loga düşmez. Başarılı sonuç `Object.freeze` edilir.
  - Mod bağımlı zorunluluk (ör. `CAPTION_MODE=anthropic` iken `ANTHROPIC_API_KEY` ve `ANTHROPIC_MODEL` zorunlu) şemada `superRefine` ya da ayrık birleşimle ifade edilir. Model adı koda gömülmez, varsayılanı da yoktur.
- **Her uygulamada `src/env.ts` (M0.5, devops):** uygulamanın kendi şemasını ortak parçalardan oluşturur ve **bellekte saklanan (memoized) bir `getEnv()` fonksiyonu** dışa aktarır. Modül üst düzeyinde parse **yapılmaz**. ESM içe aktarmalar giriş noktasının gövdesinden önce değerlendirildiği için üst düzey parse `.env` yüklenmeden çalışırdı.
- **Giriş noktası sırası (api ve worker `main.ts`):** önce `loadEnvFileIfPresent(<repo kökü>/.env)`, sonra `getEnv()` (hatalıysa süreç açık bir mesajla sonlanır), sonra uygulama başlatılır. api'de env, Nest'e özel bir provider token'ı (`ENV`) ile verilir. `@nestjs/config` kullanılmaz.
- **Web:** `next.config.ts` içinde `@next/env` → `loadEnvConfig(<repo kökü>)` ile kök `.env` yüklenir. Sunucu şeması (`src/env.server.ts`, `import 'server-only'`) ile istemci şeması (`src/env.client.ts`, yalnızca `NEXT_PUBLIC_*`) ayrıdır. Sunucu env'i `instrumentation.ts` `register()` içinde doğrulanır.
- **Prisma CLI:** Prisma 7 `.env`'i otomatik yüklemez (D-PRISMA); `packages/db/prisma.config.ts` kök `.env`'i `process.loadEnvFile` ile kendisi yükler (Karar 3.5). Config API ayrıntısı (`defineConfig`, `env()`) **[D]**.
- **Tek `.env.example`** repo kökünde. Her değişken yorumla açıklanır (hangi uygulama kullanıyor, zorunlu mu, varsayılanı ne, sır mı). M0'daki değişkenler: `NODE_ENV`, `LOG_LEVEL`, `WEB_PORT`, `API_PORT`, `DATABASE_URL`, `REDIS_URL` (Valkey bağlantısı; ad bilinçli olarak `REDIS_URL` kalır, Karar 10), `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_FORCE_PATH_STYLE`, `CAPTION_MODE`, `EMAIL_MODE`, `INSTAGRAM_MODE`, `ANTHROPIC_API_KEY` (mock modda boş), `ANTHROPIC_MODEL` (mock modda boş). Sonraki taşların değişkenleri (JWT sırları, şifreleme anahtarı, SMTP, Meta) o taşlarda eklenir.

**Alternatifler:**
- *`@t3-oss/env-nextjs` / `envalid`:* Web için uygun ama api ve worker için ikinci bir desen demek. Reddedildi. Tek desen `@club/shared/env`.
- *`@nestjs/config`:* Nest'e özgü, worker'da yok. Reddedildi.
- *Uygulama başına ayrı `.env` dosyası:* PLAN tek `.env.example` istiyor, ortak değişkenler de tekrar ederdi. Reddedildi.
- *`dotenv` paketi:* Node 24'ün `process.loadEnvFile` fonksiyonu yetiyor. Reddedildi.

**Sonuçlar/riskler:** Prod'da (`NODE_ENV=production`) `.env` dosyası okunmaz. Env, Docker Compose tarafından verilir (M9).

## Karar 10 — Altyapı servisleri (dev compose): PostgreSQL, Valkey, MinIO

**Kuyruk sunucusu: Valkey (S3 kararı).** CLAUDE.md'deki "Redis (kuyruk)" ifadesinin yerine **Valkey** kullanılır (BSD lisanslı, Redis protokolüyle uyumlu). Kuyruk kütüphanesi BullMQ olarak kalır; sürüm **6.x** (kurulum günündeki en güncel yama, tam sürüm; bilgi: `latest` 6.3.8, D-GENEL). Valkey uyumu M0.6'da doğrulanır.
- İmaj: resmi `valkey/valkey`, kurulum günündeki en güncel kararlı sürüm **[D: sürüm; 8.x ya da 9.x]**, **etiket + digest** ile sabit.
- Yapılandırma: `maxmemory-policy noeviction` (BullMQ zorunluluğu), `appendonly yes` (dev'de de kuyruk kaybını önlemek için).
- Bağlantı env değişkeni adı **`REDIS_URL`** olarak kalır (`redis://` şeması). Gerekçe: BullMQ/ioredis ekosistemindeki yerleşik ad, Valkey protokol olarak Redis uyumlu, ileride geri dönüş ya da yönetilen bir Redis uyumlu servis kullanılırsa ad değişmez. Kod ve dokümanda "Redis" kelimesi yalnızca protokol/env adı için geçer; sunucu adı Valkey'dir.
- **[D] BullMQ uyumluluğu:** devops, kurulacak BullMQ sürümünün seçilen Valkey sürümünü desteklediğini resmi kaynaktan (BullMQ dokümanı/sürüm notları) teyit eder ve M0.6'da dev compose'daki Valkey'e karşı basit bir kuyruk turu (iş ekle → işle → tamamlandı) çalıştırır. Olumsuzsa durur ve bildirir. Sonuç D-VALKEY'e yazılır.

**PostgreSQL:** `postgres:18.x` (tam etiket + digest) **[D: veri dizini yolu değişikliği]**.

**MinIO (S1 kararı):** Geliştirmede **doğrulanmış son MinIO imajı digest ile sabitlenir** (yalnızca yerel, `127.0.0.1`'e bağlı, dış ağa kapalı). Bucket init servisi aynı şekilde sabitlenmiş istemci imajıyla çalışır **[D: istemci imajının durumu]**. Sonuç D-MINIO'ya yazılır. **Prod depolama M9'da ayrı bir ADR ile belirlenir**; adaylar arasında **Contabo Object Storage** (S3 uyumlu) bulunur. Kod S3 API'sine yazıldığı için (`@club/server/storage`, `S3_*` env) sağlayıcı değişikliği kod değişikliği gerektirmemelidir.

---

## Duman testleri (M0 kabul kriteri)

S-1 ve S-2'nin kodu `apps/api/src/smoke.ts` ve `apps/worker/src/smoke.ts` dosyalarındadır (S-3 bir Vitest testidir, aşağıda). Bu dosyalar build'e dahildir, `smoke` betiğiyle (`node dist/smoke.js`) çalışır, iş mantığı içermez. M0'da devops yazar; M0 sonrasında sahibi api için backend-dev, worker için pipeline-dev olur. Başarıda `smoke ok` yazıp 0 ile çıkar, herhangi bir hatada 0 dışı kodla çıkar. `.env` okumaz; gereken değerleri kod içinde sabit sahte değerler olarak verir.

### S-1 — ESM içe aktarma (api → @club/shared)

`apps/api/src/smoke.ts`, `@club/shared/env`'den `parseEnv` ve `envSchemas`'ı içe aktarır ve gerçek bir çağrı yapar: sabit bir `source` nesnesiyle (ör. `{ NODE_ENV: 'test', API_PORT: '3001' }`) geçerli bir şemayı ayrıştırıp beklenen değeri kontrol eder, ardından geçersiz bir `source` ile `EnvValidationError` fırlatıldığını kontrol eder (`instanceof` kontrolü, iki modül örneği olmadığını da gösterir).

Üç kontrolün **hepsi** geçmelidir:

| # | Kontrol | Komut | Başarı ölçütü |
| --- | --- | --- | --- |
| 1 | Tip denetimi | `pnpm --filter @club/api typecheck` (`tsc --noEmit`, `module: node20`, `moduleResolution: nodenext`) | Çıkış kodu 0, hata yok (TS6046 gibi tsconfig hataları dahil) |
| 2 | Build | `pnpm --filter @club/api build` (`nest build`, `tsc` builder, ESM çıktı) | Başarılı; `dist/smoke.js` içinde `@club/shared/env` için `import` ifadesi var |
| 3 | Çalışma zamanı | `node apps/api/dist/smoke.js` ve ayrıca `node apps/api/dist/main.js` (geçerli env ile) | `smoke.js`: `smoke ok`, çıkış kodu 0. `main.js`: 3001'de dinlemeye başlar; `ERR_MODULE_NOT_FOUND`, `ERR_PACKAGE_PATH_NOT_EXPORTED`, `ERR_UNSUPPORTED_DIR_IMPORT` yok, `ExperimentalWarning` yok |

Çıktı yolu **`apps/api/dist/main.js`** ve **`apps/api/dist/smoke.js`** (`tsconfig.build.json` `rootDir: src`; D-NESTREF'te `tsc` builder ile teyit edildi). Build (2) typecheck'in (1) yerine geçmez (Karar 8).

### S-2 — @club/db içe aktarma (api ve worker)

M0.4'te şema ve `prisma.config.ts` yazıldıktan sonra her iki `smoke.ts` ayrıca şunu yapar (Karar 3.5 "Modelsiz şemayla duman testi"): `@club/db`'den `PrismaClient`'ı (ve gerekiyorsa adaptörü) içe aktarır, sahte bağlantı URL'siyle örnekler, `$extends`'in fonksiyon olduğunu kontrol eder, `$disconnect()` çağırır. Veritabanı çalışmıyor olmalı ya da olmasa da test geçmelidir.

| # | Tüketici | Yol | Başarı ölçütü |
| --- | --- | --- | --- |
| 1 | api (ESM) | `import` | typecheck + build + `node apps/api/dist/smoke.js` başarılı |
| 2 | worker (ESM) | `import` | typecheck + build + `node apps/worker/dist/smoke.js` başarılı |

### S-3 — DI decorator metadata (api, Vitest)

Amaç: Nest DI'nin constructor tipinden (`design:paramtypes`) çalıştığını, Vitest'te SWC olmadan (Oxc) ve `globals: false` ile kanıtlamak. Karar 6'daki dar geri dönüşün tetikleyicisi bu testtir.

**Dosyalar:**
- `apps/api/src/di-smoke/smoke-dependency.service.ts`: `@Injectable()` sınıf `SmokeDependencyService`, sabit bir değer döndüren bir metot (ör. `ping(): string` → `'pong'`).
- `apps/api/src/di-smoke/smoke-consumer.service.ts`: `@Injectable()` sınıf `SmokeConsumerService`, `constructor(private readonly dependency: SmokeDependencyService) {}`; bağımlılığın metodunu çağıran bir metot (ör. `callDependency(): string`).
- `apps/api/src/di-smoke/smoke-consumer.service.test.ts`: test.

**Zorunlu koşullar (herhangi biri sağlanmazsa test kabul edilmez):**
1. `SmokeConsumerService`, bağımlılığı **yalnızca constructor parametre tipiyle** alır. `@Inject()`, `@Inject(forwardRef(...))`, özel provider token'ı, `useFactory`/`inject` kullanılmaz.
2. `SmokeDependencyService` **değer olarak** içe aktarılır (`import { SmokeDependencyService } from './smoke-dependency.service.js'`); `import type` ya da `import { type ... }` kullanılmaz.
3. Test `Test.createTestingModule({ providers: [SmokeConsumerService, SmokeDependencyService] }).compile()` ile modülü derler ve `moduleRef.get(SmokeConsumerService)` ile örneği alır.
4. Doğrulamalar: (a) enjekte edilen bağımlılık tanımlı; (b) `SmokeDependencyService`'in örneği (`instanceof`); (c) metodu çağrılabiliyor ve beklenen değeri döndürüyor (ör. `callDependency()` → `'pong'`); (d) `Reflect.getMetadata('design:paramtypes', SmokeConsumerService)` tam olarak `[SmokeDependencyService]`.
5. Vitest `globals: false`: `describe`, `it`, `expect` `vitest`'ten açıkça içe aktarılır. Test dosyasının başında `import 'reflect-metadata';` bulunur (`Reflect.getMetadata` tipleri için).
6. Test `apps/api`'nin normal `test` görevinde (`vitest run`, `apps/api/vitest.config.ts`) çalışır; `unplugin-swc` yoktur.

**Negatif kontrol (bir kez, kayıt için):** `apps/api/tsconfig.json`'da geçici olarak `emitDecoratorMetadata: false` yapıldığında test **başarısız** olmalı, geri alınınca geçmeli. Sonuç D-SMOKE'a yazılır; değişiklik commitlenmez.

**Sahiplik:** iki servis dosyası M0'da devops (iskelet, iş mantığı yok), sonra backend-dev; test dosyası test-engineer.

CI (M0.7) `pnpm turbo run lint typecheck test build` sonrasında `pnpm smoke` çalıştırır. S-3 `test` görevinin içindedir.

---

## Sürüm özeti

| Bileşen | Karar | Doğrulama |
| --- | --- | --- |
| Node.js | 24.18.0 (LTS), `engines` `>=24.11.0 <25` | D-GENEL 1 |
| pnpm | **11.18.0**, `packageManager` + hash; `allowBuilds`, `minimumReleaseAge: 1440` | D-GENEL 4 |
| TypeScript | **6.0.3** (tam sürüm; 6.0.x'te kalır, Karar 7) | D-TS |
| Modül ayarı (Node paketleri) | tümü ESM; `module: "node20"`, `moduleResolution: "nodenext"` | D-NESTREF |
| Turborepo | 2.x (bilgi: 2.11.4) | [D] |
| Next.js / React | 16.x / 19.x (bilgi: 16.3.6 / 19.3.0) | [D] |
| NestJS | `@nestjs/common`, `core`, `platform-express`, `testing` **12.1.0**; `@nestjs/schematics` **12.0.5**; `@nestjs/cli` **12.0.7** (1 günlük sınırı geçmemişse **12.0.6**, Karar 3.4); build `tsc` builder | D-GENEL 6, D-NESTREF |
| Nest yardımcıları | `reflect-metadata` **0.2.2**, `rxjs` **7.8.2**, `@types/express` **5.0.6**; `supertest`/`@types/supertest` e2e'de gerektiğinde | D-NESTREF |
| `@types/node` | **24.13.6** | D-NESTREF |
| Prisma | **7.10.0** (`prisma`, `@prisma/client`, `@prisma/adapter-pg`) + `pg`; `prisma-client` üreteci, ESM | D-PRISMA |
| BullMQ | 6.x (bilgi: 6.3.8) | Valkey uyumu D-VALKEY |
| zod | 4.x (bilgi: 4.6.5) | — |
| Vitest | **4.1.11** + `@vitest/coverage-v8` **4.1.11**; SWC yok (Oxc metadata), `vite-tsconfig-paths` yok | D-GENEL 8, D-NESTREF |
| ESLint / typescript-eslint | **9.x** + `eslint-config-next` (= `next` sürümü) / **8.70.1** | D-GENEL 9 |
| Prettier | 3.x (bilgi: 3.9.9) | — |
| tsx | 4.x (bilgi: 4.23.15), yalnızca worker dev | — |
| Kullanılmayanlar | `@swc/core`, `unplugin-swc`, `oxlint`, `oxlint-tsgolint`, `@nestjs/mau`, `vite-tsconfig-paths`, `source-map-support`, `dotenv`, `@nestjs/config` | — |
| Docker temel imajı (M9) | `node:24.18.0-trixie-slim` + digest | etiket [D] |
| PostgreSQL (dev compose) | 18.x + digest | Veri dizini yolu değişikliği [D] |
| Valkey (dev compose) | en güncel kararlı + digest, `noeviction` | D-VALKEY |
| MinIO (dev compose) | doğrulanmış son imaj + digest | D-MINIO |

Kalın yazılan sürümler aynen kullanılır. "x" ile yazılan sürümler catalog'a ya da `package.json`'a kurulum günündeki en güncel kararlı yama ile **tam sürüm** olarak girer ("bilgi" değeri D-GENEL'deki son okuma, bağlayıcı değil). Hiçbir paket etiketsiz ya da `latest` etiketiyle kurulmaz: `typescript`, `prisma`, `vitest` ve `eslint` için `latest` bu ADR'nin kararından farklı bir major'ı gösteriyor.

## Doğrulanacaklar (M0.2/M0.3/M0.6'da devops teyit eder)

1. Node 24 LTS takvimi (Active LTS bitişi, EOL 2028-04-30) ve Node 26'nın LTS'e geçiş tarihi.
2. Node 25+ dağıtımlarında corepack'in olmaması.
3. _(Revizyon 2 ile kapsam dışı: `require(esm)` köprüsü kullanılmıyor. Numara, D-GENEL satırlarıyla eşleşme bozulmasın diye korunuyor.)_
4. pnpm 11 ayar adları (teyit edildi, D-GENEL 4). Kalan: `allowBuilds` değer biçimi ve `pnpm config` ile 11.18.0'da teyit, `packageManager` ile kendi sürüm yönetimi.
5. **TypeScript seçimi (Karar 4 kuralı):** tamamlandı → D-TS (6.0.3). Kalan: TS 6 varsayılan değişikliklerinin (`types`, `rootDir`, `strict`, `baseUrl` kullanımdan kalkması) ön ayarlarda açıkça yazılmış olması (typecheck ile).
6. NestJS 12 durumu ve `tsc` builder çıktı yolu: tamamlandı (D-GENEL 6, D-NESTREF). Kalan: kurulan `@nestjs/cli` sürümü (12.0.7 ya da 12.0.6, Karar 3.4) D-GENEL 6'ya yazılır.
7. **Prisma** → D-PRISMA: üretici seçenek adları, üretilen istemcinin modül biçimi, top-level await (bilgi), `prisma generate` için `DATABASE_URL` gerekip gerekmediği (CI'da sahte değer), modelsiz şemada client üretimi, üretilmiş kodun `verbatimModuleSyntax`/`erasableSyntaxOnly` ile derlenmesi, TS 6.0.3 ile derleme.
8. Vitest 4.1.11 `test.projects` API'si. (Oxc decorator metadata desteği D-NESTREF'te teyit edildi; projede S-3 ile yeniden doğrulanır.)
9. `next lint`'in kaldırılması, `next typegen` komutu, `eslint-config-next`'in ESLint 9 flat config'i. (ESLint major kararı verildi: 9.)
10. Satori'nin modül formatı ve yoga bağımlılığında top-level await bulunup bulunmadığı (yalnızca bilgi amaçlı; worker ESM olduğu için karar değişmez).
11. Docker imajları: `node:24.18.0-trixie-slim`, `postgres:18.x` (veri dizini yolu), Valkey → D-VALKEY, MinIO → D-MINIO.
12. **Valkey** → D-VALKEY: sürüm, digest, BullMQ 6.x uyumluluğu.
13. **Lint kural testleri** (M0.2/M0.3): top-level await seçicileri (Karar 5) ve `no-restricted-imports` yasakları. Sonuç D-GENEL'e 13 numaralı satır olarak yazılır.

## Doğrulama sonuçları (devops doldurur)

> Bu bölümdeki tablolar ve `Doğrulama sonucu:` satırları devops tarafından doldurulur. Her satıra kaynak (URL, `npm view` çıktısı, `peerDependencies` alıntısı, sürüm notu) yazılır. Tarih: doğrulamanın yapıldığı gün.

### D-TS — TypeScript seçimi (M0.2)

Kontrol listesi, kullanıcının 2026-09-25 düzeltmesine göre: **Next.js, Nest 12 CLI/schematics, typescript-eslint, Prisma 7.10.0.** SWC, TypeScript derleyicisini kullanmadığı için kuraldan çıkarıldı. Karar 4 metni architect tarafından güncellenecek.

| Araç | Sürüm | `typescript` peer aralığı | TS 6 resmi desteği | Kaynak |
| --- | --- | --- | --- | --- |
| Next.js | 16.3.6 (npm `latest`; kurulmadı) | yok (ölçüt b) | evet | v16.3.0 sürüm notu: "Bump TypeScript to 6.0: #91257", "Fix TS6 baseUrl deprecation for extended tsconfig: #91855"; v16.3.4: "Fix build error when aliasing typescript to @typescript/typescript6 (#97997)" (github.com/vercel/next.js/releases). Next deposu `typescript: 6.0.2` kullanıyor (v16.3.6 `package.json`). `eslint-config-next@16.3.6` peer `typescript: ">=3.3.1"` (isteğe bağlı) |
| Nest 12 CLI / schematics | `@nestjs/cli` 12.0.7, `@nestjs/schematics` 12.0.5 (ikisi de `latest`; kurulmadı) | CLI: peer yok, `dependencies.typescript: "~6.0.2"`. Schematics: **`">=6.0.0"`** | evet | `npm view @nestjs/schematics@12.0.5 peerDependencies` → `{ prettier: '^3.0.0', typescript: '>=6.0.0' }` (kullanıcı iddiası doğrulandı). `nest new` şablonu (`ts-esm/package.json` ve `ts/package.json`) `"typescript": "^6.0.2"` yazıyor: iskelet TS 6 üretiyor (kullanıcı iddiası şablon düzeyinde doğrulandı; iskelet kurulmadı, bkz. D-NESTREF) |
| typescript-eslint | 8.70.1 (npm `latest`; kurulmadı) | `">=4.8.4 <6.1.0"` (`@typescript-eslint/parser@8.70.1` aynı) | evet (yalnızca 6.0.x) | `npm view typescript-eslint@8.70.1 peerDependencies` |
| Prisma | `prisma` / `@prisma/client` 7.10.0 | `">=5.4.0"` (isteğe bağlı peer) | evet | `npm view prisma@7.10.0 peerDependencies`, `npm view @prisma/client@7.10.0 peerDependencies` |
| ~~SWC yolu (`@swc/core` + `unplugin-swc`)~~ | — | — | **kuraldan çıkarıldı** (2026-09-25 kullanıcı kararı; SWC TS derleyicisini kullanmıyor). Eski sonuç bilgi için: resmi TS 6 beyanı bulunamamıştı | — |

TS 6 kararlı sürümleri (`npm view typescript versions`): 6.0.2, 6.0.3 (6.1 yok; `latest` = 7.0.2). Kesişim: Next (peer yok) ∩ `~6.0.2` (Nest CLI) ∩ `>=6.0.0` (schematics) ∩ `>=4.8.4 <6.1.0` (typescript-eslint) ∩ `>=5.4.0` (Prisma) → 6.0.2 ve 6.0.3.

Doğrulama sonucu: 2026-09-25. Dört aracın dördü de TS 6'yı resmi olarak destekliyor; tek bir tam sürüm dört aracın aralığına da uyuyor. Seçilen: **TypeScript 6.0.3** (en güncel kararlı 6 minor'ın en güncel yaması; tam sürümle catalog'a yazılacak). Kısıt: typescript-eslint 8.70.1'in üst sınırı `<6.1.0`; TS 6.1 çıkarsa typescript-eslint peer'ı genişlemeden yükseltilemez. Önceki sonuç (5.9.3) geçersizdir. Catalog'a henüz yazılmadı (M0.2 kurulumu başlamadı). Ek (2026-09-25, D-NESTREF ikinci deneme): Nest 12 iskeleti TS 6.0.3'ü fiilen kurdu; `nest build`, `tsc --noEmit`, Vitest 4.1.11 ve decorator metadata TS 6.0.3 ile çalıştı. TS 6.0.3'te `moduleResolution` yalnızca `node16`, `nodenext`, `bundler` kabul ediyor (`node20` → TS6046); `module: node20` geçerli. Tek TS 6 peer uyumsuzluğu: `vite-tsconfig-paths` 5.1.4 → `tsconfck` 3.1.6 `typescript ^5.0.0` (eklenti gereksiz, bkz. D-NESTREF).

### D-PRISMA — Prisma (M0.3/M0.4)

| Alan | Sonuç | Kaynak |
| --- | --- | --- |
| Kurulan Prisma ana/tam sürümü | Kurulmadı. Kararlı son sürüm 7.10.0 (`prisma`, `@prisma/client`, `@prisma/adapter-pg`). **Dikkat:** npm `latest` etiketi ön sürümü gösteriyor: `prisma` `latest` = `8.0.0-rc.17` (2026-09-24), `prev` = `7.10.0`. `pnpm add prisma` etiketsiz çalıştırılırsa 8 RC kurulur; sürüm açıkça `7.10.0` yazılmalı | `npm view prisma dist-tags` |
| Üretici (`prisma-client` / `prisma-client-js`) | Dokümana göre `prisma-client` (yeni, Rust'sız); `prisma-client-js` "future releases"da kaldırılacak. Kurulumla teyit edilmedi | prisma.io "Upgrading to Prisma ORM 7" → Schema changes |
| Üretilen istemcinin modül biçimi (ESM/CJS) ve ilgili üretici seçenekleri | Doküman: "Prisma ORM now ships as an ES module". Üretici seçenek adları (`moduleFormat`, `importFileExtension`) kurulumla teyit edilmedi (M0.3 kurulumu sonrası) | aynı kaynak → ESM support |
| Üretilen istemcide ya da runtime'ında top-level await | Teyit edilmedi (kurulum ve M0.4 şeması sonrası) | — |
| Sürücü adaptörü gerekli mi (`@prisma/adapter-pg`) | Evet, dokümana göre tüm veritabanları için zorunlu: "require a driver adapter for all databases" | aynı kaynak → Driver adapters |
| `prisma.config.ts` ve `.env` yükleme davranışı | `.env` otomatik yüklenmiyor: "environment variables are not loaded by default"; `prisma.config.ts` varsayılan yapılandırma yeri, `env()` yardımcısı `prisma/config`'ten | aynı kaynak → Environment variables, Prisma config |
| `prisma generate` için `DATABASE_URL` gerekli mi | M0.4 sonrası | — |
| Modelsiz şemada client üretiliyor mu | M0.4 sonrası | — |
| Seçilen TS sürümüyle uyum | 7.10.0: `prisma` ve `@prisma/client` `peerDependencies.typescript: ">=5.4.0"` (isteğe bağlı) → **TS 6.0.3 aralık içinde, peer uyumlu** (2026-09-25). Not: doküman "Prerequisites" bölümü "TypeScript 5.9.x" öneriyordu (önceki kontrol); peer aralığı 6'yı kapsadığı için Karar 4 ölçütüne göre uyumlu sayılır. Derleme ile teyit M0.4 sonrası | `npm view prisma@7.10.0 peerDependencies`, `npm view @prisma/client@7.10.0 peerDependencies`; prisma.io "Upgrading to Prisma ORM 7" → Prerequisites |

Doğrulama sonucu: 2026-09-25, kısmi (yalnızca paket bilgisi ve doküman; kurulum yapılmadı). Ön değerlendirme: 7.10.0 ile `prisma-client` üreteci ESM çıktı veriyor; top-level await bilgisi kurulum ve generate'ten sonra netleşir. Karar 3.5 tablosunun ilk satırı (ESM kalır) muhtemel ama teyit edilmedi; CJS geri dönüşü için şu an bir gösterge yok. `latest` etiketinin 8 RC'yi göstermesi, catalog'da sürümün açıkça `7.10.0` yazılmasını gerektiriyor.

### D-VALKEY — Valkey (M0.6)

| Alan | Sonuç | Kaynak |
| --- | --- | --- |
| Valkey sürümü (etiket) | | |
| İmaj digest'i | | |
| BullMQ sürümü ve Valkey desteği | | |
| Kuyruk turu (ekle → işle → tamamlandı) | | — |

Doğrulama sonucu: _(doldurulacak)_

### D-MINIO — MinIO (M0.6)

| Alan | Sonuç | Kaynak |
| --- | --- | --- |
| Sunucu imajı etiketi + digest | | |
| Bucket init istemci imajı etiketi + digest | | |
| Dağıtım durumu notu (bakım modu vb.) | | |

Doğrulama sonucu: _(doldurulacak)_

### D-SMOKE — Duman testleri (M0.3 sonrası ve M0.5 sonrası)

| Test | typecheck | build | çalışma zamanı | Not (gerçek çıktı yolu, hata çıktısı) |
| --- | --- | --- | --- | --- |
| S-1 api → @club/shared | | | | |
| S-2 api → @club/db | | | | |
| S-2 worker → @club/db | | | | |
| S-3 api DI metadata (Vitest; "build" sütunu yerine `test`, negatif kontrol sonucu nota) | | | | |

Doğrulama sonucu: _(doldurulacak)_

### D-GENEL — Diğer [D] maddeleri (Doğrulanacaklar 1–4, 6, 8–11)

| # | Sonuç | Kaynak |
| --- | --- | --- |
| 1 | ADR ile uyumlu. Node 24 "Krypton": LTS 2025-10-28, Maintenance 2026-10-20, EOL 2028-04-30. Node 22 EOL 2027-04-30. Node 26 LTS 2026-10-28 | github.com/nodejs/Release `schedule.json` |
| 2 | **ADR'den farklı / çelişkili.** Node v25 dokümanı "Corepack will no longer be distributed starting with Node.js v25" diyor, ama CHANGELOG_V25 (ör. 25.6.x "deps: update corepack to 0.34.6") ve CHANGELOG_V26 ("deps: update corepack to 0.36.0") hâlâ paketli corepack'i güncelliyor. Node 25/26'da corepack'in dağıtımda olup olmadığı kesinleşmedi | nodejs/node `doc/api/corepack.md` (v25.x), `doc/changelogs/CHANGELOG_V25.md`, `CHANGELOG_V26.md` |
| 3 | ADR ile uyumlu. Node 24 `modules.md`: modül sistemi "Stability: 2 - Stable"; `require(esm)` uyarı üretip üretmediği çalışma zamanında teyit edilecek (S-1) | nodejs/node `v24.x/doc/api/modules.md` |
| 4 | Ayar adları (pnpm.io/settings, sayfa "Version: 12.x"): `engineStrict` var; `savePrefix` var; build izin listesi **`allowBuilds`** (+ `strictDepBuilds`, `dangerouslyAllowAllBuilds`), `onlyBuiltDependencies` listede yok; `minimumReleaseAge` var (+ `minimumReleaseAgeExclude`); `.npmrc`'den yalnızca auth/registry okunuyor. 11.18.0 ile `pnpm config` üzerinden teyit kurulum adımında yapılacak. **`minimumReleaseAge` varsayılanı pnpm 11.18.0'da 1440 dakika (1 gün)** (paket içeriği: `"minimum-release-age": 24 * 60`; `pnpm config get minimumReleaseAge` → `undefined`, yani varsayılan etkin). Pratik etkisi D-NESTREF'te görüldü: 4 saatlik `@nestjs/cli` 12.0.7 yerine 12.0.6 kuruldu. Catalog'a yazılacak tam sürümler 1 günden genç olmamalı ya da `minimumReleaseAgeExclude` kullanılmalı. Bilgi: pnpm `latest` = 12.6.0, 11 hattının sonu 11.27.1 | pnpm.io/settings; `npm view pnpm dist-tags`; pnpm 11.18.0 paket içeriği (corepack önbelleği) |
| 6 | **ADR'den farklı.** NestJS 12 kararlı olarak yayımlandı: `@nestjs/core`/`common`/`platform-express` `latest` = 12.1.0 (2026-09-23), 11 hattı `legacy` etiketiyle 11.2.6 (2026-09-23). `@nestjs/core@12.1.0` ve `@nestjs/common@12.1.0` **`"type": "module"`**, `exports` yalnızca ESM dosyalarına (`./index.js`) çözülüyor; `@nestjs/core` peer'ları `^12.0.0`. `@nestjs/cli` `latest` = 12.0.7 (`typescript ~6.0.2` bağımlılığı), 11 hattı 11.0.24. Karar 3.4 ("NestJS 11 CJS olarak yayımlanıyor") ve sürüm özetindeki "NestJS 11.x" bu sonuca göre architect tarafından yeniden değerlendirilmeli. SWC builder çıktı yolu teyit edilmedi (SWC builder kullanıcı kararıyla düştü). **`tsc` builder çıktı yolu (D-NESTREF, 2026-09-25): `dist/main.js`** (`tsconfig.build.json` `rootDir: ./src`). İskelet Nest 12.1.0 paketlerini kurdu; ESM + `tsc` builder ile build/çalıştırma/test geçti | `npm view @nestjs/core dist-tags`, `npm view @nestjs/core@12.1.0 type exports peerDependencies`, `npm view @nestjs/cli dist-tags` |
| 8 | **ADR'den farklı.** Vitest güncel major 5: `latest` = 5.0.2 (5.0.0: 2026-09-03), 4 hattı `V4` etiketiyle 4.1.11. Vitest 5 peer: `vite ^6.4 \|\| ^7 \|\| ^8`, `engines.node ^22.12 \|\| ^24 \|\| >=26`. `test.projects` kurulumla teyit edilmedi. Kullanıcı kararı Vitest **4.1.11** (V4). D-NESTREF (2026-09-25): 4.1.11 + Vite 8.3.0 ile Nest decorator metadata (`design:paramtypes`) SWC'siz üretiliyor, `unplugin-swc` gerekmiyor (kurulmadı, uyumu test edilmedi). `globals: false` + açık import'lar geçti. Vite 8 `resolve.tsconfigPaths: true` yerleşik, `vite-tsconfig-paths` gereksiz | `npm view vitest dist-tags`, `npm view vitest@5.0.2 peerDependencies engines` |
| 9 | ESLint `latest` 10.11.0, `@eslint/js` 10.0.1. typescript-eslint 8.70.1 peer `eslint ^8.57 \|\| ^9 \|\| ^10`. `eslint-config-next` 16.3.6 peer `eslint >=9` ama bağımlılıkları `eslint-plugin-react` 7.37.5, `eslint-plugin-import` 2.32.0, `eslint-plugin-jsx-a11y` 6.10.2 ESLint'i en fazla `^9`'a kadar destekliyor. `@next/eslint-plugin-next` 16.3.6'nın `eslint` peer'ı yok. Sonuç: `eslint-config-next` ile 9.x, yalnızca `@next/eslint-plugin-next` ile 10.x mümkün (Karar 5 ikisine de izin veriyor; seçim kurulumda yapılacaktı). `next lint` kaldırılması ve `next typegen` kurulumla teyit edilmedi | `npm view <paket>@latest peerDependencies` |
| 10 | Teyit edilmedi (M0.3 kurulumu durduruldu). Satori `latest` 0.33.5 | `npm view satori dist-tags.latest` |
| 11 | M0.6/M9 kapsamı; bu görevde bakılmadı | — |
| — | Bilgi (M0.6 için, sürüm özeti): BullMQ `latest` 6.3.8; 5 hattının sonu 5.81.5 (2026-09-10). ADR "BullMQ 5.x [D]" diyor | `npm view bullmq dist-tags.latest`, `npm view bullmq time` |
| — | Bilgi: Turborepo 2.11.4, Next.js 16.3.6, React 19.3.0, zod 4.6.5, Prettier 3.9.9, tsx 4.23.15 (hepsi ADR'deki major ile uyumlu) | `npm view <paket> dist-tags.latest` |

#### D-NESTREF — Nest 12 referans iskeleti (2026-09-25)

Amaç: repo dışında (`/tmp/nest12-ref`) Nest 12 iskeleti üretip derleme, çalıştırma, test, decorator metadata (DI), `globals: false` ve `module: node20` sonuçlarını görmek. İlk deneme (`--type esm`) durma kuralıyla kesilmişti (`--type` CLI seçeneği yok; ESM schematic varsayılanı; varsayılan builder `tsc`). Kullanıcı kararlarından sonra (TS 6.0.3, `tsc` builder, SWC yok, Vitest 4.1.11, `globals: false`, ESLint 9) ikinci deneme yapıldı. İskelet dosyaları repoya kopyalanmadı; dizin iş sonunda silindi. Ortam: Node 24.18.0, pnpm 11.18.0.

| Alan | Sonuç | Kaynak |
| --- | --- | --- |
| Üretim komutu | `cd /tmp && pnpm dlx @nestjs/cli@12.0.7 new nest12-ref --package-manager pnpm --skip-git --strict` → başarılı, soru sorulmadı (`--observe` istemi çıkmadı). CLI 12.0.7 bayrakları: `--directory`, `-d/--dry-run`, `-g/--skip-git`, `-s/--skip-install`, `-p/--package-manager`, `-l/--language`, `-c/--collection`, `--strict` (varsayılan true), `-t/--skip-tests`, `--format`, `--observe`/`--no-observe`. `--type` yok; `ts-esm` şablonu seçildi | `nest new --help`, komut çıktısı |
| Kurulan tam sürümler | `@nestjs/common`/`core`/`platform-express`/`testing` **12.1.0**, `@nestjs/cli` **12.0.6** (proje içine; `dlx` 12.0.7), `@nestjs/schematics` 12.0.5, `@nestjs/mau` 0.2.8, `typescript` **6.0.3**, `vitest` **4.1.11**, `@vitest/coverage-v8` 4.1.11, `vite` **8.3.0** (vitest üzerinden; `rolldown` 1.2.10), `@types/node` **24.13.6**, `@types/express` 5.0.6, `reflect-metadata` 0.2.2, `rxjs` 7.8.2, `supertest` 7.3.0, `@types/supertest` 7.2.1, `vite-tsconfig-paths` 5.1.4, `prettier` 3.9.9, `source-map-support` 0.5.21, `oxlint` 1.85.0, `oxlint-tsgolint` 7.0.2002 | `pnpm ls --depth 0`, `pnpm why vite` |
| `@nestjs/cli` 12.0.6 nedeni | 12.0.7 yayın zamanı 2026-09-25T06:36Z (kurulumdan ~4 saat önce), 12.0.6 2026-09-24T07:49Z. pnpm 11.18.0 varsayılan `minimum-release-age: 24 * 60` (dakika) 1 günden genç sürümü dışarıda bırakıyor. Hata değil, beklenen davranış | `npm view @nestjs/cli time`; pnpm 11.18.0 paket içeriği |
| TS/Vitest sabitleme | İskelet zaten 6.0.3 ve 4.1.11 kurdu; `package.json` aralıkları (`^6.0.2`, `^4.1.2`) tam sürüme (`6.0.3`, `4.1.11`, `@vitest/coverage-v8` `4.1.11`) çevrildi, kilit değişmedi. Not: `pnpm add -D -E` "Already up to date" durumunda `package.json`'ı yeniden yazmadı; tam sürüm elle yazılmalı | komut çıktısı |
| Peer uyarısı | Tek uyarı: `tsconfck@3.1.6` (`vite-tsconfig-paths` 5.1.4 bağımlılığı) `typescript ^5.0.0` istiyor, kurulu 6.0.3. Vitest ayrıca "Vite now supports tsconfig paths resolution natively via `resolve.tsconfigPaths`" uyarısı veriyor. Eklenti kaldırılıp `resolve: { tsconfigPaths: true }` yazıldığında testler geçti → eklentiye gerek yok, peer uyarısı ortadan kalkar | `pnpm peers check`, vitest çıktısı |
| Kutudan çıkan `pnpm build` | Geçti (`nest build`, `tsc` builder). Çıktı yolu **`dist/main.js`** (`tsconfig.build.json` `rootDir: ./src`); `dist/src/...` değil | komut çıktısı, `find dist` |
| Kutudan çıkan `node dist/main.js` | Geçti. `PORT=3999 node dist/main.js` → `curl localhost:3999/` → `Hello World!` HTTP 200; logda hata/uyarı yok (`main.ts` top-level await sorunsuz) | curl çıktısı |
| Kutudan çıkan `pnpm test` / `pnpm test:e2e` | İkisi de geçti (1/1, 1/1), Vitest 4.1.11 | komut çıktısı |
| Kutudan çıkan `tsc --noEmit -p tsconfig.json` | **Kaldı**, yalnızca şablonun e2e dosyasında: `test/app.e2e-spec.ts(4,21): error TS2307: Cannot find module 'supertest/types' or its corresponding type declarations.` Neden: `nodenext` ESM modunda uzantısız alt yol; `supertest`'in `exports` alanı yok. `'supertest/types.js'` yazınca geçiyor. `nest build` spec/test dosyalarını dışladığı için build etkilenmiyor. Vitest ile ilgisi yok; durma sebebi değil | `tsc --traceResolution` |
| DI metadata testi (varsayılan yol) | **Geçti, geri dönüş gerekmedi.** `BarService` + `FooService` (`constructor(private readonly bar: BarService) {}`, `@Inject()` yok, değer import'u) ve `foo.service.spec.ts` (`describe/it/expect` `vitest`'ten açık import; `Test.createTestingModule({ providers: [FooService, BarService] }).compile()`; `foo.bar` tanımlı, `BarService` örneği, `ping()` çağrılabiliyor, `Reflect.getMetadata('design:paramtypes', FooService)` = `[BarService]`) → 1/1 geçti. Vitest 4.1.11 + Vite 8.3.0 varsayılan dönüştürücüsü (Oxc) tsconfig'teki `emitDecoratorMetadata`'yı okuyor. `unplugin-swc`/`@swc/core` kurulmadı | vitest `--reporter verbose` çıktısı |
| DI negatif kontrol | tsconfig'te `emitDecoratorMetadata: false` yapılınca aynı test düştü (`AssertionError: expected undefined to be defined`), `true`'ya dönünce geçti → test gerçekten metadata'ya bağlı ve Vitest tsconfig'i izliyor | aynı |
| DI gerçek uygulamada | `AppModule` `providers: [AppService, BarService, FooService]`, `GET /di` → `foo.callBar()`. `pnpm build` geçti; `dist/foo.service.js` içinde `import { BarService } from './bar.service.js'` ve `__metadata("design:paramtypes", [BarService])` var. `curl localhost:3999/di` → `foo->bar-pong` HTTP 200 | curl çıktısı, derleme çıktısı |
| `verbatimModuleSyntax: true` (ek bilgi) | DI kodu etkilenmedi: build, `design:paramtypes` ve testler geçti. `tsc --noEmit` yalnızca şablon e2e dosyasında `TS1484` verdi (`INestApplication`, `App` tip olduğu için `import type` istiyor) | komut çıktısı |
| `globals: false` | `vitest.config.ts` ve `vitest.config.e2e.ts` → `globals: false`, tsconfig `types: ["node"]` (`vitest/globals` kaldırıldı). Açık import eklenmeden önce: çalışma zamanında `ReferenceError: describe is not defined`, `tsc`'de `TS2593 Cannot find name 'describe'` (globals gerçekten kapalı). Spec dosyalarına `import { describe, it, expect, beforeEach } from 'vitest'` (e2e: `beforeEach, afterEach`) eklenince: unit 2/2, e2e 1/1 geçti; `tsc --noEmit` temiz (`supertest/types.js` düzeltmesiyle) | komut çıktısı |
| `module: node20` + `moduleResolution: node20` | **TS 6.0.3'te geçersiz.** `tsc --noEmit` (hem `tsconfig.json` hem `tsconfig.build.json`): `tsconfig.json(4,25): error TS6046: Argument for '--moduleResolution' option must be: 'node16', 'nodenext', 'bundler'.` `module: node20` geçerli, `moduleResolution: node20` yok | komut çıktısı |
| `nest build` ve seçenek hataları | **Dikkat:** aynı geçersiz tsconfig ile `nest build` çıkış kodu **0** verdi ve `dist/` üretti (TS6046 sessizce yutuluyor). Gerçek tip hatasında (`TS2322`) `nest build` çıkış kodu 1 veriyor. Sonuç: CI'da ayrı `tsc --noEmit` (typecheck) zorunlu; `nest build` tek başına tsconfig hatasını yakalamaz. `node20/node20` ile `node dist/main.js` (`/` ve `/di` 200), `pnpm test` (2/2) ve e2e (1/1) geçti; ama tsconfig geçersiz olduğu için bu varyant kabul edilemez | komut çıktısı |
| `module: node20`, `moduleResolution` yazılmadan | Geçti. `tsc --showConfig` → `moduleResolution: "node16"` (örtük), `resolvePackageJsonExports: true`. `tsc --noEmit` 0, `pnpm build` 0, `/` ve `/di` HTTP 200, unit 2/2, e2e 1/1 | komut çıktısı |
| `module: node20` + `moduleResolution: nodenext` | Geçti. `tsc --noEmit` 0, `pnpm build` 0, `/` ve `/di` HTTP 200, unit 2/2, e2e 1/1 | komut çıktısı |

Doğrulama sonucu: 2026-09-25. **Durma yok.** Kutudan çıkan build, çalıştırma ve testler geçti (tek kusur: şablon e2e dosyasındaki `supertest/types` import'u `tsc --noEmit`'te TS2307; `.js` uzantısıyla düzeliyor). DI metadata testi `@Inject()` olmadan, `tsc` builder + Vitest 4.1.11 (Vite 8.3.0, SWC'siz) ile hem testte hem gerçek uygulamada geçti; `unplugin-swc` dar geri dönüşüne gerek kalmadı; negatif kontrol testin metadata'ya bağlı olduğunu gösterdi. `globals: false` + açık import'lar geçti. `node20`: `module: node20` çalışıyor, ama **`moduleResolution: node20` TS 6.0.3'te geçersiz (TS6046)**; `node20` tek başına iki alana birden yazılamaz. Geçerli seçenekler: `module: node20` + `moduleResolution` belirtilmez (örtük `node16`) ya da `moduleResolution: nodenext` (her ikisi de tüm kontrollerden geçti) ya da şablonun `nodenext`/`nodenext`'i. Karar architect'te (kullanıcı kuralı: `node20` geçmezse `nodenext`). `nest build` geçersiz tsconfig seçeneğini yutuyor; ayrı typecheck adımı şart. `/tmp/nest12-ref` silindi.

## Kapanan açık sorular

- **S1 — MinIO:** Kabul (a). Dev'de doğrulanmış son imaj digest ile sabit; prod depolama M9'da ayrı ADR, adaylara Contabo Object Storage eklendi (Karar 10).
- **S2 — Renovate:** Kabul. Kullanıcı GitHub App'i kuracak; politika Karar 7'de, `renovate.json` M0.7'de devops.
- **S3 — Kuyruk sunucusu:** Valkey (Karar 10). `REDIS_URL` adı korunur.
- **S4 — Paylaşılan sunucu kodu:** (a) Tenant Prisma extension `packages/db/src/`; şema/migration architect, `packages/db/src/` backend-dev. (b) Yeni paket `packages/server` (`@club/server`), public API architect, uygulama backend-dev; web'den içe aktarma lint ile yasak (Karar 3.8, Karar 5).
- **S5 — Sahiplik:** `tooling/tsconfig/` devops; paketlerdeki `vitest.config.ts` M0 sonrasında test-engineer.

## M0.2 kontrol listesi (devops — kök iskelet)

M0.2 Revizyon 2 ile yeniden başlar. Önceki denemeden kalan dosya yoksa sıfırdan kurulur; varsa bu listeye göre düzeltilir.

- [ ] `.nvmrc` → `24.18.0`
- [ ] Kök `package.json`: `"name": "club"`, `"private": true`, `"type": "module"`, `packageManager` (`corepack use pnpm@11.18.0` ile), `engines.node: ">=24.11.0 <25"`, Karar 8'deki betikler (`smoke` dahil). Kök `devDependencies`: `turbo` (2.x), `typescript` (`catalog:`), `eslint` (`catalog:`, 9.x), `@eslint/js` (9.x, `eslint` ile aynı major), `typescript-eslint` (`8.70.1`), `eslint-config-next` (`next`'in kurulacak sürümüyle aynı, ör. `16.3.6`), `eslint-config-prettier`, `globals`, `prettier` (`catalog:`), `vitest` (`catalog:`). Hepsi tam sürüm ya da `catalog:`; hiçbiri etiketsiz kurulmaz. `@next/eslint-plugin-next` ayrıca eklenmez (`eslint-config-next` içinde).
- [ ] `pnpm-workspace.yaml`:
  - `packages: ['apps/*', 'packages/*']`
  - `catalog:` Karar 7 listesi; kesin değerler: `typescript: 6.0.3`, `@types/node: 24.13.6`, `vitest: 4.1.11`, `@vitest/coverage-v8: 4.1.11`, `prisma: 7.10.0`, `@prisma/client: 7.10.0`, `eslint: <9.x son yama>`; diğerleri Sürüm özeti kuralıyla
  - `engineStrict: true`, `savePrefix: ''`
  - `allowBuilds`: `prisma`, `@prisma/engines` (betiği varsa), `esbuild` (değer biçimi D-GENEL 4'e göre). `@swc/core` yok.
  - `minimumReleaseAge: 1440` (açıkça). `minimumReleaseAgeExclude` yalnızca gerekirse, gerekçe yorumuyla (Karar 2)
- [ ] `pnpm-lock.yaml` üretildi ve commitlenecek. Kurulumdan sonra `pnpm ls typescript vitest eslint` çıktısı kararlaştırılan sürümleri gösteriyor (6.0.3, 4.1.11, 9.x)
- [ ] `turbo.json`: Karar 8 tablosu (`smoke` ve ayrı `typecheck` dahil), `globalDependencies`, `web#build` için `env: ["NEXT_PUBLIC_*"]`
- [ ] `tooling/tsconfig/{base,node-esm,library,worker,nest,nextjs}.json` (Karar 3.6 ve 4):
  - `base.json`: Karar 4 tablosundaki bayrakların hepsi açıkça, `target`/`lib` `ES2024`
  - `node-esm.json`: `"module": "node20"`, `"moduleResolution": "nodenext"` (ikisi de yazılı), `verbatimModuleSyntax: true`, `erasableSyntaxOnly: true`, `types: ["node"]`
  - `nest.json`: Karar 3.6 "`nest.json` (api) ayarları" tablosu (`extends: ./node-esm.json`, `target: ES2023`, `experimentalDecorators`, `emitDecoratorMetadata`, `erasableSyntaxOnly: false`, `strictPropertyInitialization: false`, `sourceMap: true`, `declaration: false`)
  - `moduleResolution: "node20"` hiçbir dosyada yok; `module: "nodenext"` hiçbir dosyada yok
- [ ] `eslint.config.mjs` (Karar 5): taban, ek kurallar, `no-restricted-imports` tam listesi (flat config birleştirme uyarısına göre her dosya grubu için tam liste), api/web/test override'ları, **top-level await bloğu** (`files: ['packages/*/src/**/*.{ts,tsx}']`, test dosyaları `ignores`'ta, Karar 5'teki üç seçici aynen), `eslint-config-prettier` en sonda, ignore: `dist`, `.next`, `coverage`, `**/generated/**`
- [ ] `.prettierrc.json`, `.prettierignore` (Karar 5)
- [ ] Kök `vitest.config.ts` (`test.projects`)
- [ ] `.editorconfig` (utf-8, lf, 2 boşluk), `.gitattributes` (`* text=auto eol=lf`)
- [ ] `.gitignore` güncellemesi: `node_modules`, `dist`, `.next`, `.turbo`, `coverage`, `packages/db/src/generated`, `.env`, `.env.*` (ama `!.env.example`), `playwright-report`, `test-results`
- [ ] **Top-level await lint kural testi (negatif + pozitif).** Geçici dosyalar (sonra silinir, commitlenmez; `package.json` oluşturulmaz, böylece workspace paketi sayılmazlar):
  - `packages/lint-probe/tsconfig.json`: `{ "extends": "../../tooling/tsconfig/library.json", "include": ["src"] }`
  - `packages/lint-probe/src/tla-probe.ts`:
    ```ts
    const p = Promise.resolve(1);
    async function* gen() { yield 1; }
    // RAPORLANMALI (6 adet):
    await p;
    export const a = await p;
    if (a > 0) { await p; }
    for await (const v of gen()) { void v; }
    try { await p; } catch { /* yok */ }
    await using r = { async [Symbol.asyncDispose]() {} };
    // RAPORLANMAMALI:
    export async function f() { await p; for await (const v of gen()) { void v; } }
    export const g = async () => { await p; };
    export class C { async m() { await p; } }
    export const o = { async m() { await p; } };
    ```
  - `packages/lint-probe/src/tla-probe.test.ts`: tek satır `await Promise.resolve(1);` (test dosyası, raporlanmamalı)
  - `apps/lint-probe/tsconfig.json` (`worker.json`'ı extend eder, `include: ["src"]`) ve `apps/lint-probe/src/main.ts`: `tla-probe.ts` ile aynı içerik (apps kapsam dışı, raporlanmamalı)
  - Komut: `pnpm exec eslint packages/lint-probe apps/lint-probe --format json`, çıktı `ruleId === 'no-restricted-syntax'` ile süzülür. Diğer kuralların mesajları bu testte değerlendirilmez.
  - Beklenen: `packages/lint-probe/src/tla-probe.ts`'te tam olarak "RAPORLANMALI" altındaki 6 satırda birer `no-restricted-syntax` hatası; aynı dosyanın geri kalanında, `tla-probe.test.ts`'te ve `apps/lint-probe/src/main.ts`'te sıfır. Ayrıştırıcı `await using` satırını reddederse o satır çıkarılır, beklenen sayı 5 olur ve durum D-GENEL 13'e yazılır (durma sebebi değil). Başka her sapma durma sebebidir (Karar 5).
  - Sonuç D-GENEL 13'e yazılır.
- [ ] Doğrulama: `pnpm install` → `pnpm install --frozen-lockfile` (değişiklik yok), Node 22 ile kurulum denemesi `engineStrict` yüzünden başarısız
- [ ] D-GENEL: Doğrulanacaklar 2, 4 (kalan kısım), 8, 9, 13'ün sonuçları

## M0.3 kontrol listesi (devops — uygulama ve paket iskeletleri)

Her paket için:
- [ ] `package.json`: `name` (`@club/*`), `"private": true`, `"type": "module"`, `exports` (kütüphanelerde, Karar 3.2 biçimi), betikler: `build`, `dev`, `lint`, `typecheck` (`tsc --noEmit -p tsconfig.json`; web: Karar 8), `test` (`vitest run`) (+ `clean`), `devDependencies`'de `typescript`, `eslint`, `vitest`, `@types/node` (`catalog:`)
- [ ] `tsconfig.json` (typecheck, `noEmit`, `src` + testler + `vitest.config.ts`) + `tsconfig.build.json` (web hariç; yalnızca `src`, `exclude`: `**/*.test.ts`, `**/*.test.tsx`, `**/*.int.test.ts`), `tooling/tsconfig/*` extend ediliyor
- [ ] `vitest.config.ts`: `defineProject`, `environment: 'node'`, `globals: false`, `include: ['src/**/*.test.{ts,tsx}']`, `exclude: ['src/**/*.int.test.ts']`, M0 için `passWithNoTests`. Eklenti yok (`unplugin-swc`, `vite-tsconfig-paths` yok)
- [ ] Boş giriş noktası. İş mantığı yok.

Pakete özgü:
- [ ] `packages/shared`: `src/index.ts` (boş dışa aktarım), `exports` içinde `.` ve `./env`. `src/env/` içeriği M0.4'te architect tarafından yazılır, devops yalnızca dizini ve `exports` girişini hazırlar. `sideEffects: false`. `dependencies`: `zod` (`catalog:`).
- [ ] `packages/db`: `package.json` (`db:generate`: `prisma generate`, `build`: `tsc -p tsconfig.build.json`); `devDependencies`: `prisma` (`catalog:`, 7.10.0); `dependencies`: `@prisma/client` (`catalog:`, 7.10.0), `@prisma/adapter-pg` (`7.10.0`), `pg` (tam sürüm; `@types/pg` gerekiyorsa dev). `src/index.ts` (M0: yalnızca üretilmiş `PrismaClient`, tipler ve gerekiyorsa adaptörün yeniden dışa aktarımı; M0.4 sonrasında doldurulur), `exports` içinde `.`. Şema ve `prisma.config.ts` M0.4'te architect tarafından yazılır. D-PRISMA tablosunu doldur (Doğrulanacaklar 7).
- [ ] `packages/server` (Karar 3.8): `package.json` (`"type": "module"`, `sideEffects: false`, `exports` yalnızca `"./package.json"`), `tsconfig.json` + `tsconfig.build.json` (`library.json`), `vitest.config.ts`, `src/index.ts` (`export {};`). `dependencies`: `@club/shared` (`workspace:*`). Modül dizinleri (`storage/`, `email/`) ve dış bağımlılıklar (AWS SDK vb.) M1'de eklenir.
- [ ] `packages/templates`: `package.json`, `tsconfig.json` + `tsconfig.build.json` (`library.json` + `"jsx": "react-jsx"`), `src/index.ts`. Satori/React bağımlılıkları M4'te pipeline-dev tarafından eklenir.
- [ ] `apps/api` (Nest 12 iskeleti repoya kopyalanmaz, bu listeye göre yazılır):
  - `package.json`: `"type": "module"`. `dependencies`: `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express` (`12.1.0`), `reflect-metadata` (`0.2.2`), `rxjs` (`7.8.2`), `@club/shared`, `@club/db` (`workspace:*`). `devDependencies`: `@nestjs/cli` (`12.0.7`; Karar 3.4 kuralıyla gerekirse `12.0.6`), `@nestjs/schematics` (`12.0.5`), `@nestjs/testing` (`12.1.0`), `@types/express` (`5.0.6`), `@types/node`, `typescript`, `vitest`, `eslint` (`catalog:`). `@swc/core`, `unplugin-swc`, `oxlint`, `@nestjs/mau`, `source-map-support`, `supertest` yok.
  - Betikler: `build`: `nest build`, `dev`: `nest start --watch`, `start`: `node dist/main.js`, `smoke`: `node dist/smoke.js`, `typecheck`: `tsc --noEmit -p tsconfig.json`, `test`: `vitest run`, `lint`: `eslint . --max-warnings=0`.
  - `nest-cli.json`: `"collection": "@nestjs/schematics"`, `"sourceRoot": "src"`, `"compilerOptions": { "builder": "tsc", "tsConfigPath": "tsconfig.build.json", "deleteOutDir": true }`. SWC ayarı (`builder: "swc"`, `typeCheck`) yok.
  - `tsconfig.json` ve `tsconfig.build.json`: `nest.json`'ı extend eder; build'de `rootDir: "src"`, `outDir: "dist"` (çıktı `dist/main.js`).
  - `src/main.ts` (env yükle → `getEnv()` → `NestFactory.create(AppModule)` → `listen(API_PORT)`; top-level await serbest), `src/app.module.ts` (boş), `src/smoke.ts` (S-1, S-2), `src/env.ts` (M0.5).
  - `src/di-smoke/smoke-dependency.service.ts` ve `src/di-smoke/smoke-consumer.service.ts` (S-3 koşullarına göre; `@Inject()` yok, değer import'u). Test dosyası M0.8'de test-engineer tarafından yazılır.
  - `vitest.config.ts`: ortak kalıp, eklenti yok.
- [ ] `apps/worker`: `"type": "module"`, `tsconfig` (`worker.json`), `src/main.ts` (env yükle → `getEnv()` → "hazır" logu, `SIGTERM`/`SIGINT` ile düzgün kapanış; M0'da Valkey'e bağlanmaz), `src/smoke.ts` (S-2), `dev`: `tsx watch src/main.ts`, `build`: `tsc -p tsconfig.build.json`, `start`: `node dist/main.js`, `smoke`: `node dist/smoke.js`. `dependencies`: `@club/shared`, `@club/db` (`workspace:*`); `devDependencies`: `tsx` (`catalog:`). `allowBuilds`'te `esbuild` bulunur.
- [ ] `apps/web`: `"type": "module"`, `next.config.ts` (`loadEnvConfig(<repo kökü>)`), `src/app/layout.tsx`, `src/app/page.tsx` (Türkçe yer tutucu metin, sabit renk yok), `src/env.server.ts`/`src/env.client.ts` (M0.5), `tsconfig.json` (`nextjs.json`, `paths: { "@/*": ["./src/*"] }`), `dev`: `next dev --port 3000`. Vitest'te `@/*` çözümü gerekirse `resolve: { tsconfigPaths: true }`.
- [ ] Doğrulama komutları ve beklenen sonuçlar:
  - `pnpm build` başarılı, ikinci çalıştırmada tamamı cache'ten (`FULL TURBO`)
  - `pnpm typecheck` (her pakette `tsc --noEmit`), `pnpm lint`, `pnpm test`, `pnpm format:check` başarılı
  - `pnpm dev`: web `http://localhost:3000` 200 dönüyor, api 3001'de ayakta, worker "hazır" logu yazıyor
  - `ls apps/api/dist/main.js apps/api/dist/smoke.js` ikisi de var; `grep -l "@club/shared/env" apps/api/dist/smoke.js` eşleşiyor, dosyada `require(` yok
  - Duman testleri S-1 ve S-2 (S-1'in `parseEnv` çağrısı M0.4 sonrasında, S-2 M0.4 şeması sonrasında tam haliyle çalışır; sonuçlar D-SMOKE'a)
  - `pnpm peers check` (ya da kurulum çıktısı): `typescript` için peer uyarısı yok
  - Lint kural testleri (geçici dosyalar, sonra silinir; her biri lint hatası vermeli): (1) `apps/api` → `@club/templates`, (2) `apps/web` → `@club/db`, (3) `apps/web` → `@club/server`, (4) `packages/shared` → `@club/server`, (5) `packages/db` → `@club/server`, (6) `packages/shared` → `@club/db`, (7) `apps/api` → `@club/shared/dist/index.js` (derin içe aktarma; api'ye özgü blok varken de yakalanmalı), (8) `apps/worker` → `@club/api`, (9) `packages/shared/src/` içinde tek satır `await Promise.resolve(1);` → `no-restricted-syntax`. Ayrıca (10) `apps/worker/src/` içinde aynı satır → `no-restricted-syntax` **yok**. Sonuçlar D-GENEL 13'e eklenir.
- [ ] D-GENEL: Doğrulanacaklar 5 (kalan), 6 (kurulan `@nestjs/cli` sürümü), 10'un sonuçları

## M0.4–M0.8 için notlar

- **M0.4 (architect):** `@club/shared/env` public API'si ve uygulaması (Karar 9). `packages/db/prisma/schema.prisma` (yalnızca `generator` + `datasource`; üretici `prisma-client`, çıktı `../src/generated`, ESM ve `.js` uzantı seçenekleri D-PRISMA'daki adlarla), `packages/db/prisma.config.ts` (kök `.env`'i `process.loadEnvFile` ile kendisi yükler; Prisma 7 otomatik yüklemez). Top-level await yok.
- **M0.5 (devops):** her uygulamada `env.ts` (memoized `getEnv()`, modül üst düzeyinde parse yok), kök `.env.example` (Karar 9 listesi, açıklamalı; `REDIS_URL` açıklamasında "Valkey bağlantısı" yazar). Ardından S-1/S-2 tam haliyle çalıştırılır, D-SMOKE doldurulur.
- **M0.6 (devops):** `docker-compose.dev.yml` (Karar 10): PostgreSQL, Valkey, MinIO + bucket init; portlar `127.0.0.1`'e bağlı, imajlar etiket + digest ile sabit, Valkey `maxmemory-policy noeviction` + `appendonly yes`. BullMQ **6.x** (worker `dependencies`, tam sürüm) ile kuyruk turu. D-VALKEY ve D-MINIO doldurulur.
- **M0.7 (devops):** GitHub Actions: `runs-on: ubuntu-24.04`, `permissions: contents: read`, `concurrency` ile eski çalışmalar iptal, `actions/checkout` → `pnpm/action-setup` (sürüm girdisi yok) → `actions/setup-node` (`node-version-file: .nvmrc`, `cache: pnpm`) → `pnpm install --frozen-lockfile` → `pnpm format:check` → `pnpm turbo run lint typecheck test build` (typecheck ve build ikisi de; Karar 8) → `pnpm smoke` → `.turbo/cache` için `actions/cache`. Action'lar SHA ile sabit. `db:generate` için sahte `DATABASE_URL` (gerekirse, D-PRISMA). `renovate.json` Karar 7 politikasına göre; TypeScript kuralı (`allowedVersions: "6.0.x"`, `typescript` + `typescript-eslint` + `@typescript-eslint/*` tek grup, catalog kuralından sonra), Vitest ve Prisma grupları dahil.
- **M0.8 (test-engineer):** S-3 test dosyası (`apps/api/src/di-smoke/smoke-consumer.service.test.ts`) ve negatif kontrol; her pakete duman testi, ardından `passWithNoTests` kaldırılır.

## Yeniden değerlendirme tetikleyicileri

- Node 26 LTS olduğunda ve native bağımlılıkların prebuilt desteği teyit edildiğinde (en erken 2027 başı).
- typescript-eslint'in peer aralığı TS 6.1'i kapsadığında (Karar 7 TypeScript kuralı, Karar 4 seçim kuralı yeniden işletilir).
- TypeScript 7 (yerel) typescript-eslint, Next, Nest CLI ve decorator metadata desteğiyle kararlı hale gelirse (ayrı ADR).
- Vitest 5'e geçiş: S-3 Vitest 5 ile geçtiğinde ayrı PR.
- ESLint 10: `eslint-config-next`'in bağımlılıkları ESLint 10'u desteklediğinde.
- S-3 başarısız olursa: Karar 6 dar geri dönüşü (`unplugin-swc` yalnızca Vitest'te).
- Nest provider'ları arasında döngüsel içe aktarma kaynaklı TDZ hatası görülürse: döngü tespiti için lint eklentisi (Karar 3.4).
- Build/typecheck süresi geliştirme döngüsünü belirgin biçimde yavaşlatırsa (Karar 3.2'deki kaynak koşulu alternatifi).
- M9: prod depolama ADR'si (Karar 10).
