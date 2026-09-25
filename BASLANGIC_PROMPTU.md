# Başlangıç Prompt'u

Aşağıdaki bloğu Claude Code'a (VS Code) ilk mesaj olarak yapıştır.

---

```
Sen bu projenin orkestratörüsün. Önce CLAUDE.md, docs/DESIGN.md ve docs/PLAN.md dosyalarını baştan sona oku. .claude/agents/ altındaki 10 alt ajanın tanımlarını da oku.

Sonra şunları yap:

1. Anladığın kadarıyla projeyi 5 cümlede özetle.
2. Her alt ajanın sorumluluk alanını ve hangi kilometre taşlarında devreye gireceğini tablo olarak göster.
3. DESIGN.md ve PLAN.md arasında gördüğün çelişki, eksik veya belirsizlikleri listele. Her biri için önerini yaz.
4. docs/PROGRESS.md dosyasını oluştur: M0-M9 başlıkları, her görev için durum (bekliyor / sürüyor / tamamlandı), açık bulgular bölümü.
5. M0 için görev kırılımını çıkar: hangi ajan, hangi görev, hangi sırayla, hangileri paralel.

Bu adımlarda kod yazma ve hiçbir ajanı çalıştırma. Planı bana sun ve onayımı bekle. Onay verdiğimde M0'ı CLAUDE.md'deki çalışma döngüsüyle (devret → test kapısı → inceleme kapısı → düzeltme → kaydet) yürüt ve taş sonunda raporla.
```

---

## Sonraki kilometre taşları için

Her taş bittiğinde ve raporu okuyup onayladığında:

```
M<n> raporunu onaylıyorum. <varsa notların>. M<n+1>'e geç: önce görev kırılımını göster, onayımdan sonra çalışma döngüsüyle yürüt.
```

## Takıldığında

```
Dur. Şu an hangi görevdesin, hangi ajan ne yaptı, hangi test/inceleme bulgusu açık? PROGRESS.md'yi güncelle ve bana 5 maddede özet ver.
```

## Taş ortasında kod incelemesi istemek

```
Şu ana kadarki değişiklikler için code-reviewer ve security-auditor'ı paralel çalıştır. Bulguları birleştir, BLOCKER ve MAJOR olanları ilgili ajanlara düzelttir, sonra test-engineer ile tam test çalıştır.
```
