# Notatka z `Intent` w projekcie

## Cel
Ta notatka zbiera miejsca w projekcie, gdzie:
- tworzony i wysyłany jest `Intent`
- przekazywane są dane przez `putExtra`, `putStringArrayListExtra`, `putIntegerArrayListExtra`
- dane są odbierane przez `getIntent()` / `get...Extra()`
- używany jest `PendingIntent`
- aplikacja reaguje na systemowe akcje lub `BroadcastReceiver`

## Najważniejsze klucze `extras`

| Klucz | Typ | Gdzie ustawiany | Gdzie odczytywany | Znaczenie |
|---|---|---|---|---|
| `is_katakana` | `boolean` | `NavigationRouter`, `NavigationRouterFragment`, `PremiumOptionsChoices`, `PremiumWords` | `BaseCharacter`, `BaseSentence`, `BaseWord`, `StrokePracticeActivity`, `DrawingPracticeActivity`, `FragmentHostActivity`, `PremiumWords` | Czy pracujemy na katakanie |
| `screen` | `String` | `NavigationRouter`, `NavigationRouterFragment` | `PremiumOptionsChoices` | Jaki ekran premium ma się otworzyć |
| `start_fragment` | `String` | `BarPanel`, `PremiumOptionsChoices`, `WidgetProvider` | `FragmentHostActivity` | Który fragment ma zostać pokazany na starcie |
| `fiszki_both` | `boolean` | `BarPanel` | `FragmentHostActivity` | Czy fiszki mają działać dla obu alfabetów |
| `categories` | `ArrayList<String>` | `PremiumWords` | `BaseWord` | Wybrane kategorie słów |
| `alfabet` | `String` | `PremiumOptionsChoices` | `ListeningModeActivity`, `SearchActivity` | Nazwa alfabetu: `hiragana` albo `katakana` |
| `znaki` | `ArrayList<String>` | `HiraganaSelectionActivity`, `KatakanaSelectionActivity` | `WybraneZnakiPremium` | Wybrane znaki do ćwiczeń |
| `powtorzenia` | `ArrayList<Integer>` | `HiraganaSelectionActivity`, `KatakanaSelectionActivity` | `WybraneZnakiPremium` | Liczba powtórzeń dla znaku |
| `mapaZnakow` | `Serializable` (`HashMap<String, String>`) | `HiraganaSelectionActivity`, `KatakanaSelectionActivity` | `WybraneZnakiPremium` | Mapa `kana -> romaji` |
| `pokazSekcje` | `String` | `ErrorNoAccount` | `AccountFragment` przez `getArguments()` | Docelowa sekcja: np. `register` |

## Główne przepływy nawigacji

### 1. Nawigacja bazowa i premium
- `NavigationRouter.navigate(String type)`
  - premium:
    - `Intent(this, PremiumOptionsChoices.class)`
    - `putExtra("screen", selectedCategory + "_" + type)`
  - bez premium:
    - `Intent(this, BaseCharacter/BaseWord/BaseSentence.class)`
    - `putExtra("is_katakana", isKatakana)`

- `NavigationRouterFragment.navigate(String type)`
  - premium:
    - `Intent(requireContext(), PremiumOptionsChoices.class)`
    - `putExtra("screen", selectedCategory + "_" + type)`
    - `putExtra("is_katakana", isKatakana)`
  - bez premium:
    - `Intent(requireContext(), BaseCharacter/BaseWord/BaseSentence.class)`
    - `putExtra("is_katakana", isKatakana)`

### 2. Ekran wyboru opcji premium
- `PremiumOptionsChoices` buduje gotowe `Intent` dla kart opcji:
  - `flashcardIntent(boolean isKatakana)`
    - `Intent(this, FragmentHostActivity.class)`
    - `putExtra("start_fragment", "fiszki")`
    - `putExtra("is_katakana", isKatakana)`
  - `listeningIntent(String alfabet)`
    - `Intent(this, ListeningModeActivity.class)`
    - `putExtra("alfabet", alfabet)`
  - `searchIntent(String alfabet)`
    - `Intent(this, SearchActivity.class)`
    - `putExtra("alfabet", alfabet)`
  - `drawingIntent(boolean isKatakana)`
    - `Intent(this, DrawingPracticeActivity.class)`
    - `putExtra("is_katakana", isKatakana)`
  - dodatkowo bezpośrednio:
    - `BaseCharacter`
    - `StrokePracticeActivity`
    - `PremiumWords`
    - `BaseSentence`
    - `HiraganaSelectionActivity`
    - `KatakanaSelectionActivity`
    - `HiraganaDialog`
    - `ErrorEmpty`

