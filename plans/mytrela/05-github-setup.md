# 05 — Podpięcie Claude Code do GitHub + workflow

## TL;DR — Co musisz zrobić

1. Zainstaluj GitHub App **Claude Code** w organizacji `manuscodex`.
2. Daj jej dostęp do repo `manuscodex/mytrela` (i opcjonalnie `manuscodex/manus-audit-inbox`).
3. Otwórz https://claude.ai/code, wybierz repo, branch — i działa.

To wszystko. Reszta tego dokumentu to szczegóły i opcjonalna konfiguracja.

---

## 1. Instalacja GitHub App (5 minut)

1. Otwórz w przeglądarce: https://github.com/apps/claude (oficjalny GitHub App dla Claude Code).
2. Kliknij **"Install"** lub **"Configure"** jeśli już zainstalowany dla innego konta.
3. Wybierz organizację: **`manuscodex`** (lub konto osobiste, jeśli repo będzie tam).
4. Wybierz scope:
   - **Recommended:** "Only select repositories" → zaznacz `mytrela` i `manus-audit-inbox`.
   - **Alternative:** "All repositories" — jeśli chcesz dawać Claude dostęp do nowych repo bez ręcznej zmiany.
5. Zaakceptuj uprawnienia (Read+Write na contents, PRs, issues, actions).
6. Gotowe — Claude Code może teraz operować na tych repo.

> Jeśli `claude.ai/code` jeszcze nie pyta o autoryzację GitHub przy starcie sesji — dostaniesz prompt podczas pierwszej sesji, kliknij "Authorize".

---

## 2. Założenie repo `manuscodex/mytrela`

1. github.com/organizations/manuscodex/repositories/new
2. Nazwa: `mytrela`
3. Private: tak
4. Add README: TAK (żeby był initial commit i branch `main`)
5. .gitignore: Node
6. License: brak (lub MIT do decyzji właściciela)
7. Stwórz.

**Branch protection na `main`** (Settings → Branches → Add rule):
- Branch name pattern: `main`
- ✅ Require a pull request before merging
  - ✅ Require approvals: 1
  - ✅ Dismiss stale approvals when new commits are pushed
- ✅ Require status checks to pass before merging
  - Po PR1 dodaj `ci / lint`, `ci / typecheck`, `ci / test`, `ci / build` jako required
- ✅ Require branches to be up to date before merging
- ✅ Require conversation resolution before merging
- ❌ Allow force pushes
- ❌ Allow deletions

---

## 3. Codex (ChatGPT) — workflow review

Codex ma już dostęp do repo `manuscodex/*` przez integrację OpenAI ↔ GitHub. Twoja praca z Codex:

1. Po draft PR od Claude wejdź na chat.openai.com/codex.
2. Wybierz repo `manuscodex/mytrela`, branch PR-a.
3. Wklej prompt z `plans/mytrela/03-sprint1-prompts-codex.md` odpowiadający temu PR-owi.
4. Codex robi review — komentarze inline + sumaryczny komentarz na PR.
5. Jeśli Codex znajdzie HIGH issues — przygotuje ready-fix w `codex-fixes/<branch>/`. Możesz:
   - Wkleić patch do nowego promptu Claude Code ("zaaplikuj poniższy patch i pushnij na ten sam branch") i Claude doda fix do PR.
   - Albo zmergować ready-fix bezpośrednio (jeśli prosty), jak w obecnym workflow `manus-audit-inbox`.

---

## 4. Manus.im — workflow deploy

Manus ma już dostęp do repo. Twoja praca z Manus:

1. Po merge PR do `main` w `mytrela` → odpal Manus z promptem z `plans/mytrela/04-sprint1-prompts-manus.md` (sekcja "Deploy po każdym PR merge").
2. Manus deployuje, robi smoke test, generuje audyt.
3. Manus pushuje audyt do `manus-audit-inbox` jako PR.
4. Akceptujesz audyt PR → merge.

Manus zna już ten flow — używał go w GGC Gaming (patrz `audits/2026-05-17/`).

---

## 5. Workflow end-to-end (przykład: PR1)

