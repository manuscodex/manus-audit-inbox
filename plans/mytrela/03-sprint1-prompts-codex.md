# 03 — Sprint 1: Prompty review dla Codex

> **Jak używać:** Po każdym draft PR od Claude Code odpal Codex (ChatGPT) z odpowiednim promptem. Codex ma już dostęp do `manuscodex/mytrela` przez integrację GitHub. Codex zostawia komentarze inline + sumaryczny komentarz + (jeśli trzeba) ready-fix patch w `codex-fixes/<branch>/`.
>
> **Zasada:** Codex NIE merguje PR-ów. Tylko Ty mergujesz po jego zielonej.

---

## Prompt review PR1 — Bootstrap

```
Rola: jesteś senior fullstack reviewerem. Twoja praca: krytyczne review draft PR `Sprint 1 / PR1 — Bootstrap` w repo `manuscodex/mytrela` (gałąź `claude/sprint-1-bootstrap`).

Kontekst: to pierwszy PR nowego projektu (MyTrela — PWA dietetyczna). Stack: Next.js 15 + TS strict + Tailwind 4 + Prisma 5 + Postgres + next-intl + @serwist/next. Pełny plan w `manuscodex/manus-audit-inbox` → `plans/mytrela/01-sprint1-plan.md` (sekcja PR1) i prompt dla Claude w `plans/mytrela/02-sprint1-prompts-claude.md` (sekcja PR1).

Co sprawdź (priorytet HIGH → LOW):

HIGH (blockery merge):
1. Czy `strict: true` + `noUncheckedIndexedAccess: true` w `tsconfig.json`? Czy są jakiekolwiek `any` (grep całe repo)?
2. Czy WSZYSTKIE teksty w UI przechodzą przez `useTranslations` / `getTranslations`? Czy żaden hardcoded polish/english/spanish string w JSX?
3. Czy `next-intl` poprawnie skonfigurowany (i18n.ts, request config, layout)?
4. Czy `@serwist/next` zainstalowany i `next.config.ts` zmodyfikowany zgodnie z dokumentacją (sprawdź wersję — najnowsza)?
5. Czy CI workflow działa: lint + typecheck + test + build + cache pnpm + cache .next?
6. Czy `.env.example` ma WSZYSTKIE klucze używane w kodzie? (grep `process.env.` vs `.env.example`)
7. Czy są sekrety w commitach (`.env`, `.env.local`)? Czy `.gitignore` je wyłapuje?
8. Czy `Dockerfile` jest multi-stage z Next.js standalone output (mniejszy obraz)?

MEDIUM:
9. Czy shadcn/ui zainicjalizowany poprawnie (`components.json` + Tailwind preset)?
10. Czy Prisma `schema.prisma` ma generator + datasource + minimum 1 model?
11. Czy `docker-compose.yml` używa Postgres 16 z health check?
12. Czy README.md ma quick start który DZIAŁA (zacznij od `pnpm install`, do `pnpm dev` na `localhost:3000`)?
13. Czy Husky pre-commit faktycznie odpala lint-staged?
14. Czy Playwright config + 1 e2e test obecny?
15. Czy Vitest config + 1 unit test obecny?

LOW (nice to have, nie blokujące):
16. Czy ESLint config rozszerza `eslint-plugin-tailwindcss`?
17. Czy `prettier.config.js` jest spójny z eslint?
18. Czy ikony PWA są ≥192 i ≥512 (nawet placeholder)?

Format output:
- Komentarze inline na PR (każdy z severity HIGH/MEDIUM/LOW).
- Sumaryczny komentarz na końcu: "Approve / Request Changes / Block" + lista 3-5 najważniejszych findings.
- Jeśli są HIGH findings → "Request Changes", przygotuj ready-fix patch i pushnij do `codex-fixes/sprint-1-bootstrap/` w repo `manuscodex/mytrela` (NIE do PR brancha — fix do ręcznego pickowania przez Claude lub właściciela).
- Jeśli tylko MEDIUM/LOW → "Approve with comments".

Zakończ podsumowaniem w komentarzu PR z listą gotowych fix patches (jeśli były).
```

---

## Prompt review PR2 — Schema multi-tenant

