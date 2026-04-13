# Notatka o widgetach

## Główna klasa

### `WidgetProvider`
Plik:
- `app/src/main/java/com/example/hiraganaandkatakana/Widget/WidgetProvider.java`

Rola:
- obsługuje widget aplikacji na ekranie głównym Androida
- dziedziczy po `AppWidgetProvider`

## Jak działa widget

Metoda główna:
- `onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds)`

Co robi:
- dla każdego widgetu tworzy `RemoteViews`
- ładuje layout:
  - `R.layout.widget_layout`
- przypina akcje do dwóch przycisków:
  - Hiragana
  - Katakana

## Akcje przycisków

### Przycisk Hiragana
- tworzy `Intent` do `FragmentHostActivity`
- przekazuje:
  - `start_fragment = hiragana`
- ustawia flagi:
  - `FLAG_ACTIVITY_NEW_TASK`
  - `FLAG_ACTIVITY_CLEAR_TOP`
- tworzy `PendingIntent`
- przypina go do `R.id.przyciskHiragana`

### Przycisk Katakana
- tworzy `Intent` do `FragmentHostActivity`
- przekazuje:
  - `start_fragment = katakana`
- ustawia te same flagi
- tworzy `PendingIntent`
- przypina go do `R.id.przyciskKatakana`

## Konfiguracja widgetu

Plik:
- `app/src/main/res/xml/widget_info.xml`

Najważniejsze ustawienia:
- `minWidth="250dp"`
- `minHeight="50dp"`
- `maxWidth="250dp"`
- `maxHeight="50dp"`
- `updatePeriodMillis="86400000"`
- `initialLayout="@layout/widget_layout"`
- `resizeMode="none"`
- `widgetCategory="home_screen"`

## Rejestracja w manifeście

W `AndroidManifest.xml` jest receiver:
- `.Widget.WidgetProvider`

Ma `intent-filter`:
- `android.appwidget.action.APPWIDGET_UPDATE`

I `meta-data`:
- `android:resource="@xml/widget_info"`

## Po co jest widget

Widget daje szybki skrót do dwóch części aplikacji:
- Hiragana
- Katakana

Nie uruchamia bezpośrednio konkretnej nauki, tylko otwiera `FragmentHostActivity` z odpowiednim fragmentem startowym.

## Szybkie podsumowanie

Widget w tym projekcie:
- jest prosty
- ma dwa przyciski
- działa przez `PendingIntent`
- otwiera `FragmentHostActivity`
- przekazuje `start_fragment` jako informację, co pokazać po uruchomieniu
