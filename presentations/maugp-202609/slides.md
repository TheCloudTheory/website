---
marp: true
theme: cloudtheory
paginate: true
size: 16:9
---

<!-- _class: title -->

# 5 lat czy 27 dni?
## Disaster Recovery za rozsądne pieniądze: jak nie zbankrutować, zanim przyjdzie katastrofa

--- 

# O mnie

- Technology Advisor @ Protopia
- Microsoft Azure MVP
- Autor książek

---

# Punkt wyjścia

- Platforma używana wewnątrz dużej firmy, używana przez setki wewnętrznych klientów
- **Dziś: brak planu DR, brak polityki DR**
- Wcześniejsza próba zaplanowania DR - utknęła przy oszacowanych kosztach
- Postanowiłem policzyć wszystko od nowa, z realnych danych, zero zgadywania

> Zasada, którą trzymałem się przez cały projekt: **measure, don't assume**

---

# Skala, z jaką mam do czynienia

| Warstwa | Ilość |
|---|---|
| Instancje zbierające metryki | 600+ instancji |
| Aplikacje korzystające z platformy | 260+ |
| Repozytoria w magazynie logów | 160+ |
| Dashboardy wizualizujące dane | 2000+ |

---

# Dane

Point-in-time snapshot:

| Magazyn | Rozmiar | Przyrost |
|---|---|---|
| Long-term storage (metryki) | ~69,7 TiB | ~9,6 TiB/mies. |
| Logi (skompresowane) | ~713 TiB | ~53 TiB/mies. |
| Trace'y | ~25,7 GiB | pomijalne |

---

# Gdzie faktycznie jest luka

Wszystkie magazyny: **ZRS, jeden region**

- Awaria strefy dostępności (AZ) — **już obsłużona** (ZRS + klaster rozłożony na 3 strefy)
- Awaria całego regionu — **nic dziś tego nie przetrwa**

Co konkretnie znaczy "brak DR": nie brak jakiejkolwiek odporności, tylko brak **odporności na outage całego regionu**.

Cel: **RTO 8h, RPO 4h** (wszystkie dane jednolicie)

---

<!-- _class: meme -->

<img src="meme-1.jpeg" />

---

<!-- _class: section -->

# Krok 1: najprostsza odpowiedź na papierze

---

# Włączam GZRS wszędzie

Najprostsze rozwiązanie: zamienić ZRS na GZRS (Azure sam replikuje dane do drugiego regionu).

| | ZRS (dziś) | GZRS | Różnica |
|---|---|---|---|
| Logi (Storage A) | 9 939,83 $ | ~17 830,33 $ | +7 890,50 $ |
| Logi (Storage B) | 1 548,95 $ | ~2 779,28 $ | +1 230,33 $ |
| Metryki (Storage C) | 2 219,76 $ | ~3 024,98 $ | +805,23 $ |
| **Razem** | **13 708,54 $** | **~23 634,59 $** | **+9 926,06 $/mies.** |

W relacji do całego budżetu infrastruktury, koszt storage **to tylko ~10%.**

---

<!-- _class: section -->

# Krok 2: prawdziwe pytanie — jak wgrać dane, które już tam leżą?

---

# Punkt startowy: ~150 mln obiektów, ~700 TB

GZRS/replikacja obsługuje **nowe zapisy na bieżąco**. Nie odpowiada na pytanie:

> Jak przenieść dane, które już dziś leżą w Storage Account, zanim replikacja w ogóle zacznie działać?

Naturalny kandydat: **natywny backfill Azure Object Replication** ("skopiuj istniejące obiekty").

---

# Metodologia: liczba obiektów i rozmiar obiektów testowane osobno

Dla backfillu ~150 mln obiektów wąskim gardłem może być albo koszt pojedynczego wywołania API (dużo małych obiektów), albo surowa przepustowość sieci (duże obiekty):

- **Seria liczby obiektów** ("count"): małe, płaskie obiekty 20KB, w zakresie liczności 172 → 1 720 → 17 204 (skala 100x) — sprawdza, czy przepustowość trzyma się przy większej skali.
- **Jednorazowy test realistycznego rozmiaru** ("realistic"): stała, umiarkowana liczba (500) obiektów o rozmiarach jak prawdziwe segmenty w platformie (86% ~2MB, 14% ~72MB, średnia ~11,7MB).

---

# Pomiar 1: natywny backfill Object Replication

