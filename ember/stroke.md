# StrokePractice – Notatka (przeglądanie kresek)

## Co robi

`StrokePracticeActivity` to ekran **przeglądania** wszystkich znaków kana wraz z animacją kolejności kresek. Użytkownik nie rysuje – tylko ogląda jak znak jest rysowany kreska po kresce.

---

## Struktura ekranu (`activity_stroke_practice.xml`)

```
ConstraintLayout
 ├── TextView "書" (220sp, alpha #09) – dekoracyjny znak w tle, lewy dół
 ├── TextView "筆" (140sp, alpha #06) – dekoracyjny znak w tle, prawy góra
 ├── TextView tvTitle – tytuł aktywności
 ├── LinearLayout categoryLayout – 3 zakładki: Podstawowe / Dakuten / Kombinowane
 └── ScrollView → GridLayout gridKana (5 kolumn) – siatka kart kana
```

### Dekoracja tła
Dwa duże japońskie znaki (書 – "pisanie", 筆 – "pędzel") z bardzo niską alpha jako dekoracja. `clickable="false"`, `focusable="false"` – nie przechwytują dotyku.

---

## Dane – tablice znaków

Klasa zawiera 6 statycznych tablic `String[][]`, każda z parami `{kana, romaji}`:

| Tablica | System | Kategoria | Rozmiar |
|---|---|---|---|
| `H_PODSTAWOWE` | Hiragana | Podstawowe | 55 (z pustymi) |
| `H_DAKUTEN` | Hiragana | Dakuten + Handakuten | 25 |
| `H_KOMBINOWANE` | Hiragana | Kombinowane (きゃ itd.) | 55 |
| `K_PODSTAWOWE` | Katakana | Podstawowe | 55 (z pustymi) |
| `K_DAKUTEN` | Katakana | Dakuten + Handakuten | 25 |
| `K_KOMBINOWANE` | Katakana | Kombinowane | 55 |

Puste komórki `{"",""}` służą do zachowania układu siatki 5-kolumnowej (np. wiersz や ma tylko 3 znaki).

---

## GridLayout – budowanie kart dynamicznie

`refreshGrid()` buduje siatkę programatycznie w pętli. Każda karta to `CardView` o wysokości `dp(80)`:

```java
lp.columnSpec = GridLayout.spec(GridLayout.UNDEFINED, 1, 1f);
lp.width  = 0;
lp.height = dp(80);
card.setRadius(dp(14));
card.setCardElevation(isEmpty ? 0f : 4f);
card.setCardBackgroundColor(isEmpty ? colorBg : colorSurface);
```

Każda niepusta karta zawiera `LinearLayout` z:
1. `TextView` – znak kana (26sp)
2. `TextView` – romaji (11sp, kolor tekst_dodatkowy)
3. `TextView "✏"` – tylko jeśli `StrokeData.getStrokes(kana)` zwraca niepustą listę (9sp, kolor primary)

### Animacja wejścia kart (kaskadowa)
```java
card.setAlpha(0f);
card.setTranslationY(20f);
card.animate()
    .alpha(1f).translationY(0f)
    .setDuration(220).setStartDelay(i * 18)
    .start();
```

### Kliknięcie karty – efekt "wciskania"
```java
card.animate().scaleX(0.93f).scaleY(0.93f).setDuration(80)
    .withEndAction(() ->
        card.animate().scaleX(1f).scaleY(1f).setDuration(140)
            .withEndAction(() -> showStrokeDialog(kana, romaji))
            .start())
    .start();
```

---

## Dialog animacji kresek (`dialog_stroke_animation.xml`)

Po kliknięciu karty otwiera się dialog z:

| Widok | Opis |
|---|---|
| `tvDialogKana` | Znak kana, 52sp, wyśrodkowany |
| `tvDialogRomaji` | Romaji, 18sp |
| `StrokeAnimationView` | 260×260dp – animowana canvas kresek |
| `tvStrokeCount` | Liczba kresek (np. "3 kreski"), odmiana przez przypadki |
| `tvClose` | Przycisk zamknięcia |

