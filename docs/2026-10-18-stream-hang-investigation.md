# Dochodzenie: sporadyczne zawieszenie aplikacji (2026-10-18)

Objaw zgłoszony przez użytkownika: aplikacja czasem się zawiesza i trzeba ją zrestartować, żeby wrócić do streamu.
Zwykle zdarza się raz na początku, potem działa bez problemu, czasem zawiesza się drugi raz.

Status: **przyczyna nie została udowodniona.** Poniżej tylko to, co wynika z logów i z kodu, z oceną pewności.

## 1. Co pokazują dostępne logi (`tools/moonlight-tv-logs`)

- Najnowszy log pochodzi z 2026-08-02, czyli sprzed wszystkich scaleń z upstream. Nie ma logu z opisanego zawieszenia.
- Wszystkie 7 zarejestrowanych sesji wystartowało poprawnie: od `Initializing platform` do `Started` mija 1,0-2,5 s.
- Sesje z 2026-07-29 (45 min) i 2026-08-02 (95 min) kończą się tak samo: pakiety wideo na okno 10 s spadają
  (z ok. 31000 do ok. 20000), potem cisza, pakiety audio też zanikają, potem `Failed to send ENet control packet`
  i rozłączenie ENet. W logu z 2026-08-02 widać to jako `VideoTransportStall noPacketMs=1037`, potem `6037`, oraz
  serię 31 komunikatów `Control message took over 10 ms to send`. Log z 2026-07-29 powstał jeszcze bez heartbeatu
  stall, ale ostatnie okno audio ma 67 pakietów zamiast ok. 780. W obu cleanup klienta kończy się w całości
  (ok. 160 ms). To wygląda na utratę łączności TV (Wi-Fi/sieć), a nie na zawieszenie kodu.
- Sesje z 2026-07-28 (profil 9.0/jQuery 4) urywają się bez cleanupu. Tam log po prostu przestaje powstawać.
- Pierwsze okno `AudioTransportStats` bywa bardzo długie (22-40 s, 1-250 pakietów). To artefakt raportowania: okno
  zamyka się dopiero po nadejściu pakietu, a host nie wysyła audio w ciszy. Nie jest to awaria klienta.

## 2. Dlaczego logowanie nie pokazuje przyczyny

- Zapis logów robi **wyłącznie wątek główny**: `diagnostic_logs.js:177` (`setInterval(pollNative, 1000)`), potem
  `Module.drainDiagnosticLogs()` i zapis przez `tizen.filesystem`. Natywny bufor (256 KB) zbierają wątki robocze,
  ale nikt go nie opróżnia, gdy wątek główny stoi.
- Skutek: zawieszenie wątku głównego i śmierć całego procesu dają w logu ten sam obraz, czyli nagłe urwanie.
  Wniosek z 2026-07-28 „zatrzymał się cały runtime” opierał się więc na logach, które w obu wariantach musiały się urwać.
- `beginSession` (`diagnostic_logs.js:260`) kopiuje bieżący log do „previous” przy każdym nowym streamie. Log
  zawieszonej sesji trzeba wyeksportować zaraz po restarcie aplikacji, przed kolejnym startem streamu.

## 3. Znaleziska w kodzie (od najlepiej pasującego do objawu)

### A. Jedna kolejka HTTP dla wszystkiego, w tym dla okładek (pewność mechanizmu: wysoka; że to Twoja przyczyna: nieznana)

- `wasm/dispatcher/lib.hpp:82`: `Dispatcher` ma jeden wątek (`std::thread thread_`). Przez niego idą wszystkie
  `OpenUrl`, `Pair`, `STUN`, `WakeOnLan` (`http.cpp:192`, `main.cpp:608/643/657`), w kolejności FIFO.
- `index.js:2641-2648`: po wczytaniu listy aplikacji każda aplikacja bez okładki w cache zleca osobne pobranie
  (`utils.js:589`, limit 10 s). Brak limitu współbieżności i brak anulowania. Komentarz autorów w tym miejscu
  sam opisuje skutek: `/serverinfo` stoi „behind 40+ concurrent image downloads”.
- `index.js:2970`: `startGame` zaczyna od `host.refreshServerInfo()`, a spinner (`showStreamMode`, `index.js:3080`)
  pojawia się dopiero po jego zakończeniu. Potem `resumeApp/launchApp` (`utils.js:892/901`) też idą przez tę kolejkę.
- W tym forku limit czasu żądania liczy się od jego rozpoczęcia, więc żądanie w kolejce nie wygasa, tylko czeka.
- Pasuje do wzorca „raz na początku, potem dobrze”: okładki zapisują się na dysk (`utils.js:599`), więc zimny cache
  występuje tylko po instalacji, nowym hoście lub dodaniu aplikacji, a każdy restart przynosi postęp w cache.
- Czego nie wiemy: ile żądań stoi w kolejce na TV i jak długo trwa. Nie ma logu czasu oczekiwania w kolejce.

### B. Ścieżki, które zostawiają stan „zablokowany do restartu” (pewność: wysoka, prawdopodobieństwo wystąpienia: niskie)

- `main.cpp:134-141` i `156-163`: `StopThreadFunc` wychodzi wcześnie przy błędzie `Join`, bez `onConnectionStopped` i bez
  `MarkIdle()`. JS nigdy nie dostaje `streamTerminated`, a `ConnectionLifecycle` zostaje w `Stopping`.
- `main.cpp:354-360`: przy błędzie `m_StopThread.Join()` wywoływane jest `BeginStop()` bez żadnego wątku zatrzymania,
  więc stan zostaje w `Stopping`.
- W obu przypadkach każdy kolejny `StartStream` zwraca `GS_WRONG_STATE`, aż do restartu aplikacji.

### C. Blokada statystyk trzymana podczas `AppendPacket` (pewność: wysoka, wpływ: niski)

- `wasmplayer.cpp:654-791`: `s_VideoStatsMutex` jest trzymany przez cały `AppendPacket` (linia 727).
- `TogglePerformanceStats` (`wasmplayer.cpp:994`) bierze tę blokadę na wątku głównym. Przy wolnym `AppendPacket`
  przełączenie overlayu zablokuje wątek główny.

### D. Ryzyka środowiskowe (niezweryfikowane)

- `CMakeLists.txt:269-283`: sztywna pamięć 128 MB (brak `ALLOW_MEMORY_GROWTH`), stos 16 MB, pula 20 wątków,
  `-DNDEBUG`, `-w`. Wyczerpanie pamięci lub wątków skończyłoby się cichym błędem, bez śladu w logu.
- `PostToJs` i `PostPromiseMessage` (`main.cpp:840-868`) są synchronicznymi wywołaniami wątku głównego (`MAIN_THREAD_EM_ASM`).
  Wątek roboczy, który je wywoła, stoi, dopóki wątek główny nie wykona JS. Nie znaleziono cyklu oczekiwania, ale
  każde zablokowanie wątku głównego zatrzymuje też wątki, które z niego korzystają.

### Sprawdzone i bez zarzutu

`OwnedPthread`, `OperationLatch`, `MediaOperationEpoch`, `DiagnosticLogBuffer::TryAppend` (try_lock), powiadamianie JS po
stronie `streamTerminated`, ścieżka błędu `showApps` (zawsze wywołuje `showAppsMode`, poza odrzuceniem dla `!host.paired`).

## 4. Co rozstrzygnęłoby przyczynę

1. Log zawieszonej sesji wyeksportowany zaraz po restarcie (Current session, przed kolejnym streamem).
2. Pomiar czasu oczekiwania żądań HTTP w kolejce (natywny log: czas od zlecenia do startu, dla żądań dłuższych niż 1 s).
3. Znacznik żywotności wątku głównego zapisywany co sekundę osobno od logów (np. do `localStorage`) oraz znacznik
   „nieczysty koniec poprzedniej sesji” przy starcie. Odróżni zawieszenie wątku głównego od śmierci procesu.
4. Natywny licznik żywotności wątków wideo i audio (log tylko przy anomalii), by odróżnić zatrzymanie renderera od
   zatrzymania transportu.

## 5. Odpowiedzi użytkownika i zmiana oceny

Odpowiedzi użytkownika po pierwszej wersji tej analizy:

- Co widać: zamrożony obraz, po chwili urwany dźwięk, brak reakcji na kontroler i brak możliwości powrotu do menu.
  Aplikację trzeba zamknąć przytrzymaniem Back na pilocie TV.
- Kiedy: po pierwszych minutach streamu.
- Jak często: prawie zawsze przy pierwszym streamie danego dnia, potem działa bez problemu.
- Sieć i profil: Wi-Fi, profil 5.5.

Co z tego wynika:

- To **nie jest zawieszenie na starcie**, więc kolejka okładek (punkt 3A) może być tylko czynnikiem pobocznym.
- Obraz zamarza od razu, a dźwięk urywa się dopiero po chwili, bo bufor audio w TV się opróżnia. To sygnatura
  **jednoczesnego zatrzymania wszystkiego, co dotyka wątku głównego lub potoku mediów**, a nie samej sieci.
- Utrata sieci kończy się w logach rozłączeniem ENet po ok. 8 s i powrotem do listy aplikacji. Tutaj nic takiego nie
  następuje, a lokalne wyjścia ze streamu (czerwony przycisk pilota i kombinacja na padzie) też nie działają.
  To wskazuje na zamrożony wątek główny albo cały proces, a nie na zerwane Wi-Fi.
