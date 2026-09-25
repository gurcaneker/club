# ADR-0001: Yığın kararı — sürümler, modül sistemi, monorepo yapılandırması

- **Durum:** Kabul edildi (2026-09-25)
- **Tarih:** 2026-09-25
- **Yazan:** architect
- **Onaylayan:** kullanıcı (gurcaneker), 2026-09-25. Onayla birlikte gelen kararlar bu metne işlendi (TS seçim kuralı, CJS→ESM ve Prisma duman testleri, S1–S5).
- **İlgili:** CLAUDE.md "Teknoloji yığını", `docs/PLAN.md` M0, `docs/PROGRESS.md` M0.1–M0.7
- **Etkilediği görevler:** M0.2, M0.3 (devops), M0.4 (architect), M0.5–M0.7 (devops), M0.8 (test-engineer), M1.x (backend-dev: `packages/db/src/`, `packages/server`)

## Bu belgeyi kim değiştirebilir

- Kararlar, gerekçeler ve kontrol listeleri **architect**'e aittir.
- **devops bu ADR'de yalnızca "Doğrulama sonucu" alanlarını doldurabilir** (bölüm "Doğrulama sonuçları" altındaki tablolar ve `Doğrulama sonucu:` ile başlayan satırlar). Başka bir satırı değiştiremez. Bir kararın değişmesi gerekiyorsa aşağıdaki durma kuralı uygulanır.
- **Durma kuralı (devops):** devops **[D]** işaretli bir değeri doğrularken ADR'deki karardan farklı bir sonuç bulursa ya da TypeScript, Prisma, Valkey kontrollerinden veya duman testlerinden biri başarısız olursa **durur**, sonucu ilgili "Doğrulama sonucu" alanına yazar ve raporunda bildirir. Karar architect tarafından ADR'ye işlenmeden uygulanmaz. Tek istisna Karar 4'teki TypeScript seçim kuralıdır: o kural deterministiktir, devops sonucu yazıp uygular.

## Bağlam

Yığının bileşenleri CLAUDE.md'de sabittir (pnpm workspaces + Turborepo, TypeScript strict, Next.js App Router, NestJS, BullMQ, Prisma + PostgreSQL, Satori, sharp, kuyruk sunucusu, S3/MinIO, Anthropic SDK, Vitest/Testcontainers/Playwright, Docker Compose + Nginx, tek VPS). Bu ADR yığını yeniden seçmiyor; tek istisna kuyruk sunucusunun Redis yerine **Valkey** olmasıdır (Karar 10). Yaptığı iş, bu bileşenleri devops'un M0 iskeletini tahmin yürütmeden kurabileceği kadar somut hale getirmek: sürümler, sabitleme yöntemi, modül sistemi, tsconfig hiyerarşisi, lint, test, Turborepo görev grafiği ve env doğrulama deseni.

Geliştirme makinesi: Node v24.18.0, pnpm 11.18.0, corepack 0.35.0, Docker 29.8.1. Karar tarihi: 2026-09-25.

**Sürüm bilgisi uyarısı:** Bu ADR'de "doğrulanacak" (kısaca **[D]**) diye işaretlenen her bilgi, M0.2/M0.3 kurulumu sırasında devops tarafından resmi kaynaktan (sürüm notları, `npm view <paket> version`, `engines`/`peerDependencies`) teyit edilir ve kaynağıyla birlikte "Doğrulama sonuçları" bölümüne yazılır.

## Paket adları ve dizinler

| Dizin | Paket adı | `type` | Rol |
| --- | --- | --- | --- |
| `apps/web` | `@club/web` | `module` | Next.js (App Router) |
| `apps/api` | `@club/api` | `commonjs` | NestJS REST API |
| `apps/worker` | `@club/worker` | `module` | BullMQ worker (düz Node, Nest yok) |
| `packages/db` | `@club/db` | `module` (geri dönüş: `commonjs`, Karar 3.5) | Prisma şeması, üretilmiş client, tenant extension |
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
- Node 24'te `require(esm)` varsayılan olarak açık. CJS NestJS uygulamasının ESM iç paketleri yükleyebilmesi buna dayanıyor (bkz. Karar 3).
- `process.loadEnvFile()` yerleşik olduğu için dotenv bağımlılığı gerekmiyor.

**Alternatifler:**
- *Node 22 (Maintenance LTS):* EOL 2027-04-30 **[D]**, pilot ömrü içinde bitiyor. `require(esm)` desteği var ama daha az olgun. Reddedildi.
- *Node 26:* 2026-09-25'te henüz LTS değil (LTS geçişi Ekim 2026 sonu bekleniyor **[D]**). Yerel eklentilerin (argon2, sharp, @swc/core) prebuilt ikili dosyaları gecikebilir. Reddedildi. 2027'de yeniden değerlendirilir.
- *`engines`'i yalnızca major ile sabitlemek (`24.x`), `.nvmrc`'yi `24` bırakmak:* Geliştirici, CI ve Docker arasında yama sürümü sapması olur. Reddedildi. `engines` bilinçli olarak aralık tutulur ki yama farkı kurulumu kırmasın; tam eşitliği `.nvmrc`, CI ve Docker sağlar.
- *pnpm'in Node çalışma zamanı yönetimi (`devEngines.runtime`) **[D]**:* Umut verici, ama tek araç kilidini artırıyor ve Docker/CI akışında karşılığı yok. Reddedildi, ileride yeniden bakılabilir.

