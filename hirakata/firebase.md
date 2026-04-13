# Notatka z Firebase

## Cel
Ta notatka zbiera użycie Firebase w projekcie:
- jakie usługi Firebase są podłączone
- gdzie są używane
- jakie kolekcje Firestore występują
- jakie pola są zapisywane i odczytywane

## Podłączone usługi Firebase

Na podstawie `app/build.gradle.kts` projekt używa:
- `firebase-auth`
- `firebase-firestore`
- `firebase-analytics`

Dodatkowo w pluginach:
- `com.google.gms.google-services`

## Główne obszary użycia Firebase

### 1. `FirebaseAuth`
Używany do:
- rejestracji użytkownika
- logowania użytkownika
- wylogowania
- pobierania aktualnego użytkownika
- usuwania konta
- ponownej autoryzacji przed usunięciem konta
- sprawdzania dostępnych metod logowania dla emaila

### 2. `FirebaseFirestore`
Używany do:
- przechowywania danych użytkownika
- mapowania nick -> email
- zapisywania statusu premium
- przechowywania streaka i daily
- przechowywania słowa dnia
- przechowywania feedbacku
- trzymania informacji o wymaganej wersji aplikacji

---

## Najważniejsze klasy korzystające z Firebase

## Autoryzacja i konto

### `BarPanel/Login`
Rola:
- logowanie użytkownika

Jak działa:
- jeśli użytkownik wpisze email:
  - `FirebaseAuth.signInWithEmailAndPassword(...)`
- jeśli wpisze nick:
  - najpierw odczyt z Firestore z kolekcji `nicks`
  - pobranie emaila
  - dopiero potem logowanie przez Auth

Po zalogowaniu:
- pobiera dokument z `users/{uid}`
- odczytuje pole `jezyk`
- synchronizuje język aplikacji z lokalnymi ustawieniami

### `BarPanel/Register`
Rola:
- rejestracja użytkownika w głównej części aplikacji

Jak działa:
- sprawdza unikalność nicku:
  - `db.collection("nicks").document(nickKey).get()`
- sprawdza unikalność emaila:
  - `mAuth.fetchSignInMethodsForEmail(email)`
- tworzy konto:
  - `createUserWithEmailAndPassword(...)`
- zapisuje dane do `users/{uid}`
- tworzy wpis w `nicks/{nickKey}`

### `OnboardingFragment3Account`
Rola:
- logowanie w trakcie onboardingu

Jak działa:
- loguje przez Auth
- potem pobiera `users/{uid}`
- sprawdza `czyMaPremium`
- od tego zależy dalszy przepływ onboardingu

### `OnboardingFragment3AccountRegister`
Rola:
- rejestracja w trakcie onboardingu

Jak działa:
- tworzy konto w Auth
- zapisuje dokument użytkownika do Firestore

Uwaga:
- tutaj nie ma osobnego wpisu w kolekcji `nicks`
- w odróżnieniu od `BarPanel/Register`

### `BarPanel/Profil`
Rola:
- panel profilu użytkownika

Jak działa:
- pobiera referencję do `users/{uid}`
- nasłuchuje zmian przez `addSnapshotListener(...)`
- pokazuje dane użytkownika na żywo

Obsługiwane operacje:
- odczyt nicku
- odczyt emaila
- odczyt premium
- odczyt streaka
- odczyt trybu developera
- zmiana nicku
- usunięcie konta

Przy zmianie nicku:
- aktualizuje `users/{uid}.nick`
- tworzy nowy wpis w `nicks/{newNick}`
- usuwa stary wpis w `nicks/{oldNick}`

Przy usuwaniu konta:
- kasuje dokument `users/{uid}`
- kasuje wpis z `nicks`
- wywołuje `currentUser.delete()`

### `Dialog/ReauthDialogFragment`
Rola:
- ponowna autoryzacja użytkownika

Jak działa:
- używa `EmailAuthProvider`
- po udanej reautoryzacji pozwala wykonać operację wysokiego ryzyka, np. usunięcie konta

### `Dialog/RecoverPasswordDialogFragment`
Rola:
- odzyskiwanie hasła

Jak działa:
- korzysta z `FirebaseAuth`
- dodatkowo sprawdza użytkownika / email w Firestore

---

## Ustawienia użytkownika i dane profilu

### `LanguageSelectionActivity`
Rola:
- zapis języka aplikacji

Jak działa:
- zapisuje język lokalnie do `SharedPreferences`
- jeśli użytkownik jest zalogowany:
  - aktualizuje `users/{uid}` przez `SetOptions.merge()`
  - zapisuje pole `jezyk`

### `AccountFragment`
Rola:
- pośrednio sprawdza stan logowania