- Korelacja z pierwszym streamem dnia: TV po rozruchu wykonuje w tle własne zadania (pamięć, CPU, Wi-Fi), a
  aplikacja raz na dobę sprawdza aktualizacje (`checkForAppUpdatesAtStartup`). Nie ma dowodu, że któreś z nich jest
  przyczyną. Dlatego zamiast kolejnych hipotez dodano narzędzia, które to rozstrzygną.

## 6. Co wdrożono (2026-10-18)

1. **Strażnik strumienia**: tętno zapisywane co sekundę, raport „poprzednie uruchomienie skończyło się podczas
   streamu" przy kolejnym starcie, wykrywanie przestojów wątku głównego.
2. **Zdrowie streamu** po stronie natywnej: liczniki pakietów wideo i audio, czas wywołań `AppendPacket`, czas rundy
   wątku wejścia (sonda responsywności wątku głównego), czas oczekiwania wątków roboczych na wątek główny, rozmiar sterty.
3. **Log kolejki HTTP**: żądania czekające w kolejce co najmniej 1 s lub trwające co najmniej 3 s (tylko ścieżka URL).
4. **Okładki**: najwyżej 2 pobierania naraz, porzucanie przestarzałych, anulowanie przy wejściu do streamu, natychmiastowa
   informacja „Starting …" po wciśnięciu aplikacji.
5. **Odporność cyklu życia**: zatrzymanie zawsze powiadamia interfejs i wraca do stanu bezczynności; nieudany start oddaje
   miejsce przez `AbortStart`.

Jak odczytać wynik po następnym zawieszeniu: sekcja „Diagnosing a freeze" w `docs/diagnostic-logs.md`.

## 7. Pozostałe hipotezy do rozstrzygnięcia po zebraniu danych

- Zablokowane wywołanie platformowego `AppendPacket` (potok mediów Samsunga): pokaże to `videoAppendBlockedMs`.
- Zablokowany lub zagłodzony wątek główny (pytania o gamepad są synchronicznie przekazywane do wątku głównego ok. 200
  razy na sekundę): pokaże to `inputPollMaxMs` i `inputPollBlockedMs`.
- Wyczerpanie sztywnej sterty 128 MB: pokaże to `heap`.
- Zabicie lub zamrożenie całej aplikacji przez TV: wszystkie liczniki zatrzymają się razem z tętnem.
- Jeśli wątek główny okaże się winny, kolejny krok to wyprowadzenie z wątków roboczych synchronicznych wywołań
  `PostToJs` i zapytań o gamepad.

## 8. Pytania (pierwotne, już odpowiedziane powyżej)

- Co dokładnie widać: zamrożony obraz z działającym dźwiękiem, czarny ekran ze spinnerem, nieruchoma lista aplikacji, brak reakcji pilota?
- Kiedy: zaraz po wciśnięciu aplikacji, w pierwszych minutach streamu, czy po wielu minutach?
- Czy dzieje się po świeżej instalacji lub po dodaniu aplikacji na hoście? Ile aplikacji ma host?
- Wi-Fi czy Ethernet, jaki model TV, który WGT jest zainstalowany (profil 5.5 czy 9.0)?

## 9. Regresja: start streamu wisi na „Starting …", okładki ładują się bardzo długo

Po wdrożeniu punktu 2 (build `freeze-diag`) aplikacja przestała uruchamiać streamy: Vibepollo dostawał `/launch`
i startował grę, a Moonlight zostawał na „Starting …". Okładki ładowały się znacznie wolniej niż wcześniej.

**Przyczyna (potwierdzona odtworzeniem):** `BlockingSection::End` aktualizowało maksimum pętlą
`compare_exchange_weak` na `std::atomic<int64_t>`. Samsungowy Emscripten (fastcomp 1.39.4.7) sprowadza 64-bitowe
atomiki do wywołań biblioteki, a wywołanie `___atomic_compare_exchange_8` ma tam błędną sygnaturę (ostrzeżenie
`asm2wasm adding operands in call … ___atomic_compare_exchange_8`, ukryte przez `-w` w buildzie). Pętla nigdy się
nie kończyła, więc wątek, który wszedł do sekcji, kręcił się w niej na zawsze. Dotyczyło to każdego wątku, który zmierzył
wywołanie dłuższe niż dotychczasowe maksimum: wątek połączenia przy pierwszym komunikacie „Starting connection to …"
(14 ms, bo wątek główny kończył jeszcze `StartStream`) oraz wątek HTTP przy odpowiedzi na żądanie, gdy wątek główny był
zajęty. Zablokowany wątek HTTP tłumaczy też wolne okładki.

**Ślad w logu:** `mainCalls=1`, `mainCallMaxMs=14` (to samo w każdym zrzucie, bo wątek w pętli stale ustawiał maksimum
na nowo po jego wyzerowaniu przez zrzut) i rosnące `mainCallBlockedMs`, a po ostatnim komunikacie startu brak
jakiejkolwiek linii natywnej (`RTSP port: …` nie pojawiło się), bo `LiStartConnection` nigdy nie został wywołany.

**Odtworzenie:** krótki program z tym samym toolchainem i flagami (`-pthread -Os -flto`), wątek roboczy wchodzący w
`ScopedSection` wokół `MAIN_THREAD_EM_ASM` trwającego 14 ms, uruchomiony w Chromium/Edge z nagłówkami COOP/COEP
(wątki wasm wymagają `SharedArrayBuffer`; node 12 z tego toolchainu nie obsługuje pthreadów tego Emscriptena). Stary
nagłówek: `mainCalls=1 mainCallMaxMs=14 mainCallBlockedMs` rośnie. Nowy nagłówek: 200 z 200 wywołań z dwóch wątków,
czasy 0–28 ms, kończy się bez blokady.

**Naprawa:** `stream_health.hpp` używa wyłącznie 32-bitowych atomików; czasy to dolne 32 bity zegara ms, odejmowane z
zawijaniem (poprawne dla przedziałów krótszych niż 24 dni). `tests/wasm_atomics_guard_tests.js` pilnuje, żeby w
`wasm/` nie pojawił się 64-bitowy `std::atomic` (wyjątek: `diagnostic_log.hpp`, tylko `fetch_add`/`exchange`) ani
`compare_exchange` na takim atomiku.

**Wniosek metodyczny:** błąd w kodzie diagnostycznym wyglądał jak zawieszenie mechanizmu proxy Emscriptena
(wątek roboczy w synchronicznym wywołaniu do wątku głównego). Rozstrzygnęło go dopiero odtworzenie w przeglądarce, a nie
analiza kodu źródłowego tego mechanizmu.

## 10. Zawieszenia po stronie sterty: forensyka na TV i test obciążeniowy (2026-10-19/20)

Zawieszenia długiego streamu nie zniknęły po naprawie z sekcji 9. Build `opt2b-O2` (ten sam wasm co `opt2-O2`) złapał
na TV trzy różne sygnatury. Narzędzia, którymi je złapano, są w `tools/tv-debug/` (opis w README tego katalogu):
watcher przez Chrome DevTools Protocol, walker łańcucha chunków dlmalloc wstrzykiwany do strony oraz kanał komend do
badania zamrożonego procesu.

**Złapane sygnatury.**

1. *19.10 14:07 UTC, 63 s po starcie streamu.* Wątek kontrolny ENet wywrócił się na `memory access out of bounds` w
   `free` (`enet_packet_destroy` ← `enet_protocol_remove_sent_reliable_command` ← `enet_host_service`). Wątek zginął z
   zajętą blokadą alokatora, reszta czekała w `_emscripten_futex_wait`.
2. *19.10 19:40 UTC, po 27 min.* Wątek odbioru wideo (`VideoReceiveThreadProc` → `RtpvAddPacket` → `queueRtpPacket` →
   `malloc`) wywrócił się w `malloc` 30 ms po „Sending RFI request for unrecoverable frame" i „VideoRecovery IDR frame
   assembled". Równocześnie wątek kontrolny ENet stał w `free` z tej samej ścieżki co w punkcie 1.
3. *19.10 20:20 UTC, po 23 min.* **Zawieszenie bez trapu.** Szczegóły niżej.

**Zawieszenie 3.**

- Blokada alokatora (zwykły `pthread_mutex`, ścieżka `malloc` → `__pthread_mutex_lock` → `__timedwait_cp` →
  `_emscripten_futex_wait`) miała `_m_lock = 0x8000000A` i `_m_waiters = 7`. W libc tego toolchainu `EBUSY` to
  `__WASI_ERRNO_BUSY` = 10, więc to zwyczajny stan „zajęty, są czekający"; szybka ścieżka nie zapisuje właściciela.
- Na tym samym adresie futeksa, z oczekiwaną wartością równą bieżącej, czekało sześć wątków: główny (w
  `getStreamHealth`), odbiór wideo, wysyłka wejścia ENet, wątek kontrolny (`free`), wątek wejścia kontrolera i wątek
  statystyk strat. Pozostałe wątki stały na zwykłych kolejkach i zdarzeniach, wątek odbioru audio pracował. Żaden nie
  był w alokatorze, czyli **nikt nie trzymał blokady**.
- Wszystkie 20 workerów jest rozliczone: 13 w użyciu, 7 bezczynnych. Bezczynny worker nie odpowiada na ewaluację,
  dopóki nie wykona JS, bo czeka na niego zakolejkowane `Debugger.pause`; timeout nie oznacza więc śmierci. Nie było
  zdarzeń `detached` ani wyjątków w konsolach workerów.
- Wyzerowanie słowa blokady i `_emscripten_futex_wake(adres, INT_MAX)` z wstrzymanego workera obudziło sześć wątków.
  Wątek główny ruszył od razu, workery oddały zaległe logi, aplikacja zgłosiła `streamTerminated: -1` (host zerwał
  sesję po minutach ciszy) i wróciła do listy gier. Całe zawieszenie to ta jedna zablokowana blokada.
