# Notatka z dźwięków i powiadomień

## 1. Dźwięki w aplikacji

## Muzyka w tle

### `PlaylistManager`
Plik:
- `app/src/main/java/com/example/hiraganaandkatakana/PlaylistManager.java`

Rola:
- zarządza muzyką w tle

Jak działa:
- używa `MediaPlayer`
- ma playlistę z plików:
  - `R.raw.background_song_1`
  - `R.raw.background_song_2`
  - `R.raw.background_song_3`
  - `R.raw.background_song_4`
  - `R.raw.background_song_5`
  - `R.raw.background_song_6`
  - `R.raw.background_song_7`

Funkcje:
- włączenie muzyki
- wyłączenie muzyki
- zmiana głośności
- przechodzenie do następnego utworu
- zatrzymywanie przy wygaszeniu ekranu
- zatrzymywanie przy przejściu aplikacji w tło
- wznowienie po powrocie do aplikacji

Zapisywane ustawienia:
- `glosnosc_muzyki`
- `muzyka_wlaczona`

## Sterowanie muzyką

### `SettingFragment`
Rola:
- pozwala włączyć lub wyłączyć muzykę
- pozwala otworzyć dialog zmiany głośności

### `MusicDialog`
Rola:
- okno do ustawienia głośności

Jak działa:
- używa `SeekBar`
- odczytuje głośność z `PlaylistManager`
- na bieżąco zmienia poziom głośności
- zmienia ikonę zależnie od poziomu

## TTS - syntezator mowy

### `ListeningModeActivity`
Rola:
- odczytuje japońskie zdania głosem

Jak działa:
- używa `TextToSpeech`
- próbuje uruchomić Google TTS
- ustawia język:
  - `Locale.JAPANESE`
- mówi tekst z pola `kana`

Dodatkowo:
- przy starcie TTS zatrzymuje muzykę przez `PlaylistManager.onTtsStart()`
- po zakończeniu TTS wznawia muzykę przez `PlaylistManager.onTtsEnd()`

Jeśli TTS nie działa:
- pokazuje komunikat
- proponuje instalację danych TTS
- może otworzyć sklep Play

---

## 2. Powiadomienia

## Główna klasa powiadomień

### `NotificationHelper`
Rola:
- tworzy kanał powiadomień
- buduje i pokazuje codzienne powiadomienie

### Kanał powiadomień
Metoda:
- `createNotificationChannel(Context context)`

Tworzy kanał:
- `daily_reminder_channel`

Ustawienia kanału:
- światło
- wibracje
- dźwięk systemowy powiadomienia

Do dźwięku używane jest:
- `RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION)`

## Wysyłanie powiadomienia

Metoda:
- `showDailyReminder(Context context)`

Jak działa:
- tworzy `Intent` do `MainActivity`
- opakowuje go w `PendingIntent`
- losuje wiadomość z listy `R.string.notification_daily_1 ... 50`
- buduje powiadomienie przez `NotificationCompat.Builder`
- ustawia:
  - tytuł
  - treść
  - `BigTextStyle`
  - `setAutoCancel(true)`
  - priorytet wysoki
  - kategorię reminder

## Harmonogram powiadomień

### `NotificationScheduler`
Rola:
- planuje codzienne przypomnienia

Jak działa:
- używa `WorkManager`
- tworzy:
  - `PeriodicWorkRequest` dla codziennych przypomnień
  - `OneTimeWorkRequest` dla powiadomienia natychmiastowego

Najważniejsze metody:
- `scheduleDailyReminder(Activity activity)`
- `scheduleDailyReminderAtHour(Activity activity, int hour)`
- `sendImmediateNotification(Activity activity)`
- `cancelDailyReminder(Activity activity)`

## Worker od powiadomień

### `ReminderWorker`
Rola:
- uruchamia właściwe pokazanie powiadomienia

Jak działa:
- wywołuje:
  - `NotificationHelper.showDailyReminder(...)`

## Pozwolenie na powiadomienia

### Android 13+
Projekt sprawdza:
- `POST_NOTIFICATIONS`

Obsługa jest w:
- `NotificationScheduler`
- `OnboardingFragment6Presentation`

Możliwe działania:
- sprawdzenie pozwolenia
- prośba o pozwolenie
- reakcja na wynik

## Gdzie powiadomienia są uruchamiane

### `MainActivity`
Po starcie:
- tworzy kanał powiadomień
- jeśli jest pozwolenie, planuje daily reminder

### `OnboardingFragment6Presentation`
Na końcu onboardingu:
- prosi użytkownika o zgodę na powiadomienia

### `DeveloperNavigation`
Rola:
- ekran testowy dla powiadomień

Pozwala:
- wysłać natychmiastowe powiadomienie
- anulować przypomnienie
- ustawić godzinę przypomnienia

---

## 3. Relacja między dźwiękiem i powiadomieniami

Te dwa obszary są osobne:

### Dźwięki
- `PlaylistManager`
- `MusicDialog`
- `ListeningModeActivity`

### Powiadomienia
- `NotificationHelper`
- `NotificationScheduler`
- `ReminderWorker`

Jedyny ważny punkt wspólny:
- TTS w `ListeningModeActivity` wpływa na muzykę w tle
- powiadomienia nie sterują muzyką z `PlaylistManager`

## Szybkie podsumowanie

### Dźwięki
- muzyka w tle działa przez `MediaPlayer`
- głos japoński działa przez `TextToSpeech`
- użytkownik może sterować głośnością i włączeniem muzyki

### Powiadomienia
- działają przez `NotificationChannel`
- są planowane przez `WorkManager`
- prowadzą po kliknięciu do `MainActivity`
- przypominają o daily learning
