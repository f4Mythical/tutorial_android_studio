# DrawingView – Szczegółowa Notatka Techniczna

## Mapa plików i powiązania

```
DrawingPracticeActivity.java
 ├── używa → StrokeData.java         (dane kresek jako Path)
 ├── używa → DrawingView.java        (canvas rysowania + ocena)
 └── layout → activity_drawing_practice.xml
               └── <com.example.hiraganaandkatakana.DrawingView>

StrokePracticeActivity.java
 ├── używa → StrokeData.java         (te same dane kresek)
 ├── używa → StrokeAnimationView.java (tylko animacja, bez rysowania)
 └── layout → activity_stroke_practice.xml
               └── dialog_stroke_animation.xml
                    └── <com.example.hiraganaandkatakana.StrokeAnimationView>
```

**StrokeData** jest wspólny dla obu aktywności – ta sama baza danych kresek.

**DrawingView** – rysowanie + ocena (DrawingPractice)
**StrokeAnimationView** – tylko animacja wzorca (StrokePractice)

---

## Dwie przestrzenie współrzędnych

To jest kluczowy koncept całego systemu. W kodzie istnieją **dwie różne przestrzenie**:

### Przestrzeń canvasu (300×300)
- Używana przez `StrokeData` – wszystkie dane kresek są w tej przestrzeni
- `(0,0)` = lewy górny róg, `(300,300)` = prawy dolny róg
- Niezależna od rozmiaru ekranu
- Przykład: kreska あ zaczyna się od `(85.35, 90.83)` w tej przestrzeni

### Przestrzeń ekranu (px fizyczne)
- Używana przez `onTouchEvent` – dotyk użytkownika jest w tej przestrzeni
- Zależy od rozmiaru urządzenia i gęstości pikseli
- Przykład: na telefonie 1080×2340px dotyk może być w `(540, 800)`

### Przeliczenie (obliczane w `onSizeChanged`)

```java
@Override
protected void onSizeChanged(int w, int h, int ow, int oh) {
    scale = Math.min(w, h) / CANVAS_SIZE;  // np. 360px / 300 = 1.2
    offX  = (w - CANVAS_SIZE * scale) / 2f; // wyśrodkowanie poziome
    offY  = (h - CANVAS_SIZE * scale) / 2f; // wyśrodkowanie pionowe
}
```

Jeśli widok ma 360×360px: `scale = 1.2`, `offX = 0`, `offY = 0`
Jeśli widok ma 400×360px: `scale = 1.2`, `offX = 20`, `offY = 0`

### Konwersja ekran → canvas

```java
// Używane w score() przed obliczeniem wyniku
for (int i = 0; i < drawn.length; i += 2) {
    drawn[i]     = (drawn[i]     - offX) / scale;  // x_canvas = (x_screen - offX) / scale
    drawn[i + 1] = (drawn[i + 1] - offY) / scale;  // y_canvas = (y_screen - offY) / scale
}
```

### Konwersja canvas → ekran (w `onDraw`)

```java
canvas.save();
canvas.translate(offX, offY);   // przesuń układ o offset
canvas.scale(scale, scale);     // skaluj razy scale
// teraz rysowanie w jednostkach canvasu automatycznie trafia w ekran
canvas.drawPath(refStrokes.get(i), paint);  // Path w przestrzeni 300×300
canvas.restore();
```

---

## Cykl życia DrawingView

### 1. Inicjalizacja – `setCharacter(kana)`

```java
public void setCharacter(String kana) {
    refStrokes.clear();
    refStrokes.addAll(StrokeData.getStrokes(kana));  // załaduj wzorcowe kreski
    reset();
}

public void reset() {
    userStrokes.clear();       // wyczyść narysowane kreski użytkownika
    strokeResults.clear();     // wyczyść wyniki (true/false) dla każdej kreski
    currentPath    = new Path();
    expectedStroke = 0;        // czekamy na kreskę nr 0
    isFinished     = false;
    nearDot        = false;
    showingCorrect = false;
    correctStrokeIdx = -1;
    setEnabled(true);
    invalidate();
}
```

Po `setCharacter` widok jest gotowy – `refStrokes` zawiera wzorce, `expectedStroke = 0` oznacza "czekamy na pierwszą kreskę".

---

### 2. Dotyk użytkownika – `onTouchEvent`

#### ACTION_DOWN (palec dotyka ekranu)