```
Rola: senior backend reviewer. Review PR `Sprint 1 / PR2 — Multi-tenant schema + seeds + tenant guard` w `manuscodex/mytrela`.

Kontekst: schemat domenowy Prisma + seedy + Prisma extension wymuszający tenant isolation. Plan w `manus-audit-inbox/plans/mytrela/01-sprint1-plan.md` (PR2), prompt Claude w `02-sprint1-prompts-claude.md` (PR2).

HIGH:
1. Czy każda tabela domenowa (oprócz User, Invitation, Session, AuditLog) ma `clinicId` i `@@index([clinicId])`?
2. Czy `withTenant` extension faktycznie blokuje cross-tenant reads dla ról DIETITIAN i CLINIC_ADMIN? (przeczytaj kod extension + testy)
3. Czy `withTenant` przy `create/update` na tabelach domenowych RZUCA jeśli `data.clinicId !== ctx.clinicId`?
4. Czy admin (`role === 'ADMIN'`) bypassuje `withTenant`? (powinien — admin widzi wszystko)
5. Czy `Client.dietitianId` filter jest wymuszany dla DIETITIAN (ale nie dla CLINIC_ADMIN)?
6. Czy testy tenant isolation pokrywają minimum: read-cross-tenant, read-cross-dietitian, write-cross-tenant?
7. Czy migracja działa na czystej bazie (`pnpm db:migrate dev` z drop)?
8. Czy seed jest idempotentny (drugi `pnpm db:seed` nie crashuje)?
9. Hasła w seedach: bcrypt cost ≥12, brak haseł w plaintext w kodzie (powinny być z env)?

MEDIUM:
10. Czy `Invitation.token` jest unique i ma `expiresAt` index?
11. Czy `Client.accessEndsAt` ma index (do alertów wygasania)?
12. Czy `AuditLog` ma index na `actorId` i `createdAt` (do filtrowania)?
13. Czy enum names są SCREAMING_SNAKE_CASE (Prisma convention)?
14. Czy relacje są obustronne (back-relations zdefiniowane)?
15. Czy `onDelete` jest świadomie ustawione na relacjach (default Restrict — dobrze; cascade tylko jeśli zamierzone)?

LOW:
16. Czy w README jest sekcja "Multi-tenant" z przykładem użycia `withTenant`?
17. Czy komentarze w `schema.prisma` wyjaśniają "dlaczego" przy nieoczywistych decyzjach?

Output: jak w PR1.

Dodatkowo: zaproponuj scenariusze ataków cross-tenant, których obecny `withTenant` może nie wyłapać (np. raw SQL, $queryRaw, agregaty, `_count`). Zostaw jako TODO do Sprint 5 (hardening).
```

---

## Prompt review PR3 — Autoryzacja + invitation

```
Rola: senior security + backend reviewer. Review PR `Sprint 1 / PR3 — Auth.js v5 + invitation flow` w `manuscodex/mytrela`.

Kontekst: pełna autoryzacja Auth.js v5 + flow zapraszania użytkowników + Resend email + rate limiting. Plan i prompt w `manus-audit-inbox/plans/mytrela/`.

HIGH (security blockers):
1. Czy bcrypt cost ≥12 wszędzie gdzie hashuje hasła?
2. Czy hasła NIGDY nie są logowane (grep `console.log.*password`, `logger.*password`)?
3. Czy `Invitation.token` generowany jest cryptographically secure (cuid2 lub crypto.randomBytes(32)), NIE Math.random?
4. Czy invitation tokens są jednorazowe (po `acceptedAt` nie można reusować)?
5. Czy expired tokens (>7d) odrzucane z friendly error?
6. Czy `NEXTAUTH_SECRET` wymagany (crash przy braku w prod)?
7. Czy cookies `httpOnly` + `secure` (w prod) + `sameSite: lax`?
8. Czy CSRF protection na Server Actions? (Next 15 ma origin check default — sprawdź czy nie wyłączone)
9. Czy rate limiting działa: 5/min per email + 20/min per IP na endpoint login? (jeśli in-memory — OK w MVP)
10. Czy middleware faktycznie blokuje routes per role? Sprawdź wszystkie warianty: anonim → /admin, dietitian → /admin, client → /admin, admin → /app.
11. Czy `session` callback NIE zwraca `user.password` ani innych sekretów?

MEDIUM:
12. Czy w dev (brak `RESEND_API_KEY`) email jest zapisywany do pliku lub logowany — nie crashuje?
13. Czy email template `invitation` ma i18n (pl/en/es) i poprawnie używa lokali?
14. Czy `forgot password` używa istniejącego mechanizmu invitation (DRY)?
15. Czy po accept invitation user jest auto-zalogowany?
16. Czy `acceptInvitation` jest atomowe (transaction wokół password update + invitation acceptance)?
17. Czy seed produkuje invitations dla dietetyków/klientów i loguje linki do konsoli (do testów lokalnych)?

LOW:
18. Czy strona `/login` ma link "zapomniałeś hasła"?
19. Czy `/forbidden` page jest user-friendly (nie tylko "403")?

Output: jak wcześniej. Szczególnie ostro w security findings.
```

