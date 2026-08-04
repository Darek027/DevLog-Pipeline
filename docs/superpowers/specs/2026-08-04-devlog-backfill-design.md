# Backfill zaległych wpisów DevLog — design

Data: 2026-08-04
Status: **wykonany** — 16 wpisów opublikowanych 2026-08-04

## Problem

Tabela `devlogs` w Supabase bAIplot (`mgvsglveaejhwnkvrzxs`) miała 15 wierszy, najnowszy
z 2026-06-25. Od tamtej pory w repozytoriach powstało kilkadziesiąt commitów, ale żaden
nie trafił do publicznego DevLoga na baiplot.com.

Przyczyny są dwie, niezależne od siebie:

1. **Brak prefiksu `[Milestone]`.** Węzeł `Code in JavaScript` w produkcyjnym workflow
   odrzuca każdy push, w którym żaden commit nie zaczyna się od `[Milestone]`. 12 ostatnich
   wykonań n8n zakończyło się statusem `success` w 20–30 ms — czyli pipeline działał
   poprawnie, tylko nie miał czego przepuścić.
2. **Brak webhooka na części repozytoriów.** `Proactive-cms` nie ma ani jednego wpisu
   w bazie, mimo że commit `707ae87` (2026-06-29) **miał** prefiks `[Milestone]`.
   Webhooki uzupełnia Darek ręcznie po stronie GitHuba — poza zakresem tego dokumentu.

Osobno naprawiona przy okazji: węzeł `Code` czytał wyłącznie `head_commit`, więc commit
z prefiksem wypchnięty razem z późniejszymi zmianami był gubiony. Poprawka
(skanowanie całej tablicy `commits`) wypchnięta jako `4053e4a`.

## Zakres

W zakresie: `PROACTIVE`, `Proactive-cms`, `DevLog-Pipeline`.

Poza zakresem, świadomie:

- `Optotom` / `Optotom-CMS` — projekt komercyjny klienta; publiczny changelog jego prac
  to decyzja biznesowa, nie techniczna.
- `PROACTIVE_COMPANY` — wewnętrzna baza wiedzy (strategia, MASTER_BACKLOG). Nie jest
  materiałem na publiczną stronę.
- `AiAgentRAG` — ostatni commit `b23a1a4` (2026-06-09) jest już w bazie. Zero luki.
- `B2B_Leads`, `FamilyChef-Vercel`, `Cloudflare - Template` — brak commitów w oknie
  albo brak zdalnego repo.

## Decyzja: grupy tematyczne zamiast wpisów per-commit

63 commity dają 16 wpisów-releasów. Alternatywa (jeden wpis na commit) zalałaby publiczny
DevLog pozycjami typu „doc update" i „poprawka poprawki UX" — to nie jest ruch, to szum,
a DevLog jest wizytówką firmy.

`commit_hash` wpisu = SHA **ostatniego** commita w grupie. Jest unikalny i wskazuje realny
punkt w historii, więc pozostaje weryfikowalny.

`created_at` = rzeczywisty czas commita głowy grupy (nie północ, nie czas importu).

### Tablica grup

Grupy są ciągłe — każdy commit z okna trafia dokładnie do jednej grupy, `base` kolejnej
grupy jest `head` poprzedniej. Dzięki temu nic nie wypada między wpisami.

