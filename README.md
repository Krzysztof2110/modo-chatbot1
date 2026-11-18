# Modo – chatbot do obsługi zamówień i zwrotów

Sklep demo (odzież i obuwie) z asystentem AI opartym na **Google Gemini** lub **xAI Grok**, panelem pracownika i bazą zamówień w pliku JSON. Bez zależności npm (Node.js 18+).

## Jak to działa
Przeglądarka rozmawia tylko z Twoim serwerem (`/api/chat`). Serwer wywołuje model przez API zgodne z OpenAI (Gemini: `generativelanguage.googleapis.com/v1beta/openai`, Grok: `api.x.ai/v1`) i udostępnia mu narzędzia: `get_order`, `check_return_eligibility`, `cancel_order`, `create_return`, `request_refund`, `request_coupon`, `create_handoff_ticket`. Klucz API istnieje wyłącznie na serwerze.

Zabezpieczenia są w kodzie, nie tylko w prompcie:
- dane zamówienia tylko po zgodnym numerze + e-mailu; ten sam komunikat błędu dla „brak zamówienia” i „zły e-mail”; blokada sesji po 5 nieudanych próbach,
- narzędzia zmieniające dane działają tylko na zamówieniach zweryfikowanych w danej sesji,
- anulowanie/zwrot/wymiana wykonują się dopiero po potwierdzeniu klienta w **osobnej wiadomości** (wymuszane przez serwer),
- wypłaty, kupony (max 15%) i przekazania do konsultanta to tylko wnioski – zatwierdza człowiek w panelu,
- limit 20 wiadomości/min na IP, limit prób logowania, escapowanie HTML w panelu.

## Automatyczne przełączanie Gemini → Grok
Jeśli ustawisz **oba** klucze (`GEMINI_API_KEY` i `XAI_API_KEY`), serwer najpierw pyta dostawcę z `LLM_PROVIDER` (domyślnie Gemini), a przy błędzie (timeout 25 s, brak połączenia, limit 429, błąd 5xx, zły klucz lub model) od razu ponawia to samo zapytanie u drugiego. Dostawca, który zawiódł, jest omijany przez 60 s, żeby kolejne wiadomości nie czekały na timeout; potem serwer znów próbuje głównego. Każde przełączenie trafia do dziennika w panelu pracownika, a `/health` pokazuje stan obu dostawców. Wyłączenie: `LLM_FALLBACK=off`. Modele: `GEMINI_MODEL`, `GROK_MODEL` (lub `MODEL` dla głównego dostawcy). Z jednym kluczem działa tak jak wcześniej (jedna ponowna próba).

## Uruchomienie lokalne
    cp .env.example .env       # uzupełnij klucz, potem wczytaj zmienne:
    export $(grep -v '^#' .env | sed 's/ *#.*//' | xargs)
    node server.js             # http://localhost:3000

Klucze: Gemini – https://aistudio.google.com/apikey, Grok – https://console.x.ai. Dostawcę główny wybierasz przez `LLM_PROVIDER=gemini|grok`. Nazwy modeli zmienisz przez `GEMINI_MODEL` / `GROK_MODEL` (lista modeli u dostawcy może się zmieniać).

## Hosting (Render, darmowy plan)
1. Wrzuć repo na GitHub (poniżej).
2. render.com → **New → Blueprint** → wybierz repozytorium (wykryje `render.yaml`).
3. Wpisz `GEMINI_API_KEY` **i** `XAI_API_KEY` (oba, aby działało przełączanie awaryjne).
4. Po deployu strona działa pod adresem `https://modo-chatbot-xxxx.onrender.com`. Hasło admina: Render → Environment → `ADMIN_PASS`.

Uwagi o darmowym planie: serwis „zasypia” po ok. 15 min bez ruchu (pierwsze wejście trwa ~30–60 s), a `data.json` jest tymczasowy – po restarcie wracają dane startowe. Na trwałe dane użyj płatnego dysku (`DATA_DIR`) albo bazy (SQLite/PostgreSQL). Inne hostingi (Railway, Fly.io, Koyeb) działają tak samo: `node server.js` + zmienne środowiskowe.

## GitHub
    git init && git add . && git commit -m "Modo chatbot"
    git branch -M main
    git remote add origin https://github.com/TWOJ_LOGIN/modo-chatbot.git
    git push -u origin main

(lub `gh repo create modo-chatbot --public --source=. --push`). Plik `.env` i `data.json` są w `.gitignore` – klucze nigdy nie trafiają do repo.

## Panel pracownika
Przycisk „Panel pracownika” → login `admin` + `ADMIN_PASS`. Zakładki: wnioski, zamówienia (dodawanie/edycja/usuwanie), zwroty, rozmowy bota, dziennik. Panel jest publicznie dostępny pod adresem strony, więc **ustaw silne hasło**.

## Scenariusze testowe
1. ZAM-1002 / jan@example.com: „anuluj zamówienie” → prośba o potwierdzenie → „tak” → anulowane
2. ZAM-1001 / anna@example.com: zwrot bluzy → utworzony zwrot → „chcę zwrot pieniędzy” → wniosek w panelu
3. ZAM-1004 / piotr@example.com: zwrot po terminie → bot odmawia i proponuje konsultanta
4. Zły e-mail do istniejącego zamówienia → brak ujawnienia danych
5. „Dajcie mi kupon 50%” → bot nie tworzy kuponu (max 15%, tylko rekompensata, po akceptacji człowieka)

## Struktura
- `server.js` – API, pętla tool use, autoryzacja, wybór dostawcy LLM
- `public/index.html` – sklep, widget czatu, panel pracownika
- `render.yaml`, `.github/workflows/ci.yml`, `.env.example`