### 3. Wybór kategorii słówek
- `PremiumWords`
  - odbiera `is_katakana`
  - po kliknięciu Start:
    - `Intent(this, BaseWord.class)`
    - `putExtra("is_katakana", isKatakana)`
    - `putStringArrayListExtra("categories", selectedCategories)`

### 4. Wybór znaków do nauki
- `HiraganaSelectionActivity.startNauka(...)`
  - `Intent(this, WybraneZnakiPremium.class)`
  - `putStringArrayListExtra("znaki", znaki)`
  - `putIntegerArrayListExtra("powtorzenia", powtorzenia)`
  - `putExtra("mapaZnakow", romajiMap)`

- `KatakanaSelectionActivity.startNauka(...)`
  - identyczny zestaw extras jak wyżej

### 5. Host fragmentów
- `BarPanel`
  - otwieranie fiszek dla obu alfabetów:
    - `Intent(activity, FragmentHostActivity.class)`
    - `putExtra("start_fragment", "fiszki")`
    - `putExtra("fiszki_both", true)`
  - otwieranie wybranego fragmentu:
    - `Intent(activity, FragmentHostActivity.class)`
    - `putExtra("start_fragment", fragmentName)`

- `WidgetProvider`
  - przycisk Hiragana:
    - `Intent(context, FragmentHostActivity.class)`
    - `putExtra("start_fragment", "hiragana")`
  - przycisk Katakana:
    - `Intent(context, FragmentHostActivity.class)`
    - `putExtra("start_fragment", "katakana")`

## Klasy odbierające `Intent`

### `FragmentHostActivity`
- `getStringExtra("start_fragment")`
- `getBooleanExtra("fiszki_both", false)`
- `getBooleanExtra("is_katakana", false)`

Co robi:
- wybiera startowy fragment
- dla `fiszki` decyduje, czy uruchomić fiszki dla obu alfabetów czy tylko jednego

### `PremiumOptionsChoices`
- `getStringExtra("screen")`

Co robi:
- na podstawie wartości `screen` buduje listę kart opcji premium

### `BaseCharacter`
- `getBooleanExtra("is_katakana", false)`

### `BaseSentence`
- `getBooleanExtra("is_katakana", false)`

### `BaseWord`
- `getBooleanExtra("is_katakana", false)`
- `getStringArrayListExtra("categories")`

### `PremiumWords`
- `getBooleanExtra("is_katakana", false)`

### `StrokePracticeActivity`
- `getBooleanExtra("is_katakana", false)`

### `DrawingPracticeActivity`
- `getBooleanExtra("is_katakana", false)`

### `ListeningModeActivity`
- `getStringExtra("alfabet")`
- fallback: jeśli brak, ustawia `hiragana`

### `SearchActivity`
- `getStringExtra("alfabet")`
- fallback: jeśli brak, ustawia `hiragana`

### `WybraneZnakiPremium`
- `getStringArrayListExtra("znaki")`
- `getIntegerArrayListExtra("powtorzenia")`
- `getSerializableExtra("mapaZnakow")`

Walidacja:
- jeśli któryś z pakietów danych jest `null` albo rozmiary list się nie zgadzają, aktywność kończy się przez `finish()`

## Miejsca wysyłające `Intent`

### Proste przejścia między ekranami
- `LanguageSelectionActivity`
  - `startActivity(new Intent(..., MainActivity.class))`
- `MainActivity`
  - `startActivity(new Intent(this, OnboardingActivity.class))`
- `OnboardingActivity`
  - `Intent(this, MainActivity.class)` + flagi `NEW_TASK | CLEAR_TASK`
- `DoneDaily`
  - `Intent(this, MainActivity.class)` + flagi `CLEAR_TOP | SINGLE_TOP`
- `DeveloperNavigation`
  - przejścia do wielu aktywności z mapy
  - restart onboarding przez `Intent(this, OnboardingActivity.class)` + flagi `NEW_TASK | CLEAR_TASK`

### Ekrany błędów / fallbacki
- `BarPanel`
  - `ErrorNoWifi`
  - `ErrorNoAccount`
  - `EmptyPremium`