```java
case MotionEvent.ACTION_DOWN: {
    float[] s = startScreenPt();  // punkt startowy bieżącej wzorcowej kreski w px ekranu
    nearDot = s != null && dist2d(x, y, s[0], s[1]) <= START_RADIUS;
    // START_RADIUS = 50f px – strefa tolerancji
    if (!nearDot) return true;  // zignoruj dotyk poza strefą
    currentPath = new Path();
    currentPath.moveTo(x, y);
    lastX = x; lastY = y;
    break;
}
```

**`startScreenPt()`** – przelicza punkt startowy wzorcowej kreski z przestrzeni canvasu na ekran:
```java
private float[] startScreenPt() {
    PathMeasure pm = new PathMeasure(refStrokes.get(expectedStroke), false);
    float[] p = new float[2];
    pm.getPosTan(0, p, null);  // punkt na pozycji 0 (początek ścieżki)
    return new float[]{ p[0] * scale + offX, p[1] * scale + offY };
    // przeliczenie canvas → ekran
}
```

#### ACTION_MOVE (przesunięcie palca)

```java
case MotionEvent.ACTION_MOVE: {
    if (!nearDot) return true;  // ignoruj jeśli start był poza strefą
    float mx = (lastX + x) / 2f;
    float my = (lastY + y) / 2f;
    currentPath.quadTo(lastX, lastY, mx, my);
    // quadTo tworzy krzywą kwadratową Béziera przez punkt środkowy
    // efekt: wygładzona linia zamiast kanciastych segmentów
    lastX = x; lastY = y;
    invalidate();  // przerysuj widok
    break;
}
```

Wygładzanie przez `quadTo`: zamiast linii prostej `A→B`, rysuje krzywą przez punkt środkowy `(A+B)/2`. Przy szybkim rysowaniu daje płynną linię bez "schodków".

#### ACTION_UP (palec uniesiony)

```java
case MotionEvent.ACTION_UP: {
    if (!nearDot) return true;
    currentPath.lineTo(x, y);  // domknij ścieżkę ostatnim punktem
    PathMeasure pm = new PathMeasure(currentPath, false);
    if (pm.getLength() >= MIN_LEN_PX) {  // MIN_LEN_PX = 20f
        evaluateStroke();
    } else {
        currentPath = new Path();  // za krótki ruch – zignoruj
        invalidate();
    }
    break;
}
```

---

### 3. Ocena kreski – `evaluateStroke()`

```java
private void evaluateStroke() {
    float sim = score(currentPath, refStrokes.get(expectedStroke));
    boolean ok = sim >= PASS_THRESHOLD;  // 0.52f

    userStrokes.add(new Path(currentPath));  // zapisz narysowaną kreskę
    strokeResults.add(ok);                    // zapisz wynik

    if (!ok) {
        showingCorrect   = true;           // pokaż wzorcową kreskę
        correctStrokeIdx = expectedStroke;
        setEnabled(false);                 // zablokuj dotyk
    }

    expectedStroke++;  // przejdź do następnej kreski

    if (callback != null)
        callback.onStrokeFinished(expectedStroke - 1, sim, ok);

    if (!ok) {
        currentPath = new Path();
        invalidate();
        return;  // zatrzymaj tutaj – Activity odczeka i wywoła hideCorrectAndContinue()
    }

    if (expectedStroke >= refStrokes.size()) {
        isFinished = true;
        setEnabled(false);
        if (callback != null) callback.onAllStrokesDone(totalAccuracy());
    }

    currentPath = new Path();
    invalidate();
}
```

#### Przepływ przy błędnej kresce

```
evaluateStroke() → ok=false
    → showingCorrect=true → rysuj niebieską przerywaną kreskę
    → setEnabled(false) → blokada dotyku
    → callback.onStrokeFinished(..., false)
        → Activity: handler.postDelay(1800ms):
            drawingView.hideCorrectAndContinue()
                → showingCorrect=false
                → setEnabled(true)
                → invalidate()
            jeśli expectedStroke >= totalStrokes → onAllStrokesDone()
            else → hint "Rysuj kolejną kreskę"
```

---

### 4. Rysowanie (`onDraw`)

