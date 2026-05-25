# 04 — Sprint 1: Prompty dla Manus.im (deploy + audyt po-deployowy)

> **Jak używać:** Manus deployuje po każdym merge do `main`. Po Sprincie 1 deployuje całość na `mytrela-staging.fly.dev`. Audyt po-deployowy wrzuca do `manus-audit-inbox/audits/<date>/`.

---

## Prerequisites (jednorazowo — przed PR1)

### 1. Założenie repo i aplikacji Fly

```
Manus, wykonaj następujące kroki konfiguracyjne dla nowego projektu MyTrela:

1. Załóż repo `manuscodex/mytrela` (private, default branch `main`).
2. Skonfiguruj branch protection na `main`:
   - Wymagaj PR
   - Wymagaj zielonego CI (`ci.yml` po PR1)
   - Wymagaj 1 approval (od Codex lub właściciela)
   - Wymagaj up-to-date branch
   - No force push
3. Załóż aplikację Fly.io: `mytrela-staging` w regionie `waw` (Warszawa).
4. Załóż Postgres na Fly: `mytrela-staging-db` (Hobby plan jest OK na MVP).
5. Załóż sekrety w Fly:
   - `DATABASE_URL` (z attach Postgres)
   - `NEXTAUTH_SECRET` (wygeneruj: `openssl rand -base64 32`)
   - `NEXTAUTH_URL=https://mytrela-staging.fly.dev`
   - `APP_URL=https://mytrela-staging.fly.dev`
   - `RESEND_API_KEY` (z konta Resend właściciela — poproś o klucz)
   - `ADMIN_SEED_PASSWORD`, `DIETITIAN_SEED_PASSWORD`, `CLIENT_SEED_PASSWORD` (każdy 16+ chars)
6. Załóż domenę email transactional w Resend dla `mytrela.app` (jeśli właściciel ma kontrolę nad domeną) lub użyj subdomain Fly do testów (`noreply@mytrela-staging.fly.dev`).
7. Załóż Fly Volume `clinic_uploads` 1GB mounted na `/app/public/uploads` (dla logo klinik).
8. Skonfiguruj webhook GitHub → Fly: deploy automatyczny po push na `main` (lub manualny przez `flyctl deploy` — wybierz strategię, udokumentuj w `manus-audit-inbox/plans/mytrela/`).
9. Raport: stwórz `manus-audit-inbox/plans/mytrela/SETUP-COMPLETE.md` z listą tego co skonfigurowane + linki do dashboardów.
```

---

## Deploy po każdym PR merge (PR1 → PR6)

```
Manus, PR <NR> Sprintu 1 MyTrela został zmergowany do `main`. Wykonaj deploy + smoke test + audyt:

1. Deploy:
   - Pobierz świeży `main` z `manuscodex/mytrela`.
   - Uruchom `flyctl deploy --remote-only` (lub odpowiednie polecenie z setupu).
   - Czekaj na zielony health check.
   - Jeśli to PR2+ (są nowe migracje): odpal `flyctl ssh console -C 'pnpm db:migrate deploy'` PRZED rolloutem (lub w release_command w fly.toml).
   - Po PR3 (auth) — odpal `flyctl ssh console -C 'pnpm db:seed'` na świeżej DB jeśli to pierwszy deploy z seedami.

2. Smoke test (zgodnie z DoD PR-a — patrz `plans/mytrela/01-sprint1-plan.md`):
   - PR1: otwórz https://mytrela-staging.fly.dev/ → widzi landing, zmiana cookie → zmiana języka.
   - PR2: nie ma UI zmian — sprawdź `pnpm db:studio` przez tunnel, weryfikuj seedy.
   - PR3: zaloguj się jako admin1, dietitian (przez invitation link z logów), client. Wszystkie ścieżki działają.
   - PR4: admin tworzy testową klinikę "ManusTest", zaprasza `manustest@mytrela.app`. Email przychodzi. Cleanup po teście.
   - PR5: dietitian tworzy testowego klienta, email przychodzi, klient się loguje. Cleanup.
   - PR6: PWA — zainstaluj na realnym Androidzie (Chrome → menu → Dodaj do ekranu głównego). Sprawdź offline (Airplane mode po pierwszym otwarciu). Lighthouse PWA score.