| Test | Obiekty | Przepustowość |
|---|---|---|
| Seria liczby obiektów | 172 → 17 204 | 0,49 → 56,59 obj/s |
| Realistyczny rozmiar (500 obiektów, ~11,7MB śr.) | 500 | 10,59 MB/s |

---

# Pomiar 1: pełna skala

Ekstrapolując do pełnej skali (~150M obiektów, ~1 755 TB):

- Ograniczenie liczbą obiektów: **~31 dni**
- Ograniczenie przepustowością (bajty): **~46 019 h ≈ 5,25 roku**

**Wiążące ograniczenie to zawsze większa z tych dwóch liczb: 5,25 roku.**

Natywny backfill - nie przejdzie.

---

<!-- _class: meme -->

<img src="meme-2.jpeg" />

---

# Pomiar 2: azcopy zamiast natywnego backfillu

Ten sam test, narzędziem `azcopy` (masowe kopiowanie równoległe):

| Test | Obiekty | Przepustowość |
|---|---|---|
| Seria liczby obiektów | 172 → 17 204 | 10,12 → 144,57 obj/s |
| Realistyczny rozmiar | 500 | 213,33 MB/s |

Ekstrapolując dalej:

- Ograniczenie liczbą obiektów: **~12 dni**
- Ograniczenie przepustowością: **~95 dni**

Realna poprawa (2,6×–20,1× względem Object Replication), ale wciąż **tygodnie, nie godziny**.

---

# Pomiar 3: dostosowanie współbieżności

Zmieniłem jedno ustawienie: `AZCOPY_CONCURRENCY_VALUE`

| Współbieżność | obj/s | MB/s |
|---|---|---|
| AUTO | 144,57 | 2,96 |
| 32 | 52,13 | 1,07 |
| **128** | **204,81** | **4,19** |
| 512 | 200,05 | 4,10 |
| 2048 | 204,81 | 4,19 |

**Wynik: powyżej 128 nic więcej nie zyskuję** — limit leży gdzie indziej (najpewniej po stronie konta/sieci, nie klienta).

---

# Pomiar 3: `azcopy` po "dostrojeniu"

Efekt: ~203h (~8,5 dnia) zamiast 288h — jedna linijka konfiguracji.

To poprawa dla limitu liczbą obiektów. Realistyczne dane (większe obiekty) mają inne ograniczenie — przepustowość, nie liczbę operacji.

---

# Pomiar 4: równoległe azcopy (realistyczny rozmiar obiektów)

Zamiast więcej wątków w jednym zadaniu — kilka niezależnych zadań `azcopy` naraz, każde na osobnym kontenerze.

| Zadania | MB/s |
|---|---|
| 1 | 475,89 |
| 2 | 618,66 |
| **4** | **754,97** |
| 6 | 575,38 |
| 8 | 287,50 |

---

# Pomiar 4: równoległe `azcopy`

**4 równoległe procesy to najlepszy zmierzony punkt.** dla 8 procesów — to potwierdzony (dwukrotnie) spadek wydajności: wspólne ograniczenie sieci/konta zaczyna dominować.

---

# Wynik końcowy: od 5,25 roku do 27 dni

| Etap | Ograniczenie (bajty) |
|---|---|
| Object Replication, natywny backfill | ~5,25 roku |
| azcopy, ustawienia domyślne | ~95 dni |
| azcopy, dostosowanie współbieżności | (dotyczy liczby obiektów) |
| azcopy, 4 procesy równolegle, realistyczne dane | **~27 dni** |

**Poprawa 71×** — z pomiarów, nie z założeń.

To wciąż nie jest operacja "jednego dnia" - ale to jednorazowy koszt uruchomienia DR, nie koszt każdego failoveru.

Koszt samych testów w Azure, jaki zapłaciłem za cały cykl pomiarowy: **5,89 $**.

---

<!-- _class: section -->

# Krok 3: jak zaprojektować replikację, żeby była tania na co dzień

---

# Aktywny-pasywny model z warstwowaniem

**Aktywny-pasywny** wygrywa z aktywnym-aktywnym: region podstawowy zostaje w warstwie **Hot** (bez wpływu na wydajność zapytań), region zapasowy w warstwie **Cool/Cold** — "podgrzewam" do Hot dopiero przy realnym failowerze.

To wymaga **Azure Object Replication** (nie natywnego GZRS): GZRS replikuje blob wraz z jego warstwą — nie da się mieć "Hot w regionie A, Cold w regionie B" jednym kontem GZRS. Object Replication to dwa osobne konta, każde z własną polityką na tiering obiektów.

