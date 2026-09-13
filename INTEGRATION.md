# Integracja wyeksportowanego projektu z backendem Go
# SunRiver / Lothar TeaM 
# -----------------------------------------------------
# - Z pasji do programowania .....
# 13.09.20026   

Ten przewodnik pokazuje krok po kroku, jak podłączyć prawdziwą logikę Go do
interfejsu wyeksportowanego przez SunGo GUI Builder — z przykładami kodu dla
każdego typu komponentu.

## Spis treści

1. [Jak to jest zbudowane](#1-jak-to-jest-zbudowane)
2. [Cykl życia jednego zdarzenia](#2-cykl-życia-jednego-zdarzenia)
3. [Krok po kroku: od bindingu do działającego handlera](#3-krok-po-kroku-od-bindingu-do-działającego-handlera)
4. [Przykłady dla każdego typu komponentu](#4-przykłady-dla-każdego-typu-komponentu)
   - [Przycisk](#przycisk-button)
   - [Pole tekstowe](#pole-tekstowe-input)
   - [Checkbox](#checkbox)
   - [Lista rozwijana](#lista-rozwijana-select)
5. [Zwracanie danych z backendu do interfejsu](#5-zwracanie-danych-z-backendu-do-interfejsu)
6. [Konsola: wypisywanie logów z backendu](#6-konsola-wypisywanie-logów-z-backendu)
7. [Pełny przykład: mini panel sterowania](#7-pełny-przykład-mini-panel-sterowania)
8. [Zapisywanie stanu (plik JSON)](#8-zapisywanie-stanu-plik-json)
9. [FAQ / typowe problemy](#9-faq--typowe-problemy)

---

## 1. Jak to jest zbudowane

Eksport tworzy cztery pliki:

```
<Nazwa Projektu>/SRC/
├── backend.go
└── frontend/
    ├── index.html
    ├── style.css
    └── app.js
```

`backend.go` to zwykły serwer `net/http`:

```go
func main() {
    mux := http.NewServeMux()
    mux.Handle("/", http.FileServer(http.Dir("frontend")))
    mux.HandleFunc("/api/event", handleEvent)

    log.Println("Serwer wystartował: http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

- serwuje pliki z `frontend/` (stąd trzeba go uruchamiać z wnętrza `SRC/`)
- przyjmuje zdarzenia z interfejsu pod `/api/event` (POST, JSON)

`app.js` zawiera pomocniczą funkcję, którą wywołuje każdy komponent
z ustawionym **bindingiem**:

```js
async function sendToBackend(eventName, payload) {
    const res = await fetch('/api/event', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ event: eventName, data: payload || {} })
    });
    if (!res.ok) {
        console.error('Backend event failed:', eventName, res.status);
        return null;
    }
    return res.json();
}
```

Po stronie Go, `handleEvent` dekoduje żądanie i przełącza się po nazwie
zdarzenia:

```go
type eventRequest struct {
    Event string         `json:"event"`
    Data  map[string]any `json:"data"`
}

func handleEvent(w http.ResponseWriter, r *http.Request) {
    var req eventRequest
    json.NewDecoder(r.Body).Decode(&req)

    var resp any
    switch req.Event {
    // <-- tu generator wstawia po jednym case na każdy Binding z edytora
    default:
        http.Error(w, "nieznane zdarzenie: "+req.Event, http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}
```

## 2. Cykl życia jednego zdarzenia

Na przykładzie przycisku z bindingiem `save_click`:

```
1. Użytkownik klika przycisk #btnSave w przeglądarce
2. app.js: btnSave.addEventListener('click', () => sendToBackend('save_click', {}))
3. fetch POST /api/event  { "event": "save_click", "data": {} }
4. backend.go: handleEvent odbiera żądanie, trafia w case "save_click"
5. Twój kod w tym case robi to, co ma robić (zapis pliku, obliczenia, cokolwiek)
6. Backend zwraca JSON — np. { "ok": true }
7. app.js dostaje odpowiedź (obecnie nic z nią nie robi — patrz sekcja 5,
   jak to rozszerzyć, żeby zaktualizować interfejs na podstawie odpowiedzi)
```

## 3. Krok po kroku: od bindingu do działającego handlera

1. **W edytorze** zaznacz komponent → w panelu właściwości, sekcja "Backend" →
   pole "Zdarzenie (binding)" → wpisz nazwę, np. `save_click`.
2. **Wyeksportuj** projekt (przycisk "Eksportuj…").
3. Otwórz `SRC/backend.go` — generator już wstawił dla Ciebie:
   ```go
   case "save_click":
       // TODO: obsłuż zdarzenie "save_click"
       resp = map[string]any{"ok": true}
   ```
4. **Zastąp `// TODO`** swoją logiką. To wszystko — reszta (nasłuch w JS,
   fetch, routing) już działa.

## 4. Przykłady dla każdego typu komponentu

Każdy typ wysyła inny kształt `data` — generator sam dobiera właściwy nasłuch
w `app.js` w zależności od typu komponentu, do którego przypiąłeś binding.

### Przycisk (`button`)

`data` jest zawsze puste (`{}`) — sam fakt kliknięcia to informacja.

```go
case "save_click":
    if err := os.WriteFile("output.txt", []byte("zapisano\n"), 0o644); err != nil {
        resp = map[string]any{"ok": false, "error": err.Error()}
        break
    }
    resp = map[string]any{"ok": true}
```

### Pole tekstowe (`input`)

Wysyła zdarzenie **przy każdym naciśnięciu klawisza** (`input` event), z
aktualną wartością w `data.value` (string). Dobre do walidacji na żywo albo
trzymania wartości zsynchronizowanej z backendem.

```go
// Zmienna modułu (albo pole w Twojej strukturze stanu aplikacji) —
// przechowuje ostatnią wartość wpisaną w polu #userName.
var userName string

case "name_input":
    value, _ := req.Data["value"].(string)
    userName = value
    resp = map[string]any{"ok": true, "length": len(value)}
```

> Uwaga: `input` event leci przy KAŻDYM znaku — jeśli robisz coś kosztownego
> (zapis do pliku, zapytanie do bazy), rozważ dodanie debounce po stronie
> JS (patrz sekcja 5) zamiast wysyłać przy każdej literze.

### Checkbox

`data.checked` to `bool`.

```go
var darkModeEnabled bool

case "toggle_dark_mode":
    checked, _ := req.Data["checked"].(bool)
    darkModeEnabled = checked
    resp = map[string]any{"ok": true, "darkMode": darkModeEnabled}
```

### Lista rozwijana (`select`)

`data.value` to wybrany tekst opcji (string).

```go
case "lang_change":
    lang, _ := req.Data["value"].(string)
    switch lang {
    case "PL":
        // przełącz komunikaty backendu na polski
    case "EN":
        // przełącz na angielski
    }
    resp = map[string]any{"ok": true, "lang": lang}
```

## 5. Zwracanie danych z backendu do interfejsu

Domyślnie `app.js` **wysyła** zdarzenia, ale nic nie robi z odpowiedzią.
Żeby zaktualizować interfejs na podstawie tego, co zwróci backend, dopisz
własny nasłuch obok wygenerowanych (np. na końcu `app.js`, w tej samej
funkcji `DOMContentLoaded`):

```js
document.getElementById('btnSave').addEventListener('click', async () => {
    const result = await sendToBackend('save_click', {});
    if (result && result.ok) {
        document.getElementById('statusLabel').textContent = 'Zapisano!';
    } else {
        document.getElementById('statusLabel').textContent = 'Błąd zapisu.';
    }
});
```

To NIE koliduje z wygenerowanym nasłuchem — możesz mieć dwa
`addEventListener('click', ...)` na tym samym przycisku, oba się wykonają.
Jeśli wolisz mieć jedną, spójną wersję, po prostu przenieś logikę z
wygenerowanego nasłuchu do własnej funkcji.

**Debounce dla pola tekstowego** (patrz uwaga w sekcji 4):

```js
let debounceTimer;
document.getElementById('userName').addEventListener('input', (e) => {
    clearTimeout(debounceTimer);
    const value = e.target.value;
    debounceTimer = setTimeout(() => sendToBackend('name_input', { value }), 300);
});
```

## 6. Konsola: wypisywanie logów z backendu

Komponent Konsola (`<textarea readonly class="console">`) **nie ma** własnego
bindingu — to pole wyświetlające, nie formularz. Architektura tego szkieletu
to proste żądanie/odpowiedź (`fetch`), bez stałego połączenia (WebSocket),
więc backend nie może sam "wypchnąć" tekstu do przeglądarki w dowolnym
momencie — musisz albo (A) dopisać tekst z odpowiedzi na już istniejące
zdarzenie, albo (B) odpytywać backend co jakiś czas.

**(A) Najprościej — dopisz log do konsoli przy okazji innego zdarzenia:**

```go
case "save_click":
    // ... logika zapisu ...
    resp = map[string]any{"ok": true, "log": "Zapisano plik output.txt"}
```

```js
document.getElementById('btnSave').addEventListener('click', async () => {
    const result = await sendToBackend('save_click', {});
    const consoleBox = document.getElementById('log1'); // ID Twojej Konsoli
    if (result && result.log) {
        consoleBox.value += result.log + '\n';
        consoleBox.scrollTop = consoleBox.scrollHeight;
    }
});
```

**(B) Odpytywanie (polling) — dla logów, które backend generuje sam z siebie**
(np. w tle działa jakiś proces). Dopisz `"sync"` do importów `backend.go`:

```go
var logBuffer []string
var logMu sync.Mutex

func appendLog(line string) {
    logMu.Lock()
    defer logMu.Unlock()
    logBuffer = append(logBuffer, line)
}

// Nowy endpoint — dopisz obok mux.HandleFunc("/api/event", handleEvent):
mux.HandleFunc("/api/logs", func(w http.ResponseWriter, r *http.Request) {
    logMu.Lock()
    defer logMu.Unlock()
    json.NewEncoder(w).Encode(logBuffer)
})
```

```js
setInterval(async () => {
    const res = await fetch('/api/logs');
    const lines = await res.json();
    document.getElementById('log1').value = lines.join('\n');
}, 1000);
```

Dla prawdziwego strumieniowania w czasie rzeczywistym (bez opóźnienia
odpytywania) docelowo warto rozważyć WebSocket — to już wykracza poza
szkielet generowany automatycznie, ale powyższy serwer `net/http` da się
rozszerzyć o `gorilla/websocket` albo standardowe `net/http` + Server-Sent
Events bez zmiany reszty architektury.

## 7. Pełny przykład: mini panel sterowania

Załóżmy w edytorze: pole tekstowe `nameInput` (binding `name_input`),
checkbox `notifyCheckbox` (binding `toggle_notify`), przycisk `btnRun`
(binding `run_click`), i konsola `logBox` (bez bindingu).

**`backend.go`** (fragment, po dopisaniu logiki w miejscu `// TODO`):

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

var (
    lastName     string
    notifyOn     bool
)

type eventRequest struct {
    Event string         `json:"event"`
    Data  map[string]any `json:"data"`
}

func main() {
    mux := http.NewServeMux()
    mux.Handle("/", http.FileServer(http.Dir("frontend")))
    mux.HandleFunc("/api/event", handleEvent)

    log.Println("Serwer wystartował: http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}

func handleEvent(w http.ResponseWriter, r *http.Request) {
    var req eventRequest
    json.NewDecoder(r.Body).Decode(&req)

    var resp any
    switch req.Event {
    case "name_input":
        lastName, _ = req.Data["value"].(string)
        resp = map[string]any{"ok": true}

    case "toggle_notify":
        notifyOn, _ = req.Data["checked"].(bool)
        resp = map[string]any{"ok": true}

    case "run_click":
        msg := fmt.Sprintf("Uruchomiono dla: %s (powiadomienia: %v)", lastName, notifyOn)
        resp = map[string]any{"ok": true, "log": msg}

    default:
        http.Error(w, "nieznane zdarzenie: "+req.Event, http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}
```

**Dopisek na końcu `app.js`** (wypisywanie do konsoli po kliknięciu):

```js
document.getElementById('btnRun').addEventListener('click', async () => {
    const result = await sendToBackend('run_click', {});
    if (result && result.log) {
        const box = document.getElementById('logBox');
        box.value += result.log + '\n';
        box.scrollTop = box.scrollHeight;
    }
});
```

Efekt: wpisujesz imię, zaznaczasz/odznaczasz checkbox, klikasz "Uruchom" —
backend składa komunikat na podstawie ostatnio zapamiętanych wartości i
odsyła go z powrotem, a JS dopisuje go do konsoli.

## 8. Zapisywanie stanu (plik JSON)

Jeśli chcesz, żeby stan (jak `lastName`/`notifyOn` powyżej) przetrwał
restart programu, najprościej zapisać go do pliku obok `backend.go`:

```go
import (
    "encoding/json"
    "os"
)

type AppState struct {
    LastName string `json:"lastName"`
    NotifyOn bool   `json:"notifyOn"`
}

func saveState(s AppState) error {
    data, _ := json.MarshalIndent(s, "", "  ")
    return os.WriteFile("state.json", data, 0o644)
}

func loadState() AppState {
    var s AppState
    data, err := os.ReadFile("state.json")
    if err == nil {
        json.Unmarshal(data, &s)
    }
    return s
}
```

Wywołaj `loadState()` na starcie `main()`, a `saveState(...)` przy okazji
zdarzeń, które zmieniają stan (albo osobnym bindingiem "Zapisz ustawienia").

## 9. FAQ / typowe problemy

**"Nieznane zdarzenie" mimo że dodałem `case`.**
Sprawdź dokładnie nazwę — musi być identyczna z tą wpisaną w polu "Zdarzenie
(binding)" w edytorze (wielkość liter ma znaczenie).

**Checkbox/select nie wysyłają zdarzenia.**
Upewnij się, że komponent ma ustawiony binding w edytorze PRZED eksportem —
generator dodaje nasłuch w `app.js` tylko dla komponentów z niepustym
bindingiem.

**Chcę, żeby przycisk robił coś od razu przy starcie strony, nie tylko po
kliknięciu.**
Wywołaj `sendToBackend(...)` bezpośrednio w bloku
`document.addEventListener('DOMContentLoaded', () => { ... })` w `app.js`,
tak jak wygenerowane nasłuchy — po prostu bez opakowywania w
`addEventListener('click', ...)`.

**Port 8080 zajęty.**
Zmień `http.ListenAndServe(":8080", mux)` na inny port, np. `":8090"`.