---

## Prompt review PR4 — Panel admin

```
Rola: senior fullstack reviewer (UX + backend). Review PR `Sprint 1 / PR4 — Admin panel` w `manuscodex/mytrela`.

HIGH:
1. Czy każda Server Action waliduje input przez Zod?
2. Czy upload logo sprawdza mime przez `file-type` (NIE tylko extension)?
3. Czy upload sprawdza size (≤2MB)?
4. Czy upload nie pozwala na path traversal (np. `../../etc/passwd`)?
5. Czy admin routes są w middleware faktycznie chronione (test: dietitian session na /admin → 403)?
6. Czy DataTable nie wykonuje N+1 queries (sprawdź Prisma logi przy ładowaniu listy klinik z licznikami dietetyków/klientów — powinno być `_count` lub raw aggregation)?
7. Czy `AuditLog` faktycznie loguje każdą zmianę admin (test: utwórz klinikę, sprawdź wpis)?

MEDIUM:
8. Czy formularze mają inline validation feedback (react-hook-form + zodResolver)?
9. Czy sortowanie/paginacja w URL params (deeplink-friendly)?
10. Czy mobile-first działa: sidebar → drawer na <md (sprawdź na 360px)?
11. Czy i18n: wszystkie nowe stringi w PL/EN/ES?
12. Czy revalidatePath po każdej mutacji (świeże dane bez reload)?
13. Czy `inviteDietitian` blokuje email który już istnieje w bazie (nie tworzy duplikatu User)?
14. Czy "Resetuj hasło" wysyła nowy invitation, NIE generuje hasła plain text?
15. Czy E2E test admin-flow pokrywa pełny path (create clinic → invite → accept → login)?

LOW:
16. Czy są keyboard shortcuts albo focus management na ważnych akcjach?
17. Czy są loading states (skeleton, spinner) przy server actions?
18. Czy toast notifications po sukcesie/błędzie?

Output: jak wcześniej.

Dodatkowo: zaproponuj UX improvements (jeśli widzisz coś niewygodnego) jako LOW findings — Claude pickuje co chce w PR5+.
```

---

## Prompt review PR5 — Panel dietetyka

