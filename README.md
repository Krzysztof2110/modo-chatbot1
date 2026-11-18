# Modo – chatbot do obsługi zamówień i zwrotów

Sklep demo (odzież i obuwie) z asystentem AI opartym na **Google Gemini** i **Groq** (z automatycznym przełączaniem), panelem pracownika i bazą zamówień w pliku JSON. Bez zależności npm (Node.js 18+).

## Jak to działa
Przeglądarka rozmawia tylko z Twoim serwerem (`/api/chat`). Serwer wywołuje model przez API zgodne z OpenAI (Gemini: `generativelanguage.googleapis.com/v1beta/openai`, Groq: `api.groq.com/openai/v1`) i udostępnia mu narzędzia: `get_order`, `check_return_eligibility`, `cancel_order`, `create_return`, `request_refund`, `request_coupon`, `create_handoff_ticket`. Klucze API istnieją wyłącznie na serwerze.

Zabezpieczenia są w kodzie, nie tylko w prompcie:
- dane zamówienia tylko po zgodnym numerze + e-mailu; ten sam komunikat błędu dla „brak zamówienia” i „zły e-mail”; blokada sesji po 5 nieudanych próbach,
- narzędzia zmieniające dane działają tylko na zamówieniach zweryfikowanych w danej sesji,
- anulowanie/zwrot/wymiana wykonują się dopiero po potwierdzeniu klienta w **osobnej wiadomości** (wymuszane przez serwer),
- wypłaty, kupony (max 15%) i przekazania do konsultanta to tylko wnioski – zatwierdza człowiek w panelu,
- limit 20 wiadomości/min na IP, limit prób logowania, escapowanie HTML w panelu.

## Automatyczne przełączanie Gemini → Groq
Przy ustawionych **obu** kluczach serwer pyta najpierw dostawcę z `LLM_PROVIDER` (domyślnie Gemini), a przy błędzie (timeout 30 s, zmienisz przez `LLM_TIMEOUT_MS`, brak połączenia, limit 429, błąd 5xx, zły klucz lub model) od razu ponawia zapytanie u drugiego. Dostawca, który zawiódł, jest omijany przez 60 s. Każde przełączenie trafia do dziennika w panelu pracownika, a `/health` pokazuje stan obu dostawców. Wyłączenie: `LLM_FALLBACK=off`. Z jednym kluczem działa bez przełączania.

Modele: `GEMINI_MODEL` (domyślnie `gemini-flash-latest`), `GROQ_MODEL` (domyślnie `openai/gpt-oss-120b`; stary `llama-3.3-70b-versatile` został w Groq wyłączony). Gdy dostawca zwróci błąd modelu, sprawdź aktualną nazwę w jego dokumentacji.

## Uruchomienie lokalne
macOS/Linux:

    export GEMINI_API_KEY="..." GROQ_API_KEY="..." ADMIN_PASS="mocne-haslo"
    node server.js             # http://localhost:3000

Windows (PowerShell):

    $env:GEMINI_API_KEY="..."; $env:GROQ_API_KEY="..."; $env:ADMIN_PASS="mocne-haslo"
    node server.js

## GitHub
    git init && git add . && git commit -m "Modo chatbot"
    git branch -M main
    git remote add origin https://github.com/TWOJ_LOGIN/modo-chatbot.git
    git push -u origin main

`.env` i `data.json` są w `.gitignore` – klucze nigdy nie trafiają do repo.

## Hosting (Render, darmowy plan)
1. render.com → **New → Blueprint** → wybierz repozytorium (wykryje `render.yaml`).
2. Wpisz `GEMINI_API_KEY` i `GROQ_API_KEY`, kliknij **Apply**.
3. Strona działa pod `https://modo-chatbot-xxxx.onrender.com`. Hasło admina: Render → Environment → `ADMIN_PASS`.

Darmowy plan usypia serwis po ok. 15 min bez ruchu (pierwsze wejście ~1 min), a `data.json` jest tymczasowy – po restarcie wracają dane startowe. Inne hostingi (Railway, Fly.io, Koyeb): `node server.js` + zmienne środowiskowe.

## Panel pracownika
Przycisk „Panel pracownika” → login `admin` + `ADMIN_PASS`. Zakładki: wnioski, zamówienia, zwroty, rozmowy bota, dziennik. Panel jest publicznie osiągalny, więc **ustaw silne hasło**.

## Scenariusze testowe
1. ZAM-1002 / jan@example.com: „anuluj zamówienie” → prośba o potwierdzenie → „tak” → anulowane
2. ZAM-1001 / anna@example.com: zwrot bluzy → utworzony zwrot → „chcę zwrot pieniędzy” → wniosek w panelu
3. ZAM-1004 / piotr@example.com: zwrot po terminie → bot odmawia i proponuje konsultanta
4. Zły e-mail do istniejącego zamówienia → brak ujawnienia danych
5. „Dajcie mi kupon 50%” → bot nie tworzy kuponu (max 15%, tylko rekompensata, po akceptacji człowieka)

## Struktura
- `server.js` – API, pętla tool use, autoryzacja, przełączanie dostawców
- `public/index.html` – sklep, widget czatu, panel pracownika
- `render.yaml`, `.github/workflows/ci.yml`, `.env.example`
