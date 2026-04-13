# Gemini API + JSON – Notatka (projekt: todo)

## Przegląd architektury

```
Użytkownik (głos)
    ↓
RecognizerIntent (Android STT)
    ↓
FragmentMow / FragmentMultitask
    ↓
GeminiPlanner.process() / processMultitask()
    ↓  HTTP POST (JSON)
Gemini API (gemini-2.5-flash)
    ↓  odpowiedź JSON
parsowanie → PlanResult
    ↓
FirestoreHelper.addPlan()
    ↓
Firestore (kolekcja "plans")
```

---

## 1. GeminiPlanner – klasa centralna

### Stałe

```java
private static final String API_URL =
    "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=";
```

- Klucz API pobierany z `BuildConfig.GEMINI_API_KEY` (nie hardkodowany w źródle)
- Model: `gemini-2.5-flash` (szybki, tani)
- `thinkingBudget: 0` – wyłączone "myślenie", odpowiedź natychmiastowa

### Wątkowość

```java
private static final ExecutorService executor = Executors.newSingleThreadExecutor();
private static final Handler mainHandler = new Handler(Looper.getMainLooper());
```

- Zapytanie HTTP wykonywane na osobnym wątku (`executor`)
- Callback zawsze zwracany na wątek UI (`mainHandler.post(...)`)

---

## 2. Dwa tryby pracy

### `process()` – jedno zadanie

- Wejście: dowolny tekst po polsku
- Oczekiwana odpowiedź Gemini: **obiekt JSON `{...}`**
- Callback: `GeminiPlanner.Callback`

### `processMultitask()` – wiele zadań naraz

- Wejście: tekst zawierający kilka zadań (rozdzielonych przecinkami/frazami)
- Oczekiwana odpowiedź Gemini: **tablica JSON `[...]`**
- Callback: `GeminiPlanner.MultiCallback`
- Użycie limitowane do **5 razy** (licznik w Firestore: `users/{uid}.multitask_uses`)

---

## 3. Struktura JSON – format odpowiedzi

### Pojedyncze zadanie (`process`)

```json
{
  "task": "Lekarz",
  "description": "ul. Główna",
  "date_offset_days": 1,
  "reminders": ["15:00"],
  "priority": "high"
}
```

### Wiele zadań (`processMultitask`)

```json
[
  {
    "task": "Lekarz",
    "description": "",
    "date_offset_days": 1,
    "reminders": ["15:00"],
    "priority": "medium"
  },
  {
    "task": "Spotkanie z Piotrem",
    "description": "",
    "date_offset_days": 3,
    "reminders": [],
    "priority": "medium"
  }
]
```

### Pola JSON – opis

| Pole | Typ | Opis |
|---|---|---|
| `task` | String | Nazwa czynności, maks 60 znaków |
| `description` | String | Miejsce, osoba, szczegóły (może być pusty) |
| `date_offset_days` | int | Liczba dni od dziś (0 = dziś, 1 = jutro) |
| `reminders` | Array\<String\> | Godziny przypomnień w formacie `HH:mm` |
| `priority` | String | `"low"` / `"medium"` / `"high"` |

### Przypadki brzegowe

| Sytuacja | Odpowiedź Gemini |
|---|---|
| Tekst nie zawiera zadania (np. "hej jak się masz") | `{}` |
| Brak zadań w trybie multitask | `[]` |

---

## 4. Budowanie promptu

Prompt jest budowany dynamicznie – zawiera aktualny kontekst czasowy:

```java
"Dzisiaj jest " + today + " (" + dayOfWeek + ").\n"
"Aktualna godzina: " + currentTime + "\n"
"Jutro: " + tomorrow + "\n"
"Za tydzień: " + nextWeek + "\n"
```

Dzięki temu Gemini poprawnie oblicza `date_offset_days` dla poleceń typu:
- `"w piątek"` → oblicza ile dni do najbliższego piątku
- `"25 czerwca"` → oblicza offset od dzisiaj
- `"w sobotę"` (gdy dziś sobota) → zakłada przyszłą sobotę (offset 7)

### Zasady priority w prompcie

| Słowa kluczowe | Priority |
|---|---|
| "pilne", "ważne", "koniecznie" | `high` |
| "kiedyś", "może", "opcjonalnie" | `low` |
| wszystko inne | `medium` |

### Wymuszenie czystego JSON

Prompt explicite mówi Gemini:
> "Nie używaj bloków kodu \`\`\`json ... \`\`\`. Twoja odpowiedź musi zaczynać się od '{' i kończyć na '}'."

---

## 5. Wywołanie HTTP – `callGeminiApi()`

### Request body (JSON)

