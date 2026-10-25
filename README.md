[README.md](https://github.com/user-attachments/files/33933098/README.md)

# LoRa Field Test — Heltec Vision Master E213

Samodzielny tester zasięgu LoRa na płytce **Heltec Vision Master E213** (ESP32-S3, SX1262, ekran e-ink 2,13" 250×122, dwa przyciski, bez dotyku i bez GPS). To wersja projektu „CYD Field for HeltecE213” przeniesiona na e-ink.
To **własny firmware, nie Meshtastic**: płytka sama obsługuje radio. Nie ma mapy ani czatu. Pasywnie nasłuchuje sieć Meshtastic, pokazuje listę słyszanych węzłów, pozwala wybrać jeden i go obserwować oraz pingować. Działa z jedną płytką, nie potrzebuje drugiej płytki.

> Uwaga: firmware został napisany i przetestowany na hoście (symulacja radia i ekranu, ASan/UBSan). **Nie był kompilowany dla ESP32 ani uruchamiany na płytce**, więc pierwsze wgranie może wymagać drobnych poprawek. Warstwa e-ink (`include/epd_hw.h`) przeszła tylko kontrolę składni względem prawdziwych nagłówków GxEPD2 i Adafruit_GFX.

## Co potrafi

| Ekran | Opis |
|---|---|
| HOME | kafelki: liczba słyszanych węzłów, ostatni pakiet (RSSI, SNR, ocena GOOD / MARGINAL / WEAK względem granicy demodulacji dla SF), szum, zajętość kanału, parametry radia, bateria; w nagłówku nazwa Twojego węzła |
| NOISE | **prawdziwy pomiar szumu** (chwilowy RSSI radia 10×/s): teraz, podłoga, szczyt, zajętość, wykres z oknami 1 min / 10 min / 1 h |
| SCAN | przemiatanie pasma 863–870 MHz (krok 100 kHz) z zapamiętaniem szczytów; zaznaczona częstotliwość kanału Meshtastic |
| MESH | podsłuch pakietów Meshtastic (nagłówki jawne): nadawca, RSSI, SNR, hopy, wiek; nazwy węzłów tylko z domyślnym kluczem kanału; **ping wybranego węzła** z pomiarem ACK |
| NODE | szczegóły wybranego węzła Meshtastic: nazwa, identyfikator, RSSI, SNR, hopy, średnie RSSI, min/max, wykres ostatnich 40 odczytów, liczba pakietów, wiek, wynik pingu |
| SETUP | moc pingu, preset Meshtastic, tryb ciemny (biały na czarnym), obrót ekranu o 180°, częstość odświeżania ekranu (2 / 5 / 10 / 30 s), beacon (odstęp), automatyczny ping, Bluetooth (stały PIN 123456); zapis w NVS |

### Ekran e-ink
Ekran jest czarno-biały i odświeża się wolno (ok. 0,5–1 s częściowo, 2–3 s w całości), więc interfejs jest uproszczony: bez animacji, bez kolorów (ważne rzeczy to pogrubienie, odwrócone pola i półtony). Zmiana treści trafia na ekran najwcześniej co N sekund (SETUP → Screen refresh, domyślnie 2 s), po naciśnięciu przycisku po ok. 1 s. Co 10 odświeżeń częściowych wykonywane jest jedno pełne (usuwa smugi). Podczas odświeżania pętla główna stoi, więc pojedynczy pakiet może zostać pominięty. Stan niskiej baterii (≤10%) pokazuje odwrócony kafelek.

### Ekran powitalny
Przy starcie statyczny ekran „MESH FIELD” z grafiką sieci mesh (ok. 4 s, bez animacji, bo e-ink jest wolny).

### Zegar
W nagłówku (po prawej) widać godzinę „HH:MM”, a „--:--” dopóki czas nie jest ustawiony. Płytka nie ma GPS ani podtrzymywanego zegara, więc czas ustawiasz w konfiguratorze przyciskiem „Set time from this device” (wymaga PIN-u; czas lokalny telefonu). Zegar chodzi dalej w trybie uśpienia, ale zeruje się po całkowitej utracie zasilania i wtedy trzeba go ustawić ponownie. Dryf wewnętrznego zegara RC jest rzędu minut na dobę.

### Bateria i ładowanie
Kafelek BATTERY na HOME pokazuje napięcie i orientacyjny procent. Gdy wykryte jest ładowanie, zamiast procentu pojawia się „CHARGING” (również w konfiguratorze i w stanie BLE: pole `chg`). Wykrywanie jest heurystyczne, bo E213 nie ma znanego pinu statusu ładowania (szczegóły w „Do sprawdzenia na sprzęcie”).

Wyłączenie przy krytycznie niskiej baterii: gdy poziom spada poniżej 5% przez 3 kolejne odczyty (ok. 30 s) i nie wykryto ładowania, płytka pokazuje ekran „BATTERY CRITICAL”, wyłącza radio, usypia e-ink i przechodzi w deep sleep. E-ink zachowuje obraz bez zasilania, więc komunikat zostaje na ekranie. Co 30 s budzi się na ułamek sekundy, mierzy napięcie i uruchamia się ponownie dopiero, gdy napięcie wyraźnie wzrośnie (≥60 mV od minimum, czyli podłączono zasilanie) lub osiągnie ≥3,75 V. Reset przyciskiem też uruchamia płytkę normalnie (jeśli bateria nadal jest krytyczna, wyłączy się ponownie po ok. 30 s). Brak pinu statusu ładowania oznacza, że wznowienie jest heurystyką; niesprawdzone na sprzęcie.

### Nazwa węzła
Nazwę swojego węzła (do 12 znaków ASCII, bez polskich liter) wpisujesz w konfiguratorze przez Bluetooth (pole „Node name”, po podaniu PIN-u 123456). Płytka zapisuje ją na stałe, pokazuje w nagłówku zamiast „LoRa Field Test”, podaje w stanie urządzenia i w nazwie BLE (`E213-<nazwa>`, po ponownym włączeniu Bluetooth). Nie jest to nazwa rozgłaszana w sieci Meshtastic.

### Widoczność w sieci Meshtastic (beacon) i pozycja GPS
Sam nasłuch i ping nie sprawiają, że inne urządzenia widzą ten węzeł: ping to pakiet prywatnego portu do jednego węzła, a żadnych informacji o sobie płytka nie rozgłaszała. Teraz robi to **beacon**: co 15 / 30 (domyślnie) / 60 / 180 min (SETUP → Node beacon albo konfigurator) nadaje w trybie broadcast:
- **NODEINFO** z Twoją nazwą (długa nazwa = nazwa węzła, krótka = pierwsze 4 znaki; bez nazwy: `E213 xxxx`, gdzie xxxx to koniec identyfikatora),
- **POSITION** z pozycją wpisaną w konfiguratorze (szerokość, długość, wysokość; pola puste = brak pozycji; przycisk „Use phone location” bierze ją z GPS telefonu).
Pierwszy beacon wychodzi ok. 30 s po starcie, a po zmianie nazwy, pozycji lub odstępu ok. 3 s po zapisie. Na HOME przytrzymanie KEY wysyła beacon od razu (licznik wysłanych jest w stopce).
Pakiety są szyfrowane domyślnym kluczem kanału (AQ==), hop_limit 3, rola CLIENT_MUTE (węzeł niczego nie przekazuje dalej). Inne węzły pokażą go na liście, jeśli: używają **tego samego presetu** (nazwa i częstotliwość jak w „Mesh preset”) i kanału z domyślnym kluczem, są w zasięgu bezpośrednim albo przez przekaźniki, a beacon dotarł. Pozycja jest publiczna dla każdego, kto odbiera ten kanał. Ograniczenia: brak GPS (pozycja jest ręczna i stała), płytka nie odpowiada na żądania NODEINFO ani nie przekazuje cudzych pakietów, a nadawanie podlega limitowi wypełnienia (beacon to ok. 0,2–0,4 s czasu antenowego na LongFast).

### Lista węzłów i obserwacja jednego
1. Ekran **MESH** pokazuje słyszane węzły (najświeższe na górze): nazwa lub identyfikator, RSSI, SNR, hopy, wiek.
2. **KEY** (krótko) przesuwa zaznaczenie na kolejny węzeł. Wybór jest zapamiętany po identyfikatorze, więc nie ucieka, gdy lista się przestawia.
3. **PRG** przechodzi na ekran **NODE**, który obserwuje wybrany węzeł: bieżące RSSI i SNR, hopy, średnie, min/max, wykres ostatnich 40 odczytów, liczba pakietów, wiek. Na NODE KEY dalej przełącza węzły.
4. Po **wybraniu** węzła (pierwsze naciśnięcie KEY zaznacza podświetlony, kolejne przechodzą dalej) płytka **pinguje go automatycznie co 10 s**, aż go zmienisz, zmienisz preset albo wyłączysz „Auto ping” w SETUP. Pingi idą też na innych ekranach (poza SCAN). Długie przytrzymanie KEY na MESH lub NODE pinguje od razu.

### Ping węzła Meshtastic (ekran MESH)
Test działa z dowolnym węzłem Meshtastic widocznym na liście, bez drugiego E213.
1. Wybierz węzeł (KEY), przytrzymaj KEY: ping.
2. E213 wysyła do węzła pakiet z żądaniem potwierdzenia (want_ack), z własnym losowym identyfikatorem (zapisanym na stałe w pamięci, widocznym w logu szeregowym), hop_limit 0 (test bezpośredni, bez przekazywania przez inne węzły) i portem PRIVATE_APP. Węzeł docelowy nie wyświetla takiej wiadomości, ale odsyła ACK.
3. Panel na dole pokazuje: ACK z RSSI, SNR, czasem odpowiedzi i liczbą hopów odpowiedzi, albo NAK, albo brak ACK po 12 s. Liczniki wysłanych i potwierdzonych pingów są osobne dla każdego węzła.
4. Ograniczenia: działa tylko dla kanału z **domyślnym kluczem** (AQ==) i nazwą presetu zgodną z ustawieniem „Mesh preset”. Dla innego klucza pakiet jest dla węzła nieczytelny i ACK nie przyjdzie. Ping to nadawanie w sieci (moc i limit wypełnienia jak w ustawieniach), więc używaj go oszczędnie. Pomiar dotyczy drogi tam (pakiet dotarł) i z powrotem (ACK dotarł): RSSI i SNR dotyczą ACK słyszanego u Ciebie.

### Presety i regiony
Domyślnie (po świeżym wgraniu, bez zapisanych ustawień) płytka pracuje na **MediumFast** w paśmie EU 868 (869,525 MHz, SF9, BW250). Zmienisz to w SETUP lub w konfiguratorze; wybór jest zapamiętywany.
Lista presetów odpowiada aktualnemu firmware Meshtastic dla Europy: **EU_868** (LongFast, LongSlow, MediumSlow, MediumFast, ShortSlow, ShortFast, LongMod; 869,4–869,65 MHz), **EU_866** (LiteFast, LiteSlow; 865,6–867,6 MHz, 4 sloty co 600 kHz) i **EU_N_868** (NarrowFast, NarrowSlow; 62,5 kHz, 869,4–869,65 MHz). Częstotliwość kanału na ekranie MESH liczona jest jak w firmware: slot = hash(nazwa presetu) % liczba slotów. VeryLongSlow i ShortTurbo nie są dostępne w regionach EU.

### Konfigurator przez Bluetooth
Zamiast aplikacji Meshtastic (która rozmawia tylko z firmware Meshtastic) płytka ma własną usługę BLE, a ustawienia zmieniasz stroną `tools/configurator.html` (Web Bluetooth).
1. Na płytce: SETUP → Bluetooth → KEY. Na ekranie pojawia się `ON  PIN 123456` (PIN jest stały: 123456).
2. Otwórz `tools/configurator.html` w Chrome lub Edge (Mac, Windows, Linux, Android). Safari nie obsługuje Web Bluetooth; na iPhonie użyj przeglądarki Bluefy. Gdyby przeglądarka blokowała plik lokalny, uruchom w folderze `tools` polecenie `python3 -m http.server` i otwórz `http://localhost:8000/configurator.html`.
3. „Connect to E213”, wybierz urządzenie `E213-…` (po nazwie węzła), wpisz PIN i „Authorize”. Potem wpisujesz nazwę węzła i pozycję GPS (współrzędne ręcznie albo z telefonu), ustawiasz beacon i automatyczny ping, zmieniasz te same ustawienia co w SETUP i klikasz „Apply”; płytka stosuje je od razu i zapisuje. Strona (interfejs po angielsku) pokazuje też na żywo stan: ostatni pakiet, szum, baterię.
4. Bez PIN-u można tylko czytać ustawienia i stan. Trzy błędne PIN-y rozłączają. PIN jest wysyłany bez szyfrowania BLE, więc to ochrona przed przypadkową zmianą, a nie zabezpieczenie kryptograficzne. PIN jest stały (123456). Wyłączenie Bluetooth w SETUP kończy rozgłaszanie i blokuje autoryzację (stos BLE pozostaje w pamięci do restartu).
5. Parametry LoRa w aplikacji Meshtastic (region, preset, kanał, moc, hop limit itd.) dotyczą firmware Meshtastic. Tu odpowiednikiem są: preset Meshtastic (nasłuch, szum, ping) i moc pingu.

### Limity EU868 (pilnowane przez firmware)
- 869,4–869,65 MHz: wypełnienie 10 %; 868,0–868,6 MHz: 1 % i maks. 14 dBm; pozostałe podpasma 0,1 %.
- Firmware liczy czas nadawania z ostatniej godziny i wstrzymuje wysyłanie po przekroczeniu limitu (na HOME pojawia się DUTY LIMIT). Moc jest przycinana do limitu pasma.

### Szum
Szum to chwilowy RSSI radia w trybie odbioru, agregowany do sekundowych kubełków (średnia, min, max, zajętość) na godzinę. „Podłoga” to 10. percentyl minimów z okna; „zajętość” to odsetek odczytów powyżej podłogi + 8 dB i jednocześnie głośniejszych niż −110 dBm. Podłoga to 25. percentyl sekundowych średnich z ostatnich 5 minut; przez pierwsze 20 s pomiaru zajętość wynosi 0, bo podłoga nie jest jeszcze znana. Stały, wysoki szum nie jest „zajętością”: jego poziom widać w polach FLOOR i NOW. Pomiar nie jest wykonywany podczas własnego nadawania i na ekranie SCAN.

## Obsługa (dwa przyciski)
| | PRG (GPIO0) | KEY (GPIO21) |
|---|---|---|
| zwykłe ekrany | krótko: następny ekran, długo: poprzedni | krótko: akcja ekranu (okno czasu / pauza skanu / następny węzeł na MESH i NODE), długo: wyczyszczenie (NOISE, SCAN) lub ping (MESH, NODE) |
| SETUP | krótko: następna pozycja, długo: wyjście | krótko: zmiana wartości, długo: poprzednia pozycja |

Gdyby obraz był do góry nogami, użyj SETUP → Flip screen.

## Budowanie
```
pio run -e e213 -t upload
pio device monitor             # log CSV co 5 s
```
Zależności (RadioLib 7.1.2, Adafruit GFX i BusIO, fork GxEPD2 z Meshtastic w tym samym commicie co firmware Meshtastic, platforma espressif32 6.9.0) są przypięte w `platformio.ini`. Płytka zgłasza się jako `esp32-s3-devkitc-1` z 8 MB flash i PSRAM OPI. Do wgrania może być potrzebne wejście w tryb bootloadera (przytrzymaj PRG, naciśnij RST).

## Piny (z definicji wariantu Meshtastic)
E-ink: CS 5, BUSY 1, DC 2, RST 3, SCK 4, MOSI 6; zasilanie e-ink i złącza QuickLink (VEXT) GPIO18, HIGH = włączone. Dwie wersje panelu rozpoznawane w locie jak w Meshtastic (BUSY przy wymuszonym RESET: LOW = V1 LCMEN2R13EFC1, HIGH = V1.1 E0213A367). LoRa SX1262: CS 8, SCK 9, MOSI 10, MISO 11, RST 12, BUSY 13, DIO1 14, TCXO 1,8 V, DIO2 jako przełącznik RF. Bateria: ADC GPIO7, sterowanie dzielnikiem 46. Przyciski: 0 i 21.

## Do sprawdzenia na sprzęcie
- Kompilacja i uruchomienie na płytce (`pio run -e e213`), wykrycie wersji panelu e-ink i orientacja obrazu (domyślnie rotacja 1; jeśli do góry nogami, Flip screen).
- Czas odświeżania i ghosting e-ink: dobierz „Screen refresh”; pełne odświeżenie co 10 częściowych (`EPD_FULL_EVERY`) wzorowane na Meshtastic.
- Zegar: czy po deep sleep nadal chodzi i jak mocno dryfuje.
- Mnożnik napięcia baterii (4,9 × 1,03) jest wzięty z Meshtastic; wskazanie procentowe jest orientacyjne.
- Ładowanie: zamiast „~100%” kafelek BATTERY pokazuje „CHARGING”. Nie znalazłem pinu statusu ładowania w E213, więc wykrywanie jest heurystyczne (napięcie rośnie o ≥20 mV między odczytami co 10 s albo ≥4,18 V; wyłączenie przy spadku lub po 60 s bez wzrostu poniżej 4,15 V). Niesprawdzone na sprzęcie.
- Nazwy węzłów w MESH pojawiają się tylko dla kanału z domyślnym kluczem i po odebraniu pakietu NODEINFO.

## Testy na hoście
```
g++ -std=c++17 -fsanitize=address,undefined -Istubs -Iinclude test/units.cpp -o units && ./units
g++ -std=c++17 -DHOST_SIM -DARDUINO=10800 -fsanitize=address,undefined -Istubs -Iinclude -Ithird_party/Adafruit_GFX test/sim.cpp third_party/Adafruit_GFX/Adafruit_GFX.cpp third_party/Adafruit_GFX/glcdfont.c -o sim && mkdir -p shots && ./sim && python3 tools/render.py shots
```
Konfigurator: `node test/configurator_test.js json_options.txt json_status.txt` (pliki JSON zapisuje `sim`) i `node test/configurator_dom_test.js tools/configurator.html json_options.txt json_status.txt`.
`units` sprawdza AES (FIPS-197, tryb CTR zgodny z biblioteką Pythona), nagłówki Meshtastic, ocenę jakości odbioru, szum i limity wypełnienia. `sim` uruchamia cały firmware z wirtualnym SX1262, przyciskami i zegarem; `render.py` składa podglądy ekranów (czcionki zastępcze).