**Mała uwaga** — backfill Object Replication - dogranie 150M *już istniejących* obiektów - zbyt wolne. Tutaj Object Replication odpowiada tylko za replikację *nowych, bieżących zapisów* — małych, ciągłych porcji danych, a nie jednorazowego catch-upu. `azcopy` robi jednorazową repliację, Object Replication przejmuje utrzymanie od tego momentu.

---

# Porównanie kosztów replikacji (wolumen magazynu logów)

| Podejście | Koszt dodatkowy/mies. |
|---|---|
| Natywny GZRS (Hot w obu regionach) | 7 809 $ – 12 473 $ |
| Object Replication, warstwa Cool | 6 501 $ – 9 669 $ |
| **Object Replication, warstwa Cold** | **2 893 $ – 3 907 $** |

**Cold-tier Object Replication: 63–70% taniej niż natywny GZRS.**

Zawiera koszt Cold ZRS po stronie zapasowej plus bieżący egress między regionami (~1 196 $/mies. przy obecnym przyroście danych).

---

# Cena, której nie widać w rachunku miesięcznym

Failover wymaga "podgrzania" danych **Cold → Hot** przed uruchomieniem klastra zapasowego.

Azure liczy to jako **Data Retrieval**: 0,03 $/GB.

| Szacunek wolumenu | Koszt jednorazowy failoveru |
|---|---|
| Niski (424 TB) | ~14 430 $ |
| Środkowy (672 TB) | ~22 848 $ |
| Wysoki (802 TB) | ~27 251 $ |

**To nie jest koszt miesięczny — to koszt jednego zdarzenia.** Dopóki failovery są rzadkie, oszczędność z warstwy Cold i tak wygrywa.

---

# Największe ryzyko nie jest kosztowe — jest czasowe

Object Replication kopiuje **pliki**. Nie odtwarza samo z siebie operacyjności platformy.

Platforma trzyma globalny katalog (lokalizacje segmentów, offsety, metadane) jako **stan wewnętrzny klastra**, nie jako gotowy indeks. Świeży klaster zapasowy musi ten katalog **odbudować, skanując dane** ("digest") — czas tej operacji rośnie z liczbą plików segmentów, nie tylko z ich rozmiarem.

Kolejność failoveru: **[Cold→Hot] → [digest]** — sekwencyjnie, nie równolegle.

**Czas digestu przy milionach segmentów: nieznany. To większe ryzyko dla 8h RTO niż jakikolwiek koszt w tej prezentacji.**

---

<!-- _class: meme -->

<img src="meme-3.jpg" />

---

# Inne ryzyka, które zostawiam otwarte

| Ryzyko | Opis |
|---|---|
| Lokalny bufor (3h) vs. RPO (4h) | Margines tylko 1h na opóźnienie replikacji |
| Replikacja danych w kolejkach | Co z danymi których retencja przekracza zadane RTO/RPO? |
| Digest danych do platformy | work-in-progress |

Wszystkie wymagają zmierzenia przed wdrożeniem produkcyjnym.

---

<!-- _class: section -->

# Krok 4: czy wszystkie dane zasługują na tę samą ochronę?

---

# Koncentracja danych

**Mniej niż 15% wszystkich klientów odpowiada za ~95,8% całego wolumenu**.

| Opcja | Opis |
|---|---|
| **A — jednolita ochrona** | Ten sam RTO/RPO dla wszystkiego. Prostsze, ale koszt rośnie z wolumenem, nie z ważnością danych |
| **B — warstwy wg krytyczności** | Krytyczne/standardowe/best-effort. Nie chodzi o oszczędność — chodzi o elastyczność i kolejność wdrożenia |

Obecne założenie robocze: **Opcja A** — dla prostoty. 

---

# Wnioski

1. **Najprostsza cyfra (9 926 $/mies. za GZRS) nie była problemem.** Problem znalazłem w operacyjnej wykonalności — jak przenieść dane, które już tam są.
2. **Miary, nie założenia** — z 5,25 roku do 27 dni, tym samym narzędziem, tylko inaczej skonfigurowanym.
3. **Tiering magazynu (Hot/Cold) obniża koszt bieżący o ~65%**, ale przenosi koszt na jedno zdarzenie: failover.
4. **Koszt failoveru jest policzony. Czas failoveru — najważniejszą liczbę w całym projekcie — wymaga dopracowania.**

---

<!-- _class: closing -->

<div>

# Dziękuję

Pytania?

</div>

<img src="qrcode.jpeg" />