```
Rola: senior fullstack + multi-tenant security reviewer. Review PR `Sprint 1 / PR5 — Dietitian panel` w `manuscodex/mytrela`.

HIGH (multi-tenant correctness):
1. Czy KAŻDA query na `Client` w `app/dietitian/**` przechodzi przez `withTenant(prisma, { role: 'DIETITIAN', clinicId, dietitianId })`? Grep `prisma.client.` poza `withTenant` → jeśli są bezpośrednie wywołania, to bug.
2. Czy próba dostępu do cudzego klienta (innej kliniki LUB innego dietetyka tej samej kliniki) zwraca 404 (NIE 403, NIE 500, NIE crash)?
3. Czy Server Actions waliduja Zod + sprawdzają ownership PRZED operacją?
4. Czy `createClient` nie pozwala na podanie własnego `clinicId` w request (forced z `session.user.clinicId`)?
5. Czy `createClient` waliduje że email nie istnieje już jako klient innego dietetyka? (Jeśli email globalnie unique → user już istnieje gdzie indziej, blokuj.)
6. Czy default access time = 32 dni (z wizji)? Sprawdź `app/dietitian/clients/new/page.tsx`.
7. Czy `extendClientAccess` i `revokeClientAccess` są audytowane?
8. Czy E2E test pokrywa create → email → login klienta?
9. Czy test separacji testuje WSZYSTKIE Server Actions na cudzych klientach?

MEDIUM:
10. Czy dashboard counts (aktywni / wygasają / nowi) są poprawne dla SQL boundary conditions (np. dokładnie `now() + 7d`)?
11. Czy confirm dialog przed "Cofnij dostęp" i "Resetuj hasło"?
12. Czy "Wyślij ponownie zaproszenie" działa tylko jeśli klient nigdy nie ustawił hasła?
13. Czy notatki dietetyka NIE są widoczne klientowi (sprawdź relację — `Client.notes` powinno być readable tylko przez dietetyka/admin)?
14. Czy lista klientów ma paginację (≥10 klientów → 2 strony)?
15. Czy filtry/sort w URL params?
16. Czy i18n nowe klucze w PL/EN/ES?

LOW:
17. Czy są loading states + empty states (brak klientów → CTA "Dodaj pierwszego")?
18. Czy w `clients/[id]` placeholder sekcje (Jadłospis/Pomiary/Materiały) są wyraźnie oznaczone "Sprint 2/3" — żeby klient na demo wiedział że to coming soon?

Output: jak wcześniej.

Dodatkowo: zaproponuj jak rozszerzyć `withTenant` żeby pokrywało raw SQL (`$queryRaw`) — TODO do Sprint 5 hardening.
```

---

## Prompt review PR6 — PWA shell

```
Rola: senior frontend + PWA reviewer. Review PR `Sprint 1 / PR6 — PWA shell + client app` w `manuscodex/mytrela`.

HIGH:
1. Czy Lighthouse PWA score ≥90 (screenshot w PR body lub odpal lokalnie)?
2. Czy manifest.json poprawny (name, short_name, start_url, display: standalone, theme_color, background_color, icons 192+512)?
3. Czy service worker zarejestrowany w prod build (devtools → Application)?
4. Czy SW NIE rejestruje się w dev (`NODE_ENV !== 'production'` skip)?
5. Czy offline page działa (kill network → wejdź na `/app` → widzi offline page lub cached `/app`)?
6. Czy bottom nav nie zasłania content (`padding-bottom` na main + safe-area-inset-bottom)?
7. Czy `<meta name="viewport">` ma `width=device-width, initial-scale=1, viewport-fit=cover` (dla iPhone notch)?

MEDIUM:
8. Czy ikony PWA są real (nie 1x1 transparent), generowane przez skrypt z `sharp`?
9. Czy `<LocaleSwitcher>` jest w KAŻDYM layoucie (admin, dietitian, app, landing)?
10. Czy `setLocale` Server Action update zarówno `User.locale` (zalogowani) jak i cookie?
11. Czy placeholder ekrany (Pomiary, Materiały) wyraźnie komunikują "wkrótce" + nie crashują?
12. Czy "Plan" pokazuje `accessEndsAt` klienta z czytelnym formatem (data + ile dni zostało)?
13. Czy `/app/profile` pozwala zmienić hasło (przez current password + new password)?
14. Czy mobile-first działa na 360px (devtools)?
15. Czy E2E `pwa.spec.ts` pokrywa: login klienta → /app → nav action → offline scenario?

LOW:
16. Czy są shortcuts w manifest (quick action z home screen)?
17. Czy theme color w `<head>` matchuje manifest theme_color?
18. Czy w README jest sekcja "How to install on phone" (dla Manusa do smoke testu)?

Output: jak wcześniej.

Po review tego PR (ostatni w Sprincie 1) dodatkowo:
**SPRINT 1 SUMMARY REVIEW** — w komentarzu sumarycznym:
- Status Definition of Done sprintu z `01-sprint1-plan.md` (każdy punkt: ✅/⚠️/❌)
- Top 3 zaległości techniczne do Sprint 2 (jeśli są)
- Top 3 ryzyka architektoniczne które widzisz na horyzoncie
- Rekomendacja: czy iść do Sprint 2 czy najpierw 1-tygodniowy hardening sprint
```

---

## Sprint review (po merge wszystkich 6 PR-ów)

```
Rola: senior fullstack architect. Sprint 1 MyTrela zamknięty (6 PR-ów merged). Twoje zadanie: end-to-end audyt sprintu na main.

1. Sklonuj `manuscodex/mytrela`, branch `main`.
2. Odpal lokalnie: `pnpm install && docker compose up -d db && cp .env.example .env && pnpm db:migrate && pnpm db:seed && pnpm dev`.
3. Przejdź pełen flow:
   - Login admin1 → utwórz klinikę → zaproś dietetyka → akceptuj email link → login dietetyk → utwórz klienta → akceptuj klient → login klient → `/app`.
4. Sprawdź na telefonie (Chrome remote debugging lub urządzenie fizyczne):
   - Czy PWA się instaluje?
   - Czy działa offline po pierwszym otwarciu?
   - Czy responsive na 360/375/414 px?
5. Sprawdź tenant isolation:
   - Otwórz devtools → wykonaj fetch `/dietitian/clients/<id_cudzego_klienta>` z session Oli → oczekiwany 404.

Output:
- Komentarz w issue `manus-audit-inbox` issue "Sprint 1 Final Review" (utwórz jeśli nie ma).
- Sekcje: ✅ Co działa, ⚠️ Co działa z zastrzeżeniami, ❌ Co nie działa, 🔧 Propozycje fix, 📋 Backlog do Sprint 2.
- Verdict: Ready for Sprint 2 / Needs hardening sprint.
```
