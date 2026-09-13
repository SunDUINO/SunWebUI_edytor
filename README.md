# SunGo GUI Builder

Edytor GUI działający jako aplikacja desktopowa (webview_go + serwer HTTP w Go),
który generuje statyczny frontend (`index.html`, `style.css`, `app.js`) oraz
szablon backendu Go (`backend.go`), gotowe do dalszego rozwijania.

<img width="1720" height="971" alt="8f85cee7ef9f9552c3ca965a3a0be412" src="https://github.com/user-attachments/assets/01ea373b-b931-4ff1-a754-79eb818a7c65" />


## Spis treści

- [Struktura projektu edytora](#struktura-projektu-edytora)
- [Uruchomienie](#uruchomienie)
- [Praca z projektami](#praca-z-projektami)
- [Komponenty](#komponenty)
- [Rozmieszczanie komponentów](#rozmieszczanie-komponentów)
- [Panel właściwości](#panel-właściwości)
- [Rozmiar okna, Podgląd, okno testowe](#rozmiar-okna-podgląd-okno-testowe)
- [Tłumaczenia PL/EN](#tłumaczenia-plen)
- [Eksport](#eksport)
- [Integracja z backendem Go](#integracja-z-backendem-go)
- [Rozwiązywanie problemów](#rozwiązywanie-problemów)

## Struktura projektu edytora

```
go-gui-builder/
├── go.mod                     # moduł: SunWebUI_edytor
├── media/                     # ikony, splash (konwencja SunGo)
├── src/
│   ├── main.go                # start webview + serwer HTTP + dialogi plików/folderów
│   ├── editor/
│   │   ├── server.go           # /api/save /api/load /api/projects /api/export
│   │   ├── model.go            # Component, StyleProps, Project, Meta
│   │   └── generator.go        # generatory HTML/CSS/JS/backend.go (eksport)
│   ├── Frontend/               # UI SAMEGO edytora (osadzone w binarce przez go:embed)
│   │   ├── index.html
│   │   ├── editor.js
│   │   ├── editor.css
│   │   ├── nls_pl.json          # tłumaczenia PL
│   │   └── nls_en.json          # tłumaczenia EN
│   └── templates/
└── projects/
    └── <Nazwa Projektu>.json   # jeden plik na projekt — nazwa pliku = nazwa projektu
```

## Uruchomienie

```bash
go mod tidy
go run ./src
```

Wymagania: kompilator C (cgo) dla `webview_go`/`sqweek/dialog` — na Windows to
zwykle MinGW-w64 (gcc) w PATH; na Windows potrzebny jest też zainstalowany
Microsoft Edge WebView2 Runtime (zwykle jest domyślnie z Edge).

## Praca z projektami

- Pole **"Projekt"** w topbarze to nazwa bieżącego projektu.
- **"Zapisz"** zapisuje pod tą nazwą jako osobny plik `projects/<Nazwa>.json` —
  zmiana nazwy przed zapisem tworzy nowy plik, nie nadpisuje poprzedniego.
- **"Nowy"** czyści canvas i pyta o nazwę nowego projektu.
- Rozwijana lista obok **"Wczytaj"** pokazuje wszystkie zapisane projekty.
- Pliki projektów zapisują się obok pliku wykonywalnego (folder `projects/`),
  niezależnie od katalogu, z którego program uruchomiono.

## Komponenty

| Komponent | Opis |
|---|---|
| Przycisk | `<button>` — typowo z ustawionym bindingiem (patrz [Integracja](#integracja-z-backendem-go)) |
| Label | zwykły tekst |
| Pole tekstowe | `<input type="text">` |
| Lista rozwijana | w edytorze pokazuje się jako statyczna atrapa (żeby dało się ją przeciągać, bez otwierania natywnego dropdownu), w Podglądzie/eksporcie to prawdziwy `<select>` |
| Checkbox | `<label><input type="checkbox"></label>` — pozycja/rozmiar ustawiane są na `<label>`, żeby opis nie "uciekał" |
| Obraz | domyślnie 100×100px; wybór pliku (tylko PNG/GIF) przez natywny dialog, obraz osadzany jako base64 w `index.html` (eksport jest samowystarczalny — nie ma osobnych plików graficznych) |
| Kontener | prawdziwy groupbox: `<fieldset>` + opcjonalny `<legend>` (pusty tytuł = sama ramka) |
| Panel | zwykła, bezstylowa skrzynka na komponenty |
| Belka | pozioma belka/toolbar — domyślnie układ "Wiersz" |
| Konsola | duże, przewijalne pole w stylu terminala (`<textarea readonly>`, ciemne tło, monospace) — treść edytujesz w polu "Zawartość" |

Kontener/Panel/Belka akceptują przeciągane dzieci (patrz niżej).

## Rozmieszczanie komponentów

- Canvas to "tablica robocza" o dokładnym rozmiarze okna (pola Szer./Wys. w topbarze).
- **Komponenty najwyższego poziomu** (bezpośrednio na canvasie) przeciągasz w
  dowolne miejsce — pozycja to bezwzględne `left`/`top` (px), identyczne w
  edytorze i w eksporcie.
- **Komponenty w Kontenerze/Panelu/Belce** układają się według wybranego
  "Układu dzieci" (widoczny w panelu właściwości po zaznaczeniu kontenera):
  - **Jeden pod drugim** — zwykły przepływ blokowy (domyślne dla Kontenera/Panelu)
  - **Wiersz** — obok siebie (domyślne dla Belki)
  - **Kolumna** — pionowo, ale z równym odstępem (flex)
  - **Siatka** — `display: grid` z konfigurowalną liczbą kolumn (np. siatka 3×3
    przycisków: Panel → Układ "Siatka" → Liczba kolumn "3" → wrzuć 9 przycisków)
  - **Nieregularne układy** (np. 2 elementy jeden nad drugim obok 3 w rzędzie)
    osiągasz przez **zagnieżdżanie**: Panel z układem "Wiersz" zawierający
    3 przyciski plus jeden zagnieżdżony Panel z układem "Kolumna" trzymający
    pozostałe 2.
- Nowo dodany komponent ma domyślne Szerokość/Wysokość/Margines/Padding
  zmierzone z jego faktycznego wyglądu (nie zaczynasz od pustych pól) —
  wyjątek: `button`/`input` nie dostają wymuszonej grubości obramowania/promienia
  zaokrąglenia, żeby nie tracić natywnego, systemowego wyglądu.

## Panel właściwości

W zależności od typu zaznaczonego komponentu, panel pokazuje różne pola:

- **ID / nazwa techniczna** — zmiana nazwy (np. `button_1` → `btnStart`),
  walidowana (litery/cyfry/`_`/`-`, unikalność), propaguje się automatycznie
  wszędzie (ten sam identyfikator w HTML/CSS/JS).
- **Usuń komponent** — z potwierdzeniem.
- **Tekst / zawartość** (lub **Zawartość** dla Konsoli, **Obraz** dla typu Obraz).
- **Tytuł ramki** — tylko dla Kontenera.
- **Kolory**: kolor tekstu, tło.
- **Wymiary i odstępy**: przełącznik jednostek Piksele/Procent na górze sekcji —
  wpisujesz samą liczbę, jednostka dokleja się automatycznie. Szerokość,
  Wysokość, Margines, Padding, Zaokrąglenie rogów, Grubość obramowania,
  Kolor obramowania.
- **Układ dzieci** — tylko dla Kontenera/Panelu/Belki (patrz wyżej).
- **Typografia**: czcionka z listy (typowe fonty Windows/Linux + generyczne).
- **Backend**: pole "Zdarzenie (binding)" — patrz [Integracja](#integracja-z-backendem-go).

Kliknięcie **pustego miejsca canvasu** pokazuje zamiast tego właściwości
samego okna (kolor tła `#app-root`).

## Rozmiar okna, Podgląd, okno testowe

- Pola **Szer./Wys.** w topbarze ustawiają rozmiar docelowego okna — canvas
  edytora ma dokładnie taki rozmiar, a `#app-root` w eksporcie/Podglądzie też.
- **"Podgląd"** przełącza canvas na w pełni wyrenderowany, samodzielny HTML
  (dokładnie to, co wyjdzie z eksportu) w ramce o zadanym rozmiarze. Przyciski
  z ustawionym bindingiem są klikalne — pokazują krótki komunikat, jaki event
  poleciałby do backendu (prawdziwego backendu nie ma w Podglądzie, więc nie
  próbujemy prawdziwego `fetch()`, tylko sygnalizujemy co by się stało).
- **"Otwórz w oknie"** robi to samo, ale w osobnym oknie przeglądarki
  (`window.open` z Blob URL) — przydatne do przetestowania w realnym rozmiarze
  poza panelami edytora. Zachowanie zależy od backendu webview (WebView2 na
  Windows zwykle otwiera osobne okno; WebKitGTK na Linuksie czasem otworzy
  zwykłą zakładkę zamiast okna — w takim wypadku użyj zwykłego "Podglądu").

## Tłumaczenia PL/EN

- `nls_pl.json` / `nls_en.json` obok plików frontendu edytora — pokrywają
  chrom UI edytora (przyciski, etykiety, nazwy komponentów). Nie tłumaczą
  treści samego projektowanego GUI (to dane użytkownika).
- Przełącznik w belce statusu (dół okna) pokazuje język, **na jaki** program
  się przełączy (po polsku widzisz "English", po angielsku "Polski").
- Belka statusu ma też link do `forum.lothar-team.pl` po prawej.

## Eksport

Pliki lądują w typowym układzie SunGo, w podfolderze nazwanym jak projekt
(znaki niedozwolone w nazwach plików/folderów są usuwane):

```
<wybrany folder>/<Nazwa Projektu>/
└── SRC/
    ├── backend.go              # szablon backendu — patrz "Integracja z backendem Go"
    └── frontend/
        ├── index.html
        ├── style.css
        └── app.js
```

`backend.go` serwuje pliki z sąsiedniego `frontend/` (ścieżka względna do
siebie) — uruchamiaj go z wnętrza folderu `SRC/`:

```bash
cd "<Nazwa Projektu>/SRC"
go run .
```

## Integracja z backendem Go

Zobacz **[INTEGRATION.md](INTEGRATION.md)** — obszerny przewodnik krok po
kroku, jak podłączyć logikę Go do wyeksportowanego interfejsu, z przykładami
kodu dla każdego typu komponentu (przycisk, pole tekstowe, checkbox, lista
rozwijana, konsola) oraz pełnym, działającym przykładem end-to-end.

## Rozwiązywanie problemów

- **Okno się nie otwiera / widać tylko "404 page not found":** frontend
  edytora jest osadzony w binarce przez `go:embed` (nie zależy od katalogu
  uruchomienia) — jeśli mimo to widzisz ten błąd, sprawdź czy używasz
  aktualnego `main.go` z dyrektywą `//go:embed Frontend`.
- **Kompilacja się nie udaje / brak okna mimo zbudowania:** `webview_go` i
  `sqweek/dialog` wymagają cgo — sprawdź czy masz kompilator C (gcc/MinGW-w64
  na Windows) w PATH, oraz czy WebView2 Runtime jest zainstalowany (Windows).
- **Przyciski/pola tekstowe wyglądają "płasko", tracą ramkę:** to by się
  zdarzyło tylko po RĘCZNYM ustawieniu grubości obramowania na przycisku/polu
  tekstowym (jawne ustawienie border-* na natywnie stylowanej kontrolce każe
  przeglądarce porzucić systemowy wygląd) — domyślnie te dwa typy nie dostają
  wymuszonego obramowania.
- `removeZoneIdentifier()` na starcie czyści Mark-of-the-Web z własnego .exe
  na Windows, żeby SmartScreen nie blokował programu po pobraniu/skopiowaniu.