```java
@Override
protected void onDraw(Canvas canvas) {
    if (refStrokes.isEmpty()) return;

    canvas.save();
    canvas.translate(offX, offY);
    canvas.scale(scale, scale);
    // od teraz jednostki = przestrzeń canvasu (300×300)

    drawGrid(canvas);  // siatka pomocnicza

    // 1. Narysowane kreski użytkownika (zielone/czerwone)
    for (int i = 0; i < userStrokes.size(); i++) {
        boolean ok = strokeResults.get(i);
        canvas.drawPath(toCanvas(userStrokes.get(i)), ok ? donePaint : wrongPaint);
        // toCanvas() przelicza kreski użytkownika z px ekranu na przestrzeń canvasu
    }

    // 2. Aktualnie rysowana kreska (pomarańczowa)
    if (!currentPath.isEmpty()) {
        canvas.drawPath(toCanvas(currentPath), currentPaint);
    }

    // 3a. Jeśli pokazujemy wzorzec po błędzie (niebieska przerywana)
    if (showingCorrect && correctStrokeIdx >= 0) {
        canvas.drawPath(refStrokes.get(correctStrokeIdx), correctShowPaint);
        // tu NIE ma toCanvas() – refStrokes są już w przestrzeni canvasu
        // start dot (niebieski, r=14)
        // end dot (niebieski, r=8)
    }
    // 3b. Normalny tryb – pokaż punkt startowy następnej kreski
    else if (!isFinished && expectedStroke < refStrokes.size()) {
        PathMeasure pm = new PathMeasure(refStrokes.get(expectedStroke), false);
        float[] p = new float[2];
        pm.getPosTan(0, p, null);
        canvas.drawCircle(p[0], p[1], START_RADIUS / scale, hintPaint);  // półtransparentne kółko
        canvas.drawCircle(p[0], p[1], 18f, dotPaint);                    // pełna pomarańczowa kropka
        canvas.drawText(String.valueOf(expectedStroke + 1), ...);         // numer kreski
    }

    canvas.restore();
}
```

#### Dlaczego `toCanvas()` tylko dla kresek użytkownika?

Kreski użytkownika (`userStrokes`, `currentPath`) są zapisywane w **przestrzeni ekranu** (px). Wzorce `refStrokes` są w **przestrzeni canvasu** (300×300). W `onDraw` obowiązuje transformacja `translate + scale`, więc:
- `refStrokes` rysowane bezpośrednio → poprawnie
- `userStrokes` muszą być przeliczone przez `toCanvas()` → odwrotna transformacja

```java
private Path toCanvas(Path screen) {
    android.graphics.Matrix m = new android.graphics.Matrix();
    m.setTranslate(-offX, -offY);    // cofnij offset
    m.postScale(1f / scale, 1f / scale);  // cofnij skalowanie
    Path out = new Path();
    screen.transform(m, out);
    return out;
}
```

---

## Algorytm oceny – `score()` w szczegółach

### Próbkowanie ścieżek – `sample(path, 64)`

```java
private float[] sample(Path path, int n) {
    PathMeasure pm = new PathMeasure(path, false);
    float len = pm.getLength();
    if (len < 5f) return null;
    float[] out = new float[n * 2];  // 64 punkty × 2 współrzędne = 128 wartości
    float[] p   = new float[2];
    for (int i = 0; i < n; i++) {
        pm.getPosTan(len * i / (float)(n - 1), p, null);
        // getPosTan(distance, position, tangent)
        // distance = len * i/63 → równomiernie rozmieszczone wzdłuż ścieżki
        out[i*2]   = p[0];  // x
        out[i*2+1] = p[1];  // y
    }
    return out;
}
```

`PathMeasure.getPosTan()` to kluczowe API – pobiera punkt leżący na ścieżce w odległości `distance` od początku. Dzięki temu niezależnie od prędkości rysowania zawsze mamy 64 równomiernie rozłożone punkty.

### Składowa 1: `shapeScore` (waga 40%)

Porównuje **kształt** (ignorując pozycję i rozmiar).

```java
private float shapeScore(float[] d, float[] r) {
    float[] dN = normBBox(d);  // normalizuj narysowaną do [0,1]×[0,1]
    float[] rN = normBBox(r);  // normalizuj wzorcową do [0,1]×[0,1]
    float dtw = dtwDist(dN, rN);
    int n = dN.length / 2;     // = 64
    float maxD = (float) Math.sqrt(2.0) * n;  // maksymalna możliwa odległość DTW
    return Math.max(0f, 1f - dtw / maxD);
}
```

**Normalizacja `normBBox`**: przesuwa i skaluje punkty tak, żeby cała ścieżka mieściła się w kwadracie [0,1]×[0,1]. Dzięki temu kreska narysowana małym ruchem i dużym ruchem daje taki sam wynik dla kształtu.

**DTW (Dynamic Time Warping)**: algorytm porównania dwóch sekwencji punktów. Lepszy niż proste porównanie par, bo dopasowuje punkty elastycznie:

