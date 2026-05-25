# 00 — Przegląd projektu MyTrela

## Cel produktu

MyTrela to platforma dietetyczna SaaS dla klinik i dietetyków (alternatywa dla alloweat.com) z trzema warstwami użytkowników:

- **Administrator** — widzi i konfiguruje całą aplikację (kliniki, dietetyków, role, baza produktów/przepisów).
- **Dietetyk** — pracuje w panelu webowym, zarządza swoimi klientami, tworzy przepisy, jadłospisy, kolekcje.
- **Klient końcowy** — korzysta z aplikacji mobilnej (PWA w MVP, natywna docelowo) — widzi swój plan, pomiary, materiały.

**Komercyjny cel:** pierwszy płacący klient w ~4 miesiące (po Sprincie 4).

## Architektura wysokopoziomowa

```
┌──────────────────────────────────────────────────────────┐
│  System (1)                                              │
│  ├── Klinika A (np. KetoDietetyk)                        │
│  │   ├── Dietetyk Ola → Klient 1, Klient 2, …            │
│  │   └── Dietetyk Grzegorz → Klient 3, …                 │
│  └── Klinika B …                                         │
└──────────────────────────────────────────────────────────┘
```

**Bezwzględne reguły separacji danych:**
- Klient należy do jednego dietetyka w danym momencie.
- Dietetyk widzi tylko swoich klientów (chyba że admin nadał szerszy zakres).
- Klinika nie loguje się — to byt organizacyjny (logo, nazwa → na PDF-ach).
- Admin widzi wszystko, może przenosić klientów (w obrębie kliniki).

## Stack techniczny (Sprint 1)

| Warstwa | Technologia | Uzasadnienie |
|---|---|---|
| Framework | **Next.js 15 (App Router)** | SSR/Server Actions, jeden codebase dla web + PWA, świetne i18n |
| Język | **TypeScript (strict)** | Bezpieczeństwo typów end-to-end |
| Styl | **Tailwind CSS 4** + **shadcn/ui** | Szybkie UI mobile-first, dostępne komponenty |
| ORM | **Prisma 5** | Type-safe, migracje, multi-tenant przez `tenantId` na każdej tabeli |
| DB | **PostgreSQL 16** | Sprawdzony, wsparcie dla JSON, RLS w przyszłości |
| Auth | **Auth.js (NextAuth v5)** + email/hasło (`bcrypt`) | Szybki setup, sesje JWT, łatwa rozbudowa o OAuth |
| i18n | **next-intl** | Server Components compatible, typed messages PL/EN/ES |
| PWA | **next-pwa** lub `@serwist/next` | Manifest, service worker, offline-shell |
| Testy | **Vitest** (unit) + **Playwright** (e2e) | Standardowe, szybkie |
| CI | **GitHub Actions** | Lint + typecheck + test + build na każdy PR |
| Hosting | **Fly.io** (przez Manus) | Spójność ze stagingiem dietapp |
| Validation | **Zod** | Schema-first walidacja formularzy i API |

## Multi-tenant — podejście

**Decyzja: shared database, shared schema, `tenantId` (clinicId) na każdej tabeli domenowej.**

- Każda tabela domenowa (Recipes, Products, Clients, MealPlans, ...) ma kolumnę `clinicId`.
- Globalne zasoby (np. przepisy globalne, baza produktów GS1) mają `clinicId = NULL` i flagę `scope = 'GLOBAL'`.
- Middleware Auth wstrzykuje `session.user.clinicId` do każdego requesta.
- Prisma middleware/extension dodaje `WHERE clinicId = session.clinicId` automatycznie do zapytań — fail-safe.
- W przyszłości można dodać Postgres Row-Level Security jako drugą warstwę obrony.

## Decyzje produktowe (z dokumentu wizji)

1. **i18n od Sprintu 1** — wszystkie teksty przez klucze. Brak hardcoded stringów w UI.
2. **Mobile-first** — wszystkie komponenty projektowane na ekran 360px wzwyż, breakpointy `sm/md/lg`.
3. **Praca na kopiach** — przepis w jadłospisie/u klienta to zawsze kopia, nigdy oryginał. (Sprint 2)
4. **Brak czatu w MVP** — zastępujemy ogłoszeniami i wyzwaniami (Sprint 4).
5. **WooCommerce nie w MVP** — Sprint 6.
6. **Rozliczenia per aktywny klient** — Sprint 5, ale schema `BillingPeriod` zakładamy już w S1.

## Co nie jest w Sprincie 1

- Produkty, przepisy, jadłospisy — Sprint 2/3
- Pomiary klienta — Sprint 3
- Ogłoszenia/wyzwania — Sprint 4
- Logika billing — Sprint 5
- WooCommerce — Sprint 6

## Co MUSI działać po Sprincie 1

- [ ] Admin loguje się i tworzy klinikę „KetoDietetyk".
- [ ] Admin tworzy dietetyka „Ola" w klinice, dostaje on email z linkiem do ustawienia hasła.
- [ ] Ola loguje się, dodaje klienta „Anna" z dostępem na 32 dni.
- [ ] Anna otrzymuje email z danymi do PWA, loguje się i widzi ekran „Mój plan" (placeholder „Twój plan pojawi się wkrótce").
- [ ] Wszystkie teksty mają warianty PL/EN/ES — przełącznik języka w nagłówku działa.
- [ ] PWA da się zainstalować na telefonie (manifest + ikony + offline-shell).
- [ ] Drugi dietetyk w tej samej klinice nie widzi klientów Oli.