- Łańcuch chunków: 4012 chunków z poprawnymi rozmiarami i jedna anomalia. Wolny chunk 456 B (w drzewie, ze stopką 456,
  z resztkami tekstu heartbeatu `getStreamHealth` w treści) ma następnika z ustawionym bitem `PINUSE`, którego `free`
  nie zostawia. Kontrole co 15 s były czyste do ok. 20:20:37, więc anomalia powstała w ostatnich sekundach przed
  zawieszeniem.

**Test obciążeniowy.** `wasm/alloc_stress.hpp` (logika, testowana na hoście w `tests/alloc_stress_tests.cpp`, razem z
dowodem, że wykrywa wstrzyknięty wyścig licznika i uszkodzony blok) i `wasm/alloc_stress.cpp` (eksport do JS). Wchodzi
do modułu tylko z `docker build --build-arg ALLOC_STRESS=ON`. Robi na wielu wątkach: `std::mutex` wokół niepodzielnego
odczytu i zapisu, licznik 32-bitowy atomowy, spin-lock na `exchange`, oraz `malloc`/`realloc`/`free` bloków w
rozmiarach z mieszanki streamu, wypełnionych wzorcem i sprawdzanych przed zwolnieniem; co czwarty blok zwalnia inny
wątek niż ten, który go zaalokował.

| Przebieg | Wynik |
|---|---|
| 12 wątków bez pauz | Po 28 s czysto, po 43 s aplikacja zniknęła. Cały TV zaczął się zacinać i trzeba go było zrestartować. Za ciężki, nic z niego nie wynika. Stąd pauzy w wątkach i automatyczne przerwanie, gdy główny wątek odpowiada wolniej niż 2,5 s. |
| 3 wątki, 120 s | 3,46 mln iteracji, 0 błędów, liczniki zgodne, łańcuch sterty spójny. |
| 4 wątki, 20 min | 79 mln iteracji (79 mln `malloc`, 79 mln `free`, 2,5 mln `realloc`), 19,7 mln zwolnień przez inny wątek, 0 błędów wzorców, liczniki mutexa, atomika i spin-locka zgodne co do jednej jednostki, najdłuższy bezruch wątku kilka ms, łańcuch sterty spójny. |

**Wnioski.**

- Mutex, 32-bitowe atomiki i `malloc`/`free` działają poprawnie pod dużym obciążeniem samych wątków roboczych (ok.
  160 mln operacji alokatora, w tym samym rzędzie co ekspozycja streamu do awarii). Sam toolchain lub platforma psująca
  blokadę jest przez to mało prawdopodobna.
- Test **nie obejmował** wątku głównego, na którym `futex_wait` to pętla obsługująca kolejkę wywołań proxy, a nie
  `Atomics.wait`, ani kodu aplikacji piszącego do pamięci.
- Sygnatura 3 (anomalia powstała tuż przy zawieszeniu, blokada bez właściciela) pasuje do wątku zatrzymanego w środku
  operacji alokatora. Którego i dlaczego, nie ustalono.

**Otwarte hipotezy i dalsze kroki.**

1. ~~Przepełnienie stosu wątku~~ **wykluczone analizą statyczną** (`tools/tv-debug/stack_frames.js`, bez TV i bez
   streamu). Stosy wątków to bloki po 2 MB z `malloc`, a stos rośnie w górę (wskaźnik stosu startuje poniżej
   `STACK_MAX`), więc przepełnienie uderzyłoby w nagłówek następnego chunka. Ale największa ramka w module (`-O2`) to
   `reed_solomon_reconstruct` 70 KB, najcięższy łańcuch bezpośrednich wywołań (wątki odbioru wideo i audio) to ok.
   75 KB, wśród wywołań bezpośrednich nie ma rekursji, a 35 funkcji z alokacją dynamiczną na stosie to dekodowanie Opus,
   `invert_mat` z FEC i kilka drobnych. Zapas względem 2 MB to ponad 25 razy. Nie śledzono wywołań przez wskaźniki do
   funkcji, ale żadna ramka spoza 30 największych (w tym żaden callback strumienia) nie przekracza 2,3 KB.
2. Wątek główny w grze o blokadę razem z emulacją futeksa i kolejką proxy: test obciążeniowy z wywołaniami proxy i
   `malloc`/`free` wołanymi z JS na wątku głównym, w trakcie obciążenia wątków roboczych.
3. ~~Zapisy z JS do pamięci modułu~~ **przejrzane, czyste.** W kodzie projektu (`wasm/`, `wasm/platform/`) jedynym
   zapisem z JS do pamięci modułu jest `__fillGamepadEventData` w `gamepad_sampling.hpp`: najwyżej 16 slotów, krok
   `sizeof(EmscriptenGamepadEvent)`, tablica 16 elementów w `gamepad.cpp`. Pozostałe miejsca (`audio.js`, `PostPromiseMessage`
   w `main.cpp`) tylko czytają. Kod Emscriptena (`library_html5.js`, `library_pthread.js` tego toolchainu) przejrzano:
   - `_fillGamepadEventData` **nie ogranicza** liczby osi i przycisków do rozmiaru struktury (pętle po
     `e.axes.length` i `e.buttons.length` zapisują `axis[i]`, `analogButton[i]` i `digitalButton[i]` bez sprawdzenia; tylko
     napisy `id` i `mapping` są ograniczone do 64 bajtów). Pad zgłaszający ponad 64 osi lub przycisków pisałby poza
     strukturę, a nasza tablica `events[16]` leży w pamięci statycznej, obok struktur alokatora. To ten sam kod co w
     upstreamie (ścieżka `emscripten_get_gamepad_status`), a zwykłe pady mają kilka osi i kilkanaście przycisków, więc
     nie ma dowodu, że to nasza przyczyna. Tani test: sprawdzić w logach `numAxes` i `numButtons` używanego pada.
   - `emscripten_futex_wait`/`wake`: emulacja czekania wątku głównego rejestruje adres przed sprawdzeniem wartości,
     przywraca poprzedni adres przy timeoucie i `EWOULDBLOCK`, a zagnieżdżone czekanie kończy się co najwyżej
     fałszywym obudzeniem. Nie znalazłem w niej scenariusza trwałej utraty obudzenia na blokadzie alokatora.
   - Zauważone przy zawieszeniu 3: `_m_waiters = 7` przy sześciu widocznych czekających, czyli jeden zgubiony
     licznik. Któryś wątek opuścił czekanie na blokadzie bez zmniejszenia licznika (np. wyjątek w środku czekania). To
     poszlaka o wcześniejszym nietypowym wyjściu wątku, ale bez śladu w logach.
4. Niezależnie od przyczyny: mniej alokacji na gorących ścieżkach (bufory pakietów wideo, pakiety ENet) wydłuża czas
   między awariami, a doraźny watchdog w wątku natywnym, który wykryje brak postępu i zwolni zablokowaną blokadę, jest
   wykonalny (ręczne zwolnienie przywróciło aplikację), choć nie wiadomo, czy sesja by to przeżyła. Wdrożony jako
   `wasm/platform/futex_guard.js` z łatką pętli futeksa wątku głównego (sekcja 11); na prawdziwym zawieszeniu jeszcze
   nie sprawdzony.

## 11. Trop z historii: profil Tizen 9 + jQuery 4 (2026-10-20)

Użytkownik przypomniał sobie, że po przejściu na „Tizen 9 only" i nowe jQuery freezy pojawiały się po rozpoczęciu
streama za każdym razem, więc wrócił do poprzedniego profilu. Historia to potwierdza i zawęża:

- `7eff49d` (`build: target Tizen 9 with modern frontend`, 25.07) zmienił w runtime tylko `required_version` z 5.5 na
  9.0 w `res/config.xml` i jQuery z 2.2.0 na 4.0.0. `CMakeLists.txt`, toolchain (Tizen Studio 6.1, Samsung Emscripten
  1.39.4.7) i wygenerowany moduł wasm pozostały bez zmian (`docs/2026-07-28-current-session-freeze-analysis.md`).
  Powrót to `ac1a228` (28.07, `build: restore legacy Tizen runtime profile`).
- Zawieszenia z 28.07 (po 380 s i po 826 s, log urwany w środku zapisu) pochodzą z profilu 9.0 + jQuery 4. Ten sam
  kod na profilu legacy przestreamował 37 minut bez crasha (`docs/2026-07-29-legacy-runtime-session-transport-
  analysis.md`), ale to jedna sesja, a freezy na profilu legacy też się zdarzają (sekcja 10).
- `b2aa50e` zauważa, że na części TV z Tizen 9 obie ścieżki EMSS się otwierają, ale callback `Play` nigdy się nie
  kończy: runtime zachowuje się tam inaczej.

Profile da się rozdzielić, bo wasm jest ten sam. Dockerfile ma do tego argumenty `REQUIRED_VERSION` (np. 9.0),
`JQUERY_VERSION` (np. 4.0.0) i `LOCK_GUARD` (domyślnie `off`: adres blokady 0, guard z sekcji 10 nieaktywny, więc zawieszenie jest widać bez
maskowania; `on` zapisuje prawdziwy adres). Zbudowano trzy warianty bieżących źródeł: `opt3-r90` (tylko 9.0), `opt3-jq4` (tylko jQuery 4) i
`opt3-r90-jq4` (oba). **Wyniki testu na TV: jeszcze brak.** Jeśli któryś zawiesza się powtarzalnie po starcie
streama, jest reproduktorem, na którym watcher z `tools/tv-debug/` pokaże, czy to ta sama zablokowana blokada
alokatora, i na którym da się sprawdzić guard.

