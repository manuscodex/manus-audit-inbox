# 01 — Sprint 1 (4 tygodnie): Fundament

> Cel sprintu: **Stabilna architektura multi-tenant + autoryzacja + i18n + szkielet PWA**.
> Dostarczone w 6 PR-ach, każdy ~5 dni roboczych dla 1 dewelopera fullstack.

## Definition of Done (cały sprint)

- [ ] Admin może utworzyć klinikę i dietetyka; dietetyk dostaje email z linkiem do ustawienia hasła.
- [ ] Dietetyk może utworzyć klienta z określonym czasem dostępu (default 32 dni).
- [ ] Klient dostaje email i loguje się do PWA, widzi placeholder „Mój plan".
- [ ] Separacja danych działa — dietetyk Ola nie widzi klientów dietetyka Grzegorza.
- [ ] i18n PL/EN/ES — przełącznik języka działa, żadnych hardcoded stringów w UI.
- [ ] PWA instaluje się na telefonie (manifest, ikony, offline shell).
- [ ] CI: lint + typecheck + test + build zielone na każdym PR.
- [ ] Pokrycie testami: ≥60% logiki domenowej (Vitest), 1 e2e per główny flow (Playwright).
- [ ] Manus zdeployował na `mytrela-staging.fly.dev` i przeszedł smoke test.

## Podział na PR-y

| PR | Temat | Dni | Branch | Zależności |
|---|---|---|---|---|
| PR1 | Bootstrap: Next.js + Prisma + Docker + CI + i18n + shadcn | 4 | `claude/sprint-1-bootstrap` | — |
| PR2 | Schema multi-tenant + migracje + seedy | 3 | `claude/sprint-1-schema` | PR1 merged |
| PR3 | Autoryzacja (Auth.js + email/hasło + invitation flow) | 5 | `claude/sprint-1-auth` | PR2 merged |
| PR4 | Panel admin (kliniki + dietetycy) | 4 | `claude/sprint-1-admin-panel` | PR3 merged |
| PR5 | Panel dietetyka (klienci + tworzenie + czas dostępu) | 4 | `claude/sprint-1-dietitian-panel` | PR4 merged |
| PR6 | PWA shell + login klienta + ekran „Mój plan" + i18n switcher | 4 | `claude/sprint-1-pwa-shell` | PR5 merged |

**Łącznie:** 24 dni roboczych = ~5 tygodni kalendarzowych przy 5 dniach pracy. Bufor 1 tydzień na review/poprawki.

## PR1 — Bootstrap (4 dni)

**Zakres:**
- Inicjalizacja `pnpm` workspace lub single Next.js 15 project (decyzja: single project, monorepo dopiero przy natywnej apce).
- Next.js 15 + App Router + TypeScript strict + Tailwind CSS 4.
- Prisma + Postgres 16 (lokalnie przez `docker-compose.yml`).
- `next-intl` setup z folderem `messages/{pl,en,es}.json` (puste szkielety).
- shadcn/ui zainicjalizowane, pierwsze komponenty: Button, Input, Card, Dialog.
- `next-pwa` lub `@serwist/next` — pusty manifest + service worker placeholder.
- ESLint + Prettier + Husky pre-commit (lint + typecheck).
- GitHub Actions: `ci.yml` — lint, typecheck, test, build.
- `Dockerfile` + `fly.toml` przygotowane (Manus dopnie deployment).
- `README.md` z `pnpm dev`, `pnpm test`, `pnpm db:migrate` etc.
- `.env.example` z wszystkimi zmiennymi.

**DoD PR1:**
- `pnpm dev` startuje na localhost:3000, pokazuje stronę „MyTrela — coming soon" przetłumaczoną przez next-intl.
- `pnpm test` przechodzi (1 placeholder test).
- `pnpm build` produkuje paczkę produkcyjną.
- CI zielone.

## PR2 — Schema multi-tenant (3 dni)

**Modele Prisma:**