Jak działa:
- używa `FirebaseAuth.getInstance().getCurrentUser()`
- wybiera sekcję `profile` albo `login`

### `BarPanel/SettingFragment`
Rola:
- ustawienia konta

Jak działa:
- używa `FirebaseAuth` do:
  - sprawdzenia czy użytkownik jest zalogowany
  - wylogowania

### `BarPanel/BarPanel`
Rola:
- dolna nawigacja

Jak działa:
- sprawdza `FirebaseAuth.getCurrentUser()`
- od tego zależy np. dostęp do daily i stan ikony profilu

---

## Premium

### `Trackery/PremiumStatusTracker`
Rola:
- singleton śledzący status premium zalogowanego użytkownika

Jak działa:
- bierze aktualnego użytkownika z `FirebaseAuth`
- nasłuchuje dokumentu `users/{uid}`
- czyta pole:
  - `czyMaPremium`

Mechanizmy:
- `startListening()`
- `stopListening()`
- `aktualizujStatusDlaUzytkownika()`

Kto korzysta:
- `BarPanel`
- `NavigationRouterFragment`
- `PoczatekHiraganaKatakanaFragment`
- `FeedbackSheetDialogFragment`

### Pola premium
Najważniejsze pole:
- `czyMaPremium`

Występuje w:
- `users/{uid}`

---

## Daily i streak

### `BarPanel/DailyWordGeneratorFragment`
Rola:
- codzienna nauka jednego słowa

Jak działa:
- odczytuje `users/{uid}`
- czyta pola:
  - `lastDailyDate`
  - `czyWykonane`
  - `iloscDni`
- jeśli trzeba, resetuje stan dnia
- pobiera słowo z:
  - `words/{yyyy-MM-dd}`

Po poprawnej odpowiedzi:
- ustawia `czyWykonane = true`
- zwiększa `iloscDni`

### `Error/DoneDaily`
Rola:
- ekran po ukończeniu daily

Jak działa:
- pobiera `users/{uid}`
- odczytuje `iloscDni`
- pokazuje aktualny streak

### `Daily/StreakStatusTracker`
Rola:
- singleton nasłuchujący zmian streaka użytkownika

Jak działa:
- nasłuchuje `users/{uid}`
- czyta pole:
  - `iloscDni`

### `Daily/DailyResetWorker`
Rola:
- reset dzienny dla użytkowników i słowa dnia

Jak działa:
- zapisuje `words/{today}`
- usuwa stare dokumenty z `words`
- pobiera wszystkich użytkowników przez `collectionGroup("users")`
- aktualizuje u każdego:
  - `czyWykonane`
  - `iloscDni`

Uwaga:
- worker zwiększa streak, jeśli poprzednie daily było wykonane

### `Daily/DailyWordResetWorker`
Rola:
- losowanie i zapis nowego słowa dnia

Jak działa:
- sprawdza, czy `words/{today}` już istnieje
- jeśli nie:
  - wybiera słowo z JSON
  - zapisuje je w `words/{today}`
- zapisuje metadane w:
  - `words_meta/stats`

Pola meta:
- `usedIndices`
- `lastResetDate`

---

## Feedback

### `Dialog/FeedbackSheetDialogFragment`
Rola:
- wysyłanie opinii użytkownika

Jak działa:
- wymaga zalogowanego użytkownika
- pobiera:
  - `uid`
  - `email`
  - status premium
  - model telefonu
- zapisuje dokument do kolekcji `feedback`

Zapisywane pola:
- `uid`
- `email`
- `maPremium`
- `kategoria`
- `kategoriaWyswietlana`
- `tresc`
- `telefon`
- `dataWyslania`
- `przeczytane`

---

## Wersjonowanie aplikacji

### `Trackery/VersionTracker`
Rola:
- wymuszanie aktualizacji aplikacji

Jak działa:
- nasłuchuje dokumentu:
  - `versions/current`
- jeśli dokument nie istnieje:
  - tworzy go z wartościami domyślnymi
- potem słucha zmian przez `addSnapshotListener(...)`

Pola w `versions/current`:
- `version`
- `bypass_key`
- `changelog`

Jeśli wersja z Firestore różni się od lokalnej:
- pokazuje dialog aktualizacji
- pozwala przejść do sklepu
- opcjonalnie pozwala wpisać `bypass_key`

---

## Developer i funkcje administracyjne

### `ANawigacja/DeveloperNavigation`
Rola:
- panel developerski

Jak działa:
- używa `FirebaseAuth`
- używa `FirebaseFirestore`
- potrafi:
  - pobierać dane użytkownika
  - wyświetlać dane Firestore
  - wykonywać narzędzia developerskie

