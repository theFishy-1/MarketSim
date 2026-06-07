# Symulacja Rynku Giełdowego (księga zleceń)

Projekt zaliczeniowy z programowania komputerów. Program symuluje działanie rynku giełdowego, w którym
**cena instrumentu zmienia się na podstawie aktywności uczestników rynku, a nie losowo**. Cena powstaje
z relacji popytu i podaży w **księdze zleceń** (ang. *limit order book*), a użytkownik może składać własne
zlecenia i obserwować ich wpływ na rynek.

Inspiracja: film *„I made a Market Simulation to see if Patterns are Real”*
(<https://youtu.be/oWheof70O9g>).

## Jak uruchomić

To czysty JavaScript działający w przeglądarce — **nie wymaga instalacji ani budowania**.

1. **Najprościej:** kliknij dwukrotnie plik [`market_simulation.html`](market_simulation.html)
   (otworzy się w domyślnej przeglądarce).
2. **Opcjonalnie** (serwer lokalny, gdyby przeglądarka blokowała pliki lokalne):
   ```bash
   npx serve .
   # albo
   python -m http.server
   ```
   i otwórz `http://localhost:3000/market_simulation.html`.

Wykres ceny korzysta z biblioteki **Chart.js** ładowanej z sieci (CDN), więc do pełni funkcji potrzebny jest
internet. Wykres głębokości i mapa cieplna działają nawet bez niego.

Diagramy projektu: otwórz [`diagram.html`](diagram.html) (renderowane przez bibliotekę Mermaid).

## Jak to działa — model rynku

Cena **„chodzi po księdze”**: zlecenia rynkowe (takerzy) konsumują kolejne poziomy płynności i przesuwają cenę.
Kluczowa obserwacja (jak w filmie): przy samych zleceniach z limitem cena prawie stoi — realistyczny ruch pojawia
się, gdy **większość zleceń jest rynkowa**.

| Uczestnik | Zachowanie |
|-----------|-----------|
| **Animator rynku** (Market Maker) | Co turę uzupełnia płynność na drabinie poziomów po obu stronach — buduje „ściany” zleceń. Liczba poziomów, głębokość i odsunięcie od środka (spread) zależą od **profilu płynności** wybranego presetu (zob. „Tempo rynku"). |
| **Traderzy szumowi** | Losowy kierunek; ~80% zleceń **rynkowych** (napędzają ruch ceny). |
| **Podążający za trendem** | Kupują przy wzrostach, sprzedają przy spadkach → wzmacniają trendy i zmienność. |
| **Wieloryby** | Rzadkie, bardzo duże zlecenia rynkowe → wyraźny skok ceny (efekt dużego zlecenia). |
| **Użytkownik** | Składa zlecenia rynkowe/z limitem i widzi wpływ na cenę. |

### „Pamięć rynku” = wsparcie i opór

Niezrealizowane zlecenia z limitem **gromadzą się na poziomach cenowych**. Gdy cena rośnie ze 100 do 102,
a potem wraca, natrafia na nagromadzone zlecenia kupna z dołu, które działają jak **wsparcie**; analogicznie
powstaje **opór**. To nie jest wyrocznia — patterny (wsparcie, opór, trendy) to **matematyczny efekt księgi zleceń**.
Wizualizują to: wykres głębokości („ściany” płynności) oraz mapa cieplna płynności w czasie (żółta ścieżka ceny
odbija się od jasnych pasm płynności).

## Sterowanie i wykres

### Dwa niezależne zegary

Czas symulacji rozdzielono na dwa pokrętła:

- **Tempo rynku** (czas na 1 tick) — ile czasu RYNKU reprezentuje jedna tura/tick. To pokrętło realizmu: domyślnie *bardzo płynny* = **0,05 s/tick** (~20 aktualizacji/s, jak żywy płynny instrument), opcjonalnie 0,1 s lub 0,5 s/tick. „Tick" to **krok symulacji** (jedna runda agentów), a nie czas między transakcjami — w jednym ticku dzieje się wiele transakcji. Drabina interwałów jest z tego **wyliczana** (1m = 60 s, więc 1200 tur przy 0,05 s/tick). Ten preset niesie **cały profil płynności**, nie samo tempo — patrz niżej.

  **Profil płynności (wielowymiarowy).** Realna (nie)płynność to nie tylko TEMPO transakcji, lecz także **głębokość księgi**, **spread** i **wpływ pojedynczego zlecenia na cenę (impact)**. Kluczowe: *płynny rynek = szybsze ticki ORAZ głębsza księga* — gęstsza płynność wchłania większy napływ, więc cena pozostaje spokojna. Dlatego każdy preset ustawia komplet parametrów animatora rynku (`PACE_PRESETS` w `CFG`):

  | Preset | s/tick | Głębokość/poziom | Poziomy | Spread | Efekt (zmierzone) |
  |---|---|---|---|---|---|
  | Bardzo płynny | 0,05 | 35 | 18 | ~5¢ | ~20 akt./s, ~254 szt/s, **niski impact, spokojna cena** |
  | Średnio płynny | 0,1 | 18 | 13 | ~5¢ | ~10 akt./s, ~137 szt/s, średnia księga |
  | Mało płynny | 0,5 | 8 | 8 | ~20¢ | ~2 akt./s, ~36 szt/s, **cienka księga, szeroki spread, duże skoki ceny** |

  Dzięki temu „mało płynny" naprawdę zachowuje się jak instrument niepłynny (cienki arkusz, szerszy spread, większe skoki ceny na zlecenie), a „bardzo płynny" jak żywy, gęsty rynek — a nie ten sam przepływ rozciągnięty/ściśnięty w czasie.
- **Tempo symulacji** (czas rynku / s) — ile czasu rynku **odtwarzamy na 1 sekundę zegara** (kompresja). Pętla wykonuje wiele tur na klatkę i rysuje raz na klatkę. Suwak jest logarytmiczny, a jego **maksimum jest realnie osiągalne** (ograniczone limitem kroków na klatkę, by nie „ścinało”): przy 0,05 s/tick do ~1,5 godz/s, a przy grubszym Tempie rynku (np. 0,5 s/tick) do ~15 godz/s — bo wtedy jeden krok to więcej czasu rynku. Aby szybko przewijać godziny/dni, wybierz grubsze **Tempo rynku**. Obok suwaka widać **rzeczywiście** osiągane tempo (i „limit wydajności”, gdy maszyna nie nadąża).

KPI **Czas rynku** pokazuje, ile czasu rynku upłynęło (np. `3 d 04:12`) oraz numer tury.

### Pozostałe elementy

- **Start / Pauza / Reset** oraz przełączniki agentów (Szum / Trend / Wieloryb).
- **Formularz zleceń** (kupno/sprzedaż, rynkowe/z limitem, ilość) oraz przycisk **Zlecenie wieloryba** — pokazują wpływ zlecenia na cenę.
- **Typ wykresu:** linia lub **świece OHLC**.
- **Zoom** (przyciski −/+ obok kontrolek wykresu lub **kółko myszy** nad wykresem) — przybliża/oddala wykres ceny (pokazuje mniej/więcej najnowszych świec). Przydatne, gdy po dłuższym czasie świece robią się ciasne.
- **Interwał:** 1 tick, 1s, 5s, 15s, 1m, 5m, 15m, 1h, 4h, 1D, 1T (tydzień), 1Mc (miesiąc). Skala jest spójna i zmiana **nie resetuje** symulacji. Interwały drobniejsze niż 1 tick są **ukrywane** (np. przy „mało płynnym" = 2 s/tick znika „1s", bo zlałoby się z „1 tick" i dawało świece bez ciała); opcja „1 tick" pokazuje realny czas ticka, np. `1 tick (2 s)`. Interwały **poniżej 1 minuty** rysowane są z surowych ticków (okno przewijane ~3000 tur). Interwały **od 1 minuty** agregują się z trwałych **świec 1-minutowych** (do ~42 dni historii), więc 15m/1h/4h/1D gromadzą prawdziwe, stabilne świece i **wypełniają** wykres, a niekompletna najstarsza świeca jest pomijana (nie „pełza”). Zakończone świece nigdy się nie zmieniają — aktualizuje się tylko ostatnia, formująca się.
- **Księga** (nakładka na wykres ceny): **historia księgi** — poziome linie zleceń (zielone = kupno, czerwone = sprzedaż), których **przezroczystość zależy od rozmiaru** zlecenia. Migawki księgi zapisywane są przy świecach bazowych, więc nakładka pokrywa **cały wykres** i nie znika. Dodatkowo **żywa głębokość** (DOM) jako słupki przy prawej krawędzi.
- **Nakładka** (tryb historii księgi) — przełącznik *co* pokazuje nakładka:
  - **Spoczynek + ślad** (domyślnie): ściany **spoczywających** (niezrealizowanych) zleceń jak wyżej, ze **złoto-kremowym śladem** na tych poziomach, na których naprawdę doszło do transakcji. Pozwala odróżnić ściany „przetestowane" (realne wsparcie/opór) od czysto potencjalnych, ciągle odnawianych przez animatora.
  - **Wolumen**: czysty **profil wolumenu** (ang. *volume profile*) — ile **zrealizowanego** wolumenu przypadło na każdy poziom ceny. Jasne pasma to węzły wysokiego wolumenu (obszary wartości / *point of control*), w których rynek „spędził czas". To uzupełniające spojrzenie: *gdzie płynność czeka* (spoczynek) kontra *gdzie naprawdę handlowano* (wolumen).
  - Nakładka (oba tryby) jest **dokładna od interwału ≥ 1m** — dane księgi/wolumenu są zbierane raz na minutę rynku. Na interwałach sub-minutowych nakładka nie jest rysowana (zamiast tego krótka podpowiedź), a żywą płynność widać w panelu „Mapa płynności w czasie". Dlatego domyślny interwał to **1m**.

## Struktura kodu

Kod jest podzielony na osobne pliki (zero zależności do zainstalowania, **bez budowania** — klasyczne
`<script>` ładowane w kolejności zależności). [`market_simulation.html`](market_simulation.html) zawiera tylko
strukturę strony i dołącza [`styles.css`](styles.css) oraz moduły z katalogu [`js/`](js/):

| Plik | Zawartość |
|------|-----------|
| [`js/config.js`](js/config.js) | Obiekt `CFG` — **wszystkie parametry strojenia** i profile płynności w jednym miejscu. |
| [`js/utils.js`](js/utils.js) | Funkcje pomocnicze (formatowanie, zaokrąglanie do kroku ceny, próbkowanie). |
| [`js/order-book.js`](js/order-book.js) | `Order` (pojedyncze zlecenie) i `OrderBook` (dwie posortowane tablice, ceny w centach, priorytet cena-czas). |
| [`js/engine.js`](js/engine.js) | `MatchingEngine` — `executeOrder` (wspólna ścieżka „chodzenia po księdze”). |
| [`js/agents.js`](js/agents.js) | `Agent` + `MarketMaker`, `NoiseTrader`, `TrendFollower`, `Whale` — uczestnicy rynku. |
| [`js/simulation.js`](js/simulation.js) | `Simulation` — pętla, stan, zlecenia użytkownika (czysta logika, bez DOM). |
| [`js/renderer.js`](js/renderer.js) | `Renderer` — całe rysowanie i interakcja z DOM. |
| [`js/main.js`](js/main.js) | Wiązanie z interfejsem: pętla animacji, suwaki, formularz, obsługa błędów. |

Logika symulacji (`simulation.js`) jest w pełni oddzielona od warstwy widoku (`renderer.js`, `main.js`).

## Pliki

| Plik | Opis |
|------|------|
| `market_simulation.html` | Strona aplikacji — struktura i dołączenie stylów oraz modułów. |
| `styles.css` | Wygląd (style CSS). |
| `js/` | Logika aplikacji w modułach (zob. „Struktura kodu”). |
| `diagram.html` | Diagramy: schemat główny, silnik dopasowania, logika agentów, diagram klas (UML). |
| `README.md` | Ten plik. |

> Pliki `styles.css` i katalog `js/` muszą leżeć obok `market_simulation.html` (HTML ładuje je ścieżkami względnymi).