```prisma
// Domena: System
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String?  // null jeśli klient jeszcze nie ustawił
  name      String?
  locale    String   @default("pl")
  role      UserRole // ADMIN | CLINIC_ADMIN | DIETITIAN | CLIENT
  clinicId  String?  // null tylko dla ADMIN
  clinic    Clinic?  @relation(fields: [clinicId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  // relations
  dietitian Dietitian?
  client    Client?
  sessions  Session[]
  invitations Invitation[]
}

enum UserRole { ADMIN CLINIC_ADMIN DIETITIAN CLIENT }

model Clinic {
  id        String  @id @default(cuid())
  name      String
  logoUrl   String?
  active    Boolean @default(true)
  createdAt DateTime @default(now())
  users     User[]
  dietitians Dietitian[]
  clients   Client[]
}

model Dietitian {
  id        String  @id @default(cuid())
  userId    String  @unique
  user      User    @relation(fields: [userId], references: [id])
  clinicId  String
  clinic    Clinic  @relation(fields: [clinicId], references: [id])
  permissions Json  @default("{}")  // override checkboxes
  baseRole  DietitianBaseRole @default(JUNIOR)
  clients   Client[]
}

enum DietitianBaseRole { JUNIOR SENIOR CLINIC_ADMIN }

model Client {
  id              String   @id @default(cuid())
  userId          String   @unique
  user            User     @relation(fields: [userId], references: [id])
  clinicId        String
  clinic          Clinic   @relation(fields: [clinicId], references: [id])
  dietitianId     String
  dietitian       Dietitian @relation(fields: [dietitianId], references: [id])
  accessStartsAt  DateTime @default(now())
  accessEndsAt    DateTime
  status          ClientStatus @default(ACTIVE)
  notes           String?  @db.Text
}

enum ClientStatus { ACTIVE EXPIRED CANCELLED }

model Invitation {
  id        String   @id @default(cuid())
  email     String
  token     String   @unique
  userId    String?
  user      User?    @relation(fields: [userId], references: [id])
  expiresAt DateTime
  acceptedAt DateTime?
  createdAt DateTime @default(now())
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  expiresAt DateTime
  token     String   @unique
}

// Indexy: każda tabela z clinicId → @@index([clinicId])
```

**Seedy (`prisma/seed.ts`):**
- 2 adminów (admin1@mytrela.app, admin2@mytrela.app) z hasłami z env
- 1 klinika „KetoDietetyk"
- 2 dietetycy: Ola, Grzegorz (oboje SENIOR)
- 4 klientów (po 2 na dietetyka) z aktywnym dostępem na 32 dni

**DoD PR2:**
- `pnpm db:migrate` tworzy wszystkie tabele.
- `pnpm db:seed` tworzy seedowy dataset.
- Test integracyjny weryfikuje że Ola nie widzi klientów Grzegorza przez query Prisma.

## PR3 — Autoryzacja + invitation flow (5 dni)

**Zakres:**
- Auth.js v5 z Credentials provider (email + password przez bcrypt).
- Server Actions: `signIn`, `signOut`, `requestPasswordReset`, `acceptInvitation(token, newPassword)`.
- Middleware `middleware.ts` — wymusza zalogowanie na `/admin/*`, `/dietitian/*`, `/app/*`; przekierowuje do `/login`.
- Wstrzykiwanie `session.user.clinicId` do request context.
- Prisma extension `withTenant(prisma, clinicId)` — wymusza `clinicId` filter na każdym query z tabel domenowych.
- Email service: **Resend** lub **Postmark** (Resend prościej, ma free tier). Template `invitation` w `react-email`.
- Strona `/invitation/[token]` — formularz ustawienia hasła.
- Strona `/login` — formularz logowania (PL/EN/ES).
- Rate limiting na `/api/auth/*` przez `next-rate-limit` lub Upstash (jak będzie dostępne).

**DoD PR3:**
- Admin loguje się przez `/login` → ląduje na `/admin`.
- Dietetyk z seeda loguje się → ląduje na `/dietitian`.
- Klient z seeda loguje się → ląduje na `/app`.
- Próba dostępu do cudzej strony → 403.
- Email z invitation przychodzi (lokalnie do Mailhog, w staging do prawdziwego inboxa).
- Token invitation jednorazowy, wygasa po 7 dniach.
- E2E test (Playwright): pełny flow zaproszenia dietetyka.

## PR4 — Panel admin (4 dni)