| # | repo | base | head | created_at | temat |
|---|---|---|---|---|---|
| P1 | PROACTIVE | `9b3daa0` | `50a9635` | 2026-06-26T13:38:38+02:00 | Treść sterowana z CMS: Automatyzacja AI i Aktualności jako tabele, portfolio, dynamic OG |
| P2 | PROACTIVE | `50a9635` | `0739962` | 2026-07-06T13:04:02+02:00 | Dopracowanie UX/UI, kontakt, normalizacja EOL, treści oferty |
| P3 | PROACTIVE | `0739962` | `dbea5f3` | 2026-07-08T20:34:32+02:00 | Pełny pakiet SEO + cennik oferty + OG images |
| P4 | PROACTIVE | `dbea5f3` | `423c669` | 2026-07-10T10:54:51+02:00 | Hardening: fail-safe B9, hard 404, HSTS 2 lata, reguły SSOT po incydencie P0 |
| P5 | PROACTIVE | `423c669` | `41cc7fc` | 2026-07-16T22:53:20+02:00 | Karta produktu — spec, plan, wdrożenie (hero + galeria) |
| P6 | PROACTIVE | `41cc7fc` | `dfe0fe4` | 2026-07-23T14:58:17+02:00 | Plany wdrożenia GA4 i formularza SmartPitch |
| P7 | PROACTIVE | `dfe0fe4` | `dc91169` | 2026-07-29T17:28:32+02:00 | Odświeżenie hero + poprawki modułu Automatyzacji |
| P8 | PROACTIVE | `dc91169` | `b25122b` | 2026-08-02T11:09:59+02:00 | Fix trailing slash 404 (routePath SSOT) + macierz walidacyjna adresów |
| P9 | PROACTIVE | `b25122b` | `6e279d7` | 2026-08-04T10:02:12+02:00 | bot-HTML: treść i nawigacja dla botów, walidacja 200 PASS / 0 FAIL na produkcji |
| C1 | proactive-cms | `722d0b6` | `838f069` | 2026-06-26T15:31:04+02:00 | Inicjacja panelu administracyjnego Proactive CMS |
| C2 | proactive-cms | `838f069` | `8b95b58` | 2026-06-30T14:41:34+02:00 | Social AI Manager — Faza 1 i 2 + generowanie postów |
| C3 | proactive-cms | `8b95b58` | `0b032d8` | 2026-07-08T20:20:43+02:00 | Generator obrazów i opisów — jakość, URL-e postów |
| C4 | proactive-cms | `0b032d8` | `e6be516` | 2026-07-14T23:19:26+02:00 | Migracja na nowe domeny + korekty generatora obrazów |
| C5 | proactive-cms | `e6be516` | `7352f9b` | 2026-07-17T16:18:21+02:00 | Przebudowa panelu + hardening bezpieczeństwa, socialView v3.4 |
| C6 | proactive-cms | `7352f9b` | `ac20f8a` | 2026-07-29T21:36:50+02:00 | Automatyzacja LinkedIn — Workflow E i O, grid zaplanowanych artykułów |
| D1 | DevLog-Pipeline | `10982c9` | `f5a4c55` | 2026-05-29T11:41:12+02:00 | Domknięcie konfiguracji pipeline'u |

Wszystkie 19 SHA są osiągalne z `main` i wypchnięte na origin — zweryfikowane
przez `git merge-base --is-ancestor` oraz `git status -sb`.

**Znane ograniczenie C1.** `722d0b6` to commit inicjalny repo (bez rodzica), więc nie da
się go użyć jako `head` w endpoincie `compare`. C1 obejmuje diff `722d0b6...838f069`,
czyli bez zawartości samego scaffoldu. Kontekst uzupełnia lista tematów commitów doklejana
do promptu oraz pole `label`. Diff scaffoldu i tak zostałby ucięty przez limit rozmiaru.

**`project_name`.** Wartość musi odpowiadać nazwie repo z payloadu GitHuba, żeby wpisy
backfillu były spójne z tym, co wstawia produkcyjny pipeline: `PROACTIVE`,
`proactive-cms`, `DevLog-Pipeline`.

## Architektura

Osobny, jednorazowy workflow **DevLog Backfill**. Produkcyjny `DevLog Pipeline`
(`6py1WRdPxr1lEQ52`) pozostaje nietknięty — backfill nie może zepsuć działającej ścieżki
webhookowej.

```
Manual Trigger
  → Code "Groups"            tablica 16 grup (repo, base, head, created_at, label)
  → Loop Over Items          batch = 1
      → HTTP GET  https://api.github.com/repos/Darek027/{repo}/compare/{base}...{head}
                  Accept: application/vnd.github.v3.diff     [cred: GitHub Token]
      → Code "Prepare"       truncate diff + doklejenie tematów commitów i label
      → HTTP POST Gemini 2.5 Flash                            [cred: Gemini_Google]
      → Supabase INSERT      commit_hash, project_name, message,
                             created_at (jawne), status = 'draft'   [cred: Supabase account]
      → Wait 7 s             limit ~10 RPM na darmowym planie Gemini
```

