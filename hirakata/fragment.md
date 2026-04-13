# Notatka z fragmentów `BarPanel` i `OnBoarding`

## Cel
Ta notatka opisuje:
- jakie fragmenty występują w `BarPanel`
- za co odpowiadają
- jak działa przełączanie między nimi
- jak działa onboarding i jakie ma etapy

## Architektura ogólna

W projekcie są dwa główne obszary pracy z fragmentami:

### 1. `BarPanel`
- to główna nawigacja w aplikacji po wejściu do części właściwej
- fragmenty są hostowane głównie przez `FragmentHostActivity`
- dolny pasek nawigacyjny obsługuje klasa `BarPanel`

### 2. `OnBoarding`
- to sekwencja ekranów startowych dla nowego użytkownika
- fragmenty są hostowane przez `OnboardingActivity`
- każdy krok onboardingowy to osobny `Fragment`

---

## `BarPanel` - najważniejsze klasy

### `FragmentHostActivity`
Rola:
- kontener dla fragmentów z głównej części aplikacji
- decyduje, który fragment ma być pokazany na starcie

Stałe fragmentów:
- `FRAGMENT_ACCOUNT`
- `FRAGMENT_DAILY`
- `FRAGMENT_HIRAGANA`
- `FRAGMENT_KATAKANA`
- `FRAGMENT_NAVIGATION`
- `FRAGMENT_SETTINGS`
- `FRAGMENT_FISZKI`

Ważne rzeczy:
- odczytuje `start_fragment` z `Intent`
- dla Hiragany i Katakany tworzy `PoczatekHiraganaKatakanaFragment` z argumentem `tryb`
- dla fiszek uruchamia `FlashcardFragment`

### `BarPanel`
Rola:
- steruje dolnym paskiem nawigacji
- obsługuje kliknięcia przycisków
- zmienia aktywny stan ikon
- uruchamia fragmenty lub ekrany błędów

Najważniejsze zachowania:
- `login`:
  - bez internetu -> `ErrorNoWifi`
  - z internetem -> `AccountFragment`
- `daily`:
  - bez internetu -> `ErrorNoWifi`
  - bez konta -> `ErrorNoAccount`
  - z kontem -> `DailyWordGeneratorFragment`
- `hiragana` / `katakana`:
  - pokazuje `PoczatekHiraganaKatakanaFragment`
  - albo przełącza tryb w już otwartym fragmencie
- `settings`:
  - otwiera `SettingFragment`
- `fiszki`:
  - bez internetu -> `ErrorNoWifi`
  - bez premium -> `EmptyPremium`
  - z premium -> `FlashcardFragment`

---

## Fragmenty w `BarPanel`

### `AccountFragment`
Rola:
- ekran konta
- zarządza trzema sekcjami:
  - rejestracja
  - logowanie
  - profil

Jak działa:
- tworzy obiekty:
  - `Register`
  - `Login`
  - `Profil`
- na podstawie argumentu `pokazSekcje` pokazuje odpowiedni widok
- jeśli użytkownik jest zalogowany, domyślnie pokazuje profil
- jeśli nie, pokazuje logowanie

Sekcje:
- `register`
- `login`
- `profile`

### `NavigationRouterFragment`
Rola:
- główny ekran wyboru rodzaju nauki
- wybór:
  - Hiragana / Katakana
  - znaki / słowa / zdania

Jak działa:
- trzyma aktualną kategorię: `hiragana` albo `katakana`
- przy kliknięciu przechodzi:
  - do `PremiumOptionsChoices`, jeśli użytkownik ma premium i internet
  - do `BaseCharacter`, `BaseWord`, `BaseSentence`, jeśli nie ma premium albo internetu

Dodatkowo:
- korzysta z `PremiumStatusTracker`
- zmienia opisy kafelków zależnie od premium i internetu

### `PoczatekHiraganaKatakanaFragment`
Rola:
- ekran przeglądania znaków Hiragany i Katakany
- pokazuje znaki w siatce

Obsługiwane tryby:
- `HIRAGANA`
- `KATAKANA`

Obsługiwane kategorie:
- `PODSTAWOWE`
- `DAKUTEN`
- `KOMBINOWANE`

Jak działa:
- korzysta z `CharacterManager`
- ładuje zestaw znaków według systemu i kategorii
- renderuje karty dynamicznie w `GridLayout`
- pozwala przełączać alfabet i kategorię bez zmiany aktywności

Dodatkowo:
- nasłuchuje stanu logowania przez `FirebaseAuth.AuthStateListener`
- uruchamia `PremiumStatusTracker`, gdy użytkownik jest zalogowany i online

### `DailyWordGeneratorFragment`
Rola:
- codzienne słowo do przepisania / odgadnięcia