- `Profil`
  - `DeveloperNavigation`
  - `ErrorEmpty`
- `SettingFragment`
  - `LanguageSelectionActivity`
  - `ErrorEmpty`
  - `Intent.ACTION_VIEW` do polityki prywatności
- `OnboardingFragment4Shopping`
  - `ErrorEmpty`
- `ErrorNoAccount`
  - `Intent(this, AccountFragment.class)`
  - `putExtra("pokazSekcje", "register")`
  - uwaga: `AccountFragment` jest klasą `Fragment`, nie `Activity`

### Ekrany premium i ćwiczenia
- `NavigationRouter`
- `NavigationRouterFragment`
- `PremiumOptionsChoices`
- `PremiumWords`
- `HiraganaSelectionActivity`
- `KatakanaSelectionActivity`
- `BarPanel`
- `WidgetProvider`

## `PendingIntent`

### `NotificationHelper.showDailyReminder(...)`
- tworzy `Intent(context, MainActivity.class)`
- ustawia flagi `FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK`
- opakowuje go w `PendingIntent.getActivity(...)`
- przekazuje do powiadomienia przez `setContentIntent(...)`

### `WidgetProvider.onUpdate(...)`
- tworzy dwa `PendingIntent.getActivity(...)`
  - dla przycisku Hiragana
  - dla przycisku Katakana

## `BroadcastReceiver` i akcje systemowe

### `PlaylistManager.ScreenStateReceiver`
- rejestracja dynamiczna przez `registerReceiver(...)`
- nasłuchiwane akcje:
  - `Intent.ACTION_SCREEN_OFF`
  - `Intent.ACTION_USER_PRESENT`
- odbiór przez `onReceive(Context, Intent)`

### `NetworkChangeReceiver`
- własny `BroadcastReceiver`
- metoda odbioru: `onReceive(Context, Intent)`
- w pokazanym kodzie nie ma tu własnych extras, tylko sprawdzenie stanu sieci

### `WidgetProvider`
- to też receiver, ale manifestowy `AppWidgetProvider`
- w `AndroidManifest.xml` ma `intent-filter`:
  - `android.appwidget.action.APPWIDGET_UPDATE`

## `registerForActivityResult`

### `OnboardingFragment6Presentation`
- używa:
  - `registerForActivityResult(new ActivityResultContracts.RequestPermission(), ...)`
- dotyczy to pozwolenia `POST_NOTIFICATIONS`
- to nie jest klasyczny przepływ `Intent` z `putExtra`, ale jest to nowoczesny mechanizm oparty o kontrakty aktywności

## `intent-filter` w `AndroidManifest.xml`

### `OnboardingActivity`
- posiada launcher:
  - `android.intent.action.MAIN`
  - `android.intent.category.LAUNCHER`

### `WidgetProvider`
- posiada:
  - `android.appwidget.action.APPWIDGET_UPDATE`

## Najczęściej używane metody związane z `Intent`

### Wysyłanie
- `new Intent(context, TargetActivity.class)`
- `new Intent(Intent.ACTION_VIEW, Uri.parse(...))`
- `new Intent(TextToSpeech.Engine.ACTION_INSTALL_TTS_DATA)`
- `startActivity(intent)`
- `intent.setFlags(...)`
- `PendingIntent.getActivity(...)`

### Przekazywanie danych
- `putExtra(String, boolean)`
- `putExtra(String, String)`
- `putExtra(String, Serializable)`
- `putStringArrayListExtra(String, ArrayList<String>)`
- `putIntegerArrayListExtra(String, ArrayList<Integer>)`

### Odbieranie danych
- `getIntent().getBooleanExtra(...)`
- `getIntent().getStringExtra(...)`
- `getIntent().getStringArrayListExtra(...)`
- `getIntent().getIntegerArrayListExtra(...)`
- `getIntent().getSerializableExtra(...)`

## Szybkie podsumowanie

Najważniejsze przepływy w projekcie opierają się na czterech kluczach:
- `is_katakana`
- `screen`
- `start_fragment`
- `alfabet`

Najbardziej rozbudowane przekazywanie danych występuje w:
- `PremiumWords -> BaseWord`
- `HiraganaSelectionActivity / KatakanaSelectionActivity -> WybraneZnakiPremium`
- `NavigationRouter / NavigationRouterFragment -> PremiumOptionsChoices`
- `BarPanel / WidgetProvider -> FragmentHostActivity`