```
Ty (właściciel):
  1. Otwierasz https://claude.ai/code
  2. Wybierasz repo: manuscodex/mytrela, branch: main
  3. Wklejasz prompt PR1 z 02-sprint1-prompts-claude.md
  4. Czekasz (Claude pracuje 30-90 min)

Claude Code (automatycznie):
  5. Tworzy branch claude/sprint-1-bootstrap
  6. Robi commits, push, otwiera draft PR do main
  7. Raportuje Ci w sesji: link do PR + co zrobione

Ty:
  8. Otwierasz chat.openai.com/codex
  9. Wybierasz repo + branch PR-a
  10. Wklejasz prompt review PR1 z 03-sprint1-prompts-codex.md

Codex:
  11. Review, komentuje PR, ewentualnie ready-fix do codex-fixes/

Ty:
  12. Czytasz review Codexa
  13. Jeśli OK → mark PR ready → merge do main
  14. Jeśli są fixy → wracasz do Claude z nowym promptem "zaaplikuj patch X"
      lub mergujesz ready-fix bezpośrednio

Manus.im (po merge):
  15. Wykrywa merge przez webhook lub odpalasz ręcznie
  16. Deploy na mytrela-staging.fly.dev
  17. Smoke test + audyt
  18. PR z audytem do manus-audit-inbox

Ty:
  19. Akceptujesz audyt
  20. Przechodzisz do PR2 (powtarzasz od kroku 1)
```

---

## 6. Jak Claude Code zachowuje się w sesji

- **Środowisko:** uruchamia się w ephemeral container w cloud (nie na Twoim laptopie). Każda sesja to świeży clone repo.
- **Sieć:** ograniczona policy. Pakiety npm/pip — OK. Wywołania do prywatnych API — zależy od policy (możesz skonfigurować).
- **Sekrety:** możesz wstrzyknąć environment variables per environment (Settings → Environments w claude.ai/code).
- **Czas:** sesja ginie po godzinach bezczynności. Zawsze pushuje commit przed końcem (więc nic nie tracisz).
- **Branche:** Claude pracuje na branchu, który mu wskażesz. Nigdy nie pushuje na `main` bez Twojej zgody (chyba że jawnie poprosisz).
- **PR-y:** Claude może otwierać PR — robi to przez GitHub MCP w sesji.

---

## 7. Konfiguracja środowiska Claude Code (opcjonalna)

W `claude.ai/code` → Settings → Environments możesz utworzyć "MyTrela" environment z:

- **Setup script:**
  ```bash
  pnpm install
  docker compose up -d db
  cp .env.example .env
  # wstrzyknij sekrety z env Claude Code:
  echo "DATABASE_URL=postgresql://mytrela:mytrela@localhost:5432/mytrela" >> .env
  echo "NEXTAUTH_SECRET=$NEXTAUTH_SECRET" >> .env
  echo "RESEND_API_KEY=$RESEND_API_KEY" >> .env
  pnpm db:migrate
  pnpm db:seed
  ```
- **Environment variables:**
  - `NEXTAUTH_SECRET` (test secret, generuj raz)
  - `RESEND_API_KEY` (test key z Resend — możesz mieć osobny do dev/test)
  - `ADMIN_SEED_PASSWORD` = `test1234`
- **Network policy:** "Public" (dostęp do registry npm, GitHub, ewent. Resend API).

Dzięki temu każda nowa sesja Claude Code na tym repo startuje już z gotowym dev env i może odpalać testy/dev natychmiast.

---

## 8. Wskazówki operacyjne

- **Jeden prompt = jeden PR.** NIE łącz PR1 i PR2 w jednej sesji — Codex review wymaga separacji.
- **Czytaj raport Claude.** Po sesji zawsze sprawdź co pushnął — nawet jeśli wygląda na sukces.
- **Jeśli Claude utknie:** wpisz w sesję "zacznij od nowa z czystej main" lub "abort and rollback uncommitted changes". Claude rozumie.
- **Sekrety NIGDY w promptach.** Jeśli Claude musi użyć sekretu — wstrzyknij przez environment, nigdy nie wklejaj w prompcie (logi sesji mogą być cache'owane).
- **Branche krótko żyjące.** Po merge PR-a zachęć Claude w następnej sesji do `git branch -D claude/sprint-1-bootstrap` żeby nie zaśmiecać.
- **Czas między PR-ami:** zostaw 1 dzień na Codex review + Manus deploy. NIE odpalaj 6 PR-ów w jeden weekend — review traci jakość.

---

## 9. Co dalej (po Sprincie 1)

- Wracasz do mnie (Claude w tej sesji `claude/keen-dijkstra-dzKps` w `manus-audit-inbox`) z raportami z S1.
- Razem planujemy Sprint 2 (produkty + przepisy + skalowanie) z analogicznym poziomem szczegółu.
- Dodajemy `plans/mytrela/sprint-2/` z analogiczną strukturą plików.

Albo — jeśli wszystko poszło sprawnie i nie potrzebujesz mojego inputu — przejdź sam do Sprintu 2 z prostszymi promptami opartymi na sekcji "Sprint 2" z `wizji.docx` + plan ze Sprintu 1 jako templates.
