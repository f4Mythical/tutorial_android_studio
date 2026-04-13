# Android Drawable – Shapes, Gradienty i Style

## 1. Shape Drawable (`<shape>`)

Podstawowy typ drawable definiowany w XML. Dostępne kształty:

- `rectangle` – prostokąt (domyślny)
- `oval` – elipsa / koło
- `line` – linia
- `ring` – pierścień

### Właściwości wewnątrz `<shape>`:

| Element | Opis | Przykład użycia |
|---|---|---|
| `<solid>` | Jednolite wypełnienie kolorem | `bg_accent_line`, `bg_avatar_circle` |
| `<stroke>` | Obramowanie (kolor, grubość) | `bg_display_card`, `bg_input_card` |
| `<corners>` | Zaokrąglenie narożników | `bg_bottom_sheet_rounded`, `bg_dialog` |
| `<gradient>` | Wypełnienie gradientem | `bg_button_gradient`, `bg_gradient_orange` |
| `<padding>` | Wewnętrzny margines | `bg_dialog`, `edittext_rounded` |
| `<size>` | Rozmiar kształtu | `bg_avatar_circle`, `bg_logo_circle` |

---

## 2. Gradienty

### Typy gradientów (`android:type`):

```xml
android:type="linear"   <!-- wzdłuż osi, kąt = android:angle (wielokrotność 45°) -->
android:type="radial"   <!-- promieniowy, wymaga android:gradientRadius -->
android:type="sweep"    <!-- okrągły, jak radar -->
```

### Atrybuty gradientu:

```xml
android:startColor="#FF8A65"
android:centerColor="#FFAB91"   <!-- opcjonalny, punkt środkowy -->
android:endColor="#FFCCBC"
android:angle="135"             <!-- tylko dla linear; 0° = lewa→prawa, 90° = dół→góra -->
android:gradientRadius="250%p"  <!-- tylko dla radial; px lub %p (% względem rozmiaru) -->
```

### Przykłady z projektu:

```xml
<!-- bg_button_gradient.xml – przycisk z gradientem pod kątem 45° -->
<gradient
    android:angle="45"
    android:startColor="@color/pomaranczowy_zaznaczony"
    android:centerColor="@color/przycisk_glowny"
    android:endColor="@color/przycisk_dodatkowy"
    android:type="linear" />

<!-- bg_background_settings.xml – złożone tło z dwóch gradientów -->
<!-- gradient radialny nałożony na gradient liniowy (layer-list) -->
```

---

## 3. Zaokrąglenia narożników

```xml
<!-- Jednolity promień dla wszystkich -->
<corners android:radius="28dp" />

<!-- Selektywne zaokrąglenia (np. tylko górna krawędź) -->
<corners
    android:topLeftRadius="28dp"
    android:topRightRadius="28dp"
    android:bottomLeftRadius="0dp"
    android:bottomRightRadius="0dp" />
```

Przykłady:
- `bg_bottom_sheet_rounded` – zaokrąglony tylko góra (bottom sheet)
- `bg_nav_side_panel` – zaokrąglony tylko lewa strona (panel boczny)
- `bg_dialog_rounded` – `radius="28dp"` pełny
- `bg_nav_panel` – `radius="36dp"` (efekt pigułki)

---

## 4. Layer-list (`<layer-list>`)

Pozwala nakładać wiele kształtów na siebie (jak warstwy w Photoshopie).

```xml
<!-- bg_background_settings.xml -->
<layer-list>
    <item>
        <!-- warstwa 1: gradient liniowy -->
        <shape><gradient ... /></shape>
    </item>
    <item>
        <!-- warstwa 2: gradient radialny nałożony transparentnie -->
        <shape><gradient android:type="radial" ... /></shape>
    </item>
</layer-list>
```

Inne zastosowania `layer-list` w projekcie:
- `bg_navbar` – biały gradient + linia separatora na górze (`android:top="0dp" android:height="1dp"`)
- `bg_nav_icon_active` – pojedyncza warstwa z wypełnieniem
- `bg_card_toggle_selected` – gradient + obramowanie jako osobne warstwy

---

## 5. Selector (`<selector>`)

Zmienia wygląd w zależności od stanu widoku.

```xml
<!-- bg_char_selector.xml -->
<selector>
    <item android:state_selected="true">
        <shape>
            <solid android:color="@color/linia_jasna" />
            <stroke android:width="2dp" android:color="@color/tekst_glowny" />
            <corners android:radius="8dp" />
        </shape>
    </item>
    <item>
        <!-- stan domyślny -->
        <shape>
            <solid android:color="@color/tlo_powierzchnia" />
            <stroke android:width="1dp" android:color="@color/linia_glowna" />
            <corners android:radius="8dp" />
        </shape>
    </item>
</selector>
```