### `ANawigacja/DeveloperStatusTracker`
Rola:
- tracker stanu developera

Jak działa:
- nasłuchuje dokumentu użytkownika
- odczytuje status z Firestore

Najważniejsze pole:
- `developerTryb`

---

## Inne miejsca użycia Firebase

### `MainActivity`
Użycie:
- sprawdzenie stanu zalogowania przez `FirebaseAuth`

### `PoczatekHiraganaKatakanaFragment`
Użycie:
- `FirebaseAuth.AuthStateListener`
- uruchamianie lub zatrzymywanie `PremiumStatusTracker`

### `HiraganaDialog`
Użycie:
- odczyt `uid`
- zapis / odczyt danych dialogów w Firestore

---

## Kolekcje Firestore w projekcie

### `users`
Dokument:
- `users/{uid}`

Najczęstsze pola:
- `nick`
- `email`
- `czyMaPremium`
- `jezyk`
- `developerTryb`
- `iloscDni`
- `lastDailyDate`
- `czyWykonane`
- `dataUtworzenia`

### `nicks`
Dokument:
- `nicks/{nickLower}`

Pola:
- `email`

Cel:
- logowanie po nicku
- walidacja unikalności nicku

### `words`
Dokument:
- `words/{yyyy-MM-dd}`

Pola:
- `japanese`
- `romaji`
- `data`

Cel:
- słowo dnia

### `words_meta`
Dokument:
- `words_meta/stats`

Pola:
- `usedIndices`
- `lastResetDate`

Cel:
- kontrola, które słowa dnia były już użyte

### `feedback`
Dokument:
- generowany automatycznie przez `add(...)`

Pola:
- `uid`
- `email`
- `maPremium`
- `kategoria`
- `kategoriaWyswietlana`
- `tresc`
- `telefon`
- `dataWyslania`
- `przeczytane`

### `versions`
Dokument:
- `versions/current`

Pola:
- `version`
- `bypass_key`
- `changelog`

---

## Najważniejsze operacje Firestore

### Odczyt dokumentu
- `.collection(...).document(...).get()`

### Zapis dokumentu
- `.set(...)`

### Zapis częściowy
- `.set(..., SetOptions.merge())`

### Aktualizacja pola
- `.update(...)`

### Nasłuchiwanie zmian
- `.addSnapshotListener(...)`

### Dodanie dokumentu z automatycznym ID
- `.add(...)`

### Zapytanie po polu
- `.whereEqualTo(...)`

### Zapytanie grupowe
- `.collectionGroup("users")`

---

## Najważniejsze operacje Firebase Auth

### Rejestracja
- `createUserWithEmailAndPassword(...)`

### Logowanie
- `signInWithEmailAndPassword(...)`

### Wylogowanie
- `signOut()`

### Sprawdzenie aktualnego użytkownika
- `getCurrentUser()`

### Sprawdzenie, czy email już istnieje
- `fetchSignInMethodsForEmail(...)`

### Usunięcie konta
- `currentUser.delete()`

### Reautoryzacja
- przez `EmailAuthProvider`

---

## Zależności logiczne między danymi

### Konto użytkownika
Auth:
- trzyma tożsamość użytkownika

Firestore:
- trzyma profil użytkownika i ustawienia

### Premium
Źródło prawdy:
- `users/{uid}.czyMaPremium`

### Streak
Źródło prawdy:
- `users/{uid}.iloscDni`

### Daily
Źródło prawdy:
- `users/{uid}.czyWykonane`
- `users/{uid}.lastDailyDate`
- `words/{today}`

### Logowanie po nicku
Źródło prawdy:
- `nicks/{nickLower}.email`

---

## Rzeczy warte zauważenia

### Niespójność rejestracji
- `BarPanel/Register` zapisuje też wpis do `nicks`
- `OnboardingFragment3AccountRegister` zapisuje tylko `users`

Skutek:
- konto założone w onboardingu może nie działać z logowaniem po nicku, jeśli kolekcja `nicks` nie zostanie utworzona osobno

### Premium i streak są oparte o snapshot listenery
- dzięki temu UI może reagować automatycznie na zmiany w Firestore

### Firestore jest używany jako baza stanu aplikacji
- nie tylko jako backup użytkownika
- ale też do daily, wersji, feedbacku i statusów

---

## Szybkie podsumowanie

Firebase w projekcie odpowiada głównie za:
- autoryzację użytkownika
- profil i ustawienia użytkownika
- status premium
- streak i daily word
- feedback
- kontrolę wersji aplikacji

Najważniejsze kolekcje:
- `users`
- `nicks`
- `words`
- `words_meta`
- `feedback`
- `versions`
