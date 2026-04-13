# Tworzenie elementów wyglądowych w Javie zamiast XML

## Wprowadzenie

W Androidzie interfejs można tworzyć na dwa główne sposoby:

### 1. Przez XML
- definiujemy układ w plikach `res/layout/*.xml`
- potem w Javie pobieramy elementy przez `findViewById(...)`

### 2. Programowo w Javie
- tworzymy widoki przez `new TextView(...)`, `new Button(...)`, `new LinearLayout(...)`, `new CardView(...)`
- ustawiamy im parametry ręcznie
- dodajemy je do kontenera przez `addView(...)`

W tym projekcie oba podejścia są używane razem:
- szkielet ekranu zwykle jest w XML
- dynamiczne elementy są budowane w Javie

To oznacza, że XML daje bazowy układ, a Java generuje te części interfejsu, które:
- zmieniają się zależnie od danych
- powtarzają się wiele razy
- są wygodniejsze do zbudowania dynamicznie

---

## Po co tworzyć UI w Javie zamiast XML

Tworzenie UI programowo jest przydatne, gdy:

### 1. Liczba elementów nie jest stała
Przykład:
- lista kart opcji premium
- siatka znaków
- dynamiczny changelog aktualizacji

### 2. Elementy zależą od danych runtime
Przykład:
- ile znaków ma zostać pokazanych
- jakie opcje premium mają być wyświetlone
- czy dodać podpis, ikonę albo badge

### 3. Chcesz łatwo generować wiele podobnych widoków
Przykład:
- wiele `CardView` o podobnym stylu
- wiele `TextView` w pętli

### 4. Chcesz modyfikować układ bardzo dynamicznie
Przykład:
- dodawanie i usuwanie widoków
- animowanie nowo dodanych kart
- zmiana struktury widoku zależnie od trybu

---

## Typowy schemat tworzenia UI programowo

Najczęstszy schemat wygląda tak:

1. pobranie kontenera z XML
2. usunięcie starych elementów, jeśli trzeba
3. utworzenie nowych widoków przez `new`
4. ustawienie stylu i rozmiaru
5. ustawienie `LayoutParams`
6. dodanie widoków do rodzica przez `addView(...)`
7. ustawienie listenerów i animacji

Przykład logiczny:

```java
LinearLayout container = findViewById(R.id.container);
container.removeAllViews();

TextView tv = new TextView(this);
tv.setText("Przykład");
tv.setTextSize(16);
tv.setTextColor(getColor(R.color.tekst_glowny));

LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(
        LinearLayout.LayoutParams.MATCH_PARENT,
        LinearLayout.LayoutParams.WRAP_CONTENT
);
tv.setLayoutParams(lp);

container.addView(tv);
```

---

## Najważniejsze klasy używane przy budowie UI w Javie

W tym projekcie najczęściej pojawiają się:

### Podstawowe widoki
- `TextView`
- `Button`
- `ImageView`
- `View`
- `EditText`

### Kontenery
- `LinearLayout`
- `GridLayout`
- `ScrollView`

### Widoki material / kartowe
- `CardView`

### Parametry układu
- `LinearLayout.LayoutParams`
- `GridLayout.LayoutParams`

---

## Najczęściej używane metody

Przy budowaniu widoków w Javie bardzo często używa się:

### Tworzenie
- `new TextView(context)`
- `new Button(context)`
- `new LinearLayout(context)`
- `new CardView(context)`
- `new View(context)`

### Styl
- `setText(...)`
- `setTextSize(...)`
- `setTextColor(...)`
- `setBackgroundColor(...)`
- `setPadding(...)`
- `setGravity(...)`
- `setAlpha(...)`

### Układ
- `setLayoutParams(...)`
- `setOrientation(...)`
- `setColumnCount(...)`
- `setMargins(...)`

### Hierarchia
- `addView(...)`
- `removeAllViews()`

### Interakcja
- `setOnClickListener(...)`
- `setOnLongClickListener(...)`