```java
private float dtwDist(float[] a, float[] b) {
    int n = a.length / 2;  // 64
    float[] prev = new float[n];
    float[] curr = new float[n];
    // programowanie dynamiczne O(n²)
    Arrays.fill(prev, Float.MAX_VALUE / 2f);
    prev[0] = pd(a, b, 0, 0);
    for (int j = 1; j < n; j++) prev[j] = prev[j-1] + pd(a, b, 0, j);
    for (int i = 1; i < n; i++) {
        curr[0] = prev[0] + pd(a, b, i, 0);
        for (int j = 1; j < n; j++)
            curr[j] = pd(a, b, i, j) + Math.min(prev[j], Math.min(curr[j-1], prev[j-1]));
        float[] tmp = prev; prev = curr; curr = tmp;
    }
    return prev[n-1];  // koszt optymalnego dopasowania
}
```

`pd(a, b, i, j)` = odległość euklidesowa między punktem `i` z ścieżki `a` i punktem `j` z ścieżki `b`.

### Składowa 2: `directionScore` (waga 38%)

Porównuje **kierunek ruchu** segment po segmencie.

```java
private float directionScore(float[] d, float[] r) {
    int n = SAMPLE_N - 1;  // 63 segmenty
    float total = 0f;
    for (int i = 0; i < n; i++) {
        // wektor segmentu narysowanego
        float dDx = d[(i+1)*2]   - d[i*2];
        float dDy = d[(i+1)*2+1] - d[i*2+1];
        // wektor segmentu wzorcowego
        float rDx = r[(i+1)*2]   - r[i*2];
        float rDy = r[(i+1)*2+1] - r[i*2+1];
        float dl = sqrt(dDx²+dDy²);
        float rl = sqrt(rDx²+rDy²);
        if (dl < 1e-6f || rl < 1e-6f) continue;  // pomiń punkty stacjonarne
        float dot = (dDx/dl)*(rDx/rl) + (dDy/dl)*(rDy/rl);
        // dot ∈ [-1, 1]:  1=ten sam kierunek,  0=prostopadłe,  -1=przeciwny
        total += (dot + 1f) / 2f;  // mapuj na [0, 1]
    }
    return total / n;
}
```

To jest **najostrzejsza** składowa – wymaga żeby każdy z 63 segmentów był skierowany prawie tak samo jak wzorzec. Przy odręcznym rysowaniu ręka drżeje i wiele segmentów ma lekko odchylony kierunek → duże straty punktów.

### Składowa 3: `positionScore` (waga 14%)

Sprawdza czy kreska jest **narysowana w odpowiednim miejscu** canvasu.

```java
private float positionScore(float[] d, float[] r) {
    float[] dc = centroid(d);  // średni punkt narysowanej ścieżki
    float[] rc = centroid(r);  // średni punkt wzorcowej ścieżki
    float dist = dist2d(dc[0], dc[1], rc[0], rc[1]);
    return Math.max(0f, 1f - dist / (CANVAS_SIZE * 0.28f));
    // tolerancja: 300 * 0.28 = 84 jednostki canvasu
}
```

### Składowa 4: `coverageScore` (waga 8%)

Sprawdza czy kreska ma **podobne proporcje** (szerokość/wysokość) do wzorca.

```java
private float coverageScore(float[] d, float[] r) {
    float[] db = bbox(d);  // [minX, minY, maxX, maxY]
    float[] rb = bbox(r);
    float dW = db[2]-db[0]; float dH = db[3]-db[1];
    float rW = rb[2]-rb[0]; float rH = rb[3]-rb[1];
    float rw = Math.min(dW,rW)/Math.max(dW,rW);  // stosunek szerokości
    float rh = Math.min(dH,rH)/Math.max(dH,rH);  // stosunek wysokości
    return (rw + rh) / 2f;
}
```

---

## Stany DrawingView – kompletna lista pól

```java
private final List<Path>    refStrokes    = new ArrayList<>();  // wzorcowe kreski (canvas 300×300)
private final List<Path>    userStrokes   = new ArrayList<>();  // narysowane kreski (px ekranu)
private final List<Boolean> strokeResults = new ArrayList<>();  // wyniki oceny

private Path    currentPath    = new Path();  // kreska aktualnie rysowana
private float   lastX, lastY;                 // poprzedni punkt dotyku (do wygładzania)
private int     expectedStroke = 0;           // indeks oczekiwanej kreski
private boolean isFinished     = false;       // czy wszystkie kreski ukończone
private boolean nearDot        = false;       // czy ACTION_DOWN trafił w strefę startową

private boolean showingCorrect   = false;     // czy pokazujemy wzorzec po błędzie
private int     correctStrokeIdx = -1;        // który wzorzec pokazujemy

private float scale, offX, offY;             // przelicznik przestrzeni
```

