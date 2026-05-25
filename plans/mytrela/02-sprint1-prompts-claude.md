# 02 — Sprint 1: Prompty dla Claude Code

> **Jak używać:** Otwórz https://claude.ai/code, wybierz repo `manuscodex/mytrela`, podaj branch zgodnie z tabelą poniżej, wklej prompt 1:1. Claude Code zrobi commit, push i otworzy draft PR. Następnie odpal Codex review (patrz `03-sprint1-prompts-codex.md`).
>
> **Zasada:** Jeden prompt = jeden PR. NIE wklejaj dwóch promptów w jednej sesji.

| Prompt | Branch | Bazowy branch |
|---|---|---|
| PR1 | `claude/sprint-1-bootstrap` | `main` |
| PR2 | `claude/sprint-1-schema` | `main` (po merge PR1) |
| PR3 | `claude/sprint-1-auth` | `main` (po merge PR2) |
| PR4 | `claude/sprint-1-admin-panel` | `main` (po merge PR3) |
| PR5 | `claude/sprint-1-dietitian-panel` | `main` (po merge PR4) |
| PR6 | `claude/sprint-1-pwa-shell` | `main` (po merge PR5) |

---

## Prompt PR1 — Bootstrap

```
Rola: jesteś głównym deweloperem fullstack budującym MyTrela — aplikację dietetyczną SaaS (PWA mobile-first + panel webowy). Pracujesz solo, 35h/tydzień, sprint 3-4 tyg. Stack: Next.js 15 + TypeScript strict + Tailwind 4 + Prisma 5 + Postgres 16. Repo jest puste (świeży `main`).

Twoje zadanie: zbudować fundament projektu (PR1 ze Sprintu 1). NIE implementuj jeszcze schematu domenowego ani auth — to PR2 i PR3. Tu chodzi o szkielet, który dalej rośnie.

ZAKRES PR1:
1. `pnpm` jako package manager. `package.json` z scriptami: `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `test:e2e`, `db:migrate`, `db:seed`, `db:studio`, `format`.
2. Next.js 15 (App Router) + TypeScript strict (`strict: true`, `noUncheckedIndexedAccess: true`).
3. Tailwind CSS 4 + `app/globals.css` z resetem + zmiennymi CSS dla theme (light tylko w MVP).
4. shadcn/ui zainicjalizowane (`components/ui/`) z komponentami: Button, Input, Label, Card, Dialog, Form, Select, Toast.
5. Prisma 5: `prisma/schema.prisma` z minimalnym placeholderem (tylko `model HealthCheck { id String @id @default(cuid()) ts DateTime @default(now()) }`). DataSource: Postgres przez `DATABASE_URL`.
6. `docker-compose.yml` z Postgres 16 (port 5432, hasło z `.env`).
7. `next-intl` setup:
   - `i18n.ts` z `getRequestConfig`
   - `messages/pl.json`, `messages/en.json`, `messages/es.json` — każdy z kluczem `common.appName: "MyTrela"` i `home.tagline: "Twoja dieta, Twój sukces" | "Your diet, your success" | "Tu dieta, tu éxito"`.
   - bez prefixu URL — locale z cookie `NEXT_LOCALE` (default `pl`).
8. `next-pwa` LUB `@serwist/next` (wybierz to drugie, jest nowsze i lepiej supportowane w Next 15). Pusty `manifest.json` w `public/`:
   ```json
   { "name": "MyTrela", "short_name": "MyTrela", "start_url": "/", "display": "standalone", "background_color": "#ffffff", "theme_color": "#16a34a", "icons": [{"src":"/icons/192.png","sizes":"192x192","type":"image/png"},{"src":"/icons/512.png","sizes":"512x512","type":"image/png"}] }
   ```
   Wygeneruj proste placeholder PNG ikony (zielony kwadrat z "M" — możesz wygenerować przez node script z `sharp`, albo wstaw 1x1 transparentny z TODO komentarzem).
9. `app/page.tsx` — landing minimalny:
   - Hero z `t('common.appName')` i `t('home.tagline')`
   - `<LocaleSwitcher>` w prawym górnym rogu (na razie tylko UI, persist w cookie przez Server Action `setLocale`)
10. ESLint (`eslint-config-next` + `eslint-plugin-tailwindcss`) + Prettier + `husky` pre-commit (`lint-staged` na zmienionych plikach).
11. Vitest + Testing Library — 1 placeholder test w `app/page.test.tsx` (renderuje stronę).
12. Playwright — `playwright.config.ts` + 1 placeholder e2e w `e2e/smoke.spec.ts` (otwiera `/`, sprawdza tytuł).
13. GitHub Actions `.github/workflows/ci.yml`:
    - triggery: push/PR na każdy branch
    - jobs (matrix Node 20):
      - lint (`pnpm lint`)
      - typecheck (`pnpm typecheck`)
      - unit (`pnpm test` z postgres service container)
      - build (`pnpm build`)
    - cache pnpm store + Next.js `.next/cache`
