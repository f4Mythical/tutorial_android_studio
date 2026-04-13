# Trackery – Notatka

## Przegląd

Projekt zawiera 3 trackery w pakiecie `com.example.hiraganaandkatakana.Trackery`:

| Klasa | Odpowiedzialność |
|---|---|
| `NetworkChangeReceiver` | Nasłuchuje zmian połączenia internetowego |
| `PremiumStatusTracker` | Śledzi status premium użytkownika w Firestore |
| `VersionTracker` | Sprawdza aktualność wersji aplikacji i wymusza update |

---

## 1. NetworkChangeReceiver

### Co robi
Rozszerza `BroadcastReceiver`. Reaguje na systemowy broadcast zmiany sieci i informuje słuchacza czy urządzenie ma dostęp do internetu.

### Schemat działania
```
System broadcast (CONNECTIVITY_CHANGE)
        ↓
onReceive(context, intent)
        ↓
NetworkUtils.isInternetAvailable(context)
        ↓
listener.onNetworkChanged(isOnline)
```

### Interfejs
```java
public interface NetworkChangeListener {
    void onNetworkChanged(boolean isOnline);
}
```

### Użycie
```java
NetworkChangeReceiver receiver = new NetworkChangeReceiver(isOnline -> {
    // reaguj na zmianę połączenia
});
// zarejestruj w onResume, wyrejestruj w onPause
```

### Uwagi
- Wymaga rejestracji w `AndroidManifest.xml` lub dynamicznie przez `registerReceiver()`
- Logika sprawdzania faktycznego dostępu do internetu ukryta w `NetworkUtils` (osobna klasa pomocnicza)

---

## 2. PremiumStatusTracker

### Co robi
Singleton nasłuchujący zmian dokumentu użytkownika w Firestore (pole `czyMaPremium`). Informuje resztę aplikacji o zmianie statusu premium w czasie rzeczywistym.

### Architektura
- **Singleton** – jedna instancja na całą aplikację (`getInstance()`)
- **Firestore snapshot listener** – reaguje natychmiast gdy zmieni się dokument w chmurze
- **Handler (main thread)** – callback zawsze wywołany na wątku UI

### Cykl życia

```
getInstance()
    ↓
startListening()  ← wywołaj przy logowaniu / starcie aplikacji
    ↓
[zmiana w Firestore: users/{uid}.czyMaPremium]
    ↓
zaktualizujStatusPremium(bool)
    ↓
statusListener.onPremiumStatusChanged(bool)  ← na main thread
    ↓
stopListening()   ← wywołaj przy wylogowaniu
```

### Metody publiczne

| Metoda | Opis |
|---|---|
| `getInstance()` | Zwraca singleton |
| `startListening()` | Startuje listener Firestore dla zalogowanego użytkownika |
| `stopListening()` | Usuwa listener, resetuje status do `false` |
| `getCzyMaPremium()` | Zwraca aktualny status (bez zapytania do sieci) |
| `setOnPremiumStatusChangedListener(listener)` | Rejestruje callback na zmiany |
| `aktualizujStatusDlaUzytkownika()` | Jednorazowy fetch (bez stałego nasłuchiwania) |

### Interfejs
```java
public interface OnPremiumStatusChangedListener {
    void onPremiumStatusChanged(boolean czyMaPremium);
}
```

### Przykładowe użycie
```java
PremiumStatusTracker tracker = PremiumStatusTracker.getInstance();
tracker.setOnPremiumStatusChangedListener(hasPremium -> {
    // pokaż/ukryj funkcje premium
});
tracker.startListening();

// przy wylogowaniu:
tracker.stopListening();
```

### Ważne szczegóły
- `startListening()` jest idempotentne – drugie wywołanie nic nie robi (`nasluchiwanieAktywne`)
- Zmiana statusu notyfikuje listener **tylko jeśli wartość faktycznie się zmieniła** (guard `czyMaPremium != nowyStatus`)
- Firestore: kolekcja `users`, dokument `{uid}`, pole `czyMaPremium` (Boolean)

---

## 3. VersionTracker

### Co robi
Porównuje wersję zainstalowanej aplikacji (`versionName` z `PackageInfo`) z wymaganą wersją zapisaną w Firestore. Jeśli się różnią – pokazuje niepomijalny dialog wymuszający aktualizację.

### Architektura
- **Brak singletona** – tworzony jako zwykły obiekt, zazwyczaj w `MainActivity`
- **Firestore snapshot listener** – reaguje na każdą zmianę dokumentu `versions/current`
- **Dialog niepomijalny** – `setCancelable(false)`, blokuje UI do czasu aktualizacji lub wpisania klucza bypass

### Struktura dokumentu Firestore (`versions/current`)

| Pole | Typ | Opis |
|---|---|---|
| `version` | String | Wymagana wersja (np. `"1.2.0"`) |
| `bypass_key` | String | Klucz developerski omijający update (opcjonalny) |
| `changelog` | List\<String\> | Lista zmian wyświetlana w dialogu |

### Cykl życia

```
start(activity)
    ↓
pobierz dokument versions/current
    ↓ (jeśli nie istnieje → utwórz z domyślnymi wartościami)
startListener()
    ↓
[zmiana w Firestore]
    ↓
currentVersion == requiredVersion?
   ├── TAK → dismissDialog()
   └── NIE → showUpdateDialog()
              ├── przycisk "Aktualizuj" → Play Store
              └── bypass panel (jeśli bypass_key ustawiony)
                  └── wpisz klucz → dismiss()
    ↓
stop()  ← wywołaj w onDestroy
```

### Metody publiczne

| Metoda | Opis |
|---|---|
| `start(activity)` | Startuje tracker, podpina listener Firestore |
| `stop()` | Usuwa listener, zamyka aktywny dialog |

### Dialog aktualizacji
- Wyświetla wymaganą wersję i listę zmian (changelog)
- Przycisk **"Aktualizuj"** otwiera Play Store (`Intent.ACTION_VIEW`)
- **Bypass panel** – ukryte pole do wpisania klucza developerskiego; widoczne po kliknięciu `bypassTrigger`; jeśli `bypass_key` jest pusty w Firestore – panel w ogóle nie jest klikalny

### Zabezpieczenia
- Sprawdza `activity.isFinishing() || activity.isDestroyed()` przed wyświetleniem dialogu
- Pomija cache Firestore przy pierwszym połączeniu (`MetadataChanges.INCLUDE` + guard `isFromCache && !exists`)
- Dialog nie jest tworzony ponownie jeśli już jest wyświetlony (`activeDialog.isShowing()`)

### Przykładowe użycie
```java
// w MainActivity
private VersionTracker versionTracker = new VersionTracker();

@Override
protected void onStart() {
    super.onStart();
    versionTracker.start(this);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    versionTracker.stop();
}
```

---

## Podsumowanie porównawcze

| Cecha | NetworkChangeReceiver | PremiumStatusTracker | VersionTracker |
|---|---|---|---|
| Wzorzec | BroadcastReceiver | Singleton | Zwykły obiekt |
| Źródło danych | System Android | Firestore (realtime) | Firestore (realtime) |
| Callback | interfejs `NetworkChangeListener` | interfejs `OnPremiumStatusChangedListener` | Dialog UI |
| Cykl życia | rejestracja/wyrejestrowanie ręcznie | `startListening()` / `stopListening()` | `start()` / `stop()` |
| Efekt dla użytkownika | brak bezpośredniego UI | odblokowuje/blokuje funkcje | blokujący dialog update |