3. Audyt:
   - Folder `manus-audit-inbox/audits/<YYYY-MM-DD>/` z plikami:
     - `DEPLOY_REPORT.md` — co zdeployowano, czas, czy migracje przeszły, czy seed odpalony.
     - `SMOKE_TEST_REPORT.md` — checklist DoD PR-a z ✅/⚠️/❌ + screenshoty (PNG do podfolderu `screenshots/`).
     - `lighthouse-<route>.html` (jeśli PR6 — raport Lighthouse mobile).
   - Branch `manus/audit-<date>` z tymi plikami, PR do `main` w `manus-audit-inbox` (NIE w `mytrela`).
   - Powiadom właściciela (Slack/email) z linkiem do PR audit + verdict (Pass/Fail/Pass with warnings).

4. Jeśli FAIL:
   - Rollback Fly (`flyctl releases rollback`).
   - Otwórz issue w `manuscodex/mytrela` z `bug: <opis>` + reproduction steps + linki do logów Fly.
   - Tag właściciela + Claude (poprzez issue) — Claude może pickować naprawę w next session.

5. Cleanup po smoke testach: usuń test klinikę/dietetyka/klienta z bazy staging żeby nie zaśmiecać dashboard counts.
```

---

## Sprint 1 final deploy + acceptance test (po merge PR6)

```
Manus, Sprint 1 MyTrela kompletny (6/6 PR-ów merged). Wykonaj:

1. Pełny redeploy z czystej DB:
   - Backup obecnej DB staging (snapshot Fly).
   - Drop + recreate DB.
   - Deploy świeżego `main`.
   - Run migrations.
   - Run seed.
2. Pełny acceptance test (przejdź wszystkie scenariusze DoD ze sprintu — patrz `01-sprint1-plan.md` sekcja "Definition of Done (cały sprint)"):
   - [ ] Admin tworzy klinikę
   - [ ] Admin zaprasza dietetyka, email przychodzi, dietetyk akceptuje, loguje się
   - [ ] Dietetyk tworzy klienta, email przychodzi, klient akceptuje, loguje się do PWA
   - [ ] Separacja: drugi dietetyk w tej samej klinice nie widzi klientów pierwszego
   - [ ] i18n: PL/EN/ES działa w UI admin + dietitian + app
   - [ ] PWA instaluje się na Androidzie + iOS
   - [ ] Offline shell działa
3. Lighthouse na 4 routes:
   - `/` — Performance + A11y + SEO
   - `/login` — A11y
   - `/admin` (z sesją admina przez puppeteer) — Performance
   - `/app` (z sesją klienta) — Performance + PWA
4. Generuj raport:
   - `manus-audit-inbox/audits/<date>/SPRINT-1-ACCEPTANCE.md`
   - Sekcje: Deploy summary, Acceptance results, Lighthouse scores, Screenshots, Issues found, Verdict.
5. Verdict:
   - ✅ Ready for Sprint 2 — wszystkie DoD spełnione, Lighthouse zielone.
   - ⚠️ Conditional — DoD spełnione, ale są drobne issues do naprawy w S2.
   - ❌ Block — DoD nie spełnione, blocker(s) zidentyfikowane, lista do natychmiastowej naprawy.
6. PR z raportem do `manus-audit-inbox/main`.
7. Powiadomienie do właściciela + Claude + Codex z verdict.
```

---

## Deploy konwencje (na przyszłość)

- **Staging:** `mytrela-staging.fly.dev` — deploy automatyczny po merge do `main`.
- **Production:** `mytrela.app` — deploy MANUALNY przez Manus, po acceptance test właściciela na staging. Strategia: blue/green (Fly supportuje).
- **Database migracje:** zawsze `release_command` w `fly.toml` (`pnpm db:migrate deploy`) — zero-downtime tylko jeśli migracja jest backward-compatible (Claude pilnuje w PR).
- **Rollback:** jeśli release nieudany → `flyctl releases rollback` + revert PR w GitHub.
- **Secrets rotation:** co kwartał, log w `manus-audit-inbox/operations/secret-rotations.md`.