Dialog jest transparentny (`android.R.color.transparent`), tło pochodzi z `@drawable/bg_dialog_rounded`.

### Odmiana liczby kresek
```java
tvCount.setText(count + getString(
    count == 1 ? R.string.stroke_unit_one :   // "kreska"
    count < 5  ? R.string.stroke_unit_few :   // "kreski"
                 R.string.stroke_unit_many     // "kresek"
));
```

### Zatrzymanie animacji przy zamknięciu
```java
dialog.setOnDismissListener(d -> av.stopAnimation());
```

---

## StrokeAnimationView – silnik animacji

Custom `View`, rysuje animację kolejności kresek znaku na `Canvas`.

### Paints
| Paint | Kolor | Użycie |
|---|---|---|
| `strokePaint` | `#E65100`, 18f, ROUND | Aktywna/ukończona kreska |
| `numberPaint` | `#FF7043`, 28f | Numery startowe |
| `dotPaint` | `#FF5722`, FILL | Punkty startowe kresek |

### Siatka pomocnicza (`drawGrid`)
Na canvasie 300×300: linia pionowa + pozioma przez środek + obie przekątne + ramka zewnętrzna. Kolor `#FFCCBC`, alpha 60–120.

### Algorytm animacji kresek

```
setCharacter(kana)
    → strokes = StrokeData.getStrokes(kana)
    → handler.postDelay(animateNextStroke, 300ms)

animateNextStroke()
    ├── wszystkie gotowe → czekaj 1500ms → resetuj → zacznij od nowa (pętla)
    └── nie → ValueAnimator(0 → pathLength), duration=500ms, LinearInterpolator
              → co klatkę: pm.getSegment(0, val, animatedPath) → invalidate()
              → po zakończeniu: opóźnienie 250ms → następna kreska
```

### Skalowanie w `onDraw`
```java
float scale = Math.min(getWidth(), getHeight()) / 300f;
canvas.translate(offsetX, offsetY);
canvas.scale(scale, scale);
```
Dane kresek są w przestrzeni 300×300 – skalowanie do rozmiaru widoku.

### Warstwy rysowania (kolejność)
1. Siatka pomocnicza
2. Ukończone kreski (alpha=80 – blade)
3. Aktualnie animowana kreska (alpha=255)
4. Kropki startowe + numery dla ukończonych i bieżącej kreski

---

## Przełączanie systemu i kategorii

System (Hiragana/Katakana) ustalany przy starcie z Intenta:
```java
boolean isKatakana = getIntent().getBooleanExtra("is_katakana", false);
currentSystem = isKatakana ? "KATAKANA" : "HIRAGANA";
```

Kategorie: kliknięcie zakładki → `switchCategory()` → aktualizacja kolorów + `refreshGrid()`.

Kolory zakładek: aktywna = `przycisk_glowny` + `tekst_na_glownym`, nieaktywna = `przycisk_dodatkowy` + `tekst_na_dodatkowym`.

---

## StrokeData – źródło danych kresek

Klasa auto-generowana (nie edytować ręcznie), źródło: **KanjiVG** (CC BY-SA 3.0).

```java
public static List<Path> getStrokes(String kana) {
    switch (kana) {
        case "あ": return build_3042();
        // 100+ przypadków
        default: return new ArrayList<>();
    }
}
```

Każda metoda `build_XXXX()` zwraca listę `Path` z krzywymi Béziera (`moveTo`, `cubicTo`).

### Znaki kombinowane (きゃ, シュ itp.)
Składane z dwóch znaków przez macierze transformacji:

```java
// 'きゃ' = 'き' (duży, lewy) + 'や' (mały, prawy)
private static List<Path> build_304d3083() {
    list.addAll(scaleLeft(build_304d()));   // き → 60% skali, offset (10, 20)
    list.addAll(scaleRight(build_3084()));  // や → 45% skali, offset (165, 140)
    return list;
}
```

| Metoda | Skala | Przesunięcie |
|---|---|---|
| `scaleLeft` | 60% | `postTranslate(10, 20)` |
| `scaleRight` | 45% | `postTranslate(165, 140)` |