## 12. Zawieszenie na buildzie 9.0 + jQuery 4 i uszkodzony bufor pakietu wideo (2026-10-20)

Wynik testu z sekcji 11 (`opt3-r90-jq4`, guard wyłączony, watcher z `tools/tv-debug/`):

- Pierwszy stream skończył się po 16 s rozłączeniem ENet od hosta (TV responsywny, bez zawieszenia). Drugi działał
  33,5 min (od 18:33:24 do 19:06:51 UTC) i się **zawiesił**. Sam profil 9.0 + jQuery 4 nie daje więc powtarzalnego
  freeza po starcie streama: ten sam typ zawieszenia występuje na profilu legacy (sekcja 10), a tu po kilkudziesięciu
  minutach. Brak błędów JS w logu. Samo `7eff49d` nie jest przyczyną.
- Sygnatura identyczna jak zawieszenie 3 z sekcji 10, bez trapu wasm: sześć wątków czeka w `malloc` (główny w
  `std::string::reserve`, kontrolny ENet w `enet_peer_queue_acknowledgement`, wejście kontrolera, wysyłka wejścia ENet,
  statystyki strat, odbiór wideo), pozostałe na zwykłych kolejkach, odbiór audio pracuje. Słowo blokady alokatora
  `0x8000000A` pod adresem 2666176, **`_m_waiters = 7` przy sześciu widocznych czekających**, tak samo jak za pierwszym
  razem.
- Zwolnienie blokady (`Atomics.store(0)` i `_emscripten_futex_wake`) obudziło sześć wątków i aplikację, ale host zdążył
  w międzyczasie zerwać sesję (po ok. 2,5 min ciszy, `streamTerminated: -1`).

**Tym razem sterta jest uszkodzona, i widać czym.** Skan łańcucha chunków zatrzymał się na chunku 1280 B pod adresem
21632104 (0x14A1468): jego nagłówek mówi „wolny", ale stopka się nie zgadza, a nagłówek następnego chunka to losowe
bajty (`head=2960565264`). W treści tego chunka, dokładnie tam, gdzie wolny chunk ma wskaźniki `fd`/`bk`, leży
**nagłówek pakietu RTP wideo** (`90 00 8c 38 8c 24 c2 0a 00 00 00 00 …`), a po nim ok. 1400 bajtów skompresowanych danych,
które wychodzą poza chunk i nadpisują nagłówek następnego. Pakiet wideo zapisany w chunku mniejszym niż bufor pakietu
(`StreamConfig.packetSize` 1392, bufor `1392 + 16 + sizeof(RTPV_QUEUE_ENTRY)` = 1448 B, czyli chunk 1456 B).

To przesuwa podejrzenie z alokatora na ścieżkę odbioru wideo: albo bufor pakietu został zwolniony i nadal był używany
(`VideoReceiveThreadProc` → `RtpvAddPacket` → kolejka FEC → depakietyzator), albo odbiór UDP zapisuje więcej niż
dostał. Odbiór idzie przez `wasm_recvfrom`, czyli natywne wiązanie hosta Samsunga (`tizentvwasm.SocketsHostBindings.
recvFrom`, `library_tizen_socket_host.js`), którego kodu nie ma w źródłach toolchainu. Przejrzano kod
`VideoStream.c` i `RtpVideoQueue.c` (`RtpvAddPacket`, `reconstructFrame`, własność bufora) i nie znaleziono w nim
oczywistego błędu własności: bufor przechodzi do kolejki tylko przy `RTPF_RET_QUEUED`, odrzucone pakiety są ponownie
używane. Nie wykluczono ścieżki FEC (bufor odzyskanego pakietu ma ten sam rozmiar) ani zbiegu z `notifyFrameLost`.

**Dalsze kroki, od najtańszego:**

1. Build diagnostyczny z kanarkiem za buforem pakietu wideo (`bufferSize + 16` z wzorcem na końcu), sprawdzanym tuż po
   `recvUdpSocket` i przy każdym `free` bufora pakietu. Rozstrzygnie, czy zapis wychodzi poza bufor od razu po odbiorze
   (wiązanie Samsunga), czy dopiero później (use-after-free w kolejce lub depakietyzatorze), i zaloguje wtedy rozmiar,
   numer sekwencyjny i ramkę.
2. Walker sterty rozszerzony o sprawdzanie wolnych chunków (wskaźniki `fd`/`bk` muszą wskazywać na chunk lub kosz),
   żeby takie uszkodzenie wykrywać w ciągu 15 s od zapisu, a nie dopiero przy zawieszeniu.
3. Guard z sekcji 11 jako zabezpieczenie niezależne od przyczyny, po sprawdzeniu na TV (`futex_guard_probe.js`).

**Kod `recvFrom` i sonda UDP (20.10).** `tizentvwasm.SocketsHostBindings.recvFrom` to `[native code]` także w inspektorze
działającej aplikacji, więc jego źródła nie da się przeczytać. Aplikacja woła `recvfrom(s, buffer, size, 0, NULL, NULL)`,
a opakowanie `wasm_recvfrom` robi `*addrlen` na wskaźniku `NULL`; po zapisie przez `NULL` nie ma śladu (pierwszy kilobajt
pamięci modułu to same zera). Zbudowano więc sondę (`wasm/udp_recv_probe.hpp`, build diagnostyczny
`opt3-diag`): odbiera jeden datagram tak jak wątek wideo, do bufora z wzorcem 256 B po obu stronach, a
`tools/tv-debug/udp_probe_run.js` wysyła z PC datagramy różnej długości.

**Wynik: odbiór UDP respektuje `len`.** W 20 z 20 przypadków (bufory 512 i 1408 B, datagramy 100…1472 B, krótsze, równe i
dłuższe od bufora, z adresem źródłowym i bez, na wątku roboczym) `recvFrom` zwrócił `min(len, rozmiar datagramu)`, nie
zmienił ani jednego bajtu poza buforem (ani przed nim) i oddał poprawne dane. Hipoteza „natywny odbiór pisze więcej,
niż dostał" nie znajduje potwierdzenia w tym teście. Zostaje podejrzenie o bufor zwolniony i nadal używany (kolejka
FEC, depakietyzator, `notifyFrameLost`), które ma rozstrzygnąć kanarek w `VideoStream.c` (też w `opt3-diag`, wymaga
streamu). Ograniczenie: sonda odbiera pojedyncze datagramy, nie 15 tys. pakietów na sekundę.


## 13. Zawieszenie na buildzie z kanarkiem pod dławieniem QoS (2026-10-20)

Warunki: `opt3-diag` (sonda UDP, kanarek 16 B za buforem pakietu wideo, guard wyłączony), PC dławi ruch do TV polityką
QoS (`tools/Start-QosThrottle.ps1`, 8 Mbit/s przy strumieniu ponad 14 Mbit/s, więc ciągłe straty i prośby o IDR/RFI).
Zawieszenie wystąpiło o 20:41:08 UTC, w trakcie streamu.

- **Kanarek milczy.** W logu aplikacji nie ma ani jednego `VideoCanary`: bajty za buforem pakietu były nietknięte
  zarówno tuż po `recvUdpSocket`, jak i przed ponownym użyciem bufora. Odbiór (natywne `recvFrom`) nie wychodzi poza
  bufor, co zgadza się z wynikiem sondy z sekcji 12.
- **Sygnatura blokady ta sama co poprzednio:** słowo blokady alokatora `0x8000000A` pod adresem 2682440, `_m_waiters = 7`,
  nikt nie jest w środku, wątki czekają w `malloc`.
- **Uszkodzony jest nagłówek chunka, a nie jego koniec.** Skan łańcucha zatrzymał się na żywym buforze pakietu wideo:
  nagłówek chunka ma `0x5B9` (rozmiar 1464, `PINUSE`, **bez** `CINUSE`), a powinien mieć `0x5C3` (1472 = 1464 B żądania z
  kanarkiem + 4 B nagłówka, zaokrąglone do 8, `PINUSE|CINUSE`). Ogon z kanarkiem w tym samym buforze jest cały. Nagłówek
  wygląda więc jak nagłówek wolnego chunka o rozmiarze równym rozmiarowi żądania, zapisany na miejsce nagłówka żywego
  chunka.
- W obu zarejestrowanych przypadkach (sekcja 12 i ta) w chunku bezpośrednio przed uszkodzonym nagłówkiem leży bufor
  tekstowy kończący się na granicy. To słaby trop: może to być zapis wstecz (underrun) albo zapis „ogona" bufora
  tekstowego o jeden rekord za daleko, ale z jednego przypadku więcej niż dopasowanie adresów nie wynika.
- Po zwolnieniu blokady (`Atomics.store(0)` + `_emscripten_futex_wake`, obudzone sześć wątków) aplikacja ruszyła, a po
  chwili (20:45:50) wątek roboczy rzucił `RuntimeError: memory access out of bounds`, i o 20:45:56 główny
  wątek znów przestał odpowiadać. Zwolnienie blokady nie leczy więc uszkodzonej sterty: guard z sekcji 11 daje co najwyżej
  kilka sekund życia, nie rozwiązanie.

**Co z tego wynika.** Odpada: przepełnienie przy odbiorze UDP (sonda, kanarek) oraz prymitywy blokady i alokatora (test
obciążeniowy). Zostaje zapis, który trafia w 4 bajty nagłówka żywego chunka w oknie między jego alokacją a zwolnieniem:
zapis wsteczny z poprzedniego chunka, stary wskaźnik na chunk o rozmiarze 1464 (ktoś nadal zapisuje metadane chunka,
który został już zwolniony i ponownie podzielony) albo pojedynczy zbłąkany zapis małej wartości.