Sekrety wyłącznie w n8n Credential Manager — zgodnie z regułą bezpieczeństwa z README.
Nic nie trafia na dysk lokalny ani do transkryptu rozmowy.

### Dlaczego `status = 'draft'`

Publiczne API (`api/devlogs.js` w baiplot-platform) filtruje `status=eq.published`.
Wstawienie jako `draft` oznacza, że 16 tekstów wygenerowanych przez Gemini nie pojawi się
na baiplot.com, dopóki nie zostaną przeczytane. Publikacja to jeden `UPDATE` na końcu.
To treść marketingowa na stronie firmowej — przegląd przed publikacją jest tańszy niż
wycofywanie wpisu z produkcji.

## Idempotencja

Migracja `devlogs_unique_commit_hash` — zastosowana:

```sql
create unique index if not exists devlogs_commit_hash_key on public.devlogs (commit_hash);
```

Przed założeniem sprawdzono, że w 15 istniejących wierszach nie ma duplikatów `commit_hash`.

Węzeł Supabase ustawiony na `onError: continueRegularOutput`, więc powtórne uruchomienie
workflow odbije się o indeks (409 z PostgREST) i pętla pójdzie dalej zamiast się wywrócić.
Indeks chroni też produkcyjny pipeline przed podwójną dostawą webhooka.

## Obsługa błędów

| Ryzyko | Reakcja |
|---|---|
| Diff większy niż limit kontekstu / kilka MB | Truncate do 150 000 znaków w węźle `Prepare` + adnotacja `[diff truncated]` w prompcie |
| `compare` zwraca błąd (za dużo plików, 404) | Fallback: do Gemini idzie sama lista tematów commitów + `label`, bez diffu |
| Gemini 429 / 5xx | `retryOnFail: true`, `waitBetweenTries: 5000` — jak w produkcji |
| Limit RPM Gemini | Węzeł `Wait` 7 s w pętli |
| Duplikat `commit_hash` | Unikalny indeks + `continueRegularOutput` |

## Weryfikacja

1. Po przebiegu: `select project_name, count(*), min(created_at), max(created_at)
   from devlogs where status='draft' group by project_name` — oczekiwane 16 wierszy
   w rozbiciu 9 / 6 / 1.
2. Przegląd 16 tekstów pod kątem sensu i tonu (angielski, 2–3 zdania, język korzyści).
3. `update devlogs set status='published' where status='draft'`.
4. `GET /api/devlogs?limit=500` zwraca 32 wpisy w kolejności `created_at desc`.

## Wynik

Workflow **DevLog Backfill** (`srxPd657orpheEJ8`) uruchomiony 2026-08-04, execution `636`:
status `success`, czas 5 min 19 s, 16/16 grup przetworzonych. Żadna grupa nie wpadła
w fallback — GitHub `compare` zwrócił pełny diff dla wszystkich zakresów, więc limit
150 000 znaków ani ścieżka bez diffu nie zostały uruchomione.

Jedna ręczna korekta treści: wpis `0739962` (2026-07-06) powtarzał informację o sekcji
portfolio, ogłoszonej już we wpisie `50a9635` (2026-06-26). Wzmianka usunięta przed
publikacją.

Stan końcowy tabeli `devlogs`: **32 wiersze, wszystkie `published`**, zakres
2026-04-15 → 2026-08-04. Publiczne `GET /api/devlogs?limit=500` zwraca komplet 32 wpisów.

Workflow zostaje w n8n jako nieaktywny. Jest idempotentny dzięki unikalnemu indeksowi,
więc da się go bezpiecznie uruchomić ponownie po podmianie tablicy grup — to gotowe
narzędzie, gdyby luka kiedyś powstała ponownie.

## Czego ten dokument nie rozwiązuje

Przyczyna źródłowa zostaje: nic nie przypomina o prefiksie `[Milestone]` przy commicie.
Git hook `commit-msg` był rozważany i świadomie odrzucony w tej iteracji. Jeśli problem
wróci za miesiąc, to jest pierwszy kandydat do wdrożenia.