14. `Dockerfile` (multi-stage: deps → builder → runner, standalone output Next.js).
15. `fly.toml` z minimalną konfiguracją (Manus dopnie sekrety i volumes osobno — tylko placeholder app name `mytrela-staging`).
16. `.env.example` z: `DATABASE_URL`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`, `RESEND_API_KEY` (jeszcze nieużywany), `APP_URL`.
17. `README.md`:
    - Co to MyTrela (2 zdania)
    - Quick start (`pnpm install`, `docker compose up -d db`, `cp .env.example .env`, `pnpm db:migrate`, `pnpm dev`)
    - Skrypty
    - Struktura katalogów
    - Link do `manus-audit-inbox/plans/mytrela/` po koordynację

WAŻNE ZASADY:
- TypeScript strict, BEZ `any` (jeśli musisz, dodaj komentarz dlaczego).
- Wszystkie teksty w UI przez `useTranslations()` / `getTranslations()` — żadnych hardcoded stringów po polsku w komponentach (kara w review).
- Tailwind 4 (nowy config przez `@theme` w CSS, nie `tailwind.config.ts`).
- Mobile-first: każdy widok testowany na 360px wzwyż, breakpointy tylko `sm:` `md:` `lg:`.
- App Router, Server Components default, `'use client'` tylko gdzie potrzebne.
- NIE dodawaj autoryzacji, schematu klinik/dietetyków/klientów, PWA service workera implementacji offline — to kolejne PR-y.

PROCES:
1. Pracujesz na branchu `claude/sprint-1-bootstrap`.
2. Commituj atomowo (1 commit = 1 logiczna zmiana): "chore: init next.js 15", "chore: add prisma + postgres", "feat: setup next-intl", "feat: add shadcn ui base", "ci: add github actions", "docs: README + env example", itd.
3. Po zakończeniu pushuj branch.
4. Otwórz **draft PR** do `main` z tytułem `Sprint 1 / PR1 — Bootstrap: Next.js 15 + Prisma + i18n + PWA shell` i body opisującym co zrobiłeś + checklistę DoD z `plans/mytrela/01-sprint1-plan.md` (sekcja PR1).
5. NIE merguj.

DEFINICJA DONE PR1:
- [ ] `pnpm dev` startuje, `/` renderuje hero w PL/EN/ES (zmiana cookie → zmiana języka)
- [ ] `pnpm test` zielone
- [ ] `pnpm test:e2e` zielone
- [ ] `pnpm build` zielony
- [ ] CI zielone na pushu
- [ ] `docker compose up -d db && pnpm db:migrate` tworzy tabelę HealthCheck
- [ ] Lighthouse na `/` ≥85 perf, ≥90 a11y na mobile

Po skończeniu daj mi krótki raport w odpowiedzi: linki do commitów, link do PR, ewentualne wątpliwości.
```

---

## Prompt PR2 — Schema multi-tenant