**Zawartość uszkodzonego chunka (zrzut z zawieszenia).** Bufor zaczyna się nagłówkiem RTP (`90 00 0e e2 …`, numer 3810) i
nagłówkiem pakietu NV z flagami `0x05` (początek ramki i dane obrazu), czyli to pierwszy pakiet ramki. Bezpośrednio przed
nim, w czterech ostatnich bajtach poprzedniego chunka (miejsce, które po nagłówku chunka zajmuje stopka poprzednika), leży
koniec tekstu `...Unrecoverable frame 41997: 23+0=23 received < 31 needed` ze znakiem nowej linii, komunikatu z
`RtpVideoQueue.c:636`, czyli z tej samej ścieżki, która zwalnia bufory pakietów przy utracie ramki. Następny chunk, za kanarkiem, jest poprawnym wolnym chunkiem
4928 B z wskaźnikami drzewa wskazującymi na siebie. Stare słowo nagłówka różni się od nowego o dokładnie 10 (`0x5C3` →
`0x5B9`). Kod FEC, kolejki i depakietyzatora w `moonlight-common-c` jest bez zmian względem bazy (`git log` pokazuje tylko
dodane diagnostyki), a jego przegląd (`reconstructFrame`, `purgeListEntries`, `queueRtpPacket`, `queueFragment`,
`reed_solomon_reconstruct`) nie wskazał błędu własności ani indeksowania.

**Strażnik buforów pakietów (build diagnostyczny).** Zamiast jednego kanarka za buforem wątku odbioru, każdy bufor pakietu
(odbiorczy i odzyskany przez FEC) dostaje 16 bajtów za sobą z kopią słowa nagłówka chunka (bez bitu `PINUSE`, który zmienia
sąsiad), wpis w rejestrze żywych buforów i wzorzec `0xA5`. `videoCanarySweep()` sprawdza wszystkie żywe bufory co 32 pakiety
odbioru (`moonlight-common-c/src/VideoCanary.c`), `free` sprawdza bufor przed zwolnieniem, a bufor zwolniony drugi raz jest
zgłaszany i **nie** trafia ponownie do `free` (wynik sprawdza hipotezę podwójnego zwolnienia). Raport `VideoCanary: ...` podaje
adres, wiek bufora, stary i nowy nagłówek, nagłówek następnego chunka, 24 bajty przed nagłówkiem, początek bufora oraz
ostatnie zdarzenia kolejki (`purge`, `recover`, `submit`). Co 30 s jest linia stanu (`buffers registered`). Wszystkie bufory
przechodzą przez makra `PACKET_BUFFER_*` z `VideoCanary.h`, które w zwykłym buildzie są `malloc`/`free`.

**Jak czytać wynik:** raport „a registered buffer changed” z krótkim wiekiem i zdarzeniem tuż przed nim wskazuje, co robiła
kolejka w chwili uszkodzenia; „freed a buffer that is not registered” oznacza podwójne zwolnienie (`where` mówi którym
miejscem); brak jakiegokolwiek raportu przy zawieszeniu oznacza, że nagłówek psuje się po ostatnim przeglądzie i zawieszenie
następuje w ciągu ułamka sekundy, więc trzeba częstszych przeglądów albo innego pomiaru.

## 14. Stan na koniec 2026-10-20 i jak wznowić

**Ustalone dotąd (skrót sekcji 10-13).**
- Sygnatura zawieszenia jest ta sama za każdym razem (trzy razy): słowo blokady alokatora `0x8000000A`, `_m_waiters = 7`,
  nikt nie jest w środku, wątki czekają w `malloc`. Przyczyną jest uszkodzona sterta, nie blokada: po jej ręcznym
  zwolnieniu aplikacja żyje kilkadziesiąt sekund i wątek roboczy rzuca `memory access out of bounds`.
- Uszkodzony jest nagłówek chunka **żywego** bufora pakietu wideo (`0x5C3` → `0x5B9`, ogon z kanarkiem cały). Odpadły:
  stos, prymitywy blokady i alokatora (test obciążeniowy), zapisy z JS, przepełnienie przy `recvFrom` (sonda UDP 20/20 i
  kanarek). Profil Tizen 9 + jQuery 4 nie jest przyczyną (sekcja 12). Kod FEC, kolejki i depakietyzatora jest bez zmian
  względem bazy i po przeglądzie nie wskazuje błędu (sekcja 13).
- Zawieszenie da się wywołać na zawołanie przez straty pakietów (QoS 8 Mbit/s przy strumieniu ponad 14 Mbit/s).

**Gotowe i czeka na test na TV.** `artifacts/Moonlight-1.18.0-opt3-diag3.wgt` (commit `089245e`, obraz Dockera
`moonlight-tizen-diag3`, zbudowany z `--build-arg ALLOC_STRESS=ON`): strażnik wszystkich buforów pakietów wideo (sekcja
13). Zainstalowany na TV przez Apps2Samsung. Na koniec dnia aplikacja nadal działa na TV w trybie debug (port 36923),
bez podpiętego watchera: zamknąć ją przed następnym `debug`.

**Dlaczego dzisiaj nie doszło do testu: start streama pada z `-9`.** Od 21:12 UTC aplikacja po instalacji `diag3`
nie potrafi uruchomić ani wznowić gry: `/launch` i `/resume` kończą się limitem czasu po ok. 4 s (`CURL request completed:
Timeout was reached`, błąd `-9`), `/serverinfo` też bywa wolne (TLS 2650 ms, kolejka do 3,4 s). Fakty:
- Host to **Apollo** (`C:\Program Files\Apollo\config\logs\sunshine-*.log`), zrestartowany o 23:15 lokalnie przez
  `/api/restart` (po tym starcie ma ostrzeżenie „could not acquire the temporary virtual-display lease"). W jego logu nie ma
  ani jednej linii o `/launch` ani `/resume` z prób TV, są tylko odpytywania `serverinfo` (HTTP) co 5 s. Żądanie HTTPS z
  TV nie dochodzi więc do obsługi po stronie hosta.
- Z PC: HTTP `serverinfo` odpowiada w 91 ms, HTTPS od razu odrzuca bez certyfikatu klienta, ping do TV 2 ms bez strat.
- Podpięty debugger (watcher) spowalnia moduł (tryb debug wasm), więc najpierw podejrzewałem jego. Po odpięciu watchera
  użytkownik zgłosił, że jest „dalej to samo", więc **to nie sam debugger** (albo aplikacja nie wróciła do pełnej
  prędkości po odpięciu).
- Wolne żądania HTTP z TV były już w sesji 20:21 UTC, przed zawieszeniem (`dns 2641 ms`, `host 2864 ms`), a o 21:13:31 UTC
  `/cancel` przez HTTPS zadziałał (`tls 288 ms`). To nie jest więc nowość z `diag3`, ale `diag3` jest jedyną zmianą
  w aplikacji od tamtej sesji, której nie wykluczyłem.

**Plan na jutro, w tej kolejności (od najtańszego):**
1. Zamknąć aplikację na TV, zrestartować TV, uruchomić aplikację zwykłym sposobem (bez `debug`) i spróbować streamu.
2. Jeśli dalej `-9`: A/B z poprzednim `Moonlight-1.18.0-opt3-diag.wgt` (działał o 20:26 UTC). Jeśli on startuje, a `diag3` nie, błąd
   siedzi w nowym kodzie (mało prawdopodobne, bo HTTP/TLS go nie dotyka); jeśli oba padają, winne jest otoczenie (host
   Apollo po restarcie, wirtualny wyświetlacz, certyfikat klienta, TV).
3. Sprawdzić po stronie hosta: parowanie klienta `TV` w panelu Apollo, ostrzeżenie o dzierżawie wirtualnego wyświetlacza,
   logi przy próbie startu (czy `/launch` w ogóle się pojawia po restarcie TV).
4. Dopiero gdy stream startuje: **najpierw uruchomić stream, potem podłączyć watchera** (debug na starcie wydłuża
   żądania), uruchomić `tools/Start-QosThrottle.ps1 -TvAddress <adres TV> -Mbps 12 -Minutes 20` w oknie
   administratora **przed** zamrożeniem obrazu (UAC użytkownik klika w streamie Desktop), i czekać na zawieszenie.
   Przed startem uruchomić `tools/Stop-QosThrottle.ps1` (jako administrator), żeby wiedzieć, że nie została stara polityka.

**Komendy (z `tools/tv-debug/README.md`).** Instalacja: `tools/tv-debug/install_wgt_apps2samsung.ps1 -Wgt <plik>` (UI Automation do Apps2Samsung z katalogu Downloads;
użytkownik instaluje też ręcznie). Debug: `docker run --rm mt-sdk-probe bash -lc 'S=/home/
moonlight/tizen-studio/tools/sdb; $S connect <adres TV>; $S -s <adres TV>:26101 shell 0 debug MoonLightS.
MoonlightWasm'` (wypisuje port inspektora). Watcher: `node tools/tv-debug/forensics.js <host:port/devtools/page/ID> <log>
[plik poleceń]`. Odbiornik logów aplikacji: `tools/Receive-MoonlightLogs.ps1 -OutputDirectory tools/moonlight-tv-logs`
(port 48150).

**Jak czytać wynik testu z `diag3`** (rozwinięcie w sekcji 13): raport „a registered buffer changed" z krótkim wiekiem i
zdarzeniami kolejki tuż przed nim wskazuje, co robiła kolejka; „freed a buffer that is not registered" to podwójne
`free`; brak raportu przy zawieszeniu oznacza uszkodzenie szybsze niż co 32 pakiety (zagęścić przeglądy). Co 30 s w logu
jest linia stanu `VideoCanary: N buffers registered ...`, której brak oznacza, że strażnik nie działa.

