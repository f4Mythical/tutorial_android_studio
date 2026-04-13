# Notatka z `Spannable`

## Co to jest `Spannable`

`Spannable` w Androidzie pozwala stylować fragment tekstu, np.:
- zmienić kolor pojedynczych znaków
- pogrubić część tekstu
- wyróżnić poprawne i błędne litery

W projekcie `Spannable` jest używany głównie do:
- kolorowania odpowiedzi użytkownika
- pokazywania, które litery są poprawne, a które błędne

## Najczęściej używane klasy

W projekcie pojawiają się:
- `Spannable`
- `SpannableString`
- `ForegroundColorSpan`
- `StyleSpan`

Najczęstszy wzorzec:
- tworzenie `SpannableString` z odpowiedzi użytkownika
- przejście po znakach
- ustawienie:
  - zielonego koloru dla poprawnych liter
  - czerwonego koloru dla błędnych liter
  - `StyleSpan(Typeface.BOLD)` dla pogrubienia

## Gdzie jest używany

### `BaseCharacter`
Rola:
- kolorowanie odpowiedzi w nauce znaków

### `BaseWord`
Rola:
- kolorowanie odpowiedzi w nauce słówek

### `BaseSentence`
Rola:
- kolorowanie odpowiedzi w nauce zdań

### `DailyWordGeneratorFragment`
Rola:
- kolorowanie błędnej odpowiedzi przy daily word

### `WybraneZnakiPremium`
Rola:
- kolorowanie wpisanego romaji podczas nauki wybranych znaków

### `HiraganaDialog`
Rola:
- kolorowanie odpowiedzi w trybie dialogów

## Jak działa ten mechanizm

Typowy schemat:

1. pobierana jest odpowiedź użytkownika
2. pobierana jest poprawna odpowiedź
3. tworzony jest `SpannableString`
4. dla każdego znaku ustawiany jest styl:
   - poprawny znak:
     - zielony
     - pogrubiony
   - błędny znak:
     - czerwony
     - pogrubiony
5. gotowy tekst jest ustawiany do `TextView`

## Najczęściej używane flagi

W projekcie używane jest:
- `Spannable.SPAN_EXCLUSIVE_EXCLUSIVE`

Znaczenie:
- styl obowiązuje dokładnie na podanym zakresie znaków

## Po co to jest w aplikacji

To rozwiązanie daje użytkownikowi szybką informację:
- które litery wpisał dobrze
- gdzie zrobił błąd

Jest to lepsze niż zwykły komunikat „źle”, bo pokazuje dokładne miejsce błędu.

## Szybkie podsumowanie

W tym projekcie `Spannable` służy głównie do:
- porównywania odpowiedzi znak po znaku
- kolorowania tekstu
- pogrubiania poprawnych i błędnych fragmentów

Najczęściej jest używany w ekranach ćwiczeń językowych.