### Animacje
- `animate().alpha(...)`
- `animate().translationX(...)`
- `animate().translationY(...)`
- `setStartDelay(...)`

---

## Bardzo ważna rzecz: `LayoutParams`

Samo utworzenie widoku nie wystarcza. Widok musi jeszcze wiedzieć:
- jaką ma szerokość
- jaką ma wysokość
- jaki ma margines
- jak ma się zachowywać w kontenerze

Dlatego prawie zawsze trzeba ustawić `LayoutParams`.

### Przykład dla `LinearLayout`

```java
LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(
        LinearLayout.LayoutParams.MATCH_PARENT,
        LinearLayout.LayoutParams.WRAP_CONTENT
);
lp.bottomMargin = 12;
view.setLayoutParams(lp);
```

### Przykład dla `GridLayout`

```java
GridLayout.LayoutParams lp = new GridLayout.LayoutParams();
lp.width = 0;
lp.height = dp(140);
lp.columnSpec = GridLayout.spec(GridLayout.UNDEFINED, 1, 1f);
lp.setMargins(8, 8, 8, 8);
view.setLayoutParams(lp);
```

To jest bardzo ważne, bo:
- `LinearLayout` używa własnych `LayoutParams`
- `GridLayout` używa własnych `LayoutParams`

Nie powinno się ich mieszać.

---

## Znaczenie `dp` przy tworzeniu UI

Gdy budujesz widoki programowo, często musisz podawać rozmiary ręcznie.

Nie powinno się wpisywać “surowych” pikseli bez przeliczenia, bo na różnych ekranach wszystko będzie miało inny rozmiar.

Dlatego w projekcie często występuje pomocnicza metoda:

```java
private int dp(int dp) {
    return Math.round(dp * getResources().getDisplayMetrics().density);
}
```

albo:

```java
private int dpToPx(int dp) {
    return Math.round(dp * requireContext().getResources().getDisplayMetrics().density);
}
```

To przelicza `dp` na piksele.

Bez tego:
- interfejs byłby niespójny
- na niektórych urządzeniach elementy byłyby za małe albo za duże

---

## Tworzenie dynamicznych kart - przykład z projektu

Jednym z najlepszych przykładów w projekcie jest:

### `PremiumOptionsChoices`

Ta klasa buduje karty opcji premium w całości programowo.

### Jak to działa

1. aktywność pobiera listę `Option`
2. dla każdej opcji wywołuje `buildCard(opt)`
3. karta jest tworzona od zera:
   - `CardView`
   - wewnętrzny `LinearLayout`
   - pasek akcentowy `View`
   - blok tekstu `LinearLayout`
   - `TextView` na tytuł
   - `TextView` na opis
   - `ImageView` na strzałkę
4. wszystko jest składane ręcznie przez `addView(...)`
5. karta dostaje kliknięcie i animację

### Dlaczego to jest dobry przykład

Bo pokazuje cały proces:
- tworzenie głównego widoku
- budowanie zagnieżdżonej hierarchii
- ustawianie `LayoutParams`
- stylowanie kolorami
- dodawanie animacji
- podpinanie akcji

### Uproszczona struktura tej karty

```text
CardView
└── LinearLayout (row)
    ├── View (accent)
    ├── LinearLayout (textBlock)
    │   ├── TextView (label)
    │   └── TextView (desc)
    └── ImageView (chevron)
```

To jest dokładnie to, co zwykle robi się w XML, tylko tutaj zostało zrobione ręcznie w Javie.

---

## Tworzenie siatki dynamicznych elementów - przykład z projektu

Drugim bardzo dobrym przykładem jest:

### `PoczatekHiraganaKatakanaFragment`

Tutaj dynamicznie tworzona jest siatka kart ze znakami.

### Jak działa

1. pobierana jest lista znaków z `CharacterManager`
2. kontener `GridLayout` jest czyszczony:
   - `container.removeAllViews()`
3. dla każdego znaku tworzona jest nowa karta:
   - `CardView`
   - wewnętrzny `LinearLayout`
   - `TextView` z kana
   - opcjonalny `TextView` z romaji