**Otwarte wątki.** (a) Guard blokady (`LOCK_GUARD=on`) nadal niesprawdzony na TV (`tools/tv-debug/futex_guard_probe.js`),
a po ręcznym zwolnieniu blokady host i tak zrywa sesję po ok. 2 min. (b) Tryb cykliczny dławienia (np. 20 s ciasno, 20 s
swobodnie) do symulacji zrywów strat. (c) Jeśli strażnik nic nie złapie, następny krok to częstsze przeglądy albo
strażnik także dla innych małych alokacji w ścieżce wideo (kolejka decode unit, `std::vector` dekodera).

## 15. Zawieszenie z pułapką w `free()` pakietu ENet (2026-10-21)

Test: `opt3-diag3` (strażnik buforów pakietów wideo), dławienie QoS 12 Mbit/s od 08:17:52 lokalnie, stream Desktop,
watcher podłączony po starcie streama. Start streama działał (wcześniejsze `-9` zniknęły po zwykłym uruchomieniu
aplikacji). Zawieszenie po 6 minutach, 06:23:41 UTC. Pierwszy raz mamy wątek, który padł w środku alokatora.

**Co się stało.**
- Wątek kontrolny (`controlReceiveThreadFunc` → `serviceEnetHost` → `enet_host_service` →
  `enet_protocol_remove_sent_reliable_command` → `enet_packet_destroy` → `free`) rzucił `memory access out of bounds` w
  `free()` wskaźnika `0x14d93e0`. Trzymał blokadę alokatora, więc wątek główny (w `getStreamHealth`), odbiór wideo,
  wysyłka wejścia i statystyki strat stanęły na `malloc`. To wyjaśnia sygnaturę „blokada zablokowana, nikogo w środku": jej
  właściciel to wątek zatrzymany pułapką. Przy zawieszeniach 1-3 nie zarejestrowano pułapki, ale wniosek o uszkodzeniu
  sterty, po którym jakiś wątek pada w środku alokatora, pasuje do wszystkich.
- Strażnik buforów pakietów wideo **nie zgłosił nic** (0 raportów do chwili pułapki), więc to nie bufor pakietu wideo.
- Wskaźnik dany do `free` jest polem `outgoingCommand->packet` komendy ENet (kanał 3, numer 0xB9A0, 37 B, wysłana
  dwukrotnie), czyli oczekującym na potwierdzenie `ENetPacket`. Pod tym adresem leży jednak nagłówek **wolnego** chunka
  32 B (`flags` = 0x121 = 0x21 z dodanym przez ENet bitem `SENT`, `data`/`dataLength` = wskaźniki kosza, `referenceCount`
  zmniejszony z 1 do 0). Pakiet został więc zwolniony, a komenda nadal na niego wskazywała, albo alokator uznał jego
  pamięć za wolną, gdy pakiet żył.
- Region wokół ma układ, który tłumaczy wszystko, co zobaczyliśmy: wolny chunk 808 B pod `0x14d90d8` (z nagłówkiem
  `0x329` i stopką `0x328` w następniku) to suma trzech sąsiadów: bufor `std::string` ze snapshotem zdrowia (392 B), blok
  embind z jego kopią (384 B) i chunk pakietu ENet (32 B). 392 + 384 + 32 = 808. W treści widać tekst snapshotu
  (`controllerSkipped=7809 … heap=47MB/128MB`) w dwóch egzemplarzach, a metadane drzewa chunka 808 B leżą na początku
  pierwszego z nich. Ten sam układ (wolny chunk z resztkami tekstu `getStreamHealth` i następnik z błędnym `PINUSE`)
  opisuje sekcja 10 dla zawieszenia 3 (chunk 456 B).
- Sterta przed tym regionem (3923 chunki) i za nim (26 chunków) jest spójna. Uszkodzenie jest lokalne, rozmiaru ok.
  800 B, i dotyczy trzech obiektów zwalnianych w tym samym czasie przez dwa wątki: wątek główny
  (`std::string` i blok embind snapshotu) i wątek kontrolny (pakiet ENet).

**Hipotezy.**
1. Pakiet ENet został zwolniony przed potwierdzeniem (ścieżka ENet, `enet_packet_destroy` wywołane dwa razy albo za
   wcześnie). Wtedy jego chunk wchodzi w sklejenie z sąsiadami przy zwolnieniu bufora `std::string` i bloku embind, a
   późne potwierdzenie niszczy go po raz drugi.
2. Dwa wątki zwalniają sąsiednie chunki w tym samym momencie bez wzajemnego wykluczenia (np. wątek główny w pętli
   emulowanego `futex_wait`, która obsługuje kolejkę proxy w trakcie czekania na blokadę), a sklejanie na podstawie
   bitów `PINUSE` rozjeżdża się. Test obciążeniowy z sekcji 10 nie obejmował wątku głównego.
3. Zapis z zewnątrz w nagłówek chunka (jak w sekcjach 12 i 13). Przy zawieszeniu 4 nic na to nie wskazuje.

**Instrumentacja do sprawdzenia (build `diag4`).** Do `VideoCanary.c` dodane śledzenie alokacji ENet:
`enet_initialize_with_callbacks` z alokatorem, który wpisuje każdy blok ENet do rejestru (rozmiar, wątek, czas, słowo
nagłówka chunka), pamięta 512 ostatnich zwolnień z wątkiem, który zwolnił, zgłasza **drugie zwolnienie tego samego bloku**
razem z pierwszym (nie przekazuje go do `free`) i zgłasza blok żywy, którego nagłówek chunka się zmienił (przy `free` i w
przeglądzie co 32 pakiety wideo). Wątki Moonlighta mają nazwy w raportach (`PACKET_THREAD_NAME` w `ThreadProc`). Raporty
mają prefiks `ENetTrace:`, linia stanu co 30 s też. Build z nazwami funkcji: `--build-arg PROFILING_FUNCS=ON`
(`--profiling-funcs`, ta sama numeracja funkcji, kilka procent różnic w rozmiarach ciał), do czytania stosów.
Odczyt: „freed again by X; allocated by Y, lived N ms and was freed by Z" rozstrzyga hipotezę 1 (kto i kiedy zwolnił jako
pierwszy); „a block was damaged while it was in use" przy braku podwójnego zwolnienia wskazuje hipotezę 2 lub 3 (nagłówek
żywego bloku ENet zmieniony przez alokator lub z zewnątrz).

**Narzędzia.** Stosy z nazwami: `named_stacks.js` (w katalogu roboczym sesji) czyta sekcję nazw z modułu zbudowanego z
`PROFILING_FUNCS=ON`. Odczyt pamięci zatrzymanego procesu: plik poleceń watchera (`{"thread": "<prefiks workera>",
"expression": "..."}`) działa na zatrzymanym workerze, także po zawieszeniu.

## 16. Drugie zawieszenie z pułapką, tym razem w `sendMessageEnet` (2026-10-21)

Test: `opt3-diag4` (strażnik pakietów wideo, śledzenie bloków ENet z historią zwolnień, nazwy funkcji), dławienie QoS
12 Mbit/s od 08:49:55, stream Desktop, watcher podłączony przed dławieniem. Pułapka po 45 s od startu dławienia
(06:50:40 UTC), a sam stream i wideo szły dalej, więc to nie był pełny freeze, tylko śmierć wątku wejścia.

**Fakty.**
- Pułapka `memory access out of bounds` w wątku `InputSend`: `inputSendThreadProc` → `sendInputPacket` →
  `sendInputPacketOnControlStream` → `sendMessageEnet` (na wierzchu sama `sendMessageEnet`, bo funkcje ENet są w niej
  wstawione). Wątek trzymał `enetMutex`, więc wątek kontrolny przestał zwalniać pakiety 26-33 ms przed zgłoszeniem.
  Reszta aplikacji żyła, kolejka wejścia zapełniała się i log zalewały wpisy „Input queue reached maximum size limit" (to
  skutek śmierci wątku, nie przyczyna).
- Przegląd rejestru (wątek odbioru wideo, co 32 pakiety) zgłosił jednocześnie trzy żywe bloki ENet zaalokowane przez
  `InputSend` 24-32 ms wcześniej (37 B pod `0x301a330`, 84 B pod `0x301a360` i `0x301a2b8`), których słowa nagłówków
  chunków są śmieciami (`0x2841f24d`, `0x00010025`, `0`) zamiast `0x32`/`0x5a`. Podwójnego zwolnienia ENet nie było (0
  zgłoszeń).
- Zrzut pamięci obszaru: normalny ruch ENet to powtarzające się trójki alokowane od góry sterty, jedna na komunikat
  wejścia (pakiet 24 B w chunku 32, dane 37 B w chunku 48, komenda 84 B w chunku 88 lub 96), numery komunikatów
  0x5d76, 0x5d77, 0x5d78, ... Wskazane przez rejestr adresy **leżą wewnątrz innych, prawidłowo wyglądających obiektów**
  (np. `0x301a330` w środku danych komunikatu 0x5d78, `0x301a360` w środku komendy). Pamięć została więc przydzielona
  dwa razy, albo wpis rejestru jest przeterminowany. Zaraz za ostatnią komendą słowo nagłówka chunka na szczycie sterty
  (`top`) wynosi 1, czyli rozmiar zero (walker: `size below 16 at 0x301a398`).
- Ślady z `getStreamHealth` tym razem nie ma.

