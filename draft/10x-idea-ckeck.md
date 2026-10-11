# 
/10x-idea-check I want to create a tool to support managing my finances. I could drop csv file
  with my monthly expenses from my banking app and this application would classify each record of expense. After the
  cassification made by external AI API, I could correct particular records and save it.  Then, these records with
  its categries would be stored and will be available to present in my web app.
  Personally, I am a programmer with commercial experience for 5 years and I have 5 hours weekly and I want to pass
  this project in second deadline.

1. Chcę się nauczyć dobrej współpracy z agentem AI. Mam małe doświadczenie w pracy z AI w
  projekcie od samego początku. Natomiast myślę, że na podstawie dobrze przygotowanego PRD oraz CLAUDE.md koordynuję
  prace agenta i wyniki sprawdzam testami. Gdy poprawka nie działa proszę agenta o wyjaśnienie problemu celem
  lepszego zrozumienia problemu.
2. CSV będzie w jednolitym formacie z jednego banku. Poszczególne kolumny zawierają daty, sumę PLN, tytuł, opis,
  adres. Nie mam zamiaru przygotować uniwersalnego narzędzia wspierającego wiele formatów CSV. Zależy mi na jednem
  znanym mi formacie z mojego banku.

wynik:

Rekomendacja: przejdź do sesji planistycznej (/10x-shape) z tym pomysłem, z jednym dodatkiem opisanym niżej.

Dlaczego. Masz 5 lat doświadczenia, jeden znany format CSV i szkielet projektu (Astro, Supabase, AI SDK), więc dojście do pierwszego działającego przepływu jest realne. Twój sposób pracy z agentem (PRD, CLAUDE.md, testy, proszenie o wyjaśnienie błędu) to rozsądny punkt wyjścia. Doświadczenie z agentami od zera w projekcie jest jeszcze małe, więc ten projekt dobrze to ćwiczy.

Główne ryzyko to pusty CRUD. Opis "wgraj CSV, AI klasyfikuje, popraw, zapisz, pokaż" zawiera jedną regułę domenową, czyli klasyfikację. Przydałaby się reguła, która sprawia, że poprawki mają znaczenie. Na przykład po Twojej korekcie kategorii kolejne podobne transakcje (ten sam kontrahent) dostają ją automatycznie, bez wywołania AI. Zdecydujesz o tym w sesji planistycznej. Zostaje to Twoja decyzja, nie wymóg.
Pierwszy działający przepływ. Wgrywasz CSV, parser czyta Twój format, API zwraca kategorie, a Ty widzisz tabelę. Najpierw sprawdź jedną rzecz: wyślij kilkanaście prawdziwych opisów transakcji do modelu i zobacz, czy kategorie są sensowne. To najmniejsza kontrola największej niepewności. Prywatne dane finansowe możesz wcześniej zanonimizować.

MVP. Pierwszy działający przepływ plus korekta i zapis w Supabase, a potem lista z filtrem po miesiącu i kategorii. Pominąłbym wykresy, budżety i alerty. Katalog kursu wymienia "aplikację do zarządzania finansami" jako wzorcowy przykład rozrośniętego MVP.

Termin 6 grudnia 2026. Zostało około 8,5 tygodnia, czyli mniej więcej 40–45 godzin. To tylko pojemność, nie wycena. Trzeba z niej odjąć logowanie (Supabase Auth), testy, dokumentację i wdrożenie na Cloudflare. Jeśli do połowy listopada nie będzie działał zapis poprawek, rozważ termin 3 (10 stycznia 2027). Zakładam strefę Europe/Warsaw, więc przy samym terminie potwierdź ją u prowadzących.

Zgodność z wymaganiami Buildera. Ścieżka jest wiarygodna:
- Logowanie: Supabase Auth.
- CRUD: transakcje, kategorie, poprawki.
- Logika: klasyfikacja AI i ewentualnie reguła z poprawek.
- Dokumenty: PRD z /10x-shape i /10x-prd.
- Test: jeden test z perspektywy użytkownika, na przykład wgranie CSV i poprawienie kategorii.

Ostateczną decyzję podejmują recenzenci kursu.

Następny krok. Uruchom:

/10x-shape Narzędzie do zarządzania własnymi finansami: wgrywam miesięczny CSV z jednego banku (kolumny: data, kwota PLN, tytuł, opis, adres), zewnętrzne AI klasyfikuje wydatki, ja poprawiam kategorie i zapisuję, a dane widzę w aplikacji webowej. Cel: nauka współpracy z agentem, termin 2 (6 grudnia 2026), 5 h tygodniowo. Pytanie otwarte: czy poprawki mają wpływać na kolejne klasyfikacje (to moja sugestia, jeszcze nie decyzja).





### 10x-shape - example
US-01's acceptance criteria are saved. Now I'll run the challenge round—14 questions across four batches, each presenting the strongest argument against a given FR, with "it stands" as the final option.

● User answered Claude's questions:
· FR-001: Admin can create a user account. What's the strongest case against it? → No counter-argument; it stands
· FR-002: User can log in with login and password. What's the strongest case against it? → Login vs email unclear
· FR-003: User can log out. What's the strongest case against it? → No counter-argument; it stands
· FR-004: User can change their own password. What's the strongest case against it? → Not needed for a family MVP

Two answers need a decision on what to do about them:

● User answered Claude's questions:
· FR-002 resolution: is the login a username or an email address? → Email address
· FR-004 resolution: you said changing passwords isn't needed for a family MVP. What should happen to FR-004? → Demote to nice-to-have
· FR-005: User can manage their own category list (starts from defaults). What's the strongest case against it? → Removing breaks old sets
· FR-006 (nice-to-have): rules like "LIDL → Shopping". In Phase 1 you said rules + AI together is what existing tools lack. What's the strongest case against keeping rules out of the MVP? → Rules are the differentiator

I've logged FR-002 (email login), FR-004 (demoted), and FR-006 (promoted to must-have). Next up: deciding how to handle FR-005's outstanding issue.
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
←  ☐ FR-005 fix  ☐ FR-007  ☐ FR-008  ☐ FR-009  ✔ Submit  →

│ FR-005 resolution: when a user removes or renames a category that saved sets already use, what happens?

❯ 1. Rename propagates; remove blocked if used
     Renaming updates every record. A category can't be removed while any record uses it.
  2. Rename propagates; removed → '?'
     Renaming updates every record. Records in a removed category go back to '?'.
  3. Only add in MVP
     Users can add categories but not rename or remove them, which avoids the problem for now.
  4. Not sure / haven't decided
     Keep FR-005 and record it as an open question.
  5. Type something.
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  6. Chat about this

Enter to select · Tab/Arrow keys to navigate · Esc to cancel