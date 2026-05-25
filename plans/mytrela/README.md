# MyTrela — Plan pracy (Sprint 1)

Plan budowy aplikacji dietetycznej MyTrela (PWA mobile-first + panel dietetyka/admina, multi-tenant: System → Klinika → Dietetyk → Klient, i18n PL/EN/ES od pierwszego sprintu).

> Źródło wizji: `807d22d5-Pe_na_wizja_aplikacji__Panel_administracyjny.docx` (przekazane przez właściciela projektu).
> Stack docelowy: **Next.js 15 (App Router) + TypeScript + Tailwind + Prisma + PostgreSQL**.
> Repo aplikacji: **nowe — `manuscodex/mytrela`** (do założenia).
> Repo koordynacyjne: **`manuscodex/manus-audit-inbox`** (tu — plan, audyty, ready-fixy).

## Role i workflow

| Aktor | Zadanie | Gdzie pracuje |
|---|---|---|
| **Claude Code** | Główny developer — pisze kod, otwiera PR-y | branche `claude/sprint-N-<temat>` w `manuscodex/mytrela` |
| **Codex (ChatGPT)** | Review PR-ów, ready-fixy | komentarze w PR + `codex-fixes/` w razie potrzeby |
| **Manus.im** | Deploy na staging/prod, audyty po-deployowe | `manuscodex/manus-audit-inbox` (raporty) + fly.dev (deploy) |
| **Właściciel** | Akceptacja PR, decyzje produktowe, merge | GitHub UI |

```
Claude Code ─push─▶ PR draft ─review─▶ Codex ─approve─▶ Właściciel merge ─▶ main ─▶ Manus deploy ─▶ staging/prod
                                  │
                                  └─▶ Audyt po-deployowy do manus-audit-inbox
```

## Spis treści

1. [`00-overview.md`](./00-overview.md) — cel produktu, architektura, decyzje
2. [`01-sprint1-plan.md`](./01-sprint1-plan.md) — szczegółowy plan Sprintu 1 (6 PR-ów, 4 tygodnie)
3. [`02-sprint1-prompts-claude.md`](./02-sprint1-prompts-claude.md) — **6 promptów dla Claude Code** (gotowe do wklejenia)
4. [`03-sprint1-prompts-codex.md`](./03-sprint1-prompts-codex.md) — prompty review dla Codex (per PR)
5. [`04-sprint1-prompts-manus.md`](./04-sprint1-prompts-manus.md) — prompt deploy + audyt dla Manus
6. [`05-github-setup.md`](./05-github-setup.md) — **jak podpiąć Claude Code do GitHub** + branch protection

## Szybki start

1. Przeczytaj [`05-github-setup.md`](./05-github-setup.md) — podepnij Claude Code.
2. Utwórz repo `manuscodex/mytrela` (puste, z `main` jako default).
3. Otwórz https://claude.ai/code, wybierz repo `mytrela`, branch `main` → wklej **Prompt PR1** z [`02-sprint1-prompts-claude.md`](./02-sprint1-prompts-claude.md).
4. Po pushu Claude Code zrobi draft PR — odpal Codexa z **Prompt review** z [`03-sprint1-prompts-codex.md`](./03-sprint1-prompts-codex.md).
5. Po akceptacji — merge — Manus deploy ze [`04-sprint1-prompts-manus.md`](./04-sprint1-prompts-manus.md).
6. Powtórz dla PR2–PR6.