**Sonuçlar/riskler:**
- Alpine (musl) imaj kullanılmaz. sharp, argon2, Prisma ve @swc/core için glibc (Debian trixie) daha az sürpriz çıkarıyor.
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
- **Kurulum betikleri izin listesi:** pnpm 10'dan beri bağımlılıkların `postinstall` betikleri varsayılan olarak çalışmıyor. İzin listesi `pnpm-workspace.yaml` içinde açıkça tutulur **[D: pnpm 11'deki ayar adı, `onlyBuiltDependencies` mı yoksa yerini alan ayar mı]**. İlk liste: `prisma`, `@prisma/engines` (hâlâ betiği varsa), `@swc/core`, `esbuild`, `sharp`, `argon2` (M1'de eklenir). Listeye ekleme PR açıklamasında gerekçelendirilir. `@nestjs/core` gibi yalnızca reklam amaçlı betikler izinli değil.
- **Tedarik zinciri gecikmesi:** `minimumReleaseAge` 1440 dakika (1 gün) **[D: pnpm 11 ayar adı ve varsayılanı]**. Güvenlik yamaları gerektiğinde istisna listesine eklenir.

**Alternatifler:**
- *Yalnızca corepack:* Node 25'ten itibaren corepack Node dağıtımıyla birlikte gelmiyor **[D]**. Uzun vadede tek dayanak olamaz. Reddedildi. Yerelde kolaylık olarak kalıyor.
- *pnpm 10.x:* Makinede 11 var. 10'u sabitlemek geriye gitmek olur. Reddedildi.
- *Hash'siz `packageManager`:* Hash, indirilen pnpm ikilisinin bütünlüğünü doğruluyor. Hash'li hali seçildi.

**Sonuçlar/riskler:** pnpm 11'in ayar adları pnpm 10'dan farklı olabilir. Devops M0.2'de `pnpm config` ve sürüm notlarından teyit eder, sonucu "Doğrulama sonuçları → D-GENEL" tablosuna yazar.

## Karar 3 — Modül sistemi ve paket tüketim biçimi

### 3.1 Paket bazında modül sistemi

| Paket | Çıktı | `module` / `moduleResolution` | Derleyici (build) | Geliştirme | Not |
| --- | --- | --- | --- | --- | --- |
| `apps/api` (NestJS) | **CJS** | `node20` / `node20` | Nest CLI, `builder: "swc"` | `nest start --watch` (swc, `typeCheck: false`) | `experimentalDecorators` + `emitDecoratorMetadata`. Metadata'yı SWC üretir. SWC modül çıktısı `commonjs`. |
| `apps/worker` | **ESM** | `node20` / `node20` | `tsc -p tsconfig.build.json` | `tsx watch src/main.ts` | Decorator yok. Nest kullanılmaz. |
| `apps/web` (Next.js) | Next yönetir (ESM) | `ESNext` / `Bundler` | `next build` (Turbopack) | `next dev` | `noEmit`. Typecheck `tsc --noEmit` ile. |
| `packages/shared` | **ESM** | `node20` / `node20` | `tsc -p tsconfig.build.json` | `tsc -p tsconfig.build.json --watch` | `sideEffects: false` |
| `packages/db` | **ESM** (geri dönüş CJS, Karar 3.5) | `node20` / `node20` | `prisma generate` + `tsc` | `tsc --watch` | Üretici ve modül biçimi D-PRISMA sonucuna bağlı |
| `packages/server` | **ESM** | `node20` / `node20` | `tsc -p tsconfig.build.json` | `tsc -p tsconfig.build.json --watch` | Karar 3.8 |
| `packages/templates` | **ESM** | `node20` / `node20` | `tsc` (`jsx: react-jsx`) | `tsc --watch` | Satori; **yalnızca worker tüketir** |

Her `package.json`'da `"type"` açıkça yazılır (`"module"` veya `"commonjs"`). Node'un sözdizimi sezme davranışına güvenilmez.

**`module: node20` seçimi (api dahil tüm Node paketleri):**
- `node20`, TypeScript 5.9'da eklenen **sabit** moddur: Node 20.19+ davranışını modeller. Bu davranışa `require(esm)` dahildir; yani CJS bir dosyanın (api) ESM bir paketi (`@club/shared`, `@club/db`, `@club/server`) içe aktarması tip denetiminde hata vermez. api'nin tüm modül köprüsü buna dayanıyor.
- `nodenext` ise "Node'un en güncel davranışı" demek ve TS sürümüyle birlikte kayıyor (TS 6'da `target`'ı da `esnext`'e çekiyor **[D]**). `node20` hem TS 5.9.x'te hem 6.x'te aynı anlamı taşıdığı için Karar 4'teki TS seçimi hangi sonucu verirse versin yapılandırma değişmez.
- Node 24'ün `node20`'nin modellediği davranışa eklediği bir modül çözümleme farkı bu projede kullanılmıyor. `target`/`lib` ayrıca `ES2024` olarak açıkça yazıldığı için `node20`'nin ima ettiği `target` varsayılanı devreye girmez.
- Göreli içe aktarmalarda `.js` uzantısı ve `exports` haritası kuralları `nodenext` ile aynıdır.

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

- Koşul olarak yalnızca `types` ve `default` kullanılır; `import`/`require` ayrımı yapılmaz. Böylece api'nin `require` çağrısı da worker'ın `import`'u da aynı ESM dosyasına çözülür (tek modül örneği).
- Alt yol listesi `@club/shared` public API'sinin parçasıdır ve architect tarafından yönetilir. M0.3'te devops yalnızca `.` ve `./env` girişlerini oluşturur. `./state` ve `./theme` ilgili public API yazıldığında (M1.1b, M8.0) eklenir.
- `exports` dışındaki derin içe aktarmalar (`@club/shared/src/...`, `@club/shared/dist/...`) yasaktır ve lint ile engellenir.
- `declarationMap: true` sayesinde editörde "tanıma git" kaynak `.ts` dosyasına gider.
- ESM paketlerde göreli içe aktarmalar **`.js` uzantısıyla** yazılır (`import { x } from './x.js'`).

**Gerekçe:**
- api (Nest + SWC CLI) ve worker (tsc) bundler kullanmıyor, çalışma zamanında Node'un kendisi çözümlüyor. Kaynak TS'i doğrudan tüketmek bu iki uygulamada ya bundler ya da çalışma zamanı transpile gerektirir. Nest'te decorator metadata yüzünden esbuild tabanlı araçlar da elenir.
- Tek bir çözümleme yolu (dist) Next, Nest, worker ve Vitest'te aynı davranışı veriyor. Turborepo cache'iyle derleme maliyeti düşük.

### 3.3 CJS api'nin ESM paketleri tüketmesi

**Karar:** api CJS kalır. `@club/shared`, `@club/server` ve (varsayılan olarak) `@club/db` ESM paketleridir. api bunları Node 24'ün `require(esm)` desteğiyle yükler. Tip denetiminde bu köprüyü `module: node20` sağlar (Karar 3.1).

**Zorunlu kurallar:**
1. `packages/shared`, `packages/server` ve `packages/db` içinde **top-level await yasak.** Top-level await içeren bir ESM modülü `require` ile yüklenemez (`ERR_REQUIRE_ASYNC_MODULE`). Bu kural bu paketlerin bağımlılıkları için de geçerlidir; api'nin yüklediği bir bağımlılık top-level await içeriyorsa duman testi bunu yakalar.
2. **api, `@club/templates` paketini içe aktaramaz.** Satori'nin yoga/wasm bağımlılığı top-level await içeriyor olabilir **[D]**. Kural lint ile uygulanır (`no-restricted-imports`).
3. **web, `@club/db` ve `@club/server` paketlerini içe aktaramaz.** Veri erişimi ve sunucu adaptörleri API üzerinden kullanılır, tenant izolasyonu tek noktada (api) kalır. Kural lint ile uygulanır (Karar 5).
4. **CJS→ESM köprüsü duman testi** (M0 kabul kriteri, ayrıntı: "Duman testleri" bölümü). Üç kontrolün hepsi geçmelidir: `tsc --noEmit`, SWC build ve çalışma zamanı.

### 3.4 Neden api CJS, worker ESM?

- **NestJS 11 CJS olarak yayımlanıyor ve belgeleniyor** **[D: Nest 12'nin durumu ve ESM desteği]**. ESM'de `emitDecoratorMetadata`, döngüsel içe aktarmalarda (Nest modülleri arasında sık görülür) TDZ kaynaklı `ReferenceError` üretebiliyor. `forwardRef` bunu her zaman çözmüyor. CJS en az sürprizli yol.
- **Worker'da decorator yok.** Satori ve bazı modern bağımlılıklar ESM öncelikli. Worker'ı ESM yapmak bu bağımlılıkları doğal biçimde (gerekirse top-level await dahil) kullanmayı sağlıyor.
- **Worker neden Nest değil:** BullMQ'nun kendi API'si yeterli. Nest eklemek worker'ı CJS'e zorlar, bu da Satori tüketimini karmaşıklaştırır. Bedeli: api ve worker arasında paylaşılan sunucu kodu Nest modülü olarak değil, `packages/*` içinde düz TS olarak yazılıyor: tenant kapsamlı Prisma extension `packages/db/src/`'de, S3 istemcisi ve `EmailSender` `packages/server`'da (Karar 3.8). api bunları Nest provider'larına kendisi sarar.

### 3.5 Prisma client

**Varsayılan karar (D-PRISMA sonucuyla kesinleşir):**
- Prisma **7.x** **[D: kurulacak ana sürüm]**, yeni **`prisma-client`** üreteci **[D]**. Çıktı `packages/db/src/generated/` altında (gitignore'da), `@club/db`'nin `tsc` derlemesine dahil. Modül biçimi ESM, göreli import uzantısı `.js` (üretici seçenekleri, ör. `moduleFormat`, `importFileExtension` **[D: seçenek adları]**).
- PostgreSQL sürücü adaptörü `@prisma/adapter-pg` (+ `pg`) **[D: zorunlu mu]**.
- Bağlantı URL'si ve `.env` yükleme `packages/db/prisma.config.ts` içinde **[D: Prisma 7 `.env`'i otomatik yüklüyor mu]**.
- `prisma generate` Turborepo'da `db:generate` görevi olarak çalışır (Karar 8).

**D-PRISMA sonucuna göre @club/db modül biçimi (deterministik):**

| Sonuç | @club/db |
| --- | --- |
| Üretici `prisma-client`, ESM çıktıda (kendisi ve runtime bağımlılıkları) top-level await **yok** | ESM kalır (varsayılan) |
| Üretici `prisma-client`, ESM çıktıda top-level await **var** | Üretici `cjs` modül biçimine alınır, `@club/db` `"type": "commonjs"` olur, tsconfig'i `node20` ile CJS çıktı verir. ESM worker CJS paketi sorunsuz içe aktardığı için bu tek yönlü güvenli geri dönüş. |
| Kurulan ana sürümde `prisma-client` yok ya da kararlı değil, `prisma-client-js` kullanılıyor | `@club/db` ESM kalır; üretilmiş CJS client'ı (`@prisma/client`) ESM'den içe aktarır. Geri dönüş gerekmez. |

Hangi satır geçerli olursa olsun devops sonucu D-PRISMA'ya yazar; ilk satır dışındaki durumlarda **durur ve bildirir**, architect bu bölümü ve Karar 3.1 tablosunu günceller.

**@club/db dışa aktarımları:** M0'da yalnızca üretilmiş `PrismaClient` sınıfı, üretilmiş tipler ve (gerekiyorsa) sürücü adaptörünün yeniden dışa aktarımı. Client fabrikası ve tenant kapsamlı extension M1'de backend-dev tarafından `packages/db/src/` altında yazılır; tenant çözümleme sözleşmesi ADR-0002'de tanımlanır.

**Modelsiz şemayla (M0) duman testi:** M0.4'teki şema yalnızca `generator` + `datasource` içerir, model yoktur. Duman testi için geçici ya da sahte bir model **eklenmez**. Beklenen **[D]**: `prisma generate` modelsiz şemada da (uyarıyla) bir client üretir. Bu client'ta model erişimcileri olmaz, ama sınıf ve `$connect`/`$disconnect`/`$extends` gibi yerleşik üyeler bulunur. Duman testi bu yüzden yalnızca şunları kapsar: `@club/db`'yi içe aktarmak, `PrismaClient`'ı (gerekiyorsa adaptörle) örneklemek, `typeof client.$extends === 'function'` kontrolü ve `await client.$disconnect()`. `$connect` ya da sorgu çağrılmaz; bağlantı URL'si olarak ulaşılamaz sahte bir değer verilir (`postgresql://smoke:smoke@127.0.0.1:1/smoke`). Prisma istemcisi ve pg adaptörü bağlantıyı ilk sorguda ya da `$connect`'te açar **[D]**, bu yüzden veritabanı gerekmez. Modelsiz şemada `prisma generate` client üretmiyorsa devops durur ve bildirir.

### 3.6 tsconfig hiyerarşisi

`tooling/tsconfig/` altında düz JSON dosyaları bulunur (sahibi devops). Paketler bunları göreli yolla `extends` eder (`"extends": "../../tooling/tsconfig/library.json"`).

```
tooling/tsconfig/
  base.json          # strict + ek bayraklar (Karar 4), target/lib, isolatedModules, skipLibCheck
  node-esm.json      # extends base: module/moduleResolution node20, verbatimModuleSyntax, erasableSyntaxOnly, types ["node"]
  library.json       # extends node-esm: declaration, declarationMap, sourceMap, outDir dist, rootDir src   (shared, db, server, templates)
  worker.json        # extends node-esm: outDir dist, rootDir src, sourceMap
  nest.json          # extends base: module/moduleResolution node20, experimentalDecorators, emitDecoratorMetadata, types ["node"]
  nextjs.json        # extends base: module ESNext, moduleResolution Bundler, jsx preserve, lib DOM + DOM.Iterable, noEmit, plugins [next]
```

Her pakette iki dosya bulunur:
- `tsconfig.json`: editör ve `typecheck` için. `src` + testler + yapılandırma dosyaları, `noEmit: true`.
- `tsconfig.build.json` (web hariç): yalnızca `src`, testler hariç, emit açık.

**Kurallar:**
- `baseUrl` kullanılmaz (TS 6'da kullanımdan kalkıyor **[D]**). Paketler arası içe aktarma yalnızca paket adıyla (`@club/shared`). `paths` yalnızca `apps/web` içinde `@/*` → `./src/*` için kullanılabilir.
- `moduleResolution: node`/`node10` kullanılmaz. `nodenext` de kullanılmaz (Karar 3.1).
- TS project references (`composite`, `tsc -b`) kullanılmaz. Derleme sırasını Turborepo `^build` bağımlılığıyla yönetir.
- `types` her ön ayarda açıkça yazılır (`["node"]`). Vitest `globals: false` olduğu için test tipleri gerekmez.
- `target`/`lib`: `ES2024` (Node 24 tamamını destekliyor). `nextjs.json` ayrıca `DOM`, `DOM.Iterable` ekler.
- `verbatimModuleSyntax` ve `erasableSyntaxOnly` **yalnızca ESM ön ayarlarında** açık. `nest.json`'da kapalı: CJS çıktıda `verbatimModuleSyntax` ESM içe aktarma sözdizimini reddediyor, `erasableSyntaxOnly` ise Nest'in constructor parameter property kullanımını (`constructor(private readonly x: X)`) yasaklıyor.
- `erasableSyntaxOnly` sonucu olarak ESM paketlerde TS `enum` ve `namespace` kullanılmaz. Durum/rol değerleri `as const` nesne + birleşim tipi + `z.enum(...)` ile ifade edilir. Veritabanı enum'ları Prisma şemasındadır.
- @club/db CJS geri dönüşüne geçerse (Karar 3.5) `packages/db/tsconfig*.json` `library.json`'ı extend etmeye devam eder, yalnızca `verbatimModuleSyntax: false` ile ezer. Ayrı bir ön ayar eklenmez.

### 3.7 Satori / sharp / ESM-only bağımlılıkların etkisi

- Satori ve JSX yalnızca `packages/templates` ve worker'da. Worker ESM olduğu için Satori modül formatı **[D]** ne olursa olsun sorun çıkmaz.
- SVG→PNG/JPEG dönüşümü için yeni bir yerel bağımlılık (`@resvg/resvg-js`) eklemeden sharp kullanılabilir. Seçim M4'te pipeline-dev'in. Yeni bir yerel bağımlılık eklenirse pnpm build izin listesine girer.
- sharp CJS/ESM ikisini de destekliyor, prebuilt ikili dosyalar `@img/sharp-*` opsiyonel bağımlılıklarından geliyor.
- api'nin ihtiyaç duyduğu dış paketler (Nest, BullMQ, AWS SDK v3, argon2, zod) CJS tüketimine uygun.

### 3.8 `packages/server` (`@club/server`)

**Amaç:** api ve worker'ın ortak kullandığı, sunucuya özel (tarayıcıda çalışmayan, sır ya da ağ istemcisi taşıyan) adaptörler: S3 istemcisi (`storage`), `EmailSender` (`email`), ileride benzer adaptörler. Instagram ve caption adaptörleri **buraya girmez**; onlar kendi sahiplerinin dizinlerinde kalır (`apps/worker/src/publish/`, `apps/worker/src/caption/`).

**Modül biçimi:** ESM, önceden derlenmiş (Karar 3.2), `sideEffects: false`, top-level await yasak (Karar 3.3 kural 1).

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

**Karar (deterministik seçim kuralı):** devops M0.2'de aşağıdaki **dört aracın** TypeScript 6'yı resmi olarak desteklediğini doğrular:

1. **Next.js** (kurulacak 16.x sürümü)
2. **Nest CLI** (`@nestjs/cli`)
3. **SWC yolu:** `@swc/core` + `unplugin-swc` (api'nin build ve test derleyicisi)
4. **typescript-eslint**

- **Dördü de destekliyorsa:** TypeScript **6.x** (kurulum günündeki en güncel kararlı 6 minor'ın en güncel yaması, catalog'da tam sürüm).
- **Biri bile desteklemiyorsa:** TypeScript **5.9.x** (en güncel yama, catalog'da tam sürüm).

**"Resmi destek" tanımı:** aracın kurulacak sürümünde (a) `peerDependencies` içindeki `typescript` aralığı `6.x`'i kapsıyor, ya da (b) aracın `typescript`'e peer bağımlılığı yoksa (ör. SWC) resmi sürüm notu, changelog ya da dokümanı TS 6 desteğini açıkça belirtiyor. Yalnızca "uyarıyla çalışıyor" ya da topluluk issue'sunda "çalışıyor" demek destek sayılmaz. Belirsiz kalan araç **desteklemiyor** sayılır. Aracın kendi `dependencies`'inde sabit bir `typescript` sürümü taşıması (ör. Nest CLI) tek başına destek yok anlamına gelmez; bu durumda (b) ölçütü uygulanır.

Kural deterministik olduğu için devops sonucu uygular ve durmaz; sonucu ve her araç için kaynağı "Doğrulama sonuçları → D-TS" tablosuna yazar. Prisma bu kurala girmez; Prisma'nın seçilen TS sürümüyle uyumu D-PRISMA'da ayrıca kontrol edilir ve uyumsuzluk durma sebebidir.

Yapılandırma iki sürümde de aynı çalışacak biçimde yazılır (`module: node20` iki sürümde de var, Karar 3.1). TypeScript 7 (Go ile yazılmış yerel derleyici) ayrı bir ADR ile, ekosistem desteği teyit edilince değerlendirilir.

`tooling/tsconfig/base.json` bayrakları:

| Bayrak | Değer | Not |
| --- | --- | --- |
| `strict` | `true` | Açıkça yazılır (TS 6 varsayılanına güvenilmez) |
| `noUncheckedIndexedAccess` | `true` | Dizi/kayıt erişiminde `undefined` zorunlu ele alınır |
| `noImplicitOverride` | `true` | |
| `noImplicitReturns` | `true` | |
| `noFallthroughCasesInSwitch` | `true` | |
| `useUnknownInCatchVariables` | `true` | (`strict` içinde, açıkça yazılır) |
| `isolatedModules` | `true` | SWC, tsx ve Vite dosya bazlı transpile ediyor |
| `forceConsistentCasingInFileNames` | `true` | |
| `skipLibCheck` | `true` | |
| `resolveJsonModule` | `true` | |
| `esModuleInterop` | `true` | |
| `exactOptionalPropertyTypes` | `false` | Bkz. alternatifler |
| `noPropertyAccessFromIndexSignature` | `false` | Bkz. alternatifler |

**Gerekçe:** Tenant ve rıza verisinin işlendiği bir kod tabanında `noUncheckedIndexedAccess` ve `switch-exhaustiveness-check` (lint) en yüksek getirili ek güvenceler. Seçim kuralı, "hangi TS" sorusunu yoruma bırakmadan kurulum anındaki ekosistem durumuna bağlıyor.

**Alternatifler:**
- *`exactOptionalPropertyTypes: true`:* Prisma ve zod'un ürettiği tiplerle sürekli sürtünme yaratıyor. Reddedildi.
- *`noPropertyAccessFromIndexSignature: true`:* `process.env` erişimi zaten `env.ts` ile tek noktada toplandığı için getirisi düşük, gürültüsü yüksek. Reddedildi.
- *TS 7 (yerel) şimdi:* Decorator metadata, typescript-eslint ve Next desteği henüz teyit edilmedi **[D]**. Reddedildi. Hızlı typecheck için isteğe bağlı olarak denenebilir, CI kapısı olmaz.
- *Her koşulda TS 5.9:* Güvenli ama TS 6'nın `baseUrl`/varsayılan değişikliklerine geçişi erteler. Seçim kuralı, destek varsa 6'yı alıyor.

**Sonuçlar/riskler:** TS 6'nın varsayılan değişiklikleri (`types`, `rootDir`, `strict` vb. **[D]**) nedeniyle tüm bu ayarlar ön ayarlarda açıkça yazılır. Hiçbir davranış varsayılana bırakılmaz.

## Karar 5 — Lint ve format

**Karar:**
- **ESLint flat config**, kökte **tek dosya**: `eslint.config.mjs`. Paket bazında ESLint yapılandırma dosyası yok. Paylaşılan config paketi de yok. Alanlara özgü kurallar aynı dosyada `files` glob'larıyla tanımlanır.
- ESLint major: typescript-eslint ve `eslint-config-next`/`@next/eslint-plugin-next` tarafından desteklenen en güncel major **[D: 10.x mi 9.x mi]**.
- **typescript-eslint v8** **[D: güncel major]**, tip farkındalıklı lint: `parserOptions.projectService: true`, `tsconfigRootDir: import.meta.dirname`.
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
- `apps/api/**` override'ları: `consistent-type-imports` kapalı. Bu kural, Nest DI'nin ihtiyaç duyduğu sınıf importlarını `import type`'a çevirerek decorator metadata'yı bozabiliyor. `@typescript-eslint/no-extraneous-class` kapalı (Nest modülleri boş sınıf).
- `apps/web/**`: Next ESLint eklentisinin flat config'i (`core-web-vitals`). Next 16 ile `next lint` kaldırıldı, lint doğrudan `eslint` ile çalışır **[D]**.
- Test dosyaları (`**/*.test.ts(x)`): `@typescript-eslint/unbound-method` kapalı.

**Prettier ayarı** (`.prettierrc.json`): `{ "singleQuote": true, "trailingComma": "all", "printWidth": 100 }`. `.prettierignore`: `dist`, `.next`, `coverage`, `.turbo`, `pnpm-lock.yaml`, `**/generated/**`.

**Gerekçe:** Flat config dosyası kökten aşağıya doğru bulunduğu için tek dosya bütün monorepoyu kapsıyor. Tek kural kaynağı, tek bağımlılık listesi. Paylaşılan config paketi yalnızca birden çok repo arasında paylaşımda anlamlı olurdu.

**Alternatifler:**
- *`packages/eslint-config` paylaşılan paketi:* Fazladan paket, sürümleme ve sahiplik yükü getiriyor. Reddedildi.
- *Biome (lint + format):* Hızlı ama tip farkındalıklı kuralları (`no-floating-promises`, exhaustiveness) typescript-eslint kadar olgun değil. Next eklentisi de yok **[D]**. Reddedildi.
- *Git hook'ları (husky/lefthook + lint-staged):* M0'da eklenmez. Kalite kapısı CI. İstenirse ayrı karar.

**Sonuçlar/riskler:**
- ESLint eklentileri kök `devDependencies`'de bulunur (config kökten çözümlüyor). `eslint` ikilisi ise her paketin `devDependencies`'inde `catalog:` ile yer alır ki `pnpm --filter <paket> lint` çalışsın.
- Tip farkındalıklı lint, bağımlı paketlerin `dist/*.d.ts`'ine ihtiyaç duyduğu için `lint` görevi `^build`'e bağlı.

## Karar 6 — Test (Vitest)

**Karar:**
- **Vitest 4.x** **[D: güncel major; `test.projects` API'si]**.
- **Paket bazında yapılandırma:** her pakette `vitest.config.ts` (`defineProject`), `environment: 'node'`, `globals: false` (testler `import { describe, it, expect } from 'vitest'` kullanır). Turborepo her paketin `test` görevini ayrı çalıştırır ve cache'ler. Bu dosyaların sahibi M0'da devops, M0 sonrasında test-engineer'dır.
- **Kök `vitest.config.ts`:** `test.projects: ['apps/*', 'packages/*']`. Yalnızca IDE ve tek komutla tüm testleri çalıştırmak için. CI Turborepo üzerinden çalışır. `vitest.workspace.ts` kullanılmaz (Vitest 3.2'de kullanımdan kalktı).
- **Dosya adlandırma:**
  - Unit: `src/**/*.test.ts(x)` → `test` görevi. M0 CI'da çalışan tek test katmanı.
  - Entegrasyon (Testcontainers): `src/**/*.int.test.ts` → ayrı `vitest.int.config.ts` ve `test:int` görevi. Docker gerektirir, M1'de CI'a ayrı job olarak eklenir. Unit config bu dosyaları `exclude` eder.
  - E2E (Playwright): `e2e/**/*.spec.ts` → `e2e` görevi. Playwright kurulumu ilk e2e ihtiyacında (M3) test-engineer tarafından yapılır. `turbo.json`'da görev tanımı M0'da bulunur.
- **NestJS + Vitest:** Vite'ın varsayılan transpile'ı (esbuild/Oxc) `emitDecoratorMetadata` üretmediği için **yalnızca `apps/api/vitest.config.ts`** içinde `unplugin-swc` eklentisi kullanılır. SWC ayarları `legacyDecorator: true`, `decoratorMetadata: true` **[D: Vitest 4 / Vite sürümüyle uyum; Vite'ın Oxc dönüştürücüsü decorator metadata'yı artık destekliyorsa eklenti gereksiz olabilir]**.
- **Testlerde env:** testler geliştiricinin `.env` dosyasını okumaz. Gerekli env değerleri `vitest.config.ts` içinde `test.env` ile ya da test içinde açıkça verilir.
- **M0 geçici ayarı:** M0'da test dosyası olmayan paketlerde `vitest run --passWithNoTests`. Bayrak M0.8'de test-engineer her pakete duman testi ekledikten sonra kaldırılır.
- Kapsam raporu: `@vitest/coverage-v8`. M0'da eşik tanımlanmaz.

**Alternatifler:**
- *Jest (Nest'in varsayılanı):* İkinci test koşucusu, ESM desteği zayıf. Reddedildi.
- *Yalnızca kökte `projects` ile tek Vitest çağrısı:* Turborepo paket bazlı cache'i kaybolur. Reddedildi.

**Sonuçlar/riskler:** `unplugin-swc` ile Vitest'in güncel sürümü uyumsuz çıkarsa geri dönüş planı: api testleri için `@swc/core` ile yazılmış küçük bir Vite eklentisi. Bu durum ADR'ye işlenir.

## Karar 7 — Bağımlılık sürüm sabitleme politikası

**Karar:**
- **Tam sürüm:** tüm doğrudan bağımlılıklar tam sürümle yazılır (`^`/`~` yok). `pnpm-workspace.yaml` → `savePrefix: ''` **[D: pnpm 11 ayar adı]**.
- **pnpm catalog:** birden fazla pakette kullanılan her bağımlılık `pnpm-workspace.yaml` → `catalog:` altında tam sürümle tanımlanır, paketlerde `"catalog:"` ile referans verilir. Asgari catalog: `typescript`, `@types/node`, `zod`, `vitest`, `@vitest/coverage-v8`, `eslint`, `prettier`, `tsx`, `prisma`, `@prisma/client` (ya da seçilen Prisma sürümündeki karşılığı, D-PRISMA), `react`, `react-dom`, `@types/react`. Tek pakette kullanılan bağımlılıklar (ör. `@nestjs/*`, `next`, `bullmq`, `satori`) o paketin `package.json`'ında tam sürümle durur.
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
| `typecheck` | `^build`, `db:generate` | — | evet | Web: `next typegen && tsc --noEmit` **[D: `next typegen`]**. Diğerleri: `tsc --noEmit` |
| `lint` | `^build`, `db:generate` | — | evet | `eslint . --max-warnings=0` |
| `test` | `^build`, `db:generate` | `coverage/**` | evet | Yalnızca unit |
| `smoke` | `build` | — | **hayır** | Yalnızca api ve worker: `node dist/smoke.js` (Duman testleri bölümü) |
| `test:int` | `^build`, `db:generate` | — | **hayır** | Docker gerektirir |
| `e2e` | `build` | `playwright-report/**`, `test-results/**` | **hayır** | M3'ten itibaren |
| `dev` | `^build`, `db:generate` | — | **hayır**, `persistent: true` | Paketler `tsc --watch`, uygulamalar dev sunucusu |
| `clean` | — | — | **hayır** | |

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
- **Her uygulamada `src/env.ts` (M0.5, devops):** uygulamanın kendi şemasını ortak parçalardan oluşturur ve **bellekte saklanan (memoized) bir `getEnv()` fonksiyonu** dışa aktarır. Modül üst düzeyinde parse **yapılmaz**. ESM ve SWC-CJS içe aktarmaları öne taşıdığı için üst düzey parse `.env` yüklenmeden çalışırdı.
- **Giriş noktası sırası (api ve worker `main.ts`):** önce `loadEnvFileIfPresent(<repo kökü>/.env)`, sonra `getEnv()` (hatalıysa süreç açık bir mesajla sonlanır), sonra uygulama başlatılır. api'de env, Nest'e özel bir provider token'ı (`ENV`) ile verilir. `@nestjs/config` kullanılmaz.
- **Web:** `next.config.ts` içinde `@next/env` → `loadEnvConfig(<repo kökü>)` ile kök `.env` yüklenir. Sunucu şeması (`src/env.server.ts`, `import 'server-only'`) ile istemci şeması (`src/env.client.ts`, yalnızca `NEXT_PUBLIC_*`) ayrıdır. Sunucu env'i `instrumentation.ts` `register()` içinde doğrulanır.
- **Prisma CLI:** `packages/db/prisma.config.ts` kök `.env`'i yükler **[D: Prisma config API'si]**.
- **Tek `.env.example`** repo kökünde. Her değişken yorumla açıklanır (hangi uygulama kullanıyor, zorunlu mu, varsayılanı ne, sır mı). M0'daki değişkenler: `NODE_ENV`, `LOG_LEVEL`, `WEB_PORT`, `API_PORT`, `DATABASE_URL`, `REDIS_URL` (Valkey bağlantısı; ad bilinçli olarak `REDIS_URL` kalır, Karar 10), `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_FORCE_PATH_STYLE`, `CAPTION_MODE`, `EMAIL_MODE`, `INSTAGRAM_MODE`, `ANTHROPIC_API_KEY` (mock modda boş), `ANTHROPIC_MODEL` (mock modda boş). Sonraki taşların değişkenleri (JWT sırları, şifreleme anahtarı, SMTP, Meta) o taşlarda eklenir.

**Alternatifler:**
- *`@t3-oss/env-nextjs` / `envalid`:* Web için uygun ama api ve worker için ikinci bir desen demek. Reddedildi. Tek desen `@club/shared/env`.
- *`@nestjs/config`:* Nest'e özgü, worker'da yok. Reddedildi.
- *Uygulama başına ayrı `.env` dosyası:* PLAN tek `.env.example` istiyor, ortak değişkenler de tekrar ederdi. Reddedildi.
- *`dotenv` paketi:* Node 24'ün `process.loadEnvFile` fonksiyonu yetiyor. Reddedildi.

**Sonuçlar/riskler:** Prod'da (`NODE_ENV=production`) `.env` dosyası okunmaz. Env, Docker Compose tarafından verilir (M9).

## Karar 10 — Altyapı servisleri (dev compose): PostgreSQL, Valkey, MinIO

**Kuyruk sunucusu: Valkey (S3 kararı).** CLAUDE.md'deki "Redis (kuyruk)" ifadesinin yerine **Valkey** kullanılır (BSD lisanslı, Redis protokolüyle uyumlu). Kuyruk kütüphanesi BullMQ olarak kalır.
- İmaj: resmi `valkey/valkey`, kurulum günündeki en güncel kararlı sürüm **[D: sürüm; 8.x ya da 9.x]**, **etiket + digest** ile sabit.
- Yapılandırma: `maxmemory-policy noeviction` (BullMQ zorunluluğu), `appendonly yes` (dev'de de kuyruk kaybını önlemek için).
- Bağlantı env değişkeni adı **`REDIS_URL`** olarak kalır (`redis://` şeması). Gerekçe: BullMQ/ioredis ekosistemindeki yerleşik ad, Valkey protokol olarak Redis uyumlu, ileride geri dönüş ya da yönetilen bir Redis uyumlu servis kullanılırsa ad değişmez. Kod ve dokümanda "Redis" kelimesi yalnızca protokol/env adı için geçer; sunucu adı Valkey'dir.
- **[D] BullMQ uyumluluğu:** devops, kurulacak BullMQ sürümünün seçilen Valkey sürümünü desteklediğini resmi kaynaktan (BullMQ dokümanı/sürüm notları) teyit eder ve M0.6'da dev compose'daki Valkey'e karşı basit bir kuyruk turu (iş ekle → işle → tamamlandı) çalıştırır. Olumsuzsa durur ve bildirir. Sonuç D-VALKEY'e yazılır.

**PostgreSQL:** `postgres:18.x` (tam etiket + digest) **[D: veri dizini yolu değişikliği]**.

**MinIO (S1 kararı):** Geliştirmede **doğrulanmış son MinIO imajı digest ile sabitlenir** (yalnızca yerel, `127.0.0.1`'e bağlı, dış ağa kapalı). Bucket init servisi aynı şekilde sabitlenmiş istemci imajıyla çalışır **[D: istemci imajının durumu]**. Sonuç D-MINIO'ya yazılır. **Prod depolama M9'da ayrı bir ADR ile belirlenir**; adaylar arasında **Contabo Object Storage** (S3 uyumlu) bulunur. Kod S3 API'sine yazıldığı için (`@club/server/storage`, `S3_*` env) sağlayıcı değişikliği kod değişikliği gerektirmemelidir.

---

## Duman testleri (M0 kabul kriteri)

Kod `apps/api/src/smoke.ts` ve `apps/worker/src/smoke.ts` dosyalarındadır. Bu dosyalar build'e dahildir, `smoke` betiğiyle (`node dist/smoke.js`) çalışır, iş mantığı içermez. M0'da devops yazar; M0 sonrasında sahibi api için backend-dev, worker için pipeline-dev olur. Başarıda `smoke ok` yazıp 0 ile çıkar, herhangi bir hatada 0 dışı kodla çıkar. `.env` okumaz; gereken değerleri kod içinde sabit sahte değerler olarak verir.

### S-1 — CJS→ESM köprüsü (api → @club/shared)

`apps/api/src/smoke.ts`, `@club/shared/env`'den `parseEnv` ve `envSchemas`'ı içe aktarır ve gerçek bir çağrı yapar: sabit bir `source` nesnesiyle (ör. `{ NODE_ENV: 'test', API_PORT: '3001' }`) geçerli bir şemayı ayrıştırıp beklenen değeri kontrol eder, ardından geçersiz bir `source` ile `EnvValidationError` fırlatıldığını kontrol eder (`instanceof` kontrolü, iki modül örneği olmadığını da gösterir).

Üç kontrolün **hepsi** geçmelidir:

| # | Kontrol | Komut | Başarı ölçütü |
| --- | --- | --- | --- |
| 1 | Tip denetimi | `pnpm --filter @club/api typecheck` (`tsc --noEmit`, `module: node20`) | Hata yok; özellikle TS1479 (CJS'den ESM içe aktarma) yok |
| 2 | SWC build | `pnpm --filter @club/api build` (`nest build`, SWC, CJS çıktı) | Başarılı; çıktıda `@club/shared/env` için `require(...)` çağrısı var |
| 3 | Çalışma zamanı | `node apps/api/dist/smoke.js` ve ayrıca `node apps/api/dist/main.js` (geçerli env ile) | `smoke.js`: `smoke ok`, çıkış kodu 0. `main.js`: 3001'de dinlemeye başlar, `ERR_REQUIRE_ESM`/`ERR_REQUIRE_ASYNC_MODULE` yok, `ExperimentalWarning` yok |

Çıktı yolu varsayımı `apps/api/dist/main.js` ve `apps/api/dist/smoke.js`'tir **[D: Nest CLI SWC builder'ın gerçek çıktı yolu, ör. `dist/src/main.js` olabilir]**. Gerçek yol farklıysa devops `start`/`smoke` betiklerini gerçek yola göre yazar ve yolu D-SMOKE'a işler; bu, durma sebebi değildir.

### S-2 — @club/db içe aktarma (api ve worker)

M0.4'te şema ve `prisma.config.ts` yazıldıktan sonra her iki `smoke.ts` ayrıca şunu yapar (Karar 3.5 "Modelsiz şemayla duman testi"): `@club/db`'den `PrismaClient`'ı (ve gerekiyorsa adaptörü) içe aktarır, sahte bağlantı URL'siyle örnekler, `$extends`'in fonksiyon olduğunu kontrol eder, `$disconnect()` çağırır. Veritabanı çalışmıyor olmalı ya da olmasa da test geçmelidir.

| # | Tüketici | Yol | Başarı ölçütü |
| --- | --- | --- | --- |
| 1 | api (CJS) | `require(esm)` (ya da @club/db CJS geri dönüşündeyse düz `require`) | typecheck + build + `node apps/api/dist/smoke.js` başarılı |
| 2 | worker (ESM) | `import` | typecheck + build + `node apps/worker/dist/smoke.js` başarılı |

CI (M0.7) `pnpm turbo run lint typecheck test build` sonrasında `pnpm smoke` çalıştırır.

---

## Sürüm özeti

| Bileşen | Karar | Doğrulama |
| --- | --- | --- |
| Node.js | 24.18.0 (LTS), `engines` `>=24.11.0 <25` | LTS/EOL tarihleri [D] |
| pnpm | 11.18.0, `packageManager` + hash | ayar adları [D] |
| TypeScript | 6.x ya da 5.9.x — Karar 4 seçim kuralı | D-TS |
| Turborepo | 2.x | [D] |
| Next.js / React | 16.x / 19.x | [D] |
| NestJS | 11.x | 12.x durumu [D] |
| Prisma | 7.x (`prisma-client` üreteci, `@prisma/adapter-pg`) — varsayılan | D-PRISMA |
| BullMQ | 5.x | [D], Valkey uyumu D-VALKEY |
| zod | 4.x | — |
| Vitest | 4.x (+ `unplugin-swc` yalnızca api) | [D] |
| ESLint / typescript-eslint | en güncel desteklenen major / 8.x | [D] |
| Prettier | 3.x | — |
| Docker temel imajı (M9) | `node:24.18.0-trixie-slim` + digest | etiket [D] |
| PostgreSQL (dev compose) | 18.x + digest | Veri dizini yolu değişikliği [D] |
| Valkey (dev compose) | en güncel kararlı + digest, `noeviction` | D-VALKEY |
| MinIO (dev compose) | doğrulanmış son imaj + digest | D-MINIO |

"x" ile yazılan sürümler catalog'a kurulum günündeki en güncel yama ile **tam sürüm** olarak girer.

## Doğrulanacaklar (M0.2/M0.3/M0.6'da devops teyit eder)

1. Node 24 LTS takvimi (Active LTS bitişi, EOL 2028-04-30) ve Node 26'nın LTS'e geçiş tarihi.
2. Node 25+ dağıtımlarında corepack'in olmaması.
3. Node 24'te `require(esm)` desteğinin kararlılık durumu (uyarı üretip üretmediği).
4. pnpm 11 ayar adları: `engineStrict`, `savePrefix`, build betiği izin listesi (`onlyBuiltDependencies` ya da karşılığı), `minimumReleaseAge`, `packageManager` ile kendi sürüm yönetimi, `.npmrc` desteği.
5. **TypeScript seçimi (Karar 4 kuralı):** dört aracın TS 6 desteği → D-TS. Ayrıca TS 6 varsayılan değişiklikleri (`types`, `rootDir`, `strict`, `baseUrl` kullanımdan kalkması) ve `module: node20`'nin seçilen sürümde CJS'den ESM içe aktarmaya izin vermesi.
6. NestJS'in güncel major'ı (11 mi, 12 mi) ve ESM durumu. Nest CLI SWC builder'ının çıktı yolu.
7. **Prisma** → D-PRISMA: ana sürüm, üretici, üretilen istemcinin modül biçimi, top-level await, sürücü adaptörü zorunluluğu, `prisma.config.ts` ve `.env` yükleme, `prisma generate` için `DATABASE_URL` gerekip gerekmediği (CI'da sahte değer), modelsiz şemada client üretimi, seçilen TS sürümüyle uyum.
8. Vitest 4 `test.projects` API'si; `unplugin-swc`'nin güncel Vite ile uyumu ya da Oxc'nin decorator metadata desteği.
9. ESLint 10'un durumu ve `eslint-config-next`'in flat config desteği. Next 16'da `next lint`'in kaldırılması. `next typegen` komutu.
10. Satori'nin modül formatı ve yoga bağımlılığında top-level await bulunup bulunmadığı (yalnızca bilgi amaçlı; worker ESM olduğu için karar değişmez).
11. Docker imajları: `node:24.18.0-trixie-slim`, `postgres:18.x` (veri dizini yolu), Valkey → D-VALKEY, MinIO → D-MINIO.
12. **Valkey** → D-VALKEY: sürüm, digest, BullMQ uyumluluğu.

## Doğrulama sonuçları (devops doldurur)

> Bu bölümdeki tablolar ve `Doğrulama sonucu:` satırları devops tarafından doldurulur. Her satıra kaynak (URL, `npm view` çıktısı, `peerDependencies` alıntısı, sürüm notu) yazılır. Tarih: doğrulamanın yapıldığı gün.

### D-TS — TypeScript seçimi (M0.2)

| Araç | Kurulan sürüm | TS 6 resmi desteği (evet/hayır) | Kaynak |
| --- | --- | --- | --- |
| Next.js | | | |
| Nest CLI (`@nestjs/cli`) | | | |
| SWC yolu (`@swc/core` + `unplugin-swc`) | | | |
| typescript-eslint | | | |

Doğrulama sonucu: _(doldurulacak: seçilen TypeScript tam sürümü, 6.x mi 5.9.x mi, tarih)_

### D-PRISMA — Prisma (M0.3/M0.4)

| Alan | Sonuç | Kaynak |
| --- | --- | --- |
| Kurulan Prisma ana/tam sürümü | | |
| Üretici (`prisma-client` / `prisma-client-js`) | | |
| Üretilen istemcinin modül biçimi (ESM/CJS) ve ilgili üretici seçenekleri | | |
| Üretilen istemcide ya da runtime'ında top-level await | | |
| Sürücü adaptörü gerekli mi (`@prisma/adapter-pg`) | | |
| `prisma.config.ts` ve `.env` yükleme davranışı | | |
| `prisma generate` için `DATABASE_URL` gerekli mi | | |
| Modelsiz şemada client üretiliyor mu | | |
| Seçilen TS sürümüyle uyum | | |

Doğrulama sonucu: _(doldurulacak: Karar 3.5 tablosunun hangi satırı geçerli, @club/db modül biçimi)_

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

Doğrulama sonucu: _(doldurulacak)_

### D-GENEL — Diğer [D] maddeleri (Doğrulanacaklar 1–4, 6, 8–11)

| # | Sonuç | Kaynak |
| --- | --- | --- |
| | | |

## Kapanan açık sorular

- **S1 — MinIO:** Kabul (a). Dev'de doğrulanmış son imaj digest ile sabit; prod depolama M9'da ayrı ADR, adaylara Contabo Object Storage eklendi (Karar 10).
- **S2 — Renovate:** Kabul. Kullanıcı GitHub App'i kuracak; politika Karar 7'de, `renovate.json` M0.7'de devops.
- **S3 — Kuyruk sunucusu:** Valkey (Karar 10). `REDIS_URL` adı korunur.
- **S4 — Paylaşılan sunucu kodu:** (a) Tenant Prisma extension `packages/db/src/`; şema/migration architect, `packages/db/src/` backend-dev. (b) Yeni paket `packages/server` (`@club/server`), public API architect, uygulama backend-dev; web'den içe aktarma lint ile yasak (Karar 3.8, Karar 5).
- **S5 — Sahiplik:** `tooling/tsconfig/` devops; paketlerdeki `vitest.config.ts` M0 sonrasında test-engineer.

## M0.2 kontrol listesi (devops — kök iskelet)

- [ ] `.nvmrc` → `24.18.0`
- [ ] **TypeScript seçimi:** Karar 4 kuralını uygula, D-TS tablosunu doldur, seçilen sürümü catalog'a yaz
- [ ] Kök `package.json`: `"name": "club"`, `"private": true`, `"type": "module"`, `packageManager` (`corepack use pnpm@11.18.0` ile), `engines.node: ">=24.11.0 <25"`, Karar 8'deki betikler (`smoke` dahil), kök `devDependencies` (`turbo`, `typescript`, `eslint`, `@eslint/js`, `typescript-eslint`, `eslint-config-prettier`, `@next/eslint-plugin-next` ya da `eslint-config-next`, `globals`, `prettier`, `vitest`), hepsi tam sürüm ya da `catalog:`
- [ ] `pnpm-workspace.yaml`: `packages: ['apps/*', 'packages/*']`, `catalog:` (Karar 7 listesi, tam sürümler), `engineStrict`, `savePrefix: ''`, build izin listesi, `minimumReleaseAge` (ayar adları [D])
- [ ] `pnpm-lock.yaml` üretildi ve commitlenecek
- [ ] `turbo.json`: Karar 8 tablosu (`smoke` dahil), `globalDependencies`, `web#build` için `env: ["NEXT_PUBLIC_*"]`
- [ ] `tooling/tsconfig/{base,node-esm,library,worker,nest,nextjs}.json` (Karar 3.6 ve 4; Node ön ayarlarında `module`/`moduleResolution` `node20`)
- [ ] `eslint.config.mjs` (Karar 5: taban, ek kurallar, `no-restricted-imports` tam listesi, api/web/test override'ları, `eslint-config-prettier` en sonda, ignore: `dist`, `.next`, `coverage`, `**/generated/**`)
- [ ] `.prettierrc.json`, `.prettierignore` (Karar 5)
- [ ] Kök `vitest.config.ts` (`test.projects`)
- [ ] `.editorconfig` (utf-8, lf, 2 boşluk), `.gitattributes` (`* text=auto eol=lf`)
- [ ] `.gitignore` güncellemesi: `node_modules`, `dist`, `.next`, `.turbo`, `coverage`, `packages/db/src/generated`, `.env`, `.env.*` (ama `!.env.example`), `playwright-report`, `test-results`
- [ ] Doğrulama: `pnpm install` → `pnpm install --frozen-lockfile` (değişiklik yok), Node 22 ile kurulum denemesi `engineStrict` yüzünden başarısız
- [ ] D-GENEL: Doğrulanacaklar 1–4, 8, 9'un sonuçları

## M0.3 kontrol listesi (devops — uygulama ve paket iskeletleri)

Her paket için:
- [ ] `package.json`: `name` (`@club/*`), `"private": true`, açık `"type"`, `exports` (kütüphanelerde, Karar 3.2 biçimi), betikler: `build`, `dev`, `lint`, `typecheck`, `test` (+ `clean`), `devDependencies`'de `typescript`, `eslint`, `vitest` (`catalog:`)
- [ ] `tsconfig.json` (typecheck, `noEmit`) + `tsconfig.build.json` (web hariç), `tooling/tsconfig/*` extend ediliyor
- [ ] `vitest.config.ts` (`defineProject`, `environment: 'node'`, `globals: false`, unit `include`/`exclude`, M0 için `passWithNoTests`)
- [ ] Boş giriş noktası. İş mantığı yok.

Pakete özgü:
- [ ] `packages/shared`: `src/index.ts` (boş dışa aktarım), `exports` içinde `.` ve `./env`. `src/env/` içeriği M0.4'te architect tarafından yazılır, devops yalnızca dizini ve `exports` girişini hazırlar. `sideEffects: false`. `dependencies`: `zod` (`catalog:`).
- [ ] `packages/db`: `package.json` (`db:generate`: `prisma generate`, `build`: `tsc -p tsconfig.build.json`), `prisma`/`@prisma/*` (ve D-PRISMA'ya göre `@prisma/adapter-pg` + `pg`) bağımlılıkları, `src/index.ts` (M0: yalnızca üretilmiş `PrismaClient` ve tiplerin yeniden dışa aktarımı; M0.4 sonrasında doldurulur), `exports` içinde `.`. Şema ve `prisma.config.ts` M0.4'te architect tarafından yazılır. D-PRISMA tablosunu doldur.
- [ ] `packages/server` (Karar 3.8): `package.json` (`"type": "module"`, `sideEffects: false`, `exports` yalnızca `"./package.json"`), `tsconfig.json` + `tsconfig.build.json` (`library.json`), `vitest.config.ts`, `src/index.ts` (`export {};`). `dependencies`: `@club/shared` (`workspace:*`). Modül dizinleri (`storage/`, `email/`) ve dış bağımlılıklar (AWS SDK vb.) M1'de eklenir. Lint: web'den ve shared/db'den içe aktarma yasağı çalışıyor.
- [ ] `packages/templates`: `package.json`, `tsconfig` (`library.json` + `"jsx": "react-jsx"`), `src/index.ts`. Satori/React bağımlılıkları M4'te pipeline-dev tarafından eklenir.
- [ ] `apps/api`: `"type": "commonjs"`, `nest-cli.json` (`compilerOptions.builder: "swc"`, `typeCheck: false`), `tsconfig` (`nest.json`, `module`/`moduleResolution` `node20`), `src/main.ts` (env yükle → `getEnv()` → `NestFactory.create(AppModule)` → `listen(API_PORT)`), `src/app.module.ts` (boş), `src/smoke.ts` (Duman testleri S-1, S-2), `src/env.ts` (M0.5), `vitest.config.ts` + `unplugin-swc`, `@swc/core` devDependency, `smoke` betiği. `dev` betiğinde paket `dist` değişikliklerinde yeniden başlatma denenir (Karar 3 sonuçlar).
- [ ] `apps/worker`: `"type": "module"`, `src/main.ts` (env yükle → `getEnv()` → "hazır" logu, `SIGTERM`/`SIGINT` ile düzgün kapanış; M0'da Valkey'e bağlanmaz), `src/smoke.ts` (S-2), `dev`: `tsx watch src/main.ts`, `build`: `tsc -p tsconfig.build.json`, `start`: `node dist/main.js`, `smoke`: `node dist/smoke.js`.
- [ ] `apps/web`: `"type": "module"`, `next.config.ts` (`loadEnvConfig(<repo kökü>)`), `src/app/layout.tsx`, `src/app/page.tsx` (Türkçe yer tutucu metin, sabit renk yok), `src/env.server.ts`/`src/env.client.ts` (M0.5), `tsconfig.json` (`nextjs.json`, `paths: { "@/*": ["./src/*"] }`), `dev`: `next dev --port 3000`.
- [ ] Doğrulama komutları ve beklenen sonuçlar:
  - `pnpm build` başarılı, ikinci çalıştırmada tamamı cache'ten (`FULL TURBO`)
  - `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm format:check` başarılı
  - `pnpm dev`: web `http://localhost:3000` 200 dönüyor, api 3001'de ayakta, worker "hazır" logu yazıyor
  - Duman testleri S-1 ve S-2 (S-1'in `parseEnv` çağrısı M0.4 sonrasında, S-2 M0.4 şeması sonrasında tam haliyle çalışır; sonuçlar D-SMOKE'a)
  - Lint kural testleri: geçici dosyalarda (1) `apps/api` → `@club/templates`, (2) `apps/web` → `@club/db`, (3) `apps/web` → `@club/server`, (4) `packages/shared` → `@club/server` içe aktarmaları lint hatası veriyor (dosyalar sonra silinir)
- [ ] D-GENEL: Doğrulanacaklar 6, 10'un sonuçları

## M0.4–M0.7 için notlar

- **M0.4 (architect):** `@club/shared/env` public API'si ve uygulaması (Karar 9). `packages/db/prisma/schema.prisma` (yalnızca `generator` + `datasource`, D-PRISMA'daki üretici ve seçeneklerle), `prisma.config.ts`.
- **M0.5 (devops):** her uygulamada `env.ts` (memoized `getEnv()`), kök `.env.example` (Karar 9 listesi, açıklamalı; `REDIS_URL` açıklamasında "Valkey bağlantısı" yazar). Ardından S-1/S-2 tam haliyle çalıştırılır, D-SMOKE doldurulur.
- **M0.6 (devops):** `docker-compose.dev.yml` (Karar 10): PostgreSQL, Valkey, MinIO + bucket init; portlar `127.0.0.1`'e bağlı, imajlar etiket + digest ile sabit, Valkey `maxmemory-policy noeviction` + `appendonly yes`. BullMQ kuyruk turu. D-VALKEY ve D-MINIO doldurulur.
- **M0.7 (devops):** GitHub Actions: `runs-on: ubuntu-24.04`, `permissions: contents: read`, `concurrency` ile eski çalışmalar iptal, `actions/checkout` → `pnpm/action-setup` (sürüm girdisi yok) → `actions/setup-node` (`node-version-file: .nvmrc`, `cache: pnpm`) → `pnpm install --frozen-lockfile` → `pnpm format:check` → `pnpm turbo run lint typecheck test build` → `pnpm smoke` → `.turbo/cache` için `actions/cache`. Action'lar SHA ile sabit. `db:generate` için sahte `DATABASE_URL` (gerekirse, D-PRISMA). `renovate.json` Karar 7 politikasına göre.

## Yeniden değerlendirme tetikleyicileri

- Node 26 LTS olduğunda ve native bağımlılıkların prebuilt desteği teyit edildiğinde (en erken 2027 başı).
- NestJS resmi ESM desteği yayımlarsa (api'nin ESM'e geçişi).
- TypeScript 7 (yerel) typescript-eslint, Next ve decorator metadata desteğiyle kararlı hale gelirse.
- D-TS sonucu 5.9.x çıkarsa: dört aracın hepsi TS 6'yı desteklediğinde TS 6'ya geçiş ayrı PR ile (ADR güncellemesi yeterli).
- Build/typecheck süresi geliştirme döngüsünü belirgin biçimde yavaşlatırsa (Karar 3.2'deki kaynak koşulu alternatifi).
- M9: prod depolama ADR'si (Karar 10).