Jak działa:
- pobiera z Firestore dokument użytkownika
- sprawdza:
  - `lastDailyDate`
  - `czyWykonane`
  - `iloscDni`
- jeśli daily już zrobione, przechodzi do `DoneDaily`
- jeśli nie, ładuje słowo dnia z kolekcji `words`
- porównuje wpisaną odpowiedź z `romaji`

Po poprawnej odpowiedzi:
- ustawia `czyWykonane = true`
- zwiększa `iloscDni`
- przechodzi do `DoneDaily`

### `FlashcardFragment`
Rola:
- fiszki do nauki słówek

Tryby działania:
- dla jednego alfabetu
- dla obu alfabetów naraz

Tworzenie fragmentu:
- `forBarPanel()` -> oba alfabety
- `forAlphabet(boolean isKatakana)` -> jeden alfabet

Jak działa:
- ładuje słówka z `WordRepository`
- pokazuje przód karty: `kana`
- pokazuje tłumaczenie i `romaji` po odsłonięciu
- obsługuje swipe:
  - w prawo = znam
  - w lewo = nie znam
- na końcu pokazuje podsumowanie
- pozwala powtarzać tylko nieznane

### `SettingFragment`
Rola:
- ekran ustawień

Co obsługuje:
- przejście do logowania
- przejście do rejestracji
- przejście do profilu
- wylogowanie
- zmiana języka
- przycisk kupna premium
- polityka prywatności
- ustawienia wyglądu
- feedback
- sterowanie muzyką

Dodatkowo:
- używa `NetworkChangeReceiver`
- pokazuje i ukrywa elementy zależnie od:
  - internetu
  - zalogowania

---

## Klasy pomocnicze wewnątrz `AccountFragment`

To nie są fragmenty, ale są kluczowe dla działania sekcji konta.

### `Login`
Rola:
- logowanie emailem albo nickiem

Jak działa:
- jeśli wpis zawiera `@`, loguje po emailu
- jeśli nie, szuka emaila po nicku w kolekcji `nicks`
- po zalogowaniu synchronizuje język aplikacji z Firestore

### `Register`
Rola:
- rejestracja nowego użytkownika

Jak działa:
- waliduje nick, email i hasło
- sprawdza unikalność nicku i emaila
- tworzy konto w Firebase Auth
- zapisuje dokument użytkownika w Firestore
- tworzy mapowanie nick -> email w kolekcji `nicks`

### `Profil`
Rola:
- ekran danych użytkownika

Jak działa:
- słucha zmian dokumentu użytkownika przez `addSnapshotListener`
- pokazuje:
  - nick
  - email
  - premium
  - streak
  - tryb developera

Dodatkowe akcje:
- wylogowanie
- zmiana nicku
- usunięcie konta
- wejście do `DeveloperNavigation`

---

## Przepływ fragmentów w `BarPanel`

Najczęstszy przepływ:

1. `MainActivity`
2. `FragmentHostActivity`
3. `NavigationRouterFragment`
4. dalej jeden z fragmentów:
   - `AccountFragment`
   - `DailyWordGeneratorFragment`
   - `PoczatekHiraganaKatakanaFragment`
   - `SettingFragment`
   - `FlashcardFragment`

Sterowanie tym przepływem:
- `BarPanel`
- `FragmentHostActivity`

---

## `OnBoarding` - architektura

### `OnboardingActivity`
Rola:
- host wszystkich fragmentów onboardingowych
- steruje kolejnością kroków
- wykrywa brak internetu
- zapisuje stan ukończenia onboardingu

Najważniejsze pola logiczne:
- `userLoggedIn`
- `premiumBought`
- `loginSkipped`
- `premiumSkipped`

Najważniejsze metody przejść:
- `goToGreetings()`
- `goToLanguageSelection()`
- `goToLogin()`
- `goToPremiumOffer()`
- `goToWhyPremium()`
- `goToAppIntro()`
- `goToMainActivity()`

Ważne zachowanie:
- jeśli `onboarding_ukonczone = true`, aktywność od razu przechodzi do `MainActivity`

---

## Fragmenty w `OnBoarding`

### `OnboardingFragment1Greetings`
Rola:
- pierwszy ekran powitalny

Jak działa:
- wyświetla tekst literka po literce
- po czasie automatycznie przechodzi dalej
- kliknięcie:
  - kończy animację pisania
  - albo od razu przechodzi dalej

Następny krok:
- `OnboardingFragment2Language`

### `OnboardingFragment2Language`
Rola:
- wybór języka aplikacji

Jak działa:
- pokazuje `Spinner` z językami
- zapisuje wybrany kod przez `OnboardingActivity.zapiszJezyk(...)`
- odświeża teksty interfejsu

