Krok 1: Otwórz CMD (1 okno) - Serwer
```bash
cd C:\Users\boy12\AndroidStudioProjects\HiraganaAndKatakana\feedback-viewer
npx http-server -p 8080
```
Zostaw otwarte
Krok 2: Otwórz CMD (2 okno) - Ngrok
```bash
ngrok http 8080
```
Skopiuj link z Forwarding



<h1>LUB Z ZROK.IO</h1>

Krok 1: Otwórz CMD (1 okno) – Serwer
Tutaj uruchamiasz swoje pliki.
```bash
cd C:\Users\boy12\AndroidStudioProjects\HiraganaAndKatakana\feedback-viewer
npx http-server -p 8080
```
Zostaw to okno otwarte. Pamiętaj: jeśli chcesz, aby PHP działało, musisz tu zamiast npx odpalić Apache w XAMPP na porcie 8080.

Krok 2: Otwórz CMD (2 okno) – Rezerwacja adresu (Tylko raz!)
W zrok warto raz zarezerwować nazwę, żeby link był zawsze taki sam (np. hiragana-app).
```bash
cd Desktop
zrok reserve public localhost:8080 --unique-name hirakata
```
Zrób to tylko raz. Jeśli nazwa jest zajęta, wybierz inną.

Krok 3: Otwórz CMD (2 okno) – Uruchomienie tunelu
Teraz odpalasz właściwe udostępnianie. Używaj tej komendy codziennie:

```bash
zrok share reserved hirakata
```
Skopiuj link z wiersza Access Your Share At: (będzie to https://hirakata.share.zrok.io).