```
Rola: kontynuujesz Sprint 1 MyTrela (PR2). Bazujesz na `main` po merge PR1. Stack już stoi (Next.js 15 + Prisma + Postgres + i18n).

Zadanie: zaprojektować i zaimplementować pełny schemat domenowy Sprintu 1 + seedy + Prisma extension wymuszający multi-tenant.

ZAKRES PR2:
1. Branch: `claude/sprint-1-schema` z `main`.
2. `prisma/schema.prisma` — modele dokładnie takie jak w `plans/mytrela/01-sprint1-plan.md` (sekcja PR2). Pełna lista:
   - `User` (id, email unique, password optional, name, locale, role enum, clinicId optional, timestamps)
   - `enum UserRole { ADMIN | CLINIC_ADMIN | DIETITIAN | CLIENT }`
   - `Clinic` (id, name, logoUrl optional, active, timestamps)
   - `Dietitian` (id, userId unique, clinicId, permissions JSON, baseRole enum)
   - `enum DietitianBaseRole { JUNIOR | SENIOR | CLINIC_ADMIN }`
   - `Client` (id, userId unique, clinicId, dietitianId, accessStartsAt, accessEndsAt, status enum, notes optional)
   - `enum ClientStatus { ACTIVE | EXPIRED | CANCELLED }`
   - `Invitation` (id, email, token unique, userId optional, expiresAt, acceptedAt, createdAt)
   - `Session` (id, userId, expiresAt, token unique) — będzie używane przez Auth.js w PR3
   - `AuditLog` (id, actorId, action string, entity string, entityId string, payload JSON, createdAt) — dla audytu admina (PR4)
3. Indeksy:
   - każda tabela z `clinicId` → `@@index([clinicId])`
   - `Client.dietitianId` → `@@index([dietitianId])`
   - `Client.accessEndsAt` → `@@index([accessEndsAt])` (do alertów wygasania w PR5)
   - `Invitation.expiresAt` → `@@index([expiresAt])`
4. Migracja: `pnpm db:migrate dev --name sprint1_core_schema`. Skomituj folder `prisma/migrations/`.
5. `prisma/seed.ts`:
   - 2 adminów (`admin1@mytrela.app`, `admin2@mytrela.app`) — hasła z `ADMIN_SEED_PASSWORD` env (bcrypt hash, cost 12).
   - 1 klinika "KetoDietetyk" (logo NULL).
   - 2 dietetycy: Ola (`ola@ketodietetyk.app`, SENIOR), Grzegorz (`grzegorz@ketodietetyk.app`, SENIOR).
   - 4 klienci: Anna i Bartek do Oli, Cezary i Daria do Grzegorza. Wszyscy z `accessEndsAt = now + 32 days`, status ACTIVE.
   - Wszystkie konta z hasłami z env (`DIETITIAN_SEED_PASSWORD`, `CLIENT_SEED_PASSWORD`) lub random + zalogowane do konsoli.
   - Idempotentne (upsert by email).
   - Skrypt `pnpm db:seed` w package.json.
6. `lib/db/tenant.ts` — Prisma extension `withTenant(prisma, ctx: { role, clinicId, dietitianId? })`:
   - Dla tabel `Client`, `Dietitian` (i przyszłych `Recipe`, `Product`, `MealPlan`): automatyczne `where: { clinicId: ctx.clinicId }` przy `findMany`, `findFirst`, `findUnique`, `count`, `aggregate`.
   - Dla `Client` dodatkowo: jeśli `ctx.role === 'DIETITIAN'`, dorzuca `where: { dietitianId: ctx.dietitianId }`.
   - Dla `ctx.role === 'ADMIN'` — bypass (admin widzi wszystko).
   - Przy `create/update/upsert` na tabelach domenowych: wymusza `data.clinicId = ctx.clinicId`, rzuca jeśli próba podania innego.
   - Implementacja przez `prisma.$extends({ query: { ... } })` (Prisma Client Extensions, oficjalne API).
7. Testy:
   - `tests/db/tenant.test.ts` (Vitest + testcontainers lub osobna Postgres na CI):
     - Setup: stwórz 2 kliniki, każda po 1 dietetyku, każdy po 2 klientów.
     - Test 1: Ola (kontekst tenanta jej kliniki + jej `dietitianId`) — `client.findMany()` zwraca tylko jej 2 klientów.
     - Test 2: Ola próbuje `client.findUnique({ where: { id: <Grzegorz's client> } })` — zwraca null.
     - Test 3: Admin — `client.findMany()` bez tenant context → wszyscy klienci.
     - Test 4: Ola próbuje `client.create({ data: { clinicId: <inna klinika>, ... } })` → throw.
   - Min. 8 testów pokrywających modele Client, Dietitian (Recipe i Product będą w PR Sprint 2).
8. `README.md` — dopisz sekcję "Multi-tenant": jak działa `withTenant`, kiedy używać, przykład.

WAŻNE:
- Każda tabela domenowa MUSI mieć `clinicId` (z wyjątkiem `User`, gdzie jest optional — admin nie ma kliniki, oraz `Invitation`, `Session` — bound do `userId`).
- W przyszłości (poza Sprintem 1) `Recipe` i `Product` będą miały `clinicId nullable` dla globalnych — w S1 nie implementujemy, ale schemat ma być gotowy do dodania (zostaw komentarz `// TODO: nullable clinicId for global resources in Sprint 2`).
- W kolejnym kroku (PR3) dodamy NextAuth, który będzie czytać `User` i wstrzykiwać `ctx` do `withTenant` w Server Actions.

PROCES:
1. Branch `claude/sprint-1-schema`.
2. Commity: "feat(db): add core multi-tenant schema", "feat(db): seed dataset KetoDietetyk", "feat(db): tenant prisma extension", "test(db): tenant isolation tests", "docs: multi-tenant section in README".
3. Push, draft PR do `main`, tytuł `Sprint 1 / PR2 — Multi-tenant schema + seeds + tenant guard`.

DoD:
- [ ] Migracja przechodzi czysto na pustej bazie
- [ ] `pnpm db:seed` produkuje wymagany dataset (sprawdź przez `pnpm db:studio`)
- [ ] Wszystkie testy tenant zielone
- [ ] CI zielony

Raport po skończeniu: link do PR + lista plików dodanych/zmienionych + szczególne uwagi.
```

---

## Prompt PR3 — Autoryzacja + invitation flow

```
Rola: kontynuujesz Sprint 1 MyTrela (PR3). Bazujesz na `main` po merge PR2. Masz już schemat domenowy i `withTenant`.

Zadanie: pełna autoryzacja Auth.js v5 + flow zapraszania użytkowników (admin → dietetyk → klient), email transactional przez Resend, middleware autoryzujący routes.

ZAKRES PR3:
1. Branch: `claude/sprint-1-auth` z `main`.
2. Auth.js v5 (`@auth/nextjs`, beta jest stable enough — sprawdź najnowszą wersję):
   - Credentials provider (email + password).
   - Adapter Prisma (oficjalny `@auth/prisma-adapter`).
   - JWT session strategy (prościej niż database sessions w S1).
   - `auth.config.ts` + `auth.ts` + route handler `app/api/auth/[...nextauth]/route.ts`.
   - Callbacks: `session` wzbogaca `session.user` o `id`, `role`, `clinicId`, `dietitianId` (jeśli dietetyk), `clientId` (jeśli klient).
   - JWT secret z `NEXTAUTH_SECRET`.
3. `middleware.ts` (Next.js middleware):
   - Wymaga zalogowania na `/admin/*`, `/dietitian/*`, `/app/*`.
   - Sprawdza rolę: `/admin/*` → tylko ADMIN; `/dietitian/*` → DIETITIAN lub CLINIC_ADMIN; `/app/*` → CLIENT.
   - Redirect do `/login?redirect=<path>` jeśli niezalogowany.
   - Redirect do `/forbidden` jeśli zła rola.
4. `app/login/page.tsx` — formularz email + password, z `useFormStatus` (Next 15) + Server Action `signInAction`. Pełna walidacja Zod. Komunikaty błędów przez i18n. Link "Zapomniałeś hasła?" → `/forgot-password`.
5. `app/forgot-password/page.tsx` — formularz email, Server Action `requestPasswordReset` tworzy `Invitation` z token, wysyła email.
6. `app/invitation/[token]/page.tsx` — formularz "Ustaw hasło" (password + confirm). Server Action `acceptInvitation` waliduje token, ustawia password, kasuje `Invitation`, automatycznie loguje user (po userRole redirect: admin → `/admin`, dietitian → `/dietitian`, client → `/app`).
7. `app/forbidden/page.tsx` — prosty komunikat.
8. Email service — **Resend** (najprostszy):
   - `lib/email/client.ts` — `resend = new Resend(env.RESEND_API_KEY)`.
   - `lib/email/templates/invitation.tsx` — `react-email` template z claim "Witaj w MyTrela!" + button "Ustaw hasło" linkujący do `${APP_URL}/invitation/${token}`. Pełne i18n (template przyjmuje `locale`).
   - `lib/email/send.ts` — `sendInvitation({ to, name, token, locale })`.
   - W dev: jeśli `RESEND_API_KEY` brak, loguj email do konsoli + zapisz do `tmp/emails/<timestamp>.html` zamiast wysyłać.
9. `lib/auth/invitations.ts` — service:
   - `createInvitation(email, userId?)` → token (cuid2, 32 znaki), expiresAt = now + 7 days.
   - `acceptInvitation(token, password)` → walidacja, hash bcrypt cost 12, update User.password, oznacz Invitation.acceptedAt.
10. Rate limiting:
    - Próby logowania: 5/min per email + 20/min per IP (in-memory LRU w MVP, Upstash w S5).
    - Endpoint `/api/auth/*` z rate limit middleware.
11. Update seed (`prisma/seed.ts`):
    - Dla 2 adminów: hasła set bezpośrednio (mogą się logować od razu).
    - Dla dietetyków i klientów: hasła NULL + utworzone `Invitation` z 30-day expiry. Loguj linki invitation do konsoli przy seed (do testowania lokalnie).
12. Testy:
    - Unit: `acceptInvitation` (expired token, wrong token, success).
    - E2E (Playwright): pełny flow zaproszenia dietetyka:
      - Admin loguje się → wywołuje `inviteDietitian(email)` (Server Action, w PR3 jeszcze nie ma UI — wywołaj przez API/test helper)
      - Email wpada do `tmp/emails/`
      - Wyciągnij token z emaila
      - Otwórz `/invitation/<token>` → ustaw hasło → redirect do `/dietitian`
    - Unit: separacja — zalogowany dietitian próbuje GET `/admin` → 403 redirect.
13. Update i18n — nowe klucze w `messages/{pl,en,es}.json`:
    - `auth.signIn.*`, `auth.forgotPassword.*`, `auth.invitation.*`, `errors.*`.

WAŻNE:
- BCrypt cost 12 (slow, secure). Nie używaj `crypto.scrypt` ani niczego custom.
- Nigdy nie loguj haseł nawet w dev.
- Token invitation: tylko `cuid2` lub `crypto.randomBytes(32).toString('hex')`. Nie sekwencyjne.
- Sesja JWT: max 7 dni, refresh przy aktywności.
- CSRF — Auth.js obsługuje out-of-the-box dla credentials, sprawdź czy włączone.
- Cookies: `httpOnly`, `secure` (w prod), `sameSite: lax`.
- W PR3 NIE robisz jeszcze UI dla zapraszania dietetyków/klientów — to PR4 (admin) i PR5 (dietitian). Tu tworzysz tylko Server Action `inviteDietitian(email, clinicId, baseRole)` i `inviteClient(email, name, accessDays)` dostępne do wywołania z testów.

PROCES:
1. Branch `claude/sprint-1-auth`.
2. Atomowe commity: "feat(auth): nextauth v5 with credentials + prisma adapter", "feat(auth): middleware role-based routing", "feat(auth): invitation flow service", "feat(email): resend client + invitation template", "feat(ui): login + forgot-password + invitation pages", "feat(auth): rate limiting", "test(auth): invitation flow e2e", "chore(seed): generate invitations for dietitians and clients".
3. Push + draft PR.

DoD:
- [ ] Admin z seeda loguje się → `/admin` (jeszcze pusty placeholder OK)
- [ ] Dietetyk z seeda: token z konsoli → `/invitation/...` → set password → loguje się → `/dietitian`
- [ ] Klient: jak wyżej → `/app`
- [ ] Zły token → friendly error
- [ ] Wygasły token → friendly error + opcja "wyślij nowy"
- [ ] Rate limit działa (po 5 nieudanych logowaniach → 429 na minute)
- [ ] E2E zielone
- [ ] CI zielone

Raport po skończeniu.
```

---

## Prompt PR4 — Panel admin

```
Rola: kontynuujesz Sprint 1 MyTrela (PR4). Po PR3 masz auth, invitation flow, middleware. Teraz budujesz UI dla admina.

Zadanie: panel administracyjny — zarządzanie klinikami i dietetykami (CRUD + invitation), audit log.

ZAKRES PR4:
1. Branch: `claude/sprint-1-admin-panel` z `main`.
2. Layout: `app/admin/layout.tsx` — sidebar (Dashboard | Kliniki | Dietetycy | Audyt | Wyloguj) + topbar z `<LocaleSwitcher>` + nick admina.
3. Routes:
   - `app/admin/page.tsx` — dashboard: kafle "Liczba klinik", "Liczba dietetyków", "Liczba klientów aktywnych", "Wygasające w 7 dni".
   - `app/admin/clinics/page.tsx` — tabela klinik (kolumny: nazwa, logo, # dietetyków, # klientów, status, akcje). Paginacja 10/page przez query params. Filtr po nazwie. shadcn DataTable.
   - `app/admin/clinics/new/page.tsx` — formularz: nazwa (required), logo (file upload optional, max 2MB, png/jpg/svg, w MVP zapis lokalnie do `public/uploads/clinics/<id>.<ext>` — w prod Manus podmieni na S3/Fly Volume).
   - `app/admin/clinics/[id]/page.tsx` — edycja (nazwa, logo, active) + lista dietetyków + button "Zaproś dietetyka".
   - `app/admin/clinics/[id]/dietitians/new/page.tsx` — formularz invitation: email (required), imię, baseRole (radio: JUNIOR / SENIOR / CLINIC_ADMIN), checkboxy permissions (placeholder: ["canViewAllClinicClients", "canEditGlobalRecipes", "canManageDietitians"] — w PR4 zapisujemy do JSON, użycie w PR5+).
   - `app/admin/dietitians/page.tsx` — globalna lista dietetyków z filtrami (klinika, baseRole, status active).
   - `app/admin/dietitians/[id]/page.tsx` — edycja permissions + button "Deaktywuj/Aktywuj" + przycisk "Resetuj hasło" (wysyła nowy invitation).
   - `app/admin/audit/page.tsx` — tabela `AuditLog` z filtrami (kto, kiedy, co).
4. Server Actions w `app/admin/_actions/`:
   - `createClinic`, `updateClinic`, `toggleClinicActive`
   - `inviteDietitian` (woła `inviteDietitian` z PR3)
   - `updateDietitianPermissions`, `toggleDietitianActive`, `resetDietitianPassword`
   - Każda akcja: Zod validation, `revalidatePath`, audit log.
5. `lib/audit.ts` — `logAudit({ actor, action, entity, entityId, payload })` — wywoływane z każdej Server Action admina. Action stringi enumlike: `CLINIC_CREATED`, `DIETITIAN_INVITED`, etc.
6. Upload logo:
   - Server Action `uploadLogo(formData)` — sprawdza mime/size, zapisuje do `public/uploads/clinics/<clinicId>.<ext>`.
   - W middleware Next.js zezwól na serving z `public/uploads/`.
   - W `fly.toml` Manus zmapuje to na volume — Claude tylko zostawia komentarz w README.
7. Wszystkie teksty przez i18n. Rosną klucze: `admin.dashboard.*`, `admin.clinics.*`, `admin.dietitians.*`, `admin.audit.*`.
8. Test E2E (Playwright) `e2e/admin-flow.spec.ts`:
   - Admin loguje się
   - Tworzy klinikę "TestClinic"
   - Zaprasza dietetyka `test-diet@example.com`, SENIOR
   - Sprawdza że dietetyk pojawia się w liście kliniki
   - Otwiera email z `tmp/emails/`, wyciąga token, accept invitation
   - Logout, login jako test-diet → ląduje na `/dietitian`
9. Test integracyjny `tests/admin/separation.test.ts`:
   - Klinika A z dietetykiem oraz klinika B z dietetykiem
   - Dietetyk A próbuje GET `/admin/...` → 403 (przez middleware)

WAŻNE:
- Wszystkie formularze: `react-hook-form` + `zodResolver`.
- Upload pliku: użyj Server Action z `'use server'` i `FormData`. Sprawdź mime przez `file-type` package (nie tylko extension).
- DataTable: shadcn `@tanstack/react-table` (oficjalny example shadcn).
- Mobile-first: admin też działa na telefonie (sidebar → drawer na <md).
- Zachowaj wszystkie inwariantów multi-tenant — admin używa `prisma` bezpośrednio (BEZ `withTenant`), bo admin widzi wszystko.

PROCES:
1. Branch `claude/sprint-1-admin-panel`.
2. Commity tematyczne (clinics CRUD, dietitians CRUD, audit log, layout, i18n keys, e2e).
3. Push + draft PR.

DoD:
- [ ] Admin tworzy klinikę z logo
- [ ] Admin zaprasza dietetyka, email wysłany, dietetyk akceptuje, loguje się
- [ ] Admin widzi audit log z 2 wpisami (CLINIC_CREATED, DIETITIAN_INVITED)
- [ ] Próba wejścia non-admina na `/admin/*` → 403
- [ ] Wszystko działa na 360px (sprawdź w devtools)
- [ ] i18n PL/EN/ES — przełącz language, wszystkie teksty się zmieniają
- [ ] CI zielone

Raport po skończeniu.
```

---

## Prompt PR5 — Panel dietetyka

```
Rola: kontynuujesz Sprint 1 MyTrela (PR5). Po PR4 masz panel admina, dietetycy mogą się logować. Teraz UI dla dietetyka.

Zadanie: panel dietetyka — zarządzanie klientami (CRUD + invitation + access management).

ZAKRES PR5:
1. Branch: `claude/sprint-1-dietitian-panel` z `main`.
2. Layout: `app/dietitian/layout.tsx` — sidebar (Dashboard | Moi klienci | Profil | Wyloguj) + topbar (`<LocaleSwitcher>` + imię dietetyka + logo kliniki obok).
3. Routes:
   - `app/dietitian/page.tsx` — dashboard: kafle "Liczba moich klientów aktywnych", "Wygasają w 7 dni", "Nowi w tym miesiącu". Lista 5 ostatnich klientów z linkiem do szczegółów.
   - `app/dietitian/clients/page.tsx` — tabela klientów dietetyka (kolumny: imię, email, status, dni do wygaśnięcia, akcje). Filtr: status. Sort: po `accessEndsAt`. Paginacja.
   - `app/dietitian/clients/new/page.tsx` — formularz:
     - email (required)
     - imię (required)
     - notatka (optional, textarea)
     - czas dostępu: 3 radio (30 / 60 / 90 dni) + opcja "Inny" → date picker. **Default: 32 dni** (zgodnie z wizją).
   - `app/dietitian/clients/[id]/page.tsx` — szczegóły:
     - Info klienta (imię, email, kiedy dodany, status)
     - Pasek "Dostęp do <date> (<N> dni)"
     - Sekcja "Notatki dietetyka" (edytowalna)
     - Akcje: "Przedłuż dostęp" (modal: +30 / +60 / +90 / custom), "Cofnij dostęp" (set `accessEndsAt = now`, status CANCELLED), "Resetuj hasło" (nowy invitation), "Wyślij ponownie zaproszenie" (jeśli klient nigdy nie ustawił hasła).
     - Placeholder sekcji "Jadłospis", "Pomiary", "Materiały" — empty state z "Dostępne w kolejnym sprincie".
   - `app/dietitian/profile/page.tsx` — edycja imienia, hasła, locale.
4. Server Actions `app/dietitian/_actions/`:
   - `createClient` (waliduje Zod, tworzy User + Client + Invitation, wysyła email z `inviteClient` z PR3, audit log)
   - `updateClient` (notes, imię) — `withTenant(prisma, ctx).client.update(...)` — sprawdza że klient należy do tego dietetyka
   - `extendClientAccess(clientId, daysToAdd)` — auditowane
   - `revokeClientAccess(clientId)` — audit
   - `resetClientPassword(clientId)`, `resendInvitation(clientId)`
5. Wszystkie reads i writes na `Client` przez `withTenant(prisma, { role: 'DIETITIAN', clinicId, dietitianId })`.
6. Dashboard data:
   - "Aktywni" = `status === ACTIVE && accessEndsAt > now`
   - "Wygasają w 7 dni" = `status === ACTIVE && accessEndsAt BETWEEN now AND now+7d`
   - "Nowi w tym miesiącu" = `createdAt >= startOfMonth`
7. UX detale:
   - Toast po sukcesie ("Klient utworzony — zaproszenie wysłane")
   - Optimistic update przy przedłużeniu dostępu (jeśli potrafisz przy Server Actions; jeśli komplikuje — pomiń)
   - Confirm dialog przed "Cofnij dostęp" i "Resetuj hasło"
8. Test E2E `e2e/dietitian-flow.spec.ts`:
   - Login jako Ola (z seeda)
   - Tworzy klienta "Ewa", email `ewa@test.com`, 32 dni
   - Toast się pojawia, klient w liście
   - Email w `tmp/emails/`, accept invitation, login as Ewa → `/app`
9. Test separacji `tests/dietitian/separation.test.ts`:
   - Ola wywołuje `extendClientAccess(<Grzegorz's client>.id, 30)` przez Server Action → throw 'NotFound' lub redirect 404.
   - Bezpośrednie zapytanie Prisma z `withTenant(Ola's ctx).client.findUnique({ where: { id: <Grzegorz's client>.id } })` → null.

WAŻNE:
- Wszystkie `prisma.client.*` queries przez `withTenant`. Jeśli zapomnisz — Codex review zwróci.
- 404 zamiast 403 dla "cudzych" klientów — żeby nie wyciekać że istnieją.
- i18n: rosną klucze `dietitian.*` w PL/EN/ES.
- Default 32 dni — z wizji.

PROCES:
1. Branch `claude/sprint-1-dietitian-panel`.
2. Commity tematyczne.
3. Push + draft PR.

DoD:
- [ ] Ola loguje się → widzi swoich 2 klientów (i tylko ich)
- [ ] Ola tworzy nowego klienta → email wysłany → klient loguje się do `/app`
- [ ] Przedłużenie dostępu działa, widać nowe `accessEndsAt`
- [ ] Cofnięcie dostępu działa (status CANCELLED)
- [ ] Ola NIE widzi klientów Grzegorza w UI
- [ ] Ola NIE może modyfikować klientów Grzegorza przez API (test pokazuje)
- [ ] Mobile-first działa
- [ ] CI zielone

Raport po skończeniu.
```

---

## Prompt PR6 — PWA shell + login klienta + i18n switcher

```
Rola: zamykasz Sprint 1 MyTrela (PR6). Po PR5 admin i dietetyk działają. Teraz strona klienta jako PWA mobile-first.

Zadanie: pełnoprawna PWA dla klienta (manifest, service worker, offline shell, bottom nav, placeholder „Mój plan", przełącznik języka, instalowalna na telefon).

ZAKRES PR6:
1. Branch: `claude/sprint-1-pwa-shell` z `main`.
2. Layout `app/app/layout.tsx`:
   - Mobile-first, max-width 480px na desktop (centered).
   - Topbar: logo MyTrela (lewa) + nick klienta + `<LocaleSwitcher>` (prawa).
   - Bottom nav (fixed bottom, 4 ikony): "Plan", "Pomiary", "Materiały", "Profil". Tylko "Plan" działa w S1.
   - Padding-bottom dla nav, safe-area-inset-bottom dla iPhone.
3. Routes:
   - `app/app/page.tsx` (Plan) — empty state:
     - Ikona kalendarza
     - "Cześć, {imię}! Twój dietetyk pracuje nad Twoim planem. Wkrótce pojawi się tutaj."
     - Pokazuje też "Twój dostęp do: <data> (<N> dni)" z `Client.accessEndsAt`.
   - `app/app/measurements/page.tsx` — empty state "Dostępne wkrótce"
   - `app/app/materials/page.tsx` — empty state "Dostępne wkrótce"
   - `app/app/profile/page.tsx` — edycja imienia, hasła, locale (analog. do `/dietitian/profile`)
4. PWA — `@serwist/next`:
   - `app/sw.ts` — service worker z precaching + runtime caching:
     - Precache: `/app`, `/login`, `/manifest.json`, ikony, fonty
     - Runtime: NetworkFirst dla `/api/*` z fallback do offline page; CacheFirst dla statycznych assets
   - Offline page: `app/offline/page.tsx` — minimalny "Brak połączenia"
   - `next.config.ts` zaktualizowany dla serwist
5. Manifest (`public/manifest.json`):
   - name "MyTrela", short_name "MyTrela"
   - start_url "/app"
   - display "standalone"
   - background_color "#ffffff", theme_color "#16a34a"
   - icons 192/512 (placeholder z PR1 → wygeneruj lepsze: zielony okrąg z napisem "MT" białym, użyj `sharp` w skrypcie `scripts/generate-icons.ts`)
   - shortcuts: "Mój plan" → `/app`, "Pomiary" → `/app/measurements`
   - screenshots placeholder (do uzupełnienia przez grafika później)
6. `<LocaleSwitcher>` komponent (`components/locale-switcher.tsx`):
   - Dropdown z flagami PL/EN/ES (użyj `react-flag-kit` lub własne SVG)
   - Klik → Server Action `setLocale(locale)`:
     - Jeśli user zalogowany: update `User.locale`
     - Zawsze: cookie `NEXT_LOCALE`
     - Revalidate path
   - Zaimplementuj w topbarze KAŻDEGO layoutu: admin, dietitian, app, oraz na landing/login.
7. Lighthouse:
   - Cel: PWA score ≥90, Performance ≥85 na mobile (Moto G Power profile).
   - Dodaj `scripts/lighthouse.sh` który odpala lokalnie i wypluwa raport.
   - Jeśli PWA <90 — popraw (najczęstsze: brak `<meta name="theme-color">`, brak `<meta name="viewport">`, brak ikon w manifest).
8. Test E2E `e2e/pwa.spec.ts`:
   - Klient z seeda loguje się
   - Sprawdza że jest na `/app`, widzi placeholder
   - Sprawdza że bottom nav ma 4 ikony, klik na "Pomiary" → empty state
   - Sprawdza że `<meta name="theme-color">` jest obecny
9. Smoke test instalowalności (opisowo w PR description, ręczny test do zrobienia przez Manusa na prawdziwym Androidzie po deployu).
10. i18n — nowe klucze `app.plan.*`, `app.measurements.*`, `app.materials.*`, `app.profile.*`, `app.nav.*` w PL/EN/ES.

WAŻNE:
- Serwist + Next 15 App Router — sprawdź najnowszą dokumentację @serwist/next, są kruczki z compile/runtime.
- Mobile-first → wszystko testowane na 360px (Pixel 3a). NIE używaj `min-h-screen` (problem z address bar) — użyj `min-h-dvh`.
- Bottom nav: użyj `position: fixed; bottom: 0` z `pb-[env(safe-area-inset-bottom)]`.
- `<meta name="theme-color">` musi być w `app/layout.tsx` przez `viewport` export.
- Service worker w dev domyślnie wyłączony — `process.env.NODE_ENV !== 'production'` skip register.

PROCES:
1. Branch `claude/sprint-1-pwa-shell`.
2. Commity: "feat(pwa): serwist setup + manifest + sw", "feat(app): client layout with bottom nav", "feat(app): plan + placeholder screens", "feat(app): profile page", "feat(i18n): locale switcher component", "feat(icons): generate brand icons", "test(e2e): pwa basics", "docs: PWA install instructions".
3. Push + draft PR.

DoD:
- [ ] Klient z seeda loguje się i ląduje na `/app`
- [ ] Widzi swój nick, datę wygaśnięcia, empty state planu
- [ ] Bottom nav działa, każda zakładka renderuje (placeholder OK)
- [ ] Locale switcher zmienia język + persistuje
- [ ] Manifest + service worker zarejestrowane (devtools → Application → SW: activated)
- [ ] Lighthouse PWA ≥90 (zalącz screenshot w PR body)
- [ ] CI zielone
- [ ] **W PR description: instrukcja jak zainstalować na Androidzie do testu przez Manusa**

Raport po skończeniu — to ostatni PR Sprintu 1. W raporcie również: lista zaległości technicznych do Sprintu 2 (jeśli jakieś).
```

---

## Po Sprincie 1

Po merge wszystkich 6 PR-ów do `main`:
1. Manus deployuje na `mytrela-staging.fly.dev` (patrz [`04-sprint1-prompts-manus.md`](./04-sprint1-prompts-manus.md)).
2. Codex robi end-to-end audyt sprintu (patrz [`03-sprint1-prompts-codex.md`](./03-sprint1-prompts-codex.md) — sekcja "Sprint review").
3. Demo dla właściciela na staging.
4. Akceptacja → planowanie Sprintu 2 (produkty + przepisy).
