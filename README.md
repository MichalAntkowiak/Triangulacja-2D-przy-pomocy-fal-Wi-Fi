# Triangulacja 2D przy pomocy fal Wi-Fi

Projekt pasywnego systemu lokalizacji wewnątrzbudynkowej (Indoor Localization) realizujący śledzenie pozycji człowieka na żywo. System opiera się na analizie zaburzeń pola elektromagnetycznego fal Wi-Fi (Channel State Information - CSI) w paśmie 2.4 GHz. 

Projekt realizowany w ramach studiów na kierunku Mechatronika na Politechnice Warszawskiej.

---

## Sprzęt i dane wejściowe

### Komponenty sprzętowe:
* **3x Nadajniki (TX):** Układy deweloperskie ESP32 rozsyłające pakiety radiowe w stałych punktach pomieszczenia.
* **1x Odbiornik (RX):** Układ ESP32 podłączony do stacji roboczej przez interfejs szeregowy USB. Odpowiada za ekstrakcję surowych danych CSI z nagłówków pakietów i ich transmisję do PC.

### Zbiór danych:
* **Rozmiar:** Ponad 55 000 wierszy danych (częstotliwość próbkowania ~100 Hz).
* **Kalibracja:** Pomiarów dokonano w siatce 22 fizycznych punktów w pomieszczeniu o wymiarach 1.6 m x 4.6 m. Dla każdego punktu zarejestrowano po 2500 próbek uwzględniających pełny obrót ciała o 360 stopni (symulacja tłumienia fal przez ludzkie ciało).
* **Wejście (Features):** 192 cechy (3 nadajniki * 64 podnośne amplitudy CSI).
* **Wyjście (Target):** Współrzędne X i Y wyrażone w metrach.

---

## Zasada działania systemu

1. **Generowanie pola:** Nadajniki (TX) stale emitują pakiety radiowe, tworząc w pokoju siatkę fal stojących i odbitych.
2. **Ekstrakcja cech:** Odbiornik (RX) mierzy amplitudy na 64 podnośnych dla każdego nadajnika i przekazuje je do skryptu przetwarzającego.
3. **Fuzja sygnałów:** Skrypt w Pythonie zbiera pakiety z poszczególnych adresów MAC i składa je w jeden zsynchronizowany wektor cech.
4. **Klasyfikacja przestrzenna:** Model K-Nearest Neighbors (KNN) z wagami dystansowymi dopasowuje aktualny profil radiowy do bazy danych i wyznacza przybliżone współrzędne X i Y.
5. **Filtracja i wizualizacja:** Współrzędne przechodzą przez dwuwymiarowy filtr Kalmana, który wygładza trajektorię i eliminuje szum przed wyświetleniem punktu na wykresie.

---

## Problemy i rozwiązania (Ścieżka deweloperska)

| Problem | Przyczyna | Rozwiązanie |
| :--- | :--- | :--- |
| **Zawieszanie się okna radaru** | Blokująca funkcja `readline()` nie nadążała przetwarzać strumienia przy częstotliwości ~120 pakietów/s. | Zaimplementowano asynchroniczny, nieblokujący bufor kołowy oparty na `ser.read(ser.in_waiting)`. |
| **Kropka zamrożona w centrum** | Model RandomForestRegressor przy szumie live zwracał bezpieczną średnią matematyczną zbioru. | Zastąpiono model algorytmem **KNN**, który znacznie lepiej radzi sobie z ciągłą interpolacją przestrzeni. |
| **Drżenie kropki przy braku ruchu** | Sprzętowy mechanizm automatycznej regulacji wzmocnienia (**AGC**) w chipach ESP32 nieustannie zmieniał amplitudy sygnałów. | Wprowadzono **normalizację L2** (X_norm = X / \|\|X\|\|) wektora wejściowego. Model analizuje teraz geometryczny kształt fali, a nie jej moc. |
| **Brak płynności i nagłe skoki pozycji** | Silna interferencja wielodrogowa i odbicia fal od ścian w wąskim pomieszczeniu. | Zaimplementowano **dwuwymiarowy Filtr Kalmana** na wyjściu predykcji, narzucający na pozycję kropki prawa kinematyki i bezwładności. |

---

## Wnioski

Projekt wykazał, że pasywna lokalizacja Wi-Fi CSI pozwala na orientacyjne wyznaczanie pozycji bez urządzeń ubieralnych, ale napotyka twarde ograniczenia fizyczne. Z powodu zjawiska wielodrogowości i nieliniowego tłumienia sygnału w zamkniętych przestrzeniach, analiza samej amplitudy fali najlepiej sprawdza się w klasyfikacji strefowej (np. wykrywanie obecności w konkretnym pokoju). Do uzyskania stabilnej, centymetrowej płynności w czasie rzeczywistym niezbędne jest zastosowanie metod opartych na czasie lotu fali (ToF, np. Ultra-Wideband) lub analizie przesunięć fazowych (AoA).