---

## Siatka pomocnicza (`drawGrid`)

```java
private void drawGrid(Canvas canvas) {
    gridPaint.setAlpha(140);
    canvas.drawRect(0f, 0f, CANVAS_SIZE, CANVAS_SIZE, gridPaint);  // ramka
    gridPaint.setAlpha(70);
    canvas.drawLine(150f, 0f, 150f, 300f, gridPaint);   // pionowa środkowa
    canvas.drawLine(0f, 150f, 300f, 150f, gridPaint);   // pozioma środkowa
    gridPaint.setAlpha(40);
    canvas.drawLine(0f, 0f, 300f, 300f, gridPaint);     // przekątna ↘
    canvas.drawLine(300f, 0f, 0f, 300f, gridPaint);     // przekątna ↙
}
```

Identyczna siatka jak w `StrokeAnimationView` – kolor `#FFCCBC` (jasny różowy).

---

## Interfejs DrawingCallback

```java
public interface DrawingCallback {
    void onStrokeFinished(int strokeIndex, float similarity, boolean passed);
    void onAllStrokesDone(float totalAccuracy);
}
```

`DrawingPracticeActivity` implementuje ten interfejs i obsługuje:

```java
@Override
public void onStrokeFinished(int strokeIndex, float similarity, boolean passed) {
    if (passed) {
        tvHint.setText("Dobrze!");
        tvHint.setTextColor(R.color.poprawny);
    } else {
        tvHint.setText("Niepoprawnie – popatrz na prawidłową kreskę");
        tvHint.setTextColor(R.color.bledny);
        handler.postDelayed(() -> {
            drawingView.hideCorrectAndContinue();
            if (drawingView.getExpectedStroke() >= drawingView.getTotalStrokes()) {
                onAllStrokesDone(0f);
            } else {
                tvHint.setText("Rysuj kolejną kreskę");
            }
        }, CORRECT_SHOW_MS);  // 1800ms
    }
}

@Override
public void onAllStrokesDone(float totalAccuracy) {
    handler.postDelayed(this::nextCharacter, 1200);  // auto-przejście po 1200ms
}
```

---

## Problem z PASS_THRESHOLD – szczegółowa analiza

Próg `0.52f` w praktyce działa tak:

```
Przykładowy wynik dla dobrze narysowanej kreski:
  shapeScore    = 0.65  →  0.65 × 0.40 = 0.260
  directionScore= 0.72  →  0.72 × 0.38 = 0.274
  positionScore = 0.80  →  0.80 × 0.14 = 0.112
  coverageScore = 0.85  →  0.85 × 0.08 = 0.068
  ŁĄCZNIE = 0.714 → PASS ✓

Przykład gdzie directionScore jest niski (drżąca ręka):
  shapeScore    = 0.70  →  0.70 × 0.40 = 0.280
  directionScore= 0.48  →  0.48 × 0.38 = 0.182  ← tu traci
  positionScore = 0.75  →  0.75 × 0.14 = 0.105
  coverageScore = 0.80  →  0.80 × 0.08 = 0.064
  ŁĄCZNIE = 0.631 → PASS ✓ (jeszcze przechodzi)

Przykład słabszy directionScore:
  directionScore= 0.35  →  0.35 × 0.38 = 0.133
  reszta dobra  = ~0.45
  ŁĄCZNIE = 0.583 → PASS ✓ (ledwo)

  directionScore= 0.25  →  0.25 × 0.38 = 0.095
  reszta dobra  = ~0.45
  ŁĄCZNIE = 0.545 → FAIL ✗ (0.545 ≥ 0.52 jeszcze pass...)
```

Główny problem: `directionScore` jest obliczany **przed normalizacją** – porównuje kierunki w przestrzeni ekranu gdzie narysowana i wzorcowa mogą mieć inną prędkość próbkowania. Oraz `SAMPLE_N=64` przy kreskach o długości ~100-150px daje segmenty o długości ~2-3px, które są bardzo czułe na drżenie ręki.

### Rekomendowana zmiana

```java
// Obecne:
private static final float PASS_THRESHOLD = 0.52f;
private static final float W_DIR = 0.38f;

// Sugerowane:
private static final float PASS_THRESHOLD = 0.40f;
private static final float W_SHAPE = 0.50f;  // zwiększ wagę kształtu
private static final float W_DIR   = 0.25f;  // zmniejsz wagę kierunku
private static final float W_POS   = 0.16f;
private static final float W_COVERAGE = 0.09f;
```