Dostępne stany: `state_selected`, `state_checked`, `state_pressed`, `state_enabled`, `state_focused`

---

## 6. Ripple (`<ripple>`)

Efekt dotknięcia (Material Design). Wymaga `<item android:id="@android:id/mask">` jako maski kształtu.

```xml
<!-- bg_start_button.xml -->
<ripple android:color="@color/ripple">
    <item android:id="@android:id/mask">
        <shape android:shape="oval">
            <solid android:color="@color/przycisk_glowny" />
        </shape>
    </item>
    <item>
        <!-- właściwy wygląd przycisku -->
        <shape android:shape="oval">
            <gradient android:type="radial" ... />
            <stroke ... />
        </shape>
    </item>
</ripple>
```

---

## 7. Kształty charakterystyczne – wzorce z projektu

### Koło / awatar
```xml
<shape android:shape="oval">
    <solid android:color="#FF7043" />
    <size android:width="80dp" android:height="80dp" />
</shape>
```

### Karta z cieniem wizualnym (stroke zamiast elevation)
```xml
<shape android:shape="rectangle">
    <corners android:radius="20dp" />
    <solid android:color="@color/tlo_powierzchnia" />
    <stroke android:width="1dp" android:color="@color/linia_jasna" />
</shape>
```

### Pole tekstowe / spinner
```xml
<shape android:shape="rectangle">
    <corners android:radius="12dp" />
    <solid android:color="@color/tlo_pola" />
    <stroke android:width="1dp" android:color="@color/linia_jasna" />
</shape>
```

### Przycisk gradient (pill)
```xml
<shape android:shape="rectangle">
    <gradient
        android:angle="45"
        android:startColor="@color/pomaranczowy_zaznaczony"
        android:endColor="@color/przycisk_dodatkowy"
        android:type="linear" />
    <corners android:radius="30dp"/>
</shape>
```

### Karta toggle (selected / unselected)
```xml
<!-- selected: bg_card_toggle_selected -->
<layer-list>
    <item>
        <shape><corners android:radius="18dp"/>
            <gradient android:angle="135" android:startColor="#FF8A65" android:endColor="#FFF3E0" /></shape>
    </item>
    <item>
        <shape><corners android:radius="18dp"/>
            <stroke android:width="2.5dp" android:color="#FF7043"/>
            <solid android:color="#00000000"/></shape>
    </item>
</layer-list>

<!-- unselected: bg_card_toggle_unselected -->
<shape android:shape="rectangle">
    <corners android:radius="18dp"/>
    <solid android:color="#FFF9F6"/>
    <stroke android:width="1dp" android:color="#FFE0D0"/>
</shape>
```

---

## 8. Konwencje nazewnicze w projekcie

| Prefix | Znaczenie |
|---|---|
| `bg_` | Tło widoku (background) |
| `ic_` | Ikona (vector drawable) |
| `gradient_` | Głównie gradient bez logiki stanu |
| `circle_` / `bg_..._circle` | Kształt owalny/kołowy |
| `widget_` | Drawable dedykowany widgetowi |
| `nav_` | Elementy nawigacji |
| `edittext_` | Tło pola tekstowego |

---

## 9. Kolory projektu (referencje)

Projekt używa własnych tokenów kolorów:

| Token | Użycie |
|---|---|
| `@color/pomaranczowy_zaznaczony` | Akcent, aktywne elementy, dot-y, gradient start |
| `@color/przycisk_glowny` | Główny kolor przycisku |
| `@color/przycisk_dodatkowy` | Dodatkowy kolor przycisku (gradient end) |
| `@color/tlo_powierzchnia` | Tło kart i powierzchni |
| `@color/tlo_pola` | Tło pól inputów, spinnerów |
| `@color/linia_jasna` | Subtelne obramowania |
| `@color/linia_glowna` | Wyraźniejsze linie |
| `@color/tekst_glowny` | Kolor głównego tekstu |
| `@color/ripple` | Efekt ripple |

---

## 10. Dobre praktyki

- **`stroke-width` w dp**, nie px – skaluje się z gęstością ekranu
- **`radius="30dp"`** lub wyżej przy wysokości ~50dp = efekt pigułki (pill)
- **`layer-list`** do złożonych teł zamiast kodu w Javie/Kotlinie
- **`selector`** zamiast ustawiania tła programatycznie przez `setBackgroundResource`
- **`ripple`** zawsze z maską (`@android:id/mask`) żeby efekt był przycięty do kształtu
- Gradientowe przyciski – `type="linear"`, kąt `45°` lub `135°` dla przekątnej
- Karty z delikatnym `stroke 1dp` w kolorze `linia_jasna` zamiast `elevation` – lżejszy efekt