Po kliknięciu Gotowe:
- jeśli jest internet -> przechodzi do logowania
- jeśli nie ma internetu -> `OnboardingFragmentNoWifi`

### `OnboardingFragment3Account`
Rola:
- logowanie podczas onboardingu

Jak działa:
- najpierw pokazuje wstęp tekstowy
- potem odsłania panel logowania
- pozwala:
  - zalogować się
  - przejść do rejestracji
  - pominąć logowanie
  - odzyskać hasło

Po poprawnym logowaniu:
- pobiera dokument użytkownika z Firestore
- sprawdza `czyMaPremium`
- jeśli premium jest aktywne -> przechodzi do `OnboardingFragment6Presentation`
- jeśli nie -> przechodzi do `OnboardingFragment4Shopping`

Po pominięciu:
- ustawia `loginSkipped = true`
- przechodzi do prezentacji aplikacji

### `OnboardingFragment3AccountRegister`
Rola:
- rejestracja podczas onboardingu

Jak działa:
- waliduje:
  - nick
  - email
  - siłę hasła
  - zgodność hasła i potwierdzenia
- tworzy konto w Firebase Auth
- zapisuje dokument użytkownika w Firestore

Po udanej rejestracji:
- ustawia `userLoggedIn = true`
- przechodzi do `OnboardingFragment4Shopping`

### `OnboardingFragment4Shopping`
Rola:
- ekran oferty premium

Co robi:
- `Kup premium` -> na razie przejście do `ErrorEmpty`
- `Dlaczego premium` -> `OnboardingFragment5Premium`
- `Pomiń` -> ustawia `premiumSkipped = true` i przechodzi dalej

### `OnboardingFragment5Premium`
Rola:
- slajdy wyjaśniające zalety premium

Jak działa:
- ma 5 slajdów
- każdy slajd ma:
  - tytuł
  - opis
  - numer
- działa automatyczne przechodzenie co kilka sekund
- użytkownik może:
  - iść dalej
  - pominąć
  - wrócić

Na końcu:
- przechodzi do `OnboardingFragment6Presentation`

### `OnboardingFragment6Presentation`
Rola:
- końcowy ekran prezentujący aplikację

Jak działa:
- pokazuje opis aplikacji
- po kliknięciu przycisku przechodzi do `MainActivity`
- przy Android 13+ prosi o pozwolenie na powiadomienia

Wykorzystuje:
- `registerForActivityResult`
- `NotificationScheduler`

### `OnboardingFragmentNoWifi`
Rola:
- ekran braku internetu podczas onboardingu

Jak działa:
- pokazuje komunikat z animacją pisania
- nasłuchuje powrotu internetu przez `NetworkChangeReceiver`
- gdy internet wraca:
  - zmienia treść komunikatu
  - zmienia akcję przycisku
  - automatycznie wraca do flow onboardingu

Najczęściej wraca do:
- `goToLogin()`

---

## Kolejność ekranów w onboardingu

Standardowy przepływ:

1. `OnboardingFragment1Greetings`
2. `OnboardingFragment2Language`
3. `OnboardingFragment3Account`
4. `OnboardingFragment4Shopping`
5. `OnboardingFragment5Premium`
6. `OnboardingFragment6Presentation`
7. `MainActivity`

Możliwe odchylenia:
- brak internetu -> `OnboardingFragmentNoWifi`
- pominięcie logowania -> od razu `OnboardingFragment6Presentation`
- zalogowany użytkownik z premium -> z logowania bezpośrednio do `OnboardingFragment6Presentation`

---

## Zależności logiczne

### `BarPanel` zależy od:
- `FragmentHostActivity`
- `PremiumStatusTracker`
- `FirebaseAuth`
- `NetworkUtils`

### `OnBoarding` zależy od:
- `OnboardingActivity`
- `NetworkChangeReceiver`
- `FirebaseAuth`
- `FirebaseFirestore`
- `NotificationScheduler`

---

## Szybkie podsumowanie

### Najważniejsze fragmenty `BarPanel`
- `AccountFragment`
- `NavigationRouterFragment`
- `PoczatekHiraganaKatakanaFragment`
- `DailyWordGeneratorFragment`
- `FlashcardFragment`
- `SettingFragment`

### Najważniejsze fragmenty `OnBoarding`
- `OnboardingFragment1Greetings`
- `OnboardingFragment2Language`
- `OnboardingFragment3Account`
- `OnboardingFragment3AccountRegister`
- `OnboardingFragment4Shopping`
- `OnboardingFragment5Premium`
- `OnboardingFragment6Presentation`
- `OnboardingFragmentNoWifi`

### Główna różnica
- `BarPanel` to nawigacja we właściwej aplikacji
- `OnBoarding` to sekwencja wprowadzająca użytkownika przed wejściem do aplikacji