4. ustawiane są `GridLayout.LayoutParams`
5. karta jest animowana i dodawana do siatki

### Co to pokazuje

Ten przykład uczy:
- jak budować listę lub siatkę widoków w pętli
- jak działa `GridLayout`
- jak ustawić szerokość `0` i wagę przez `columnSpec`
- jak dodać marginesy między kartami

### Ważna rzecz

Ta metoda dobrze pokazuje, że UI programowe bardzo często oznacza:
- generowanie wielu podobnych kart
- ręczne zarządzanie kolejnością i opóźnieniem animacji

---

## Tworzenie bardziej złożonych struktur - przykład z projektu

### `HiraganaSelectionActivity`

To jest jeden z najbardziej rozbudowanych przykładów budowania UI programowo.

Ta aktywność dynamicznie tworzy:
- przyciski wierszy
- przyciski kolumn
- karty znaków
- dodatkowe oznaczenia liczby powtórzeń
- układ siatki

### Co jest tu ważne

#### 1. Tworzenie przycisków wierszy
Metoda:
- `stworzPrzyciskWiersza(int row)`

Tworzy:
- `Button`
- ustawia tło
- ustawia rozmiar
- podpina kliknięcie

#### 2. Tworzenie przycisków kolumn
Metoda:
- `stworzPrzyciskiKolumn()`

Tworzy:
- pusty `View` jako spacer
- kilka `Button` w pętli
- każdemu nadaje `LinearLayout.LayoutParams`

#### 3. Tworzenie kart znaków
Metoda:
- `buildCard(int idx, Character ch)`

Tworzy:
- `CardView`
- wewnętrzny `LinearLayout`
- `TextView` z kana
- `TextView` z romaji
- `TextView` z liczbą powtórzeń

#### 4. Aktualizacja istniejącego widoku
Metoda:
- `aktualizujKarte(int position)`

To ważne, bo pokazuje, że programowe UI to nie tylko tworzenie, ale też:
- modyfikowanie już istniejącego widoku
- zmiana koloru tła
- zmiana koloru tekstu
- usuwanie i ponowne dodawanie dziecka

To jest trudniejsze niż XML, ale daje pełną kontrolę.

---

## Budowanie elementów pomocniczych w dialogach

Programowe tworzenie UI nie musi dotyczyć całych ekranów.
Może też dotyczyć małych fragmentów interfejsu, np. w oknach dialogowych.

### Przykład: `HiraganaSelectionActivity` i `KatakanaSelectionActivity`

W metodzie ustawiania liczby powtórzeń:
- tworzony jest `EditText`
- tworzony jest `LinearLayout`
- `EditText` jest dodawany do kontenera
- kontener jest przekazywany do `AlertDialog.Builder.setView(...)`

To jest bardzo praktyczne zastosowanie, bo:
- nie trzeba robić osobnego pliku XML
- można szybko zbudować prosty custom view dla dialogu

### Przykład logiczny

```java
EditText input = new EditText(this);
LinearLayout container = new LinearLayout(this);
container.setOrientation(LinearLayout.VERTICAL);
container.setPadding(dp(24), dp(12), dp(24), dp(12));
container.addView(input);
```

---

## Tworzenie treści list / changelogów programowo

### `VersionTracker`

To kolejny dobry przykład budowania UI w kodzie.

W dialogu aktualizacji:
- tworzony jest `LinearLayout row`
- tworzony jest `TextView bullet`
- tworzony jest `TextView tvItem`
- oba elementy są dodawane do `row`
- `row` jest dodawany do kontenera changelogu

### Dlaczego to jest przydatne

Bo liczba pozycji changelogu nie jest znana z góry.
Zależy od danych z Firestore.

XML nie nadaje się tu tak wygodnie, bo:
- musiałbyś wcześniej zdefiniować sztywną liczbę widoków
- albo użyć osobnego adaptera / recyclera

Java programowa daje prosty sposób:
- odczytaj listę
- wygeneruj po jednym wierszu na element