**Routes (App Router):**
- `/admin` — dashboard (liczba klinik, dietetyków, klientów)
- `/admin/clinics` — lista + filtr + paginacja (10/page)
- `/admin/clinics/new` — formularz tworzenia (nazwa, logo upload)
- `/admin/clinics/[id]` — edycja + lista dietetyków
- `/admin/clinics/[id]/dietitians/new` — zaproszenie dietetyka (email, baseRole, permissions checkboxes)
- `/admin/dietitians` — globalna lista (filtry: klinika, rola, status)
- `/admin/dietitians/[id]` — edycja uprawnień, deaktywacja

**Wymagania:**
- Server Actions + Zod validation.
- Wszystkie formularze z shadcn/ui + `react-hook-form`.
- Wszystkie teksty z i18n (`messages/*.json` rosną w tym PR).
- Audit log: każda zmiana admina → wpis `AuditLog` (nowy model — dodać w tym PR).
- Logo kliniki: upload do `/public/uploads/clinics/<id>.png` lokalnie, w staging do Fly Volume lub S3 (placeholder w MVP).

**DoD PR4:**
- E2E: admin tworzy klinikę → zaprasza dietetyka → email przychodzi → dietetyk akceptuje → loguje się.

## PR5 — Panel dietetyka (4 dni)

**Routes:**
- `/dietitian` — dashboard (liczba klientów aktywnych, wygasających w ciągu 7 dni)
- `/dietitian/clients` — lista klientów (tylko swoich!)
- `/dietitian/clients/new` — formularz (email, imię, czas dostępu default 32 dni)
- `/dietitian/clients/[id]` — szczegóły + edycja + przedłużenie/cofnięcie dostępu

**Wymagania:**
- Każde query przez `withTenant(prisma, clinicId).client.findMany({ where: { dietitianId: session.user.dietitianId }})`.
- Test integracyjny: Ola próbuje pobrać klienta Grzegorza po ID → 404 (nie 403, żeby nie wyciekać istnienia).
- Wybór czasu dostępu: dropdown 30/60/90 + custom date picker, default 32 dni.
- Po utworzeniu klienta — email z invitation do PWA.

**DoD PR5:**
- E2E: Ola tworzy klienta → klient dostaje email → loguje się do `/app` → widzi siebie w nagłówku.
- Test separacji: zalogowana Ola wywołuje API/Server Action z `clientId` Grzegorza → odmowa.

## PR6 — PWA shell + login klienta + i18n switcher (4 dni)

**Zakres:**
- `app/app/layout.tsx` — layout mobile-first (bottom nav: „Plan" | „Pomiary" | „Materiały" | „Profil" — w S1 wszystkie poza Plan to placeholdery).
- `app/app/page.tsx` — ekran „Mój plan" placeholder: „Twój dietetyk pracuje nad Twoim planem. Wkrótce pojawi się tutaj.".
- `manifest.json` z ikonami (Claude wygeneruje placeholdery 192/512, ostateczne wrzuci grafik).
- Service worker (offline shell — przynajmniej `/app` i `/login` cache'owane).
- Komponent `<LocaleSwitcher>` (flagi PL/EN/ES) — w nagłówku każdej strony.
- Persistance locale: w `User.locale` (zalogowani) lub cookie (anonimowi).
- Sprawdzić Lighthouse PWA score — cel ≥90.

**DoD PR6:**
- Otwarcie `/app` w Chrome na Androidzie → propozycja „Dodaj do ekranu głównego" działa.
- Po dodaniu — odpalenie z ikony otwiera fullscreen.
- Zmiana języka w switcherze persistuje po przeładowaniu.
- Lighthouse PWA ≥90, Performance ≥85 na mobile.

## Ryzyka Sprintu 1

| Ryzyko | Mitygacja |
|---|---|
| Auth.js v5 jest świeży, ma quirks z App Router | PR3 ma 5 dni (najwięcej), bufor 1 dzień w timeline |
| Email delivery (Resend/Postmark) wymaga DNS setup | Manus konfiguruje DNS w paraleli; w dev używamy Mailhog |
| Multi-tenant Prisma extension może mieć edge cases | Test integracyjny pokrywa wszystkie modele z `clinicId` |
| PWA offline shell vs Server Components | next-pwa supportuje, ale wymaga konfiguracji — Codex review szczególnie tu |
| i18n routes (`/pl/app` vs `/app?locale=pl`) | Decyzja: bez prefixu URL, locale z usera/cookie (prostsze SEO dla landing) |