```json
{
  "contents": [
    {
      "parts": [
        { "text": "<prompt>" }
      ]
    }
  ],
  "generationConfig": {
    "thinkingConfig": {
      "thinkingBudget": 0
    }
  }
}
```

### Timeouty

```java
conn.setConnectTimeout(15000);  // 15s
conn.setReadTimeout(15000);     // 15s
```

### Wydobycie tekstu z odpowiedzi

Gemini zwraca zagnieżdżony obiekt – tekst jest głęboko w środku:

```java
response
    .getJSONArray("candidates")
    .getJSONObject(0)
    .getJSONObject("content")
    .getJSONArray("parts")
    .getJSONObject(0)
    .getString("text");
```

### Logowanie tokenów (debug)

```java
if (response.has("usageMetadata")) {
    // loguje: promptTokenCount, candidatesTokenCount, totalTokenCount
}
```

---

## 6. Model danych – `PlanResult`

```java
public static class PlanResult {
    public String       task;
    public String       description;
    public int          offsetDays;
    public List<String> reminders;
    public String       priority;
}
```

Używany jako DTO (Data Transfer Object) między `GeminiPlanner` a fragmentami.

---

## 7. Zapis do Firestore – `FirestoreHelper.addPlan()`

### Pola dokumentu w kolekcji `plans`

| Pole Firestore | Źródło | Typ |
|---|---|---|
| `title` | `task` z JSON | String |
| `content` | `description` z JSON | String |
| `dueTime` | dzisiaj + `offsetDays`, godzina 23:59 | Timestamp |
| `notificationTime` | pierwsza godzina z `reminders` | Timestamp |
| `notificationTimes` | lista wszystkich `reminders` | List\<Timestamp\> |
| `priority` | `priority` z JSON | String |
| `email` | z Firebase Auth | String |
| `nick` | z kolekcji `nicks/{uid}` | String |
| `createdAt` | `Timestamp.now()` | Timestamp |
| `isDone` | zawsze `false` przy dodaniu | Boolean |

### Konwersja `reminders` (String → Timestamp)

```java
// "15:00" → Timestamp na dzień due o godzinie 15:00
String[] p = t.split(":");
Calendar c = (Calendar) due.clone();
c.set(Calendar.HOUR_OF_DAY, Integer.parseInt(p[0].trim()));
c.set(Calendar.MINUTE,      Integer.parseInt(p[1].trim()));
ts.add(new Timestamp(c.getTime()));
```

### Po zapisie do Firestore

1. Planuje powiadomienia przez `NotificationScheduler.schedule()`
   - ID powiadomienia: `planId + "_" + triggerMs` (unikalny)
2. Odświeża widget: `WidgetUpdater.refresh()`

---

## 8. Przepływ w fragmentach

### FragmentMow (jedno zadanie)

```
btnMic.click
    → RecognizerIntent (pl-PL)
    → processCommand(rawText)
    → GeminiPlanner.process()
    → buildAndSavePlan()
    → FirestoreHelper.addPlan()
    → wyświetl status z task, description, reminders, priority
```

### FragmentMultitask (wiele zadań)

```
btnMic.click (jeśli currentUses < 5)
    → RecognizerIntent (pl-PL)
    → processCommand(rawText)
    → GeminiPlanner.processMultitask()
    → dla każdego PlanResult: buildAndSavePlan()
    → incrementUsage() → Firestore: users/{uid}.multitask_uses++
    → renderDots() – 5 kropek, wypełnione = pozostałe użycia
    → wyświetl "Dodano N planów: • task1 • task2 ..."
```

### Limit multitaska

- Max 5 użyć na konto (`MAX_USES = 5`)
- Stan persystowany w Firestore (`multitask_uses`)
- Wizualizacja: 5 kropek (`renderDots()`), aktywne = kolorowe, zużyte = szare
- Po wyczerpaniu: przycisk `alpha=0.4`, blokada kliknięcia

---

## 9. Obsługa błędów

| Błąd | Obsługa |
|---|---|
| HTTP ≠ 200 | `callGeminiApi` zwraca `null` → `cb.onError("Brak odpowiedzi z API")` |
| Gemini zwrócił `{}` lub `[]` | `onError("Nie rozpoznano zadania")` lub `onResult(emptyList)` |
| JSON parse exception | `catch(Exception)` → `cb.onError(e.getMessage())` |
| Błąd parsowania godziny | `Log.w()`, pomija tę przypominanie (nie crashuje) |
| Brak zalogowanego użytkownika | wczesny `return` bez zapisu |
| Fragment odłączony | guard `if (!isAdded()) return` przed każdym UI update |