---

## Tworzenie UI w Javie a stylowanie

Gdy tworzysz widok programowo, sam odpowiadasz za styl:
- kolory
- marginesy
- padding
- rozmiar tekstu
- promień zaokrąglenia
- elevation

W projekcie często robi się to przez pobieranie kolorów z zasobów:

```java
colorPrimary = ContextCompat.getColor(this, R.color.przycisk_glowny);
colorSurface = ContextCompat.getColor(this, R.color.tlo_powierzchnia);
colorTextPrimary = ContextCompat.getColor(this, R.color.tekst_glowny);
```

To jest bardzo dobra praktyka, bo:
- nie wpisujesz kolorów “na sztywno”
- UI pozostaje spójne z resztą aplikacji
- łatwiej zmienić motyw później

---

## Animacje przy dynamicznie tworzonych widokach

Duża zaleta programowego UI jest taka, że od razu po utworzeniu widoku możesz:
- nadać mu stan początkowy
- uruchomić animację wejścia

W projekcie widać to np. w:
- `PremiumOptionsChoices`
- `PoczatekHiraganaKatakanaFragment`
- `HiraganaSelectionActivity`
- `StrokePracticeActivity`

### Typowy wzorzec

```java
card.setAlpha(0f);
card.setTranslationY(40f);
card.animate().alpha(1f).translationY(0f).setDuration(250).start();
```

To daje efekt, że nowy widok:
- pojawia się płynnie
- wygląda bardziej nowocześnie

---

## Zalety tworzenia UI programowo

### 1. Duża elastyczność
Możesz dynamicznie tworzyć tyle elementów, ile chcesz.

### 2. Łatwe generowanie z danych
Pętla po liście danych od razu może tworzyć widoki.

### 3. Łatwe warunki
Możesz łatwo powiedzieć:
- jeśli jest romaji -> dodaj `TextView`
- jeśli nie ma -> nie dodawaj

### 4. Pełna kontrola nad kolejnością i strukturą
Sam decydujesz:
- co jest rodzicem
- co jest dzieckiem
- w jakiej kolejności wszystko dodać

### 5. Dobre do małych dynamicznych komponentów
Świetnie sprawdza się przy:
- kartach
- badge’ach
- changelogach
- prostych dialogach
- siatkach dynamicznych elementów

---

## Wady tworzenia UI programowo

### 1. Kod robi się długi
To, co w XML zajmuje kilka linii, w Javie bywa dużo dłuższe.

### 2. Czytelność jest gorsza
W XML łatwiej od razu zobaczyć strukturę widoku.
W Javie trzeba ją “czytać” krok po kroku.

### 3. Łatwiej o błędy
Np.:
- złe `LayoutParams`
- brak marginesu
- zły `Context`
- brak `addView(...)`
- pomieszanie `dp` i px

### 4. Trudniej utrzymać duże ekrany
Jeśli cały ekran jest budowany tylko w Javie, kod może stać się ciężki do utrzymania.

### 5. Trudniej współpracować z projektantem
UI w XML jest bardziej “wizualne”.
Kod w Javie jest bardziej techniczny.

---

## Kiedy warto używać Java zamiast XML

Warto użyć UI programowego, gdy:
- elementów jest dużo i są podobne
- ich liczba jest dynamiczna
- chcesz generować UI na podstawie listy danych
- budujesz prostą zawartość dialogu
- potrzebujesz szybkiego prototypu dynamicznej sekcji

Warto zostać przy XML, gdy:
- ekran jest statyczny
- układ jest rozbudowany, ale nie dynamiczny
- zależy Ci na czytelności i prostocie

Najlepsze podejście w praktyce:
- XML dla szkieletu
- Java dla elementów dynamicznych

I właśnie tak zrobiono w tym projekcie.

---

## Dobre praktyki przy tworzeniu UI w Javie

### 1. Wyciągaj kolory do zmiennych
Zamiast wołać `getColor(...)` wiele razy, lepiej zrobić:

```java
colorPrimary = ContextCompat.getColor(this, R.color.przycisk_glowny);
```

### 2. Rób pomocniczą metodę `dp(...)`
To bardzo ułatwia ustawianie rozmiarów.

### 3. Dziel kod na małe metody
Zamiast budować wszystko w `onCreate()`, lepiej tworzyć:
- `buildCard(...)`
- `buildRepView(...)`
- `createRowButton(...)`
- `buildOptionCards(...)`

### 4. Oddziel logikę danych od logiki widoku
Najpierw przygotuj dane, potem buduj UI.

### 5. Czyść kontener przed ponownym renderowaniem

```java
container.removeAllViews();
```

Bez tego elementy będą się dublować.

### 6. Uważaj na `Context`
W aktywności najczęściej używa się:
- `this`

W fragmencie:
- `requireContext()`
- `requireActivity()`

### 7. Przy aktualizacji istniejących widoków trzymaj referencje
W projekcie przykład:
- `Map<Integer, CardView> kartyMapa`

To pozwala aktualizować pojedynczy widok bez rysowania wszystkiego od nowa.

---

## Typowe błędy

### 1. Brak `LayoutParams`
Widok może się nie wyświetlić poprawnie albo mieć zły rozmiar.

### 2. Zły typ `LayoutParams`
Np. użycie `LinearLayout.LayoutParams` w `GridLayout`.

### 3. Brak `addView(...)`
Widok istnieje w pamięci, ale nie jest dodany do rodzica.

### 4. Używanie pikseli zamiast `dp`
Na różnych ekranach UI będzie wyglądać źle.

### 5. Zbyt duża ilość logiki w jednej metodzie
Kod robi się trudny do czytania i naprawy.

### 6. Nieczyszczenie kontenera
Po ponownym wyrenderowaniu elementy będą się powielały.

### 7. Modyfikowanie widoku po zniszczeniu fragmentu
Dlatego w projekcie często są zabezpieczenia:
- `if (!isAdded()) return;`

---

## Co dokładnie warto zapamiętać z tego projektu

### `PremiumOptionsChoices`
Zapamiętaj jako przykład:
- tworzenia estetycznych kart programowo

### `PoczatekHiraganaKatakanaFragment`
Zapamiętaj jako przykład:
- dynamicznej siatki `GridLayout`

### `HiraganaSelectionActivity` i `KatakanaSelectionActivity`
Zapamiętaj jako przykład:
- rozbudowanego dynamicznego UI z kartami, przyciskami wierszy i kolumn

### `VersionTracker`
Zapamiętaj jako przykład:
- budowania prostych list danych w dialogu

### dialog z `EditText`
Zapamiętaj jako przykład:
- tworzenia małego custom view bez osobnego XML

---

## Krótkie porównanie: XML vs Java

### XML
Lepszy do:
- statycznych ekranów
- przejrzystej struktury
- łatwiejszego utrzymania

### Java
Lepsza do:
- dynamicznych list i siatek
- generowania elementów z danych
- niestandardowego budowania kart i sekcji

### Najlepsza praktyka
Łączyć oba podejścia:
- XML jako baza
- Java do dynamicznych elementów

---

## Podsumowanie końcowe

Tworzenie elementów wyglądowych w Javie zamiast XML polega na tym, że:
- widoki tworzysz ręcznie przez `new`
- nadajesz im styl i rozmiar metodami
- ustawiasz `LayoutParams`
- budujesz hierarchię przez `addView(...)`
- często robisz to w pętli, na podstawie danych

W tym projekcie takie podejście jest używane głównie do:
- kart opcji
- siatek znaków
- przycisków dynamicznych
- prostych kontenerów dialogowych
- list changelogu

Najważniejsza zaleta:
- pełna elastyczność

Największa wada:
- dłuższy i trudniejszy kod

Najrozsądniejsze podejście:
- nie zastępować XML wszędzie
- tylko używać Javy tam, gdzie interfejs naprawdę musi być dynamiczny