**Interpretacja.** Razem z sekcją 15 (pakiet ENet zwalniany w obszarze, który alokator uznał za wolny, i sklejone
sąsiednie chunki zwolnione przez dwa różne wątki w tej samej chwili) to najlepiej pasuje do **alokatora, który przydziela
lub scala bez wzajemnego wykluczenia wątków**: dwa wątki jednocześnie tną `top` (nakładające się przydziały i zerowy
rozmiar `top`) albo jednocześnie zwalniają sąsiednie chunki (sklejenie 808 B). Samo nakładanie się obu zdarzeń na wątek
wejścia (alokuje trójki) i wątek kontrolny (zwalnia je po potwierdzeniu) jest typowe dla wyścigu o mały rozmiar chunków
przy wysokim tempie. Czego jeszcze nie wiemy: czy blokada `pthread_mutex` alokatora nie wyklucza, czy któraś ścieżka
(np. `calloc`, `realloc`, `memalign`, wywołanie z JS) jej nie bierze.

**Instrumentacja do sprawdzenia (build `diag5`).** Do śledzenia ENet dodane: strefy ochronne po 16 B przed i za każdym
blokiem (przepełnienie bloku zmienia strefę), wypełnienie zwolnionego bloku wzorcem i kwarantanna na 128 zwolnień (zapis
po zwolnieniu wychodzi przy opuszczaniu kwarantanny i w przeglądzie), sprawdzanie przy każdym przydziale, czy zwrócona
pamięć nie nachodzi na blok używany albo trzymany („overlapping allocation: ... which is still in use"). Odczyt:
„overlapping allocation" potwierdza hipotezę alokatora (kolejny krok to własny `dlmalloc` ze sprawdzoną blokadą spinową
i licznikiem „kto jest w środku"); „a block was written outside its bounds" lub „after it was freed" wskazuje zapis z
kodu aplikacji; samo „a live block changed" z głową bloku bez nakładania i bez stref oznacza zapis w nagłówek z zewnątrz.

## 17. Błędy startu streama (`-9`) i zacinanie aplikacji: pełny plik logów (2026-10-21)

To wyjaśnia błędy startu z sekcji 14 i te po instalacji `diag6`/`nodiag`. **Nie miały związku z kanarkiem ani z
restartem TV**, a przetrwały restarty TV i reinstalacje.

**Objawy.** `/launch` kończył się po 4-20 s błędem `-9`, animacja „Starting Desktop" się zacinała, strona z logami się
zawieszała, wibracja pada po starcie aplikacji trwała za długo, a kolejne żądania HTTP czekały w kolejce po kilka
sekund (`ran for 19511 ms` przy limitach 8 s i 10 s: czas poszedł poza rachunek curl, bo wyjście wątków roboczych
przechodzi przez zablokowany główny wątek).

**Przyczyna.** Aplikacja trzyma log diagnostyczny w pliku `wgt-private/moonlight-diagnostics-current.log` (limit 2 MiB)
w pamięci prywatnej aplikacji (pliki sprzed 18.10, więc przeżywają instalacje). W przebiegach, w których wątek wejścia
padł albo stanął (`diag4`, `diag5`), biblioteka strumieniowa zalała log komunikatem „Input queue reached maximum size
limit" (ok. 80 tys. linii). Plik zapełnił się do limitu (2 097 115 B, 38 595 linii, z czego 37 897 to ten komunikat i 217
razy „Main thread stalled for N ms"). Odtąd **każda porcja logu (co sekundę) powodowała odczyt całego pliku (400 ms),
przycięcie go i zapis 2 MB od nowa, na głównym wątku**, bo przycięcie schodziło dokładnie do limitu i następna porcja
znów go przekraczała. Główny wątek był przez to zajęty przez większość czasu: opóźniał DNS (przechodzi przez niego),
wyjście konsoli wątków roboczych, animacje i stronę logów (która dodatkowo wczytywała cały plik jako jeden tekst).

**Sprawdzenie na TV.** `tools/tv-debug/page_eval.js` (wyrażenia wykonywane w stronie działającej aplikacji, bez
pauzowania) pokazał rozmiar plików i ich zawartość; po wyczyszczeniu obu plików aplikacja od razu działała poprawnie.

**Poprawka (`b7759ea`).**
- Ten sam komunikat powtórzony w ciągu sekundy jest pomijany i liczony („the previous message was repeated N times"),
  przed konsolą i przed logiem (`wasm/log_repeat_filter.hpp`).
- Pełny log jest przycinany do trzech czwartych limitu, więc kolejne porcje są tylko dopisywane.
- Strona logów pokazuje tylko ostatnie 128 KB i odświeża się nie częściej niż co 3 s.
- Przełącznik „Save diagnostic logs on this device" (domyślnie włączony, zapamiętany w `localStorage`) wyłącza zapis
  także po stronie natywnej, a przycisk „Clear all logs" czyści oba pliki oraz to, co trzyma strona natywna.

**Skutek uboczny dla śledztwa.** Komunikat o kolejce wejścia jest tylko skutkiem śmierci wątku wejścia (sekcja 16), nie
usterką wejścia, i nic w kodzie wejścia nie było do cofnięcia. Limit połączenia 8 s dla `launch`/`resume`/`cancel` z
`17cee3a` zostaje jako dodatkowe zabezpieczenie, ale nie był przyczyną.

## 18. Wynik `diag7` i `diag8`: blokadę alokatora zgubił główny wątek (2026-10-21)

**`diag7`.** Trap z kontroli alokatora był winą mojego trackera, nie sterty: rejestr ENet (2048 pozycji) przepełnił się
przy 2013 blokach naraz i `videoCanaryEnetFree` zwolnił wskaźnik użytkownika zamiast bazy bloku. Stan sterty po
awarii był spójny. Poprawka w `503f36a` (rejestr 16384 pozycji, mapa pokrycia pamięci, blok poza rejestrem dostaje
zwykły `malloc` bez stref).

**`diag8`, zawieszenie 3 min po włączeniu dławienia 12 Mbps (watcher podpięty przed startem).** Zrzut 14 wątków:
- Nikt nie jest w alokatorze, nie było trapa ani wyjątku, nie padła żadna kontrola `dlmalloc:`. Pięć wątków czeka w
  `malloc` → `pthread_mutex_lock` (główny w `DiagnosticLogBuffer::Drain`, wysyłka wejścia ENet, mysz, odbiór wideo,
  statystyki strat), szósty w mutexie ENet.
- Słowo blokady `gm->mutex` to `0x8000000A`, `_m_waiters = 6` przy pięciu widocznych oczekujących (znów jeden
  „fantomowy"; poprzednio 7 przy sześciu). Wszystkie inne mutexy (także `malloc_global_mutex`) są wolne.
- **Rekord właściciela z diagnostycznego dlmalloc (`diag_owner`, adres z wasm przez `find_owner.js`) wskazuje główny
  wątek** (struktura pthread w pamięci statycznej, 0x56DEE0; workery mają adresy ~24 mln). Główny wątek wziął blokadę
  jako ostatni i nigdy jej nie zwolnił, a w jego stosie nie ma alokatora. Nie padł ani jeden komunikat „thread X got the
  lock while thread Y is still inside", więc dwa wątki nie weszły do sekcji krytycznej naraz.
- Sterta ma wolny chunk 216 B (w bin 27, stopka 0x20 zamiast 216) leżący na pamięci, w której są trzy poprawne chunki
  (120 B komenda ENet, 64 B pakiet ENet, 32 B wolny). Wygląda to na operację alokatora przerwaną w połowie, a nie na
  wyścig.
- Wniosek roboczy: główny wątek opuścił `malloc`/`free` przez wyjątek (trap albo błąd JS, np. `memory access out of
  bounds` przy zepsutej stercie), który złapał kod wywołujący moduł (np. `try/catch` w `pollNative`). To wyjaśnia
  zgubioną blokadę, uszkodzoną stertę i fantomowego oczekującego (wyjątek przeszedł przez `__pthread_mutex_timedlock`
  z pominięciem `a_dec(&m->_m_waiters)`, bo główny wątek w pętli czekania wykonuje zakolejkowane wywołania innych
  wątków). Ostatnie nie jest jeszcze potwierdzone.

**Zawieszenie tuż po starcie streamu.** Użytkownik zgłosił, że aplikacja potrafiła się zawiesić zaraz po starcie także
wcześniej i bez dławienia (pierwszy test `diag8`: Apollo zakończył sesję po 5 s). Nie udało się wtedy nic odczytać z
zawieszonego renderera (nie odpowiadał nawet `Debugger.pause`, a sesja strony przechodzi przez renderer).

**Błąd w moich narzędziach.** Pauza głównego wątku na wszystkich wyjątkach (`FORENSICS_MAIN_EXCEPTIONS=1`) zatrzymuje go
także na odrzuconej obietnicy i w środku `_mediaElementSetSrc`; po niej EMSS zawiesza źródło (`kSourceSuspended`) i
obraz jest czarny (video/audio appendCalls=0). Pauzowanie wątków w trakcie startu odtwarzacza nie nadaje się do
pomiaru.

**Dalej (`diag9`, commit `9ff7c2a`).** Zamiast pauzy rejestrator w stronie (`wasm/platform/wasm_exception_recorder.js`)
owija eksporty modułu na głównym wątku, a przy wyjątku pyta moduł, czy wątek nadal jest zapisanym właścicielem blokady
(`diag_dlmalloc_owner_is_calling_thread`). Raport trafia do konsoli (`[wasm-exception] ...`), bez zatrzymywania wątku.

## 19. `diag9`: alokator zwraca pamięć nachodzącą na żywe bloki, rozjazd o 8 bajtów (2026-10-21)

Test: `diag9` w debugu, watcher bez pauzowania wątków (obraz działa), dławienie 12 Mbps. Zawieszenie ok. 2 min po
włączeniu dławienia, po raz pierwszy z pełnym zapisem kolejności zdarzeń (czasy UTC, 11:04:55):

1. `.291` tracker: żywy blok ENet 36 B (`0x32aa710`, wątek LossStats, 451 ms wcześniej) ma nadpisaną **czerwoną strefę
   przed sobą**; potem jeszcze kilka razy (raporty 2-4).
2. `.487` tracker: **`overlapping allocation`**: alokator zwrócił blok 84 B (`0x32aa668`, InputSend) nachodzący na żywy
   blok 84 B (`0x32aa698`, InputSend, 668 ms wcześniej).
3. `.525` wątek `ControlRecv` (`controlReceiveThreadFunc` → `serviceEnetHost` → `enet_protocol_send_outgoing_commands`
   → `enet_socket_send` → `free`) rzuca `memory access out of bounds`: w `free` kolejny chunk ma „nagłówek"
   `0xa5a5a5a5`, czyli wzorzec czerwonej strefy trackera. Wątek trzyma blokadę, reszta staje.

**Układ pamięci** (zrzut 0x32aa600-0x32aa800 z wątku po trapie): chunk `0x32aa650` (120 B, blok 84 B z drugiego
zdarzenia) i chunk `0x32aa6c8` (56 B, tymczasowy bufor 52 B z `enet_socket_send`) zajmują pamięć, którą tracker przypisuje
żywemu blokowi 120 B w `0x32aa680` i blokowi 72 B w `0x32aa6f8`. Wolny chunk, z którego alokator wydał te dwa chunki,
miał zapisany rozmiar 176 B, a jego koniec wypadał o 8 B za rzeczywistym początkiem następnego chunka (`0x32aa700`
zamiast `0x32aa6f8`). To jest „+8 / nagłówek wolnego chunka rozjechany" z poprzednich zawieszeń; wyścigu dwóch wątków
w alokatorze nadal nie widać (żadnego „got the lock while thread X is still inside").

**Wniosek.** Awaria zaczyna się od wolnego chunka z błędnym rozmiarem; reszta (strefy, nachodzące bloki, trap w `free`)
to skutki. Poprzednia kopia dlmalloc tego nie wykrywała, bo `DEBUG` był wyłączony i `check_*` były pustymi makrami
(włączone było tylko `ABORT` dla `RTCHECK`).

**Rejestrator wyjątków (`9ff7c2a`).** W `diag9` nie zadziałał: eksporty modułu mają podkreślnik (`_malloc`,
`_diag_dlmalloc_owner_is_calling_thread`), a rejestrator szukał nazwy bez niego, więc nic nie owinął. Naprawione w
`a631628`; zawieszenie `diag9` było i tak typu „trap w `free` na wątku roboczym", więc główny wątek nie był winny.

**Dalej (`diag10`, `a631628`).** `DEBUG 1` w kopii dlmalloc i co 64. wejście pod blokadą `do_check_malloc_state(gm)`
(wszystkie chunki, biny, drzewa) z zapisem ostatnich 256 wejść (numer, wątek, linia w dlmalloc). Pierwszy rozjazd
metadanych zatrzyma wątek i wypisze 64 ostatnie wejścia. Test na hoście (`dl_harness.c`: 4 wątki, 2,4 mln wejść) bez
fałszywych alarmów; celowo zwiększony o 8 nagłówek wolnego chunka jest wykrywany.

## 20. Przyczyna: odczyt `std::string` przez Embind zapisuje bajt za buforem (2026-10-21)

**Droga do wyniku (`diag10`-`diag15`).** Kopia dlmalloc z `DEBUG 1` i walidacją całego stanu co 16. wejście (`diag10`,
`diag11`: okno 16 wpisów z argumentami `malloc(n)`/`free(ptr)`) pokazała, że asercja potrafi zawieść, a chwilę później
stan jest spójny (`diag12`: brak zmiany stanu `gm` bez blokady, brak współbieżnego wejścia). `diag13` dodał drugi
przebieg walidacji po 20000 barierach pamięci i pełne bariery wokół blokady: pierwszy przebieg nadal widział na chwilę
zły rozmiar chunka `top` (`do_check_top_chunk`, `sz == m->topsize`), drugi przechodził. Pełne bariery nie pomogły. Test
na TV (4 workery, blokada na `Atomics.compareExchange`, 6,2 mln przejęć, 3 zwykłe słowa w różnych liniach pamięci
podręcznej) nie dał ani jednej niezgodności, więc kolejność zapisów między rdzeniami działa. `diag14` złapał prawdziwe
zawieszenie: wątek główny w `free` dostał `memory access out of bounds` z zajętą blokadą, bo następny chunk miał w
nagłówku `0xf9` („wolny, 248 B") zamiast `0x4b` („zajęty, 72 B"), a jego dane to żywy tekst loga. `free` próbował
scalić go do przodu i rozwinął wskaźniki z tekstu. `diag15` dodał przejście całego łańcucha chunków co drugie wejście
i złapał to samo w locie: za chunkiem z napisem stanu strumienia (372 B) nagłówek następnego chunka miał na chwilę
wartość `0` (po kilku ms prawidłowe `0x43`).

**Przyczyna.** `fromWireType` dla `std::string` w kodzie, który generuje Samsung Emscripten 1.39.4.7:

```
var endChar=HEAPU8[value+4+length];var endCharSwap=0;
if(endChar!=0){endCharSwap=endChar;HEAPU8[value+4+length]=0}
... UTF8ToString(decodeStartPtr) ...
if(endCharSwap!=0)HEAPU8[value+4+length]=endCharSwap
```

Bufor, który buduje strona C++ (`toWireType` w `wire.h`), to `malloc(4 + length)` bez terminatora. Odczyt wpisuje więc
zero w bajt **za** buforem i po dekodowaniu przywraca stary bajt, nie trzymając blokady alokatora. Gdy `length` dzieli się
przez 8 (napis stanu strumienia ma 368 B), ten bajt to najmłodszy bajt nagłówka następnego chunka sterty. Jeśli w tym
czasie inny wątek zmieni ten nagłówek (malloc dzielący chunk, free scalający), przywrócenie starego bajtu cofa jego zmianę:
nagłówek jest uszkodzony, a najbliższy `free` sąsiada wchodzi w śmieci. `getStreamHealth` i podobne funkcje wołane z JS
na wątku głównym wołają to co sekundę. Dławienie QoS zwiększa ruch alokatora (retransmisje, komunikaty kontrolne ENet),
więc okno wyścigu jest trafiane w minutach.

To tłumaczy wszystkie wcześniejsze ślady: właściciel blokady = wątek główny, który zginął w `free` (`diag8`, sekcja 18),
nagłówek wolnego chunka o 8 B za duży i nachodzące bloki (`diag9`, sekcja 19), para `{długość, wskaźnik}` w nagłówku
(`diag10`), chwilowe zero w nagłówku (`diag13`, `diag15`) i `0xf9` zamiast `0x4b` (`diag14`). Do tego dochodzi pułapka
diagnostyczna, która sama trzyma blokadę po wykryciu błędu: zawieszenie „po trapie" w `diag13` i `diag15` to skutek
wykrycia, nie osobna przyczyna.

**Poprawka.** `tools/patch-embind-string.js` (krok Dockera po linkowaniu, zaraz po `patch-futex-wait.js`; test
`tests/embind_string_patch_tests.js`) zamienia gałąź UTF-8 na odczyt, który nic nie zapisuje: segmenty kończą się na
bajcie zero wewnątrz napisu i na jego długości, a `UTF8ToString(ptr, liczba_bajtów)` tego Emscriptena przyjmuje limit
bajtów. Łatka zawodzi budowanie, jeśli wzorca nie ma dokładnie raz. Zastosowana na prawdziwym `moonlight-wasm.js` z
`diag9` (pozostało 0 wystąpień `endCharSwap`, składnia poprawna).

**Weryfikacja.** `diag16` = `ALLOC_STRESS` + `DIAG_MALLOC` + ta poprawka, ten sam test (Desktop 3840x2160@60 HEVC,
dławienie 12 Mbps, wcześniej zawieszenie w 1-6 min za każdym razem). Wynik poniżej.

- `diag16` (12 Mbps, 12:23-12:51 UTC, ok. 28 min pod dławieniem, dwa strumienie przerwane przez samo dławienie: rozłączenie
  kanału kontrolnego ENet, aplikacja wróciła do listy gier): **0 raportów alokatora** (`walk found a damaged chunk`,
  `check failed`), 0 zawieszeń, 0 wyjątków `[wasm-exception]`. Walidacja całej sterty co 2. wejście była włączona.
- Wersja produkcyjna (`artifacts/Moonlight-1.18.0-opt3-embindfix.wgt`: bez kopii dlmalloc i bez kanarka, tylko poprawka
  `std::string`), dławienie 15 Mbps, 12:54-13:05 UTC (ok. 11 min, ponad 7500 wpisów `Network dropped` w logu): bez
  zawieszenia, bez trapa, aplikacja odpowiadała do końca (strumień zamknięty przez użytkownika). Obraz przeskakuje przy
  dławieniu, to normalne.

Dla porównania wcześniej każde uruchomienie z dławieniem 12 Mbps kończyło się zawieszeniem w ciągu 1-6 min. Wniosek:
poprawka usuwa zawieszenie. Czego to nie dowodzi: testowano jedno TV (UE43CU7172UXXH, Tizen 9) i jeden host (Apollo).
