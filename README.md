# StockRadar

StockRadar to prywatny radar giełdowy dla GPW i wybranych spółek zagranicznych. System pobiera dane z wielu źródeł, analizuje je modułowo i wysyła alerty do Telegrama.

## Struktura projektu

```text
StockRadar/
├── src/        # kod aplikacji
├── tests/      # testy automatyczne
├── docs/       # dokumentacja, prompty i zasoby dokumentacyjne
├── data/       # lokalne dane uruchomieniowe, bez zawartości w Git
├── config.yaml
└── README.md
```

Polecenia należy uruchamiać z głównego katalogu repozytorium. Katalogi `tmp/`
i `res/` zawierają lokalne artefakty i nie są wersjonowane.

## Moduły

| Moduł | Kategoria | Sygnały |
|-------|-----------|---------|
| `TECH_INDICATORS` | Techniczna | STRONG BUY / BUY / SELL / WAIT |
| `TECH_VOLUME` | Techniczna | BUY (BULLISH) / SELL (BEARISH) / WAIT |
| `TECH_CANDLESTICK` | Techniczna | BUY / SELL / WAIT |
| `TECH_GAPS` | Techniczna | GAP UP / GAP DOWN / FILLED |
| `TECH_DIVERGENCE` | Techniczna | BUY (bycza) / SELL (niedźwiedzia) |
| `TECH_MA_CROSSOVER` | Techniczna | BUY (Golden Cross) / SELL (Death Cross) |
| `TECH_SUPPORT_BOUNCE` | Techniczna | BUY |
| `TECH_BOLLINGER` | Techniczna | BUY / SELL / WAIT |
| `TECH_ADX` | Techniczna | STRONG↑ / STRONG↓ / BUILDING / WEAK |
| `TECH_REL_STRENGTH` | Techniczna | OUTPERFORM / UNDERPERFORM |
| `TECH_PIVOT` | Techniczna | BUY (BULLISH) / SELL (BEARISH) / WAIT (CHOP) |
| `TECH_MARKET_BREADTH` | Techniczna (rynkowa) | 🔥 SILNA HOSSA / 📈 HOSSA / ⚪ NEUTRALNY / 📉 BESSA / ⚠️ SILNA BESSA |
| `FUND_OVERVIEW` | Fundamentalna | BUY / HOLD / WAIT (scoring 0–100) |
| `FUND_SCORE_PIOTROSKI` | Fundamentalna | STRONG (7-9) / WEAK (0-4) |
| `FUND_EARNINGS_DRIFT` | Fundamentalna | POSITIVE DRIFT / NEGATIVE DRIFT |
| `META_AI_VERDICT` | Meta (AI Hybrid) | STRONG_BUY / BUY / HOLD / SELL |
| `META_CONFLUENCE` | Meta | STRONG_BULLISH / BULLISH / STRONG_BEARISH / BEARISH (MIXED = brak emisji sygnalu) |
| `ALERT_PRICE_CHANGE` | Alert | UP / DOWN |
| `ALERT_PRICE_LEVEL` | Alert | BUY / SELL |
| `ALERT_RECOMMENDATIONS` | Alert | BUY / SELL / HOLD |
| `ALERT_ESPI` | Alert | informacyjny |
| `ALERT_KNF_SHORT` | Alert | SHORT (zmiana pozycji) |
| `ALERT_CALENDAR` | Alert | informacyjny |
| `REPORT_AI_RECOMMENDATIONS` | Raport (AI) | KUP / TRZYMAJ / OMIJAJ |
| `REPORT_AI_DAILY_PICK` | Raport (AI) | pick dnia |
| `REPORT_ANALYST_PICK` | Raport (źródło zewnętrzne) | BUY / SELL / HOLD |
| `REPORT_MORNING_BRIEF` | Raport | informacyjny |

### Klasyfikacja mechanizmu generowania sygnałów

Na potrzeby analizy i porównywania skuteczności moduły oraz sygnały należy klasyfikować według źródła decyzji:

| Klasa | Mechanizm | Przykłady |
|-------|-----------|-----------|
| `HEURISTIC` | Deterministyczne reguły, progi, wagi i scoringi zdefiniowane w StockRadarze | moduły `TECH_*`, `FUND_OVERVIEW`, `FUND_SCORE_PIOTROSKI`, `FUND_EARNINGS_DRIFT`, `META_CONFLUENCE` |
| `AI` | Werdykt lub wycena wygenerowana przez model AI | `META_AI_VERDICT`, `FUND_AI_FAIR_VALUE`, `REPORT_AI_RECOMMENDATIONS`, `REPORT_AI_DAILY_PICK` |
| `EXTERNAL` | Decyzja człowieka albo zewnętrznego źródła, w tym sygnał zarejestrowany ręcznie | `REPORT_ANALYST_PICK`, rekomendacje analityczne, konta społecznościowe, sygnały manualne |
| `INFORMATIONAL` | Rejestracja zdarzenia lub przekroczenia warunku bez właściwej prognozy inwestycyjnej | `ALERT_ESPI`, `ALERT_CALENDAR`, `ALERT_KNF_SHORT` oraz proste alerty cenowe |

Określenie **moduły heurystyczne** jest poprawne dla modułów regułowych. Same wzory wskaźników i modeli, np. RSI, ADX, Piotroski F-Score, DCF lub EPV, są obliczeniami ilościowymi; heurystyczna jest ich interpretacja poprzez ustalone progi, punkty i wagi prowadzące do sygnału `BUY`, `SELL` lub `HOLD`.

Wynik `confidence_score` w zakresie 0–100 pozostaje indeksem heurystycznym, a nie bezpośrednim prawdopodobieństwem sukcesu. Backtest tworzy osobne `calibrated_probability_pct`: monotoniczną estymację szansy dodatniego wyniku netto, uczoną na starszych transakcjach i ocenianą wyłącznie out-of-sample. Można ją interpretować probabilistycznie tylko wtedy, gdy próba OOS jest wystarczająca i `passes_calibration_check=true`.

Przy porównywaniu jakości predykcyjnej zalecany jest podział na `HEURISTIC`, `AI` i `EXTERNAL`. Sygnały `INFORMATIONAL` powinny być analizowane oddzielnie, ponieważ zazwyczaj nie zawierają kierunkowej prognozy inwestycyjnej.

### Telegram

System wysyła alerty na Telegram i może uruchomić interaktywnego bota.

Obsługiwane komendy:

- `/cena TICKER`
- `/analiza TICKER`
- `/wycena TICKER`
- `/arkusz TICKER`
- `/fundamenty TICKER`
- `/signal TICKER SIGNAL [MODULE] [ATRYBUCJA] [NOTATKA]` — ręczny zapis sygnału do `trade_signals`; AI opisujesz tokenem `ai:MODEL`, a analityka tokenami `type:analyst|social_account source:NAZWA platform:PLATFORMA url:URL`
- `/ai_batch [TREŚĆ]` — hurtowy zapis rekomendacji wygenerowanych przez AI w WWW w formacie `TICKER: REKOMENDACJA | powód`
- `/run MODULE_NAME` — natychmiastowe uruchomienie wybranego modułu (np. `/run TECH_DIVERGENCE`)

Bot ma blokadę wielokrotnego uruchomienia i zabezpieczenie przed konfliktem `409` przy `getUpdates`.

## Web Dashboard

System zawiera interaktywny dashboard Streamlit dostępny przez `src/app.py`.

### Funkcjonalności:
- **Wybór spółki**: Lista wszystkich tickerów z `config.yaml`
- **Okresy historyczne**: 5d, 1mo, 3mo, 6mo, 1y
- **Interwały**: 1d, 1h, 30m, 15m
- **Wykres cenowy**: Linia ceny z interaktywnym zoomem/panem
- **Poziomy Pivot**: Automatyczne obliczenie i wyświetlenie P, R1, R2, R3, S1, S2, S3
- **Dokładne wartości**: Metryki z precyzyjnymi poziomami pivot
- **VWAP**: Obliczenie Volume Weighted Average Price
- **Struktura rynku**: Detekcja HH/HL/LH/LL
- **Setupy**: LONG_PIVOT / SHORT_PIVOT z rekomendacjami
- **Odświeżanie**: Przycisk manualny + auto odświeżanie co 5 min

### Uruchomienie:
```bash
streamlit run src/app.py
```

## Opis modułów i logika sygnałów

Każdy moduł emituje sygnały widoczne w konsoli i opcjonalnie zapisuje je do tabeli `trade_signals`.

---

### TECH_INDICATORS

Analizuje klasyczne wskaźniki techniczne na danych dziennych.

**Wskaźniki:** RSI(14), EMA(200), MACD(12,26,9), OBV

| Sygnał | Warunki |
|--------|--------|
| 🟢 STRONG BUY | Close > EMA200 (uptrend) AND RSI < 40 AND MACD > 0 AND OBV rośnie |
| 🟢 BUY | Close > EMA200 AND MACD > 0 AND OBV rośnie |
| 🔴 SELL | Close < EMA200 (downtrend) AND RSI > 70 |
| ⚪ WAIT | Żaden z powyższych warunków |

---

### TECH_VOLUME

Wykrywa anomalie wolumenowe względem historycznej średniej, integując je z analizą Order Book dla potwierdzenia sygnałów.

**Wskaźniki:** 
- Wolumen bieżący vs. SMA wolumenu (20 sesji)
- Order Book Direction Score (bid-ask imbalance)
- Kierunek świecy (bullish/bearish)

**Logika Sygnałów (3-warstwowa):**

| Warstwę | Sygnał | Warunki | Confidence |
|---------|--------|---------|-----------|
| **Tier 1** | 🟢 STRONG_BUY | Spike + Order Book bullish (score > 0.1) | 0.95 |
| **Tier 1** | 🟢 BUY | Spike + Order Book neutral/bullish (score > -0.1) | 0.85 |
| **Tier 1** | 🔴 STRONG_SELL | Spike + Order Book bearish (score < -0.1) | 0.95 |
| **Tier 1** | 🔴 SELL | Spike + Order Book neutral/bearish (score < 0.1) | 0.85 |
| **Tier 2** | 🟢 OB_BUY | Silny sygnał Order Book bez spike (score > 0.3) | 0.30+ |
| **Tier 2** | 🔴 OB_SELL | Silny sygnał Order Book bez spike (score < -0.3) | 0.30+ |
| **Tier 3** | ⚪ WAIT | Brak konfiguracji lub słaby sygnał | — |

**Progi konfiguracyjne** (domyślnie):
- Spike wolumenu: ≥ 300% MA20
- Order Book confidence threshold: > 0.3 (dla samodzielnych OB sygnałów)
- MA window: 20 dni

**Konfiguracja** (`config.yaml`):
```yaml
volume_criteria:
  threshold_pct: 300.0        # Próg spike'u (%)
  ma_window_days: 20          # Okno średniej ruchomej
```

**Output metryki** w `signal_params`:
```python
{
    "combined_signal": "BULLISH" | "BEARISH",      # Ostateczny sygnał
    "combined_confidence": 0.85,                    # Confidence 0.0-1.0
    "volume_spike": True,                           # Czy spike detectowany
    "spike_pct": 320.5,                             # Wolumen % średniej
    "ob_direction_score": 0.65,                     # Order Book direction (-1 do 1)
    "ob_signal": "BULLISH",                         # OB label
    "is_green_candle": True,                        # Kierunek świecy
    "current_volume": 1200000,
    "avg_volume": 375000
}
```

**Opis zmian (v2.0+):**
- ✅ **Zawsze** pobierana analiza Order Book (nie tylko przy spike'u)
- ✅ Kombinacja dwóch niezależnych sygnałów dla wyższej jakości
- ✅ Możliwość handlu sygnałami Order Book bez volume spike
- ✅ Wbudowana pewność (confidence scoring)

---

### TECH_CANDLESTICK

Rozpoznaje klasyczne formacje świecowe.

**Dane:** OHLC

| Sygnał | Formacja | Warunki |
|--------|----------|--------|
| 🟢 BUY | Bullish Engulfing | Poprzednia świeca czerwona, bieżąca zielona i obejmuje poprzednią |
| 🟢 BUY | Hammer | Dolny cień > 2× ciało, górny cień minimalny |
| 🔴 SELL | Bearish Engulfing | Poprzednia świeca zielona, bieżąca czerwona i obejmuje poprzednią |
| 🔴 SELL | Shooting Star | Górny cień > 2× ciało, dolny cień minimalny |

---

### TECH_GAPS

Wykrywa i śledzi luki cenowe na interwale H1. Backfill analizuje także późniejsze
świece, zapisuje pełne domknięcie luki do poziomu poprzedniego zamknięcia oraz
wyświetla jedną tabelę z historycznym udziałem domknięć osobno dla każdej spółki
i kierunku luki. Czas domknięcia jest podawany jako `H` (godziny) w ramach tej
samej sesji, a po przejściu do kolejnej sesji jako `D` (dni kalendarzowe).

**Dane:** H1 OHLC, SQLite (historia luk)

| Sygnał | Warunki |
|--------|--------|
| BUY (GAP DOWN) | Open sesji < poprzednie Close o ≥ `min_gap_pct`; sygnał również przy domknięciu w pierwszej H1 |
| ALERT (GAP UP) | Open > Close poprzedniej świecy o ≥ `min_gap_pct` |
| ALERT (GAP DOWN) | Open < Close poprzedniej świecy o ≤ `-min_gap_pct` |
| 🎯 FILLED | GAP UP: `Low <= poprzednie Close`; GAP DOWN: `High >= poprzednie Close` |

Domyślna minimalna luka: 1.5%, konfigurowalny przez `gap_criteria.min_gap_pct`.
Ten sam próg jest stosowany w analizie bieżącej i podczas backfillu.

Moduł zapisuje long-only setupy mean-reversion do `trade_signals` jako
`TECH_GAPS` i wysyła je na Telegram. Dla luki spadkowej zapisuje `BUY`, z celem
na poprzednim zamknięciu. Sygnał powstaje na świecy otwarcia sesji natychmiast
po wykryciu luki. Domknięcie w tej samej świecy H1 nie blokuje sygnału.
`TECH_GAPS` może uczestniczyć w `META_CONFLUENCE` jak pozostałe moduły
techniczne. Luki wzrostowe pozostają w historii luk, ale przy `long_only: true`
nie tworzą transakcyjnego sygnału `SELL`.

Dedykowany backtest gap-fill zakłada wejście po cenie otwarcia luki, brak
stop-lossa i maksymalnie sześć miesięcy kalendarzowych na domknięcie. Uwzględnia
koszty transakcyjne oraz MAE/MFE. Luki bez pełnego sześciomiesięcznego okresu
obserwacji są oznaczane jako ocenzurowane i nie trafiają do mianownika
statystyki domknięć.

---

### TECH_DIVERGENCE

Wykrywa rozbieżności cenowe z oscylatorami — wczesne sygnały odwrócenia trendu.

**Wskaźniki:** RSI(14), MACD histogram, EMA(20) (filtr trendu)

| Sygnał | Typ rozbieżności | Warunki |
|--------|-----------------|--------|
| 🟢 BUY | Bycza (Bullish) | Cena: Lower Low, RSI/MACD: Higher Low; Close < EMA20; RSI ≤ 40 |
| 🔴 SELL | Niedźwiedzia (Bearish) | Cena: Higher High, RSI/MACD: Lower High; Close > EMA20; RSI ≥ 60 |

Minimalna odległość między pikami: `min_rsi_divergence_bars` (domyślnie 5 świec).

---

### TECH_MA_CROSSOVER

Wykrywa krzyżowanie średnich kroczących.

Od teraz sygnał jest warunkowy: crossover jest emitowany tylko wtedy, gdy rynek ma co najmniej minimalną siłę trendu (filtr ADX).

**Wskaźniki:** SMA lub EMA (domyślnie SMA 50/200)

| Sygnał | Nazwa | Warunki |
|--------|-------|--------|
| 🟢 BUY | Golden Cross | Szybka MA krzyżuje Wolną MA od dołu |
| 🔴 SELL | Death Cross | Szybka MA krzyżuje Wolną MA od góry |

Filtr trendu (anti-whipsaw):

- jeśli `ADX < adx_threshold` (domyślnie 20), sygnał crossover jest tłumiony jako szum rynku bocznego
- jeśli `ADX >= adx_threshold`, sygnał może zostać wysłany normalnie

Konfiguracja: `ma_crossover_criteria.fast_period`, `slow_period`, `use_ema`, `adx_threshold`.

---

### TECH_SUPPORT_BOUNCE

Wykrywa odbicia od poziomów wsparcia.

**Wskaźniki:** RSI(14), OBV, pivot lows (20 świec wstecz)

Trzy warunki muszą zajść jednocześnie:

1. Cena dotknęła wsparcia (pivot low ± 2%) w ostatnich 5 świecach
2. RSI był ≤ 30 (strefa wyprzedania) w ostatnich 3 świecach
3. Bieżąca świeca bycza (Close > Open AND Close > Close poprzedniej)

| Sygnał | Warunki |
|--------|--------|
| 🟢 BUY | Wszystkie 3 powyższe |

---

### TECH_BOLLINGER

Wykrywa squeeze i wybicia z wstęg Bollingera.

**Wskaźniki:** Bollinger Bands (20, std=2.0), Bandwidth, wolumen

Algorytm dwuetapowy:

1. **Squeeze:** Bandwidth = (Upper – Lower) / Middle < 0.04 przez ≥ 3 kolejne świece
2. **Wybicie po squeeze:** Close > Upper Band → BUY; Close < Lower Band → SELL
   - Potwierdzenie wolumenu: vol > 1.5× MA20 wzmacnia sygnał

| Sygnał | Warunki |
|--------|--------|
| 🟢 BUY | Squeeze + wybicie górą |
| 🔴 SELL | Squeeze + wybicie dołem |
| ⚪ WAIT | Brak squeeze lub brak wybicia |

Konfiguracja: `bollinger_criteria.squeeze_threshold`, `squeeze_lookback`.

---

### TECH_ADX

Mierzy siłę trendu za pomocą ADX (Average Directional Index). Nie wskazuje kierunku — mówi, czy trend jest silny czy słaby. Służy jako meta-filtr: gdy ADX < 20, sygnały trendowe (MA crossover, MACD) są mało wiarygodne.

**Wskaźniki:** ADX(14), +DI(14), -DI(14)

| Sygnał | Warunki |
|--------|--------|
| 🟢 STRONG↑ | ADX ≥ 25 AND +DI > -DI (silny uptrend) |
| 🔴 STRONG↓ | ADX ≥ 25 AND -DI > +DI (silny downtrend) |
| 🟡 BUILDING | ADX 20–25 AND ADX rośnie (trend się formuje) |
| ⚪ FADING | ADX 20–25 AND ADX spada (trend zamiera) |
| ⚪ WEAK | ADX < 20 (brak trendu — konsolidacja) |

Konfiguracja: `adx_criteria.length` (domyślnie 14).

---

### TECH_REL_STRENGTH

Oblicza relatywną siłę spółki vs benchmark (Jegadeesh-Titman momentum factor — jedna z najlepiej udokumentowanych anomalii rynkowych). Spółki z silnym momentum 6-12M kontynuują outperformance.

**Dane:** 6-miesięczna historia cen spółki i benchmarku

**Benchmarki:**
- PL: EPOL (iShares MSCI Poland ETF — proxy WIG)
- US: S&P 500

**Formuła:** `RS = zwrot_spółki(6M) - zwrot_benchmarku(6M)` (w punktach procentowych)

| Sygnał | Warunki |
|--------|--------|
| 🟢 OUTPERFORM | RS ≥ +15pp |
| 🔴 UNDERPERFORM | RS ≤ -15pp |
| ⚪ NEUTRAL | między -15pp a +15pp |

Konfiguracja: `relative_strength_criteria.lookback_months`, `outperform_pct`, `underperform_pct`.

---

### TECH_PIVOT

Zaawansowana analiza techniczna punktów odniesienia (Pivot Points), VWAP, struktury rynku i setupów płynności.

**Wskaźniki:** HLC (High, Low, Close) z poprzedniego dnia, Volume, OHLC bieżące.

#### Pivot Points
Obliczone z poprzedniej sesji:
- **Pivot (P)**: `(High + Low + Close) / 3`
- **Resistance 1/2/3 (R1/R2/R3)**: Poziomy oporu
- **Support 1/2/3 (S1/S2/S3)**: Poziomy wsparcia

#### VWAP (Volume Weighted Average Price)
Średnia cena ważona wolumenem — dynamiczne wsparcie/opór.

#### Market Structure
- **HH/HL**: Higher High/Low (struktura wzrostowa)
- **LH/LL**: Lower High/Low (struktura spadkowa)

#### Liquidity Sweeps
Przełamania poziomów z wysokim wolumenem potwierdzające kierunek.

| Sygnał | Bias | Warunki |
|--------|------|--------|
| 🟢 BUY (BULLISH) | LONG_PIVOT | Cena > Pivot, pullback do poziomu, reakcja wzrostowa, HH/HL struktura, VWAP wsparcie |
| 🔴 SELL (BEARISH) | SHORT_PIVOT | Cena < Pivot, odbicie od poziomu, reakcja spadkowa, LH/LL struktura, VWAP opór |
| ⚪ WAIT (CHOP) | Konsolidacja | Cena w strefie szumu wokół Pivot (± chop_zone_pct) |

Konfiguracja: `pivot_criteria.chop_zone_pct` (domyślnie `0.15`).

---

### TECH_MARKET_BREADTH

Analizuje szerokość rynku GPW — mierzy, ile spółek faktycznie uczestniczy w ruchu indeksu, a nie tylko jego giganty. Działa w dwóch trybach jednocześnie i raportuje wynik jako **jeden zbiorczy wiersz** w konsoli po zakończeniu pełnego cyklu skanowania.

**Dane:** yfinance (batch download), 1 rok historii dziennej (`period=1y`).

#### Dwa źródła danych

| Źródło | Opis |
|--------|------|
| **Radar** | Spółki ze scanera (z głównej pętli `analyze()`) — wyniki gromadzone inline |
| **WIG broad** | Stały koszyk ~81 spółek: WIG20 + mWIG40 (odchudzone) + sWIG80 proxy — pobierany jednorazowo w `finalize_analysis()` |

#### Metryki

| Metryka | Opis |
|---------|------|
| **A/D Ratio** | Stosunek liczby spółek rosnących do spadających w danym dniu |
| **% Above SMA50** | Odsetek spółek, których kurs jest powyżej 50-sesyjnej średniej kroczącej |
| **% Above SMA200** | Odsetek spółek powyżej 200-sesyjnej średniej (długoterminowy trend) |

> Spółki z `NaN` na SMA (zbyt mała historia) są **wykluczane** z licznika, nie zawyżają/zaniżają wyniku. Spółki z flat close (bez zmiany ceny) nie wliczają się ani do wzrostów, ani do spadków.

#### Logika statusu

| Warunek | Status |
|---------|--------|
| `>SMA50 > 70%` **i** `A/D > 1.5` | 🔥 SILNA HOSSA |
| `>SMA50 > 60%` | 📈 HOSSA |
| `>SMA50 30–60%` | ⚪ NEUTRALNY |
| `>SMA50 < 40%` | 📉 BESSA |
| `>SMA50 < 30%` **i** `A/D < 0.7` | ⚠️ SILNA BESSA |

#### Interpretacja sygnału ukrytej słabości

Jeśli WIG rośnie, ale `% Above SMA50` spada — tylko największe spółki ciągną indeks w górę przy słabości reszty rynku. To klasyczny sygnał ostrzegawczy poprzedzający korektę.

#### Alert Telegram

Wysyłany raz na cykl (deduplikacja przez `state_value = "{pct_above_50}|{ad_ratio}"`).

---

### FUND_OVERVIEW

Ocenia kondycję fundamentalną spółki systemem punktowym (0–100+).

**Dane:** BiznesRadar (domyślnie), Yahoo Finance lub StockAnalysis — cena, C/Z, P/B, ROE, D/E, FCF, wyniki finansowe, historia dywidend, cele analityków

#### Tabela punktowa

| Kryterium | Maks. pkt | Logika |
|-----------|-----------|--------|
| Stopa dywidendy | 30 | ≥ 8%: 30 pkt \| ≥ 6%: 20 pkt \| ≥ 3%: 10 pkt |
| Regularne dywidendy (3 lata) | 5 | Bonus: dywidenda > 0 każdego roku |
| Ryzyko cięcia dywidendy | −5 | Kara: forward yield < 70% trailing annual yield |
| Stabilność dywidendy | 5 | Bonus: trailing yield ≥ 90% średniej 5-letniej i payout ≤ 85% |
| C/Z (trailing P/E) | 20 | < 8: 20 \| < 12: 15 \| < 15: 10 \| < 20: 5 |
| DCF upside | 15 | ≥ 30%: 15 \| ≥ 10%: 10 \| > 0: 5 |
| Cel analityków (upside) | 10 | ≥ 20%: 10 \| ≥ 10%: 7 \| > 0: 3 |
| Zysk netto r/r | −5 do +10 | ≥ 10%: 10 \| ≥ 0%: 5 \| spadek: −5 |
| ROE | 10 | ≥ 15%: 10 \| ≥ 10%: 5 |
| Dług/Kapitał (D/E) | 10 | < 50%: 10 \| < 100%: 5 |
| Current Ratio | 5 | ≥ 1.5: 5 \| ≥ 1.0: 2 |
| Kwartalny zysk netto q/q | 5 | ≥ 20%: 5 \| ≥ 0%: 2 \| spadek: 0 |
| FCF Yield (Wolne Przepływy) | −5 do +15 | ≥ 10%: 15 \| ≥ 5%: 10 \| > 0: 5 \| < 0: −5 |

**Uwaga dot. stopy dywidendy:** do punktacji system preferuje yield liczony dynamicznie jako `dividendRate / currentPrice` (jeśli oba pola są dostępne). Wartość dostawcy (`dividendYield`) jest używana jako fallback.

**Fair Value upside** bazuje na composite fair value — **medianie** z zastosowanych modeli wyceny:

#### Fair Value — modele wyceny

| Model | Formuła | Wymagane dane | Warunek stosowania |
|-------|---------|---------------|--------------------|
| **DCF** (2-fazowy) | Faza 1: 5 lat prognoz + Faza 2: Terminal Value | FCF lub NI, `marketCap` | zawsze gdy FCF lub NI > 0 |
| **Graham Number** | `√(22.5 × EPS × BVPS)` | `trailingPE`, `priceToBook` | tylko gdy P/E ≤ 25 |
| **EPV** (Earnings Power Value) | `NI / r` (zero-growth baseline) | `netIncome` | zawsze gdy NI > 0 |
| **DDM** (Dividend Discount) | `DPS × (1 + g) / (r − g)` | `dividendRate` | tylko gdy yield ≥ 1% |

**DCF 2-fazowy** — porządny model wyceny:
- **Faza 1 (5 lat):** prognoza cash flow z estymowanym tempem wzrostu (mediana: NI r/r, earningsGrowth, revenueGrowth; cap 2–25%)
- **Faza 2:** Terminal Value = `CF₅ × (1 + g_terminal) / (r − g_terminal)`, gdzie `g_terminal = 3%`
- **Baza:** FCF jeśli dostępny, lub NI jako proxy (np. BiznesRadar nie udostępnia FCF)
- **Stopa dyskontowa:** `r = 10%` (WACC proxy)

**EPV** — dolna granica wyceny (zakłada zerowy wzrost): `NI / r`.

**Smart filtering:** Graham pomijany dla growth (P/E > 25), DDM pomijany gdy yield < 1%.
Modele pominięte są pokazywane poglądowo w nawiasach w kolumnie DETAILS.

**Composite** = mediana zastosowanych modeli (odporna na wartości skrajne).
Na konsoli: `FV: 180 (+15%) ` z DETAILS: `BIZNESRADAR DCF:220 EPV:61 (GRAHAM:66) (DDM:15)`.

| Sygnał | Wymagany score |
|--------|---------------|
| 🟢 BUY | ≥ 65 pkt |
| 🟡 HOLD | ≥ 40 pkt |
| ⚪ WAIT | < 40 pkt |

Progi konfigurowalny przez `fundamental_criteria.buy_threshold` i `hold_threshold`.

---

### FUND_SCORE_PIOTROSKI

Oblicza Piotroski F-Score (0–9) — jeden z najlepiej udokumentowanych sygnałów fundamentalnych. 9 binarnych kryteriów oceniających jakość i poprawę kondycji finansowej.

**Dane:** yfinance — sprawozdania finansowe (Income Statement, Balance Sheet, Cash Flow) za ostatnie 4 lata

#### Kryteria

**Rentowność (4 pkt):**

| # | Kryterium | Pass |
|---|-----------|------|
| F1 | ROA > 0 | Zysk netto / Aktywa dodatni |
| F2 | CFO > 0 | Gotówka operacyjna dodatnia |
| F3 | ΔROA > 0 | ROA rośnie r/r |
| F4 | Accruals | CFO > Zysk netto (jakość gotówki) |

**Dźwignia / Płynność (3 pkt):**

| # | Kryterium | Pass |
|---|-----------|------|
| F5 | ΔLeverage ≤ 0 | Dług długoterminowy / Aktywa spada r/r |
| F6 | ΔCurrent Ratio > 0 | Wskaźnik bieżącej płynności rośnie |
| F7 | Brak rozwodnienia | Liczba akcji nie wzrosła |

**Efektywność operacyjna (2 pkt):**

| # | Kryterium | Pass |
|---|-----------|------|
| F8 | ΔGross Margin > 0 | Marża brutto rośnie r/r |
| F9 | ΔAsset Turnover > 0 | Obrót aktywami rośnie r/r |

| Sygnał | Wymagany score |
|--------|---------------|
| 🟢 STRONG | 7–9 (zdrowe i poprawiające się fundamenty) |
| 🟡 NEUTRAL | 5–6 |
| 🔴 WEAK | 0–4 (pogarszające się fundamenty — ostrożność) |

Minimum 5 z 9 kryteriów musi być obliczalnych, by wygenerować sygnał.

---

### FUND_EARNINGS_DRIFT (dawniej PEAD)

Post-Earnings Announcement Drift — jedna z najtrwalszych anomalii rynkowych (udokumentowana od lat 60.). Spółki z dużą niespodzianką wynikową (earnings surprise) kontynuują ruch w kierunku niespodzianki przez 60-90 dni.

**Dane:** yfinance `earnings_dates` (EPS Estimate, Reported EPS, Surprise%), historia cen

**Algorytm:**
1. Znajdź ostatni opublikowany raport kwartalny
2. Sprawdź wielkość niespodzianki (actual vs consensus EPS)
3. Jeśli |surprise| ≥ próg i jesteśmy w oknie dryfu — emituj sygnał
4. Śledź faktyczny drift cenowy od daty raportu

| Sygnał | Warunki |
|--------|--------|
| 🟢 POSITIVE DRIFT | Surprise ≥ +5% i ≤60 dni od raportu |
| 🔴 NEGATIVE DRIFT | Surprise ≤ -5% i ≤60 dni od raportu |
| ⚪ MILD | Niespodzianka poniżej progu — bez sygnału transakcyjnego |

Sygnał zawiera: wielkość niespodzianki, drift cenowy od raportu, liczbę dni w oknie.

Konfiguracja: `pead_criteria.surprise_threshold_pct` (domyślnie 5%), `drift_window_days` (domyślnie 60).

---

### META_CONFLUENCE

Meta-analizator agregujący sygnały ze wszystkich pozostałych modułów dla danego tickera.

Aktualna logika działa warstwowo (2-layer scoring), zamiast prostego liczenia liczby sygnałów:

1. Warstwa 1 (decyzja TAK/NIE): kategorie fundamentalne, np. Piotroski / PEAD / FUND_OVERVIEW.
2. Warstwa 2 (timing): momentum i wejście techniczne, np. RS / ADX / Bollinger / Support Bounce.

Sygnał końcowy jest liczony jako ważona kompozycja kategorii (domyślnie FUND 35%, MOMENTUM 45%, TECH_ENTRY 20%).
Warstwa 1 pełni rolę bramki: jeśli FUND nie potwierdza kierunku, sygnał jest blokowany.
Dodatkowo działa kara konfliktu (value-trap guard): gdy FUND jest dodatni, ale MOMENTUM wyraźnie ujemne, composite jest obniżany.

**Dane:** tabela `trade_signals` w SQLite (ostatnie N dni)

#### Kluczowy parametr: Lookback (dni)

Parametr `lookback_days` (domyślnie 3) określa **okno pamięci systemu**. Ponieważ moduły działają z różną częstotliwością (np. fundamenty raz na dobę, technika co 15 minut), system musi "pamiętać" poprzednie wskazania. 

Dzięki temu `META_CONFLUENCE` może połączyć np. sygnał `BUY` z modułu fundamentalnego sprzed 2 dni z dzisiejszym wybiciem technicznym, generując finalny sygnał zbieżności. Sygnały starsze niż zdefiniowana liczba dni są ignorowane jako nieaktualne.

| Sygnał | Warunki |
|--------|--------|
| 🟢🟢 STRONG BULLISH | Composite bullish, mocny wynik i brak istotnej kontrstrony |
| 🟢 BULLISH | Composite bullish, przekroczony próg |
| 🔴🔴 STRONG BEARISH | Composite bearish, mocny wynik i brak istotnej kontrstrony |
| 🔴 BEARISH | Composite bearish, przekroczony próg |
| ⚪ BRAK SYGNAŁU | Wynik poniżej progu lub blokada przez Layer 1 |

Klasyfikacja sygnałów:
- **Bycze:** BUY, STRONG BUY, STRONG (Piotroski), OUTPERFORM (RS), POSITIVE_DRIFT (PEAD), STRONG_UP (ADX)
- **Niedźwiedzie:** SELL, WEAK (Piotroski), UNDERPERFORM (RS), NEGATIVE_DRIFT (PEAD), STRONG_DOWN (ADX)

Konfiguracja (nowa): `confluence_criteria.lookback_days`, `module_weights`, `module_category_map`, `category_weights`, `layer1_categories`, `layer1_min_score`, `composite_threshold`, `conflict_penalty`.

Fallback: jeśli nie zdefiniujesz mapowania kategorii (`module_category_map` + `category_weights`), moduł przechodzi do starszego trybu sumowania wag (`min_signals`).

#### Jak interpretować komunikaty na konsoli (META_CONFLUENCE)

- `🟢 BULLISH` / `🟢🟢 STRONG_BULLISH` = praktyczny odpowiednik kierunku BUY po agregacji sygnałów (nie jest to literalny sygnał `BUY`, tylko sygnał meta).
- `🔴 BEARISH` / `🔴🔴 STRONG_BEARISH` = praktyczny odpowiednik kierunku SELL/AVOID po agregacji.
- `⚪ BELOW THRESHOLD` = composite nie przekroczył `composite_threshold`; traktuj jako brak przewagi (`NO-TRADE`).
- `🚫 L1 GATE` = warstwa FUND (Layer 1) zablokowała sygnał mimo wskazań warstwy technicznej.
- `⚠️ CONFLICT PENALTY` = aktywna kara konfliktu FUND vs MOMENTUM obniżyła composite.

Przykład:

- `Bullish blocked — FUND: +0.20 ≤ 0.20` oznacza, że FUND nie potwierdził kierunku long przy `layer1_min_score: 0.2`.
- W tym modelu warto traktować taki przypadek jako `WAIT` / obserwację, a nie wejście.

Kolejność decyzji w modelu warstwowym:

1. Najpierw sprawdzana jest warstwa `FUND` (L1, bramka go/no-go).
2. Dopiero potem `MOMENTUM` + `TECH_ENTRY` budują finalny kierunek i siłę sygnału.

#### Tagi setupów (Setup Tags)

Gdy META_CONFLUENCE emituje sygnał byczży, automatycznie rozpoznaje jedną z czterech nazwanych konfiguracji wysokiej jakości i dołącza tag do logu konsoli, wiadomości Telegram oraz pola `signal_params` w bazie danych (umożliwia filtrowanie w backtestach).

| Tag | Warunki | Charakterystyka |
|-----|---------|----------------|
| `SWING_CORE` | `FUND_OVERVIEW=BUY/STRONG_BUY` + `FUND_SCORE_PIOTROSKI=STRONG` + `TECH_REL_STRENGTH=OUTPERFORM` | Spółka fundamentalnie silna, zdrowy bilans (Piotroski ≥ 7), relatywna siła — kluczowy setup swingowy |
| `MOMENTUM_CONT` | `FUND_EARNINGS_DRIFT=POSITIVE_DRIFT` + `TECH_REL_STRENGTH=OUTPERFORM` + `TECH_ADX=STRONG_UP` | Kontynuacja trendu po pozytywnej niespodziance wynikowej z potwierdzeniem ADX — wysoka precyzja |
| `SQUEEZE_CONTEXT` | dowolny moduł FUND byczzy + `TECH_BOLLINGER=BUY` + `TECH_ADX=STRONG_UP` | Wybicie z konsolidacji (Bollinger squeeze) z potwierdzeniem siły trendu — setup wybiciowy |
| `QUALITY_PULLBACK` | dowolny moduł FUND byczzy + `TECH_SUPPORT_BOUNCE=BUY` | Cofnięcie do wsparcia z technicznym sygnałem odbicia na tle pozytywnych fundamentów |

Brak tagu oznacza, że sygnał jest ważny, lecz nie pasuje do żadnej z czterech nazwanych konfiguracji.

Przykład logu konsoli z tagiem:

```
[META_CONFLUENCE] 🟢 BULLISH  Composite: +0.61 | FUND: +0.80 | MOMENTUM: +1.00 | TECH_ENTRY: +0.50 | 3d | 🏷 SWING_CORE
```

Przykład fragmentu wiadomości Telegram:

```
🔗 Confluence: BULLISH
  Composite: +0.61 | FUND: +0.80 | MOMENTUM: +1.00 | TECH_ENTRY: +0.50
  Bull: 4 | Bear: 0
  🏷 Setup: SWING_CORE
```

Filtrowanie backtestów po tagu (SQL):

```sql
SELECT * FROM trade_signals
WHERE module = 'META_CONFLUENCE'
  AND json_extract(signal_params, '$.setup_tag') = 'SWING_CORE';
```

---

### META_AI_VERDICT

Hybrydowy moduł decyzyjny AI. Nie liczy wskaźników od zera.

Zamiast tego:
1. Pobiera najnowsze sygnały modułowe z tabeli `trade_signals` (z ostatnich N dni).
2. Buduje ustrukturyzowany prompt z twardymi wejściami (moduł, sygnał, confidence, timestamp).
3. Prosi model AI o finalny werdykt w formacie JSON: `STRONG_BUY | BUY | HOLD | SELL`.

**Ważne:** `META_AI_VERDICT` jest celowo wykluczony z wejścia `META_CONFLUENCE`, żeby uniknąć pętli zwrotnej (AI -> confluence -> AI).

| Werdykt | Zachowanie |
|--------|------------|
| `STRONG_BUY` | zapis jako trade signal (`STRONG BUY`) |
| `BUY` | zapis jako trade signal (`BUY`) |
| `SELL` | zapis jako trade signal (`SELL`) |
| `HOLD` | alert informacyjny (bez zapisu trade signal) |

Konfiguracja: `ai_verdict_criteria` (`lookback_days`, `max_signals`, `min_signals`, `model_name`, `temperature`, `top_p`, `prompt_template`).

---

### ALERT_PRICE_CHANGE

Monitoruje nagłe zmiany ceny zamknięcia.

**Dane:** cena dzienna

| Sygnał | Warunki |
|--------|--------|
| 🚀 UP | Zmiana ≥ +`threshold`% względem poprzedniego zamknięcia |
| 🩸 DOWN | Zmiana ≤ −`threshold`% względem poprzedniego zamknięcia |

Domyślny próg: 3%, konfigurowalny przez `alert_threshold` w parametrach tickera.

---

### ALERT_PRICE_LEVEL

Pilnuje ręcznie zdefiniowanych poziomów cenowych.

**Dane:** cena bieżąca, progi z `config.yaml`

| Sygnał | Warunki |
|--------|--------|
| 🟢 BUY | Close ≤ `buy_alert` (zdefiniowany dla tickera) |
| 🔴 SELL | Close ≥ `sell_alert` (zdefiniowany dla tickera) |

#### Akcje po osiągnięciu poziomu (`buy_alert_action` / `sell_alert_action`)

Opcjonalnie można skonfigurować co ma się stać po wyzwoleniu alertu:

| Wartość | Działanie |
|---------|----------|
| brak / `null` | domyślne zachowanie — alert przy każdej zmianie stanu (powtarzalny) |
| `once` | jednorazowy alert, potem `buy_alert`/`sell_alert` ustawiany na `null` w `config.yaml` |
| `once_daily` | maksymalnie 1 alert dziennie (stan śledzony w `session_state.db`) |
| `adjust:-10%` | po alertcie obniża/podwyższa target o podany procent |
| `adjust:-5` | po alertcie przesuwa target o wartość absolutną (PLN) |

Przykład konfiguracji:

```yaml
tickers:
  PL:
    CDR:
      buy_alert: 233.0
      buy_alert_action: "once"        # jednorazowy — po alertcie ustawi buy_alert na null
      sell_alert: 250.0
      sell_alert_action: "once_daily"  # max raz dziennie
    FRO:
      buy_alert: 26
      buy_alert_action: "adjust:-10%" # po alertcie obniży buy_alert o 10% (26 → 23.40)
      sell_alert: 33.0
      sell_alert_action: "adjust:+5%" # po alertcie podwyższy sell_alert o 5% (33 → 34.65)
    PLW:
      buy_alert: 230.0
      buy_alert_action: "adjust:-5"   # po alertcie obniży o 5 PLN (230 → 225)
```

Akcja `once` i `adjust` modyfikują `config.yaml` w miejscu (z zachowaniem formatowania i komentarzy dzięki `ruamel.yaml`).

Globalne domyślne akcje (używane gdy ticker nie ma własnych):

```yaml
default_buy_alert_action: "once"        # domyślna akcja dla wszystkich buy_alert
default_sell_alert_action: "once_daily"  # domyślna akcja dla wszystkich sell_alert
```

Ticker z własnym `buy_alert_action` / `sell_alert_action` nadpisuje globalną wartość.

---

### ALERT_RECOMMENDATIONS

Pobiera i klasyfikuje rekomendacje analityczne z BiznesRadar dla GPW oraz Yahoo Finance/Benzinga dla spółek USA.

**Dane:** BiznesRadar i Yahoo Finance/Benzinga (cache 1h), cele cenowe

Źródła ustawia lista `modules.ALERT_RECOMMENDATIONS.sources`; obsługiwane wartości to `biznesradar` i `yahoo`. Dla Yahoo parametr `yahoo_lookback_days` określa ponownie sprawdzane okno historii (domyślnie 7 dni), co pozwala uzupełnić zdarzenia opublikowane podczas przerwy aplikacji. Polling Yahoo odbywa się tylko dla tickerów `.US`. Trwały `source_record_key` oraz kanoniczny fingerprint chronią przed powtórnym zapisem i alertem po restarcie procesu.

Klasyfikacja rekomendacji:

| Sygnał | Słowa kluczowe |
|--------|---------------|
| 🟢 BUY | KUPUJ, AKUMULUJ, PRZEWAŻAJ, BUY, OVERWEIGHT |
| 🔴 SELL | SPRZEDAJ, REDUKUJ, NIEDOWAŻAJ, SELL, UNDERWEIGHT |
| 🟡 HOLD | Pozostałe |

Każda pobrana rekomendacja jest zapisywana lub uzupełniana w kanonicznej tabeli `analyst_recommendations`. Alert Telegram jest wysyłany tylko dla nowej rekomendacji z dzisiaj lub wczoraj. Dla Yahoo zawiera także instytucję, oryginalny rating i cenę docelową w USD. Moduł nie tworzy już osobnego rekordu rekomendacji w `trade_signals`.

---

### ALERT_ESPI

Monitoruje komunikaty bieżące GPW z kanału RSS PAP Biznes.

**Dane:** RSS PAP Biznes

- Filtruje komunikaty po tickerze lub `keywords` z konfiguracji
- Wykrywa nowe ESPI (deduplikacja przez `session_state.db`)
- Wysyła alert Telegram ze skrótem treści

---

### ALERT_KNF_SHORT

Śledzi zmiany w rejestrze krótkiej sprzedaży KNF.

**Dane:** KNF API (rejestr krótkich pozycji)

- Dopasowanie tickera po `isin` lub słowach kluczowych
- Śledzi łączny % krótkiej sprzedaży wszystkich funduszy
- Wykrywa zmiany: nowa pozycja, wzrost, spadek, brak zmiany
- Alerty tylko dla wpisów zmodyfikowanych dzisiaj (`modifyDate`)

---

### ALERT_CALENDAR

Monitoruje nadchodzące zdarzenia korporacyjne ze źródeł rynkowych.

**Dane:** Yahoo Finance calendar

- Patrzy 14 dni w przód
- Alerty dla: dat dywidend (ex-dividend), dat wyników finansowych
- Sygnał informacyjny — bez rekomendacji transakcyjnej

---

### REPORT_AI_RECOMMENDATIONS

Generuje batchowe rekomendacje AI dla listy spółek.

**Model:** Google Gemini (5 tickerów na batch, raz dziennie)

| Sygnał | Znaczenie |
|--------|---------|
| KUP | AI rekomenduje zakup |
| TRZYMAJ | AI rekomenduje utrzymanie |
| OMIJAJ / SPRZEDAJ | AI odradza |

Wyniki są cachowane w `session_state.db` i alerty Telegram wysyłane tylko dla `KUP`.

---

### REPORT_AI_DAILY_PICK

Generuje najwyżej jeden "pick dnia" z aktywnych tickerów GPW w konfiguracji.

**Model:** Google Gemini z obowiązkowym Google Search grounding.

- Wstawia bieżącą datę do promptu; prompt nie zawiera zaszytego roku
- Waliduje ticker, datę kursu, datę raportu, confidence i metryki sektorowe
- Wymaga co najmniej dwóch źródeł Google Search dla sygnału `PICK`
- Oznacza automatyczny zapis w `signal_params` przez `manual: false`
- Stosuje osobne kryteria dla finansów, growth, value/dividend i spółek cyklicznych
- Zwraca `NO_PICK` bez sygnału transakcyjnego, gdy dane lub przewaga są niewystarczające
- Odrzuca odpowiedź bez wyszukiwania zamiast ponawiać ją z wiedzy modelu
- Zapisuje pełne metryki, tezę, ryzyka i źródła w komunikacie
- Działa w trybie `api` (Gemini) lub `prompt` (gotowy prompt na Telegram)

Jednorazowe uruchomienie niezależnie od okna harmonogramu:

```powershell
.\.venv\Scripts\python.exe src\stock_radar.py --modules REPORT_AI_DAILY_PICK --ignore-schedule
```

Dodaj `--no-session`, aby świadomie powtórzyć próbę tego samego dnia bez trwałego
stanu i deduplikacji alertu.

Konfiguracja modelu jest niezależna w `ai.daily_pick_model_name`. Integracja używa
pakietu `google-genai` i narzędzia `google_search`. Model innego dostawcy nie może
być użyty dla Daily Pick, ponieważ nie zapewnia tej ścieżki groundingu.
Gemini 3 wymusza schemat JSON bezpośrednio w jednym wywołaniu API. Dla Gemini 2.5
połączenie `google_search` ze schematem odpowiedzi nie jest obsługiwane, dlatego
integracja wykonuje dwa etapy: badanie ze źródłami przez Google Search, a następnie
formatowanie zebranego materiału do JSON ze schematem. Wynik jest dodatkowo
rygorystycznie walidowany lokalnie przed utworzeniem sygnału.

---

### REPORT_ANALYST_PICK

Udostępnia ręczny import i raportowanie rekomendacji analityków. Rekomendacje DM są zapisywane w `analyst_recommendations`; wskazania kont społecznościowych pozostają ogólnymi sygnałami w `trade_signals`.

Rekomendacje Yahoo Finance/Benzinga dla aktywnych spółek z sekcji `tickers.US` można pobrać za wskazaną liczbę miesięcy poleceniem:

```bash
python src/import_yahoo_recommendations.py --months 12
```

Importer pomija indeksy, kontrakty i wyłączone symbole, mapuje ratingi do `BUY`/`HOLD`/`SELL`, pobiera cenę wejścia z ostatniego zamknięcia w dniu publikacji lub wcześniej oraz zachowuje instytucję, dokładny czas zdarzenia, poprzedni rating i zmianę ceny docelowej. Zdarzenia tego samego DM dla jednej spółki z tego samego dnia są scalane w jeden rekord, a komplet danych źródłowych pozostaje w notatce. Przed atomowym zapisem powstaje kopia `data/backups/stock_radar_before_yahoo_import_*.db`. Flaga `--dry-run` wykonuje pobranie i walidację bez zmiany bazy.

- `source_type`: `analyst` albo `social_account`
- `source_name`: nazwa analityka lub konta (wymagana)
- `source_platform`: np. `X`, `YouTube`, `TV` albo `DM`
- `source_url`: opcjonalny link do materiału źródłowego
- `ingestion_channel`: techniczny kanał zapisu, np. `cli` albo `telegram`
- `recommendation_type`: `fundamental` albo `technical`; brak wartości oznacza `fundamental`

Moduł nie ma osobnego analizatora ani harmonogramu. Import JSON, zapis ręczny analityka i automat BiznesRadar korzystają z jednego serwisu zapisu. Deduplikacja opiera się na spółce, kanonicznym ID instytucji, dacie raportu i typie rekomendacji; ponowny zapis może uzupełnić brakującą cenę docelową, notatkę lub URL.

Menu udostępnia zarówno zapis pojedynczego wskazania, jak i opcję **Importuj rekomendacje analityka lub DM z JSON**. Atrybucja może być wspólna dla płaskiej listy `signals` albo określona osobno dla każdej rekomendacji przez pole `institution`.

Nazwy instytucji są mapowane przez dwa słowniki w SQLite:

- `analyst_institutions` — kanoniczne nazwy DM i innych instytucji analitycznych,
- `analyst_institution_aliases` — możliwe warianty nazw powiązane przez `institution_id` z nazwą kanoniczną.

Warianty `B of A Securities`, `Bank of America Securities`, `BofA`, `BofA Securities LLC` i `Bank of America Merrill Lynch` są normalizowane do `BofA Securities`.

Podczas importu klucz aliasu jest niewrażliwy na wielkość liter, znaki diakrytyczne, interpunkcję i nadmiarowe spacje. Przykładowo `DM BOŚ`, `DM BOS`, `BOŚ DM`, `DM BOŚ SA` i `Dom Maklerski BOS` są zapisywane jako `DM BOŚ`. Warianty `DM Millennium`, `Millennium DM SA` i `Millennium Dom Maklerski` są scalane do `BM Banku Millennium`, `PKO BP DM` i `BM PKO BP SA` do `BM PKO BP`, `Erste Group Research`, `Erste BM`, `Erste Bank`, `Erste / East Value Research` i `Erste Securities Polska SA` do `Erste Securities`, a `DM Trigon` i `Trigon DM SA` do `Trigon DM`. Analogicznie normalizowane są sufiksy prawne używane przez Bankier/Notoria, np. `Noble Securities DM SA`, `East Value Research GmbH` i `Ipopema Securities SA`. Samodzielne `East Value Research` pozostaje odrębną instytucją badawczą. Nieznana instytucja jest automatycznie dodawana jako nowa nazwa kanoniczna wraz z aliasem własnym. To samo mapowanie jest stosowane podczas migracji danych historycznych i budowania rankingu. Inicjalizacja bazy automatycznie scala starsze instytucje zapisane pod aliasem, przepina ich rekomendacje i zachowuje połączone metadane źródłowe.

Każda rekomendacja ma trwałe powiązanie `analyst_recommendations.analyst_institution_id` → `analyst_institutions.id`. Tabela przechowuje bezpośrednio m.in. datę raportu, sygnał, oryginalną treść rekomendacji, cenę docelową i walutę, cenę wejścia, horyzont, notatkę, URL oraz listę kanałów pozyskania. Inicjalizacja bazy idempotentnie migruje historyczne rekordy `REPORT_ANALYST_PICK` i `ALERT_RECOMMENDATIONS`, łącząc dokładne duplikaty i zachowując stare `trade_signals` wyłącznie jako dane historyczne.

Szczegółowy widok wybranej spółki pobiera dzienne notowania OHLC z Yahoo Finance. Dla rekomendacji z ceną docelową pokazuje procentową zmianę targetu względem ceny wejścia oraz sprawdza, czy target został osiągnięty. Dla targetu powyżej ceny wejścia używane jest dzienne maksimum (`High`), a dla targetu poniżej ceny wejścia dzienne minimum (`Low`), dzięki czemu trafienie intraday nie jest pomijane. Raport podaje pierwszą datę osiągnięcia celu i liczbę dni kalendarzowych od publikacji.

Notowania Yahoo są cache'owane zarówno w pamięci procesu, jak i trwale w osobnej bazie `data/yahoo_cache.db`. Klucz cache'u obejmuje symbol Yahoo, okres i interwał. Szczegółowy raport rekomendacji korzysta z wpisu przez godzinę, więc ponowne wyświetlenie raportu — także w kolejnym procesie CLI — nie odpytuje ponownie Yahoo. Pozostałe moduły zachowują domyślny TTL 60 sekund, aby alerty i widoki intraday nie pracowały na zbyt starym kursie. Cache jest techniczny i może zostać bezpiecznie usunięty; nie stanowi części kanonicznych danych aplikacji.

Panel Web udostępnia widok **Rekomendacje DM**. Pozwala filtrować kanoniczne rekomendacje po spółce oraz edytować datę publikacji, sygnał, treść rekomendacji, cenę wejścia, cenę docelową i jej walutę, horyzont oraz notatkę. Zapis odbywa się po ID istniejącego rekordu i dodaje kanał `web_ui` do `ingestion_sources`, dzięki czemu nie tworzy nowej rekomendacji. Zmiana daty aktualizuje także fingerprint rekordu i jest blokowana, jeśli spowodowałaby duplikat. Rekomendację można trwale usunąć w osobnej strefie niebezpiecznej po zaznaczeniu potwierdzenia.

Przykład ręcznego dodania aliasu w SQLite:

```sql
INSERT INTO analyst_institution_aliases (
  institution_id,
  alias,
  normalized_alias
)
SELECT id, 'BOŚ Securities', 'bos securities'
FROM analyst_institutions
WHERE canonical_name = 'DM BOŚ';
```

Opcja **Sprawdź skuteczność analityków po 3/6/12 miesiącach** pozwala wybrać w menu `3 miesiące`, `6 miesięcy`, `12 miesięcy` albo wszystkie trzy okresy jednocześnie. Wartości techniczne 90, 180 i 365 dni nie są pokazywane użytkownikowi. Dzięki temu rekomendację wystawioną wyłącznie na 12 miesięcy można ocenić tylko po pełnym roku. Backtest obejmuje rekomendacje `BUY` i `SELL`: `BUY` jest trafiony przy dodatniej zmianie kursu, a `SELL` przy ujemnej. Rekomendacje `HOLD` są pomijane, ponieważ ich ocena wymagałaby osobno zdefiniowanego pasma neutralności. Raport uwzględnia wyłącznie horyzonty, które już upłynęły. W konsoli wyświetla tabelę z pozycją, horyzontem, analitykiem, liczbą ocenionych i trafionych rekomendacji, skutecznością oraz średnim wynikiem kierunkowym. Mediana i pozostałe szczegóły są dostępne w plikach `analyst_performance*.csv`, które preset menu zapisuje w systemowym katalogu tymczasowym, w podkatalogu `StockRadar`.

Opcja menu **Wylistuj rekomendacje analityków** udostępnia pięć tabel: wszystkie rekomendacje ze szczegółami, szczegóły wybranej spółki, podsumowanie per DM, podsumowanie per spółka oraz podsumowanie per miesiąc. Każdy widok można ograniczyć do 1, 3, 6 lub 12 miesięcy, 2 lat albo całego okresu. Dla widoku pojedynczej spółki można podać ticker z sufiksem rynku lub bez niego, np. `CDR.PL` albo `CDR`; tabela pokazuje datę, rekomendację, DM, cenę docelową i opcjonalną notatkę. Raport korzysta z kanonicznego `analyst_institution_id`, uwzględnia tylko źródła typu `analyst` i pomija konta społecznościowe. W widoku wszystkich rekomendacji pokazuje również cenę wejścia, cenę docelową oraz skuteczność DM. Skuteczność oznacza odsetek osiągniętych cen docelowych wśród targetów możliwych do oceny i jest zawsze liczona z całej historii danego DM, niezależnie od okresu wybranego do filtrowania widocznych rekomendacji.

W `config.yaml` moduł ma `execution_mode: manual`. Taki moduł nie wymaga `interval_minutes` ani `active_hours`, jest pomijany przez scheduler oraz przez jednorazowy przebieg analizy.

---

### REPORT_MORNING_BRIEF

Generuje poranny przegląd rynkowy (raz dziennie).

**Dane:** Fear & Greed Index, Yahoo Finance (S&P500, VIX, USD/PLN, US 10Y)

Zawiera:
- Fear & Greed Index (nastroje rynku)
- S&P 500 (zmiana 1-dniowa i 5-dniowa)
- VIX (indeks zmienności — rośnie gdy rynek się boi)
- USD/PLN (kurs dolara)
- Rentowność US 10Y (obligacje)

Wyłącznie informacyjny — bez sygnałów transakcyjnych.

---



## Harmonogram modułów

Scheduler działa per moduł. Każdy moduł ma osobne:

- `interval_minutes`
- `active_hours`

Konfiguracja znajduje się w sekcji `module_schedules` w `config.yaml`.

Przykład:

```yaml
module_schedules:
  ALERT_ESPI:
    interval_minutes: 1
    active_hours: ["00:00-23:59"]
  TECH_INDICATORS:
    interval_minutes: 15
    active_hours: ["09:00-17:00"]
  REPORT_AI_DAILY_PICK:
    interval_minutes: 1
    active_hours: ["08:40-08:50"]
```

Uwagi:

- `module_schedules` jest wymagane
- każdy moduł z `active_modules` musi mieć własny wpis w `module_schedules`
- `active_hours` może być pojedynczym oknem lub listą okien
- `run()` i `--schedule` respektują okna czasowe modułów

## Konfiguracja tickerów

Tickery definiujesz w `config.yaml`.

Przykład:

```yaml
tickers:
  PL:
    CDR:
      isin: "PLOPTTC00011"
      keywords: ["CD\\s*PROJEKT\\w*"]
      buy_alert: 240.0
      enabled: false  # opcjonalnie: false, aby tymczasowo wyłączyć analizę
      fund_provider: "biznesradar"
      biznesradar_id: "CD-PROJEKT"
      priority: "high"
```

Obsługiwane pola zależne od modułu:

- `keywords`
- `isin`
- `enabled` - opcjonalnie: ustaw na `false`, aby tymczasowo wyłączyć spółkę z automatycznych analiz (domyślnie `true`).
- `buy_alert`
- `sell_alert`
- `buy_alert_action` — akcja po osiągnięciu poziomu buy (`once`, `once_daily`, `adjust:...`)
- `sell_alert_action` — akcja po osiągnięciu poziomu sell (`once`, `once_daily`, `adjust:...`)
- `alert_threshold`
- `fund_provider` — opcjonalne nadpisanie dostawcy danych fundamentalnych dla konkretnej spółki (`biznesradar`, `yahoo`, `stockanalysis`)
- `biznesradar_id` — identyfikator spółki na portalu BiznesRadar (wymagany, jeśli różni się od tickera)

Wszystkie interaktywne wejścia tickerów, ręczny zapis sygnałów i importy rekomendacji korzystają z jednego resolvera. Unikalny symbol bez rynku jest kanonizowany na podstawie konfiguracji, np. `XTB` → `XTB.PL`. Jeśli ten sam symbol albo alias pasuje do spółek na kilku rynkach, operacja jest zatrzymywana i komunikat pokazuje dostępne tickery wraz z nazwami spółek. Symbol spoza konfiguracji musi jawnie zawierać rynek (`.PL` albo `.US`). Przed przejściem do kolejnego kroku menu i ponownie przed zapisem system sprawdza tożsamość symbolu, typ `EQUITY`, zgodność rynku oraz dostępność ceny; ETF-y i inne instrumenty niebędące spółkami są odrzucane. Wyniki pozytywne i negatywne trafiają do tabeli SQLite `ticker_registry` na 30 dni. W tym okresie walidacja korzysta z rejestru bez ponownego wywołania Yahoo; błędy połączenia z dostawcą nie są zapisywane w cache'u. Cache identyfikacji instrumentu nie zastępuje osobnego pobrania aktualnej ceny sygnału.

## Konfiguracja analizatorów

### Fundamenty (FUND_OVERVIEW)

```yaml
fundamental_criteria:
  default_fund_provider: "biznesradar"  # Globalny domyślny dostawca: biznesradar, yahoo, stockanalysis
```

### Luki cenowe (TECH_GAPS)

```yaml
gap_criteria:
  min_gap_pct: 1.5  # Minimalna wielkość luki (%)

gap_strategy:
  trade_signals_enabled: true
  signal_module_name: TECH_GAPS
  session_open_times: {PL: "09:00", US: "09:30"}
  long_only: true
  max_holding_months: 6
  transaction_cost_pct: 0.2
```

### Rozbieżności techniczne (TECH_DIVERGENCE)

```yaml
divergence_criteria:
  min_rsi_divergence_bars: 5   # Liczba świec do analizy
  min_macd_divergence_bars: 5
```

### Krzyżowanie średnich kroczących (TECH_MA_CROSSOVER)

```yaml
ma_crossover_criteria:
  fast_period: 50         # Krótka MA
  slow_period: 200        # Długa MA
  use_ema: false          # false = SMA, true = EMA
  adx_threshold: 20       # crossover jest tlumiony, gdy ADX < prog (rynek boczny)
```

### Confluence (META_CONFLUENCE)

```yaml
confluence_criteria:
  lookback_days: 3
  min_signals: 3            # fallback dla starego trybu (bez kategorii)
  layer1_categories: [FUND]
  layer1_min_score: 0.2
  composite_threshold: 0.2
  category_weights:
    FUND: 0.55
    MOMENTUM: 0.25
    TECH_ENTRY: 0.20
  conflict_penalty:
    enabled: true
    momentum_negative_threshold: -0.3
    penalty: 0.4
```

## Sesja i deduplikacja

Stan aplikacji jest trzymany w `session_state.db`.

System pamięta m.in.:

- przetworzone komunikaty ESPI
- poprzedni stan alertów
- poprzedni procent pozycji krótkiej KNF
- cache dziennych wyników AI
- datę ostatniego `AI Daily Pick`

Opcja `--no-session` uruchamia aplikację bez zachowania tego stanu.

## Uruchamianie

Tryb standard (jednorazowy przebieg):

```bash
python src/stock_radar.py
```

Tryb harmonogramu:

```bash
python src/stock_radar.py --schedule
```

Tryb harmonogramu z wymuszeniem ignorowania okien czasowych:

```bash
python src/stock_radar.py --schedule --ignore-schedule
```

Analiza tylko wybranych spolek:

```bash
python src/stock_radar.py --ticker XTB.PL,CDR.PL
```

Uruchomienie tylko wybranych modulow:

```bash
python src/stock_radar.py --modules FEED_ESPI,FEED_KNF_SHORT,FEED_RECOMMENDATIONS
```

Wyjscie z aplikacji: **Ctrl+C** (graceful shutdown przez signal handler).

Natychmiastowe uruchomienie modulu z Telegrama:

```text
/run TECH_DIVERGENCE
/run TECH_MA_CROSSOVER
/run TECH_GAPS
```

## Tryby pracy CLI

- `standard`: domyslny przebieg analizy (z opcjonalnym `--schedule`)
- `silent service`: po podaniu `--silent` aplikacja nie wypisuje logow na konsolę i nie wysyła Telegrama; dalej zapisuje sygnaly i stan do bazy
- `backfill gaps`: po podaniu `--backfill-gaps` aplikacja wykona backfill i zakonczy proces; menu pozwala wybrać okres 3, 6 lub 12 miesięcy
- `backtest trade_signals`: po podaniu `--backtest-trade-signals` aplikacja uruchomi backtest i zakonczy proces
- `signals chart`: po podaniu `--signals-chart-days` aplikacja wygeneruje dwustronny wykres siły rynkowej (byki vs niedźwiedzie) i zakończy proces

## Wszystkie argumenty CLI

- `--menu` - wyświetla interaktywne menu konsolowe z komendami pogrupowanymi w ramki kategorii. Komendy zdefiniowane w `config.yaml` wybiera się strzałkami; obsługiwane są również dynamiczne parametry wejściowe (TICKER, MODEL itp.).
- `--schedule` - uruchamia petle harmonogramu
- `--ignore-schedule` - ignoruje `active_hours` i uruchamia aktywne moduly niezaleznie od okien czasu
- `--silent` - tryb cichy: brak logow konsolowych i brak powiadomien Telegram; sygnaly i dane dalej trafiaja do SQLite
- Gdy `startup_diagnostics: true` w `config.yaml`, każdy właściwy przebieg aplikacji bez `--silent` wysyła do konsoli i Telegrama diagnostykę procesu: czas, host, użytkownika, PID, interpreter, pełną komendę oraz dane dwóch poziomów procesów nadrzędnych. Wywołania `--help` i `--menu` nie wysyłają diagnostyki. Po zakończeniu śledztwa ustaw `startup_diagnostics: false`. Ułatwia to ustalenie, czy aplikację uruchomił terminal, plik BAT, launcher Pythona czy Harmonogram zadań Windows.
- `--no-session` - resetuje i pomija trwały stan z `session_state.db`
- `--list-tickers` - wypisuje wszystkie skonfigurowane tickery po przecinku i konczy dzialanie
- `--tickers <T1,T2,...>` - ogranicza analizę do wybranych tickerow (np. `PKO.PL,MSFT.US` lub `*.US` dla calego rynku); moduły z `analysis_scope: market` są wtedy pomijane, chyba że zostaną jawnie wskazane przez `--modules`
- `--modules <M1,M2,...>` - wlacza tylko podane moduly (pozostale sa tymczasowo wylaczane)
- `--backfill-gaps` - uruchamia backfill historii luk cenowych (`TECH_GAPS`)
- `--backfill-period <PERIOD>` - okres backfillu, np. `3mo`, `6mo`, `1y` (domyslnie `1y`)
- `--backfill-gap-signals` - idempotentnie zapisuje historyczne luki sesyjne `DOWN` jako `TECH_GAPS / BUY` w `trade_signals`
- `--list-price-gaps <TICKER>` - wyświetla tabelę luk zapisanych dla spółki
- `--gap-list-status <STATUS>` - filtr tabeli luk: `unfilled` (domyślnie), `filled` albo `all`
- `--backtest-gap-fill` - uruchamia dedykowany backtest strategii domykania luk
- `--gap-backtest-period <PERIOD>` - okres danych H1, np. `1y`, `2y`; domyślnie `2y`
- `--gap-min-pct <PCT>` - nadpisuje minimalną wielkość luki
- `--gap-max-holding-months <N>` - maksymalny czas pozycji w miesiącach kalendarzowych
- `--gap-transaction-cost-pct <PCT>` - łączny koszt wejścia i wyjścia
- `--gap-backtest-export <CSV>` - eksportuje pojedyncze transakcje i MAE/MFE
- `--register-ticker <T>` - ticker do ręcznego zapisu sygnału (np. `CDR.PL`); centralny resolver waliduje składnię, alias, rynek i jednoznaczność przed pobraniem ceny oraz zapisem
- `--register-signal <S>` - sygnał do zapisu (np. `BUY`, `SELL`, `HOLD`)
- `--register-module <M>` - moduł źródłowy sygnału (domyślnie `REPORT_AI_DAILY_PICK`)
- `--register-note <TXT>` - notatka opisująca kontekst decyzji
- `--register-price <P>` - cena wejścia; gdy brak, system pobiera ostatni close z Yahoo
- `--register-ai-model <N>` - nazwa modelu AI do statystyk (np. `gemini-2.5-flash`, `gpt-5.4-nano`)
- `--register-source-type <T>` - typ źródła: `ai`, `analyst` albo `social_account`
- `--register-source-name <N>` - nazwa modelu, analityka albo konta społecznościowego
- `--register-source-platform <P>` - platforma publikacji, np. `X`, `YouTube` albo `DM`
- `--register-source-url <URL>` - opcjonalny adres HTTP/HTTPS materiału źródłowego
- `--register-recommendation-type <T>` - typ rekomendacji: `fundamental` albo `technical`; domyślnie `fundamental`
- `--register-ai-batch <PATH>` - wczytuje zbiorcze wyniki AI z pliku JSON lub tekstowego. Format JSON pozwala przypisać osobno sygnał `BUY`, `SELL` albo `HOLD` do każdego tickera:

  ```json
  {
    "model": "gpt-5",
    "recommendation_type": "technical",
    "signals": [
      {"ticker": "XTB.PL", "signal": "BUY"},
      {"ticker": "CDR.PL", "signal": "SELL"},
      {"ticker": "PKO.PL", "signal": "HOLD"}
    ]
  }
  ```

  `recommendation_type` dotyczy całej paczki i jest opcjonalne; brak pola oznacza `fundamental`. Jawna flaga `--register-recommendation-type` ma pierwszeństwo przed wartością z JSON. Starszy format JSON z polem `"tickers"` pozostaje obsługiwany i traktuje wszystkie wpisy jako `BUY`. Format tekstowy: `TICKER: REKOMENDACJA | powód`.
- `--register-analyst-batch <PATH>` - importuje paczkę JSON jako `REPORT_ANALYST_PICK`. Atrybucja znajduje się w samym pliku. Obsługiwany jest płaski format:

  ```json
  {
    "source_type": "analyst",
    "source_name": "DM BOŚ",
    "source_platform": "DM",
    "recommendation_type": "fundamental",
    "signals": [
      {"ticker": "XTB.PL", "signal": "BUY"},
      {"ticker": "CDR.PL", "signal": "HOLD"}
    ]
  }
  ```

  Można również przekazać listę takich obiektów, gdy jeden plik zawiera osobne paczki wielu DM. W płaskim formacie pole `date` jest aliasem `report_date`, a opcjonalne `target_price` i `currency` trafiają do osobnych kolumn tabeli rekomendacji. Jeśli podano jednocześnie `date` i `report_date`, obie wartości muszą być identyczne.

  Wymagania płaskiego formatu:

  - główny element: pojedynczy obiekt paczki albo niepusta lista takich obiektów,
  - paczka: niepusta lista `signals`; `source_name` jest wymagane na poziomie paczki albo osobno przy każdym sygnale,
  - sygnał: wymagane niepuste `ticker` i `signal`,
  - pola opcjonalne paczki: `source_type` (domyślnie `analyst`), `source_platform` (domyślnie `DM`) i `recommendation_type` (domyślnie `fundamental`),
  - pola opcjonalne sygnału: `market`, `date`, `report_date`, `target_price`, `currency`, `entry_price`, `note`, `source_name`, `source_type`, `source_platform`, `source_url` i `recommendation_type`.

  Wartości `signals[].signal` oraz `recommendations_history[].recommendation` są niewrażliwe na wielkość liter i mapowane następująco:

  | Wynik zapisywany | Akceptowane wartości wejściowe |
  |---|---|
  | `BUY` | `BUY`, `KUP`, `KUPUJ`, `ACCUMULATE`, `AKUMULUJ`, `OVERWEIGHT`, `PRZEWAŻAJ`, `PRZEWAZAJ` |
  | `HOLD` | `HOLD`, `TRZYMAJ`, `NEUTRAL`, `NEUTRALNIE`, `EQUAL WEIGHT`, `EQUALWEIGHT`, `RÓWNOWAŻ`, `ROWNOWAZ` |
  | `SELL` | `SELL`, `SPRZEDAJ`, `REDUCE`, `REDUKUJ`, `UNDERWEIGHT`, `NIEDOWAŻAJ`, `NIEDOWAZAJ` |

  Importer przyjmuje również format historii `recommendations[].recommendations_history[]` z polami `institution`, `recommendation`, `target_price`, `currency`, `report_date` i `horizon_months`. W tym formacie `institution` jest mapowane przez `analyst_institution_aliases` na kanoniczne `source_name`, a domyślne wartości to `source_type=analyst`, `source_platform=DM` i `recommendation_type=fundamental`. Dla `market=GPW` importer dodaje do tickera sufiks `.PL`. Jeżeli podano `report_date`, data sygnału odpowiada dacie raportu, a ceną wejścia jest ostatnie dostępne zamknięcie z tego dnia lub wcześniejszej sesji.
- `--backtest-trade-signals` - uruchamia backtest na tabeli `trade_signals`
- `--backtest-horizons <D1,D2,...>` - globalne horyzonty oceny w dniach, np. `1,7,30,90`; gdy parametr nie jest podany, backtest bierze `backtest_horizons` z konfiguracji modułu
- `--backtest-dedup-days <D>` - globalne nadpisanie okna deduplikacji sygnalow dla pary `(ticker, module, signal)`; bez flagi obowiazuje `modules.<NAZWA>.backtest_dedup_days`, `0` wylacza deduplikacje
- `--backtest-success-threshold <P>` - prog sukcesu w procentach; transakcja jest liczona jako trafiona, gdy `net_directional_return_pct > P`; domyslnie `1.0`
- `--backtest-min-confidence <C>` - filtruje backtest do sygnalow z `confidence_score >= C`; wartość `0` wyłącza filtr i obejmuje także rekordy bez `confidence_score`
- `--backtest-transaction-cost-pct <P>` - łączny koszt wejścia, wyjścia i poślizgu odejmowany od każdej transakcji; domyślnie `0.2%`
- `--backtest-entry-execution <M>` - sposób realizacji wejścia: `next_session_close` (domyślny, bez użycia ceny niedostępnej po publikacji sygnału) albo zgodny wstecznie `recorded_price`
- `--backtest-min-ranking-trades <N>` - minimalna liczba transakcji potrzebna do przyznania modułowi miejsca w rankingu; domyślnie `30`
- `--backtest-oos-test-fraction <F>` - udział najnowszych dat przeznaczony na zamrożony holdout OOS, np. `0.30`
- `--backtest-oos-min-train-trades <N>` - minimalna liczba znanych wyników w train wymagana do wyboru modułu; domyślnie `30`
- `--backtest-oos-min-test-trades <N>` - minimalna liczba wyników w oknie OOS wymagana do oceny; domyślnie `10`
- `--backtest-walk-forward-train-days <N>` - minimalna historia przed pierwszym foldem anchored walk-forward; domyślnie `90`
- `--backtest-walk-forward-test-days <N>` - długość pełnego okna testowego; domyślnie `30`
- `--backtest-walk-forward-step-days <N>` - krok między niepokrywającymi się foldami; domyślnie `30`
- `--backtest-fdr-alpha <F>` - maksymalny poziom false discovery rate po korekcie Benjamini-Hochberga; domyślnie `0.05`
- `--backtest-calibration-min-module-trades <N>` - minimalna liczba wyników train z confidence wymagana dla kalibratora moduł + horyzont; domyślnie `30`
- `--backtest-calibration-min-horizon-trades <N>` - minimalna próba dla zapasowego kalibratora wspólnego dla horyzontu; domyślnie `50`
- `--backtest-calibration-min-unique-scores <N>` - minimalna liczba różnych wartości confidence w train; domyślnie `3`
- `--backtest-calibration-reliability-bins <N>` - liczba przedziałów raportu reliability; domyślnie `10`
- `--backtest-calibration-prior-strength <F>` - siła wygładzania prawdopodobieństw do bazowej częstości train; domyślnie `10`
- `--backtest-disable-oos-validation` - wyłącza raporty holdout i walk-forward
- `--backtest-disable-confidence-calibration` - wyłącza kalibrację confidence w holdout i walk-forward
- `--backtest-disable-benchmark` - wyłącza obliczanie relatywnego wyniku względem WIG20 lub S&P 500
- `--backtest-from <YYYY-MM-DD>` - data poczatkowa filtrowania sygnalow
- `--backtest-to <YYYY-MM-DD>` - data koncowa filtrowania sygnalow
- `--backtest-modules <M1,M2,...>` - filtr po polu `module` w `trade_signals`
- `--backtest-signals <S1,S2,...>` - filtr po polu `signal` (np. `BUY,SELL`)
- `--backtest-export <PATH.csv>` - eksport CSV wynikow backtestu, rankingow modulow i raportow confidence buckets
- `--backtest-ai-analysis` - po zakonczeniu backtestu wysyla wyniki do AI (Gemini/OpenAI) w celu analizy; w trybie `prompt` dostarcza gotowy prompt na Telegram
- `--backtest-completed-horizons-only` - zgodna wstecznie flaga wymuszająca zachowanie, które jest obecnie domyślne: pomijanie niedojrzałych horyzontów
- `--backtest-allow-incomplete-horizons` - jawny opt-in do wyceny niedojrzałych horyzontów ostatnim dostępnym kursem; rekordy otrzymują `is_horizon_complete=false` i rzeczywisty `actual_holding_days`
- `--list-analyst-recommendations` - wyświetla tabelę rekomendacji analityków; bez dodatkowych flag pokazuje wszystkie szczegóły z całego okresu
- `--analyst-list-view <V>` - wybiera widok raportu: `all`, `ticker`, `dm`, `company` albo `period`
- `--analyst-list-period <P>` - ogranicza raport do ostatnich `30`, `90`, `180`, `365`, `730` dni albo `all`
- `--analyst-list-ticker <T>` - wybiera spółkę dla widoku `ticker`; sufiks rynku jest opcjonalny
- `--signals-chart-days <D>` - wygeneruj i wyświetl w konsoli dwustronny wykres słupkowy sygnałów z ostatnich D dni
- `--signals-chart-modules <M1,M2,...>` - wymuś na wykresie konkretne moduły (zamiast domyślnych z rolą `signal`)
- `--list-signals-days <D>` - wyświetla maksymalnie 10 tickerów z największą liczbą sygnałów z ostatnich D dni; wyniki są agregowane po tickerze i sortowane malejąco według łącznej liczby sygnałów
- `--list-signals-module <M>` - wybiera moduł uwzględniany w tabeli rekomendacji
- `--anonymize-tickers <MODE>` - ustawia prezentację tickerów w tabeli: `nie` pozostawia oryginały, `pełna` generuje identyfikatory `TICKER-001`, `TICKER-002` itd., a `maskowana` pozostawia pierwszą literę i kraj oraz zawsze wstawia dwie gwiazdki, np. `GOOGL.US` → `G**.US` i `V.US` → `V**.US`. Stała liczba gwiazdek nie ujawnia długości tickera. Opcja jest dostępna w menu „Wylistuj sygnały wg okresu”; wartości `tak/nie` pozostają zgodne wstecznie.
- `--debug` - uruchamia aplikację w trybie debugowania (więcej logów i szczegółów)
- `--help` - wyświetla w konsoli listę wszystkich dostępnych komend

## Dostepne moduly (`AnalysisModule`)

- `REPORT_AI_DAILY_PICK`
- `REPORT_AI_RECOMMENDATIONS`
- `REPORT_ANALYST_PICK`
- `ALERT_PRICE_CHANGE`
- `ALERT_PRICE_LEVEL`
- `FEED_CALENDAR`
- `FEED_ESPI`
- `FEED_KNF_SHORT`
- `FEED_RECOMMENDATIONS`
- `FUND_OVERVIEW`
- `FUND_EARNINGS_DRIFT`
- `FUND_SCORE_PIOTROSKI`
- `META_AI_VERDICT`
- `META_CONFLUENCE`
- `REPORT_MORNING_BRIEF`
- `TECH_ADX`
- `TECH_CANDLESTICK`
- `TECH_DIVERGENCE`
- `TECH_GAPS`
- `TECH_INDICATORS`
- `TECH_MA_CROSSOVER`
- `TECH_PIVOT`
- `TECH_REL_STRENGTH`
- `TECH_SUPPORT_BOUNCE`
- `TECH_VOLUME`

Dla `--modules` dzialaja tez legacy aliasy (stare nazwy): `ESPI`, `PRICE_ALERTS`, `TECHNICAL`, `FUNDAMENTAL`, `AI_PICK`, `AI_VERDICT`, `VOLUME_SPIKES`, `CANDLESTICK_PATTERNS`, `PRICE_GAPS`, `DIVERGENCES`, `MA_CROSSOVERS`, `SUPPORT_BOUNCES`, `CALENDAR_EVENTS`, `KNF_SHORTS`, `RECOMMENDATIONS`, `MORNING_BRIEF` i ich warianty w liczbie pojedynczej.

## Przyklady CLI

```bash
# harmonogram tylko dla wybranych modulow
python src/stock_radar.py --schedule --modules ESPI,KNF_SHORTS,RECOMMENDATIONS

# cichy run pod task/service - bez konsoli i Telegrama
python src/stock_radar.py --schedule --silent

# jednorazowy run dla konkretnej listy spolek
python src/stock_radar.py --ticker XTB.PL,PKO.PL --modules TECH_INDICATORS,ALERT_PRICE_CHANGE

# analiza calego rynku amerykanskiego i dodatkowo wybranej spolki z PL
python src/stock_radar.py --tickers *.US,CDR.PL

# backfill luk cenowych dla 6 miesiecy wraz ze statystykami domknięć per spółka
python src/stock_radar.py --backfill-gaps --backfill-period 6mo --ticker CDR.PL,PKO.PL

# ręczny zapis sygnału BUY po promptcie AI z WWW
python src/stock_radar.py --register-ticker CDR.PL --register-signal BUY --register-module REPORT_AI_DAILY_PICK --register-note "AI WWW daily pick"

# ręczny zapis sygnału BUY z nazwą modelu AI
python src/stock_radar.py --register-ticker CDR.PL --register-signal BUY --register-module REPORT_AI_DAILY_PICK --register-ai-model gpt-5.4-nano --register-recommendation-type technical --register-note "AI WWW daily pick"

# ręczny zapis wskazania analityka z platformy X
python src/stock_radar.py --register-ticker CDR.PL --register-signal BUY --register-module REPORT_ANALYST_PICK --register-source-type social_account --register-source-name @trader_xyz --register-source-platform X --register-source-url https://x.com/trader_xyz/status/1 --register-note "Wybicie z konsolidacji"

# masowy import rekomendacji jednego domu maklerskiego z pliku JSON
python src/stock_radar.py --register-analyst-batch tmp/dm_bos.json

# backtest sygnalow BUY/SELL z eksportem CSV
python src/stock_radar.py --backtest-trade-signals --backtest-horizons 1,7,30,90 --backtest-signals BUY,SELL --backtest-export backtest_results.csv

# skutecznosc analitykow po 3/6/12 miesiacach; raporty trafiaja do Windows TEMP
python src/stock_radar.py --backtest-trade-signals --backtest-horizons 90,180,365 --backtest-success-threshold 0 --backtest-modules REPORT_ANALYST_PICK --backtest-signals BUY,SELL --backtest-export "$env:TEMP/StockRadar/analyst_performance.csv"

# backtest tylko dla mocniejszych sygnalow, z deduplikacja i progiem sukcesu 2%
python src/stock_radar.py --backtest-trade-signals --backtest-min-confidence 60 --backtest-dedup-days 7 --backtest-success-threshold 2 --backtest-export backtest_results.csv

# backtest z analiza AI wynikow (tryb api = odpowiedz z modelu; tryb prompt = gotowy prompt na Telegram)
python src/stock_radar.py --backtest-trade-signals --backtest-export backtest_results.csv --backtest-ai-analysis

# wyświetlenie wykresu sygnałów z ostatnich 14 dni dla wszystkich modułów (z rolą signal)
python src/stock_radar.py --signals-chart-days 14

# wyświetlenie wykresu sygnałów z 30 dni tylko dla konkretnych modułów
python src/stock_radar.py --signals-chart-days 30 --signals-chart-modules META_CONFLUENCE,TECH_ADX
```

Przy ręcznym zapisie ogólnych sygnałów atrybucja i znormalizowany `recommendation_type` trafiają do `trade_signals.signal_params`. Flaga `--register-ai-model` pozostaje kompatybilna i automatycznie ustawia `source_type=ai`, `source_name` oraz `ai_model`. Rekomendacje analityków korzystają z osobnych, typowanych kolumn tabeli `analyst_recommendations`.

Przykład agregacji rekomendacji analityków:

```sql
SELECT
  institutions.canonical_name,
  recommendations.recommendation_type,
  COUNT(*) AS recommendation_count,
  SUM(recommendations.target_price IS NOT NULL) AS with_target_price
FROM analyst_recommendations AS recommendations
JOIN analyst_institutions AS institutions
  ON institutions.id = recommendations.analyst_institution_id
GROUP BY institutions.id, recommendations.recommendation_type
ORDER BY recommendation_count DESC;
```

## Zasady backtestu sygnałów i rekomendacji

Backtester czyta zwykłe sygnały z `trade_signals`, a dla modułu `REPORT_ANALYST_PICK` korzysta z kanonicznej tabeli `analyst_recommendations`.

Aktualny raport referencyjny: [audyt skuteczności sygnałów z 2026-08-22](docs/signal-effectiveness-audit-2026-08-22.md).

- Backtest domyslnie ocenia tylko moduly z `module_role: signal`; alerty i raporty sa pomijane.
- Jezeli nie podasz `--backtest-horizons`, system bierze horyzonty z `modules.<NAZWA>.backtest_horizons`.
- Domyslne horyzonty sa rozdzielone per typ modułu: fundamentalne maja zwykle `30,60,90,180`, a techniczne `1,7,14,30`.
- Deduplikacja działa w ruchomym oknie czasu dla tego samego `(ticker, module, signal, source_type, source_name)` i zostawia pierwszy sygnał danego źródła z okna. Bez globalnego nadpisania każdy moduł korzysta z `backtest_dedup_days`; wartość `0` przekazana przez CLI wyłącza deduplikację. Niezależni autorzy nie usuwają wzajemnie swoich wskazań.
- Przed obliczeniami działa nieinwazyjny audyt jakości. Rekordy z nieprawidłowym tickerem, pustym modułem lub sygnałem, nieparsowalną albo przyszłą datą, niespójną datą/czasem albo bez jednoznacznego kierunku są domyślnie wykluczane z próby, lecz pozostają bez zmian w bazie wraz z powodem widocznym w raporcie.
- Neutralne `HOLD`, `WAIT`, `NEUTRAL` i nieznane nazwy sygnałów nie są automatycznie uznawane za pozycje long. Kierunek jest przypisywany wyłącznie przez jawne mapowanie kanonicznych sygnałów long/short.
- Brak historycznego `confidence_score`, atrybucji źródła albo ceny zapisanej przy sygnale jest raportowany, ale nie unieważnia transakcji. Brak ceny zapisanej nie przeszkadza domyślnemu wejściu po cenie zamknięcia następnej sesji; brak confidence ogranicza jedynie pokrycie kalibracji.
- Audyt rejestruje również problemy wykryte podczas wyceny: brak szeregu rynkowego, ceny wejścia lub ceny wyjścia oznacza `outcome_unresolved`, natomiast brak benchmarku pozostawia wynik absolutny i wyłącza tylko ocenę alfy.
- Granica zapisu nowych sygnałów normalizuje ticker, moduł i sygnał, odrzuca ticker niebędący symbolem rynkowym oraz zapisuje niepoprawną cenę lub confidence spoza zakresu jako `NULL`. Nie wykonuje automatycznej korekty ani usuwania starych rekordów.
- W panelu Web kierunek „Domyślne (Long + Short)” obejmuje wyłącznie sygnały kierunkowe z mapowania `backtest_criteria.signal_mapping`; neutralne `HOLD` nie są traktowane jak transakcje long.
- Backtest domyślnie pomija sygnały, dla których pełny horyzont jeszcze nie upłynął. Jawna flaga `--backtest-allow-incomplete-horizons` włącza wycenę mark-to-market ostatnim dostępnym kursem; takie rekordy są oznaczone `is_horizon_complete=false` i zawierają rzeczywisty `actual_holding_days`.
- Domyślne wejście następuje po sygnale, na zamknięciu następnej dostępnej sesji. Docelowy horyzont jest liczony od faktycznej daty wejścia, a wyjście po pierwszym dostępnym zamknięciu w dniu docelowym lub później.
- `directional_return_pct` jest zwrotem brutto zgodnym z kierunkiem sygnału. `net_directional_return_pct` odejmuje ustawiony koszt round-trip i jest podstawą metryk decyzyjnych.
- `profit_probability_pct` odpowiada na pytanie, jaki odsetek transakcji zakończył się wynikiem netto powyżej zera. `win_rate_pct` oznacza odsetek transakcji, które przekroczyły `--backtest-success-threshold`; przy progu `3%` zysk `+2%` netto jest więc transakcją zyskowną, ale nie jest trafieniem celu.
- Ranking modułów jest liczony osobno dla każdego horyzontu na wszystkich zakończonych wynikach danego horyzontu. Małe próby pozostają widoczne, ale poniżej `min_ranking_trades` nie otrzymują miejsca. Kolejność kwalifikujących się modułów opiera się najpierw na dolnej granicy 95% przedziału średniego zwrotu netto, a następnie na dolnej granicy szansy zysku i liczbie transakcji.
- Widok „Czas trzymania” korzysta z kohorty dopasowanej: dla danego modułu zachowuje identyczne `signal_id` zakończone we wszystkich wybranych horyzontach. Jeżeli zwykłe zestawienie ma 100 transakcji dla 14 dni i 4 dla 90 dni, zwykłe metryki 90-dniowe są liczone tylko z 4 dojrzałych transakcji; porównanie czasu trzymania liczy zarówno 14, jak i 90 dni na tych samych 4 sygnałach.
- Walidacja holdout jest wykonywana osobno dla każdego horyzontu. Najnowsza część dat wejścia stanowi OOS, a ranking modułów powstaje wyłącznie na starszej części train. Wynik OOS nigdy nie zmienia `train_rank`.
- Podział jest purged: transakcja rozpoczęta w train trafia do statystyk treningowych tylko wtedy, gdy jej `exit_date` przypada przed początkiem OOS. Wyniki, które nie były jeszcze znane, są raportowane jako `purged_outcomes` i usuwane z train.
- Anchored walk-forward zaczyna się domyślnie po 90 dniach historii i ocenia kolejne pełne, niepokrywające się okna po 30 dni. Niepełne końcowe okno nie jest raportowane.
- Dodatni wynik OOS jest testowany jednostronnie, a wartości p są korygowane metodą Benjamini-Hochberga osobno w każdej rodzinie horyzont/fold. `passes_oos_net_fdr` oznacza, że wynik netto przetrwał ustawiony limit FDR; brak flagi oznacza wynik niepotwierdzony, a nie automatycznie stratny.
- Dla tickerów GPW benchmarkiem jest `WIG20.WA`, a dla USA `^GSPC`. `excess_return_pct` porównuje zwrot kierunkowy pozycji ze zwrotem benchmarku w tym samym kierunku i okresie. Przy jednakowym koszcie round-trip koszt odejmuje się po obu stronach i nie zmienia alfy.
- Panel Web ma osobną kartę „Out-of-sample” z pełnym holdoutem, modułami wybranymi na train, foldami walk-forward, alfą oraz FDR. Confidence buckety są weryfikowane OOS niezależnie od rankingu in-sample.
- Kalibrator confidence jest dopasowywany oddzielnie w każdym podziale i horyzoncie metodą regresji izotonicznej PAVA. Preferuje model `module_horizon`; gdy próba modułu jest zbyt mała, może użyć oznaczonego `horizon_fallback`. Model nie powstaje przy zbyt małej próbie, zbyt małej liczbie różnych score albo tylko jednej klasie wyników.
- Nowe sygnały `REPORT_AI_DAILY_PICK` zapisują indywidualny confidence konkretnego wyboru jako `confidence_score`, dzięki czemu przyszła próba zawiera zmienność potrzebną do kalibracji. Historyczne `NULL` nie są rekonstruowane z aktualnej konfiguracji, ponieważ tworzyłoby to błąd look-ahead/configuration bias.
- Celem kalibracji jest `is_profitable`, czyli dodatni wynik po kosztach, a nie przekroczenie arbitralnego progu `is_success`. Wyższy score nie może otrzymać niższego skalibrowanego prawdopodobieństwa niż niższy score.
- Jakość prawdopodobieństw jest mierzona wyłącznie OOS przez Brier score, log loss, ECE, bias i wykres reliability. `passes_calibration_check` wymaga wystarczającej próby OOS oraz Brier score lepszego zarówno od surowego `confidence_score / 100`, jak i od stałej bazowej częstości z train.
- Panel Web ma osobną kartę „Kalibracja Confidence”. `coverage_pct` pokazuje, jaka część wyników OOS ma użyteczny historyczny score i kwalifikujący się model; brak pokrycia nie jest zastępowany domyślnym prawdopodobieństwem.
- Panel Web ma osobną kartę „Jakość danych” z bilansem rekordów wejściowych, dopuszczonych i wykluczonych, zestawieniem klas problemów oraz eksportem szczegółów. Sekcja `backtest_criteria.data_quality` w `config.yaml` steruje polityką audytu.
- Confidence score moze sluzyc jednoczesnie do filtrowania sygnalow i do raportowania bucketow skutecznosci. Próg `0` oznacza brak filtra i zachowuje również historyczne rekordy z `confidence_score = NULL`; dodatni próg pomija rekordy bez wyniku. Surowych bucketów in-sample nie należy interpretować jako dowodu kalibracji.
- Panel Web stylizuje maksymalnie 5000 pierwszych transakcji, aby duże backtesty nie przekraczały limitu Pandas Styler. Pełny zbiór pozostaje dostępny w eksporcie CSV.

Pliki CSV po eksporcie backtestu:

- `backtest_results.csv` - wszystkie transakcje wynikowe (`outcomes`)
- `backtest_results_summary.csv` - podsumowanie skutecznosci per horyzont
- `backtest_results_by_module.csv` - statystyki modułów per horyzont
- `backtest_results_module_ranking.csv` - historyczny przekrojowy ranking ogólny, zachowany dla zgodności eksportu; do decyzji używaj rankingu per horyzont
- `backtest_results_module_ranking_by_horizon.csv` - decyzyjny ranking modułów osobno dla każdego horyzontu, z metrykami netto, przedziałami 95% i oceną wiarygodności próby
- `backtest_results_matched_horizon_outcomes.csv` - transakcje ze wspólnej kohorty zakończonej we wszystkich wybranych horyzontach
- `backtest_results_matched_horizon_summary.csv` - porównanie efektu czasu trzymania na identycznych sygnałach
- `backtest_results_oos_module_validation.csv` - purged holdout modułów: train, OOS, alfa i FDR per horyzont
- `backtest_results_oos_confidence_validation.csv` - holdout confidence bucketów per horyzont
- `backtest_results_oos_module_confidence_validation.csv` - holdout kombinacji moduł + confidence bucket
- `backtest_results_oos_selected_modules.csv` - wyłącznie moduły wybrane na train, bez selekcji po wyniku OOS
- `backtest_results_walk_forward_module_validation.csv` - wszystkie moduły w kolejnych foldach walk-forward
- `backtest_results_walk_forward_confidence_validation.csv` - confidence buckety w kolejnych foldach
- `backtest_results_walk_forward_module_confidence_validation.csv` - kombinacje moduł + bucket w kolejnych foldach
- `backtest_results_walk_forward_selected_modules.csv` - zwycięzca train każdego horyzontu i folda wraz z późniejszym wynikiem OOS
- `backtest_results_oos_calibration_predictions.csv` - każda prognoza holdout: surowy score, prawdopodobieństwo po kalibracji, wynik netto i użyty zakres modelu
- `backtest_results_oos_calibration_metrics.csv` - Brier score, log loss, ECE, bias, pokrycie i werdykt kalibracji w holdoucie
- `backtest_results_oos_calibration_reliability.csv` - prognozowana i zaobserwowana szansa zysku w przedziałach prawdopodobieństwa holdout
- `backtest_results_oos_calibration_models.csv` - audyt modeli train, przyczyny odrzucenia i zapis mapowania izotonicznego
- `backtest_results_walk_forward_calibration_predictions.csv` - prognozy confidence dla kolejnych foldów walk-forward
- `backtest_results_walk_forward_calibration_metrics.csv` - metryki jakości prawdopodobieństw w każdym foldzie
- `backtest_results_walk_forward_calibration_reliability.csv` - reliability dla foldów walk-forward
- `backtest_results_walk_forward_calibration_models.csv` - modele train dopasowane niezależnie w każdym foldzie
- `backtest_results_data_quality_overview.csv` - bilans rekordów wejściowych, dopuszczonych i wykluczonych oraz liczba problemów według poziomu
- `backtest_results_data_quality_summary.csv` - zagregowane klasy problemów jakości, działania i liczba rekordów
- `backtest_results_data_quality_issues.csv` - szczegółowy, nieinwazyjny ślad audytowy per rekord i powód wykluczenia
- `backtest_results_source_ranking.csv` - ranking modeli, analityków i kont społecznościowych
- `backtest_results_source_ranking_by_horizon.csv` - ranking źródeł osobno dla każdego horyzontu
- `backtest_results_confidence_buckets.csv` - statystyki confidence bucketow lacznie dla wszystkich horyzontow
- `backtest_results_confidence_buckets_by_horizon.csv` - statystyki confidence bucketow osobno dla kazdego horyzontu
- `backtest_results_confidence_buckets_ranking.csv` - ranking confidence bucketow dla calego zbioru
- `backtest_results_confidence_buckets_ranking_by_horizon.csv` - ranking confidence bucketow osobno dla kazdego horyzontu
- `backtest_results_best_confidence_buckets_per_horizon.csv` - najlepszy confidence bucket dla kazdego horyzontu
- `backtest_results_ai_analysis.md` - podsumowanie AI (generowane gdy podano `--backtest-ai-analysis` w trybie `api`)

## Wazne mechanizmy runtime

- Trwaly stan aplikacji jest trzymany w `session_state.db` (deduplikacja alertow, stany modułów, cache AI).
- Sygnały transakcyjne sa zapisywane do tabeli `trade_signals` (m.in. `ticker`, `module`, `signal`, `price`, `assumed_entry_price`, data/czas).
- Kazdy `trade_signal` moze miec `confidence_score` (0-100), wyliczany z wag modułu i typu sygnalu.
- Scheduler przeladowuje `config.yaml` w petli, wiec zmiany konfiguracji sa podchwytywane bez restartu procesu.
- Komenda Telegram `/run MODULE` budzi scheduler i wymusza szybkie wykonanie wskazanego modulu.
- AI wspiera dwa tryby: `api` i `prompt` (konfiguracja `ai.request_mode`).

## Confidence Score (wagi sygnalow)

System liczy score sygnalu i zapisuje go w `trade_signals.confidence_score`.

Domyslnie score to:

- `base` za modul (`module_scores`, albo `default_score`)
- korekta za typ sygnalu (`signal_adjustments`)
- korekta za tresc rekomendacji (`recommendation_adjustments`)
- opcjonalnie `signal_strength` i `confidence_boost` z `trade_meta`

Przykladowa konfiguracja w `config.yaml`:

```yaml
signal_confidence:
  enabled: true
  default_score: 50
  min_score: 0
  max_score: 100
  module_scores:
    AT: 62
    AF: 63
    ALERT_PRICE_CHANGE: 60
    RECO: 56
  signal_adjustments:
    STRONG_BUY: 10
    BUY: 5
    SELL: 5
  recommendation_adjustments:
    BUY: 6
    ACCUMULATE: 5
    SELL: 6
    REDUCE: 4
    AVOID: 4
```

Uwagi:

- analyzer moze nadpisac score przez `trade_meta["confidence_score"]`
- score i szczegoly wyliczenia sa zapisywane w DB (`confidence_score`, `confidence_meta`)
- backtest moze filtrowac sygnaly po `--backtest-min-confidence`
- raport backtestu buduje buckety: `00_49`, `50_64`, `65_79`, `80_plus`, `unknown`
- buckety sa raportowane lacznie, per horyzont, oraz rankingowane po `win_rate`, potem po `avg_directional_return_pct`

## 💡 Baza Wiedzy (Mechaniki Wyceny w Systemie)

StockRadar nie jest tylko prostym skanerem wskaźników. Posiada wbudowane mechanizmy chroniące kapitał i faworyzujące zdrowe biznesy rynkowe.

### 1. Spółki Wzrostowe (Growth) vs Value (Ukryty PEG Ratio)
Często spółka z P/E = 10 rosnąca o 40% r/r jest lepszą inwestycją niż spółka z P/E = 5, której zyski spadają. 
Choć system nie raportuje wprost wskaźnika **PEG (Price/Earnings to Growth)**, jego wewnętrzna logika matematycznie go naśladuje. Moduł `FUND_OVERVIEW` przyznaje osobne punkty za "taniość" (Trailing P/E) oraz potężne premie za "dynamikę" (Zysk netto r/r oraz kw/kw). Dodatkowo, wbudowany model 2-fazowego DCF estymuje solidne tempo wzrostu (mediana z dynamiki przychodów i zysków), co naturalnie podbija Wartość Godziwą (Fair Value) dynamicznie rosnących biznesów.

### 2. Rentowność Wolnych Przepływów (FCF Yield)
Zyski księgowe można sztucznie "masować" operacjami papierowymi, ale żywa gotówka na koncie nie kłamie. `FCF Yield` (Free Cash Flow / Market Cap) to bezkompromisowy wskaźnik pokazujący rentowność wolnej gotówki.
- **FCF Yield > 10%:** Wybitnie tania "maszyna gotówkowa". Spółka w teorii mogłaby całkowicie spłacić swoją kapitalizację rynkową w 10 lat z samej nadwyżki finansowej. System mocno premiuje to zjawisko (+15 pkt).
- **FCF Yield < 0%:** Spółka "przepala" gotówkę (mimo np. papierowego zysku netto). System nakłada karę (-5 pkt), chroniąc portfel przed firmami na kroplówce finansowej.

### 3. Ochrona przed Value Traps (Pułapkami Wartości)
Tanie spółki często są tanie nie bez powodu (np. zwijający się biznes, problemy z długiem). StockRadar broni się przed nimi na trzy sposoby:
- **Audyt Piotroskiego:** Bada jakość księgową (m.in. czy zyskom towarzyszą przepływy operacyjne CFO). Jeśli F-Score wynosi 0-4, włącza się ostrzeżenie i punkty za taniość zostają zneutralizowane słabym audytem.
- **Odcięcie Grahama:** Model B. Grahama świetnie wycenia dojrzałe, nudne biznesy, ale zaniża wartość spółek technologicznych i innowacyjnych. System identyfikuje profil spółki i automatycznie ignoruje ten model w agregacji (FV), jeżeli P/E spółki przekracza 25.
- **Bramka Meta-Confluence:** Moduły techniczne (RSI, MACD, wybicia) **nie mogą** wygenerować silnego sygnału `KUPUJ` z poziomu warstwy `META_CONFLUENCE`, jeśli warstwa fundamentalna (FUND) odradza wejście (jest słaba/negatywna). Fundamenty muszą dać "zielone światło" timingowi technicznemu.

### 4. Cache AI i Dynamiczny Upside
Zapytania do LLM (sztucznej inteligencji) o Wartość Godziwą w module `AIFairValueAnalyzer` są kosztowne czasowo i zużywają limity API. Biorąc pod uwagę fakt, że fundamenty spółki zmieniają się zazwyczaj tylko 4 razy w roku (po publikacji raportu kwartalnego), ciągłe odpytywanie AI nie ma sensu.
Dlatego system:
1. Zapisuje (cachuje) surową wycenę dokonaną przez AI (np. na 7-30 dni zgodnie z configiem `fair_value_cache_days`).
2. Przy każdym uruchomieniu skanera błyskawicznie wczytuje te wartości.
3. Pobiera z giełdy **najświeższą cenę w czasie rzeczywistym** i w ułamku sekundy na nowo przelicza aktualny potencjał wzrostu (Upside %). 

Jeżeli wciągu 3 dni spółka urosła o 20%, jej status w cache'u dynamicznie przekształci się z `Undervalued` na `Overvalued`, bez wysyłania zbędnego promtu do serwerów Google Gemini / OpenAI.

### 5. Analiza Techniczna Pivot & Struktura Rynku (TECH_PIVOT)
Moduł ten pozwala na interpretację punktów odniesienia (pivot points), poziomów VWAP, wykrywanie struktury rynku oraz setupów opartych o płynność (liquidity sweeps).

**Podstawy Pivot Points**
Pivot points obliczane są na podstawie High, Low i Close z poprzedniego dnia sesyjnego:
- **Pivot (P)**: `(High + Low + Close) / 3`
- **Resistance 1 (R1)**: `(2 × P) - Low`
- **Support 1 (S1)**: `(2 × P) - High`
- **Resistance 2 (R2)**: `P + (High - Low)`
- **Support 2 (S2)**: `P - (High - Low)`

**Fibonacci (retracement)**
- Dla danych intraday zniesienia Fibo liczone są z aktywnego swinga bieżącej sesji, a nie z poprzedniej świecy referencyjnej.
- Dla swinga wzrostowego poziomy cofnięcia liczone są od `High` w dół, np. `Fib 38.2 = High - 0.382 × (High - Low)`.
- Dla swinga spadkowego poziomy cofnięcia liczone są od `Low` w górę.
- Dashboard i `TECH_PIVOT` używają tej samej logiki, więc wartości na wykresie i w analizie są spójne.
- Dla tickerów intraday można dodatkowo zawęzić źródło swinga do regular session przez parametry `fib_session_timezone`, `fib_session_start`, `fib_session_end` w `config.yaml`.

**Fibonacci (extension)**
- Dostępne są też poziomy `Ext 100%`, `Ext 127.2%`, `Ext 161.8%` liczone z tego samego aktywnego swinga.
- Dla swinga wzrostowego są to projekcje ponad `High`; dla spadkowego poniżej `Low`.
- W `TECH_PIVOT` poziomy extension są logowane i przekazywane w `signal_params`, a dashboard rysuje je na wykresie.

**Poziomy na kolejny dzień (D+1)**
- `TECH_PIVOT` może działać w trybie utrwalonych poziomów D+1 (`pivot_criteria.use_next_day_frozen_levels: true`).
- Po zakończonej sesji wyznaczany jest pakiet poziomów na następny dzień i zapisywany w `app_state` (`data/stock_radar.db`).
- W kolejnym dniu poziomy są odczytywane z cache i nie są rekalkulowane intraday.
- Dashboard korzysta z tej samej ścieżki, więc wartości w UI i sygnałach pozostają spójne.

**VWAP (Volume Weighted Average Price)**
VWAP to średnia cena ważona wolumenem:
- Cena powyżej VWAP = bycze momentum (bullish)
- Cena poniżej VWAP = niedźwiedzie momentum (bearish)
- VWAP działa jako dynamiczne wsparcie/opór.

**Wykrywanie struktury rynku**
- **Higher High (HH)**: Obecny szczyt > poprzedni szczyt
- **Higher Low (HL)**: Obecny dołek > poprzedni dołek
- **Lower High (LH)**: Obecny szczyt < poprzedni szczyt
- **Lower Low (LL)**: Obecny dołek < poprzedni dołek

**Liquidity Sweep Setup**
Zebranie płynności (sweep) występuje, gdy cena przebija kluczowy poziom na wysokim wolumenie i szybko zawraca:
- **Bullish Sweep**: Przebicie oporu w dół z wysokim wolumenem i powrót.
- **Bearish Sweep**: Przebicie wsparcia w górę z wysokim wolumenem i odrzucenie.

**Logika Generowania Setupów (v2)**
1. **SHORT_ATH** (Short z ekstremum): Cena blisko ATH lub wybicie nowego high i odrzucenie. Target: Pivot, S1.
2. **SHORT_R1** (Short z oporu): Test poziomu R1 z odrzuceniem (knotem). Target: Pivot, S1.
3. **LONG_S1** (Long ze wsparcia S1): Cena testuje S1 i odbija. Target: Pivot, R1.
4. **LONG_PIVOT** (Long z pivota - pullback): Pullback do pivota z pomyślnym retestem. Target: Szczyt dnia, ATH.
5. **FAKE_BIAS_SHORT** (Odrzucenie byczej struktury): Struktura LH (niższy szczyt) i nieudany retest pivota. Target: S1.

*Priorytet:* Konkretne setupy mają zawsze wyższy priorytet niż ogólny kierunek (bias). Sygnały BULLISH/BEARISH są wysyłane tylko wtedy, gdy nie wykryto dedykowanego setupu.

**Zarządzanie Ryzykiem i Odczyt**
- Wykorzystuj poziomy R1/R2 i S1/S2 jako cele zysku (take profit) i poziomy obronne (stop loss).
- Sugerowane R/R (Zysk do Ryzyka): Minimum 2:1.
- Wyjście z pozycji (invalidacja): Zamknięcie ceny poniżej pivota (dla pozycji long) lub powyżej pivota (dla pozycji short).

## Wymagane sekcje config.yaml

Minimalnie plik `config.yaml` powinien zawierac:

- `active_modules`
- `module_schedules`
- `tickers`

Sekcje opcjonalne (zalecane):

- `ai`
- `fundamental_criteria`
- `volume_criteria`
- `gap_criteria`
- `divergence_criteria`
- `ma_crossover_criteria`
- `confluence_criteria`
- `support_bounce_criteria`
- `adx_criteria`
- `relative_strength_criteria`
- `pead_criteria`
- `espi_blacklist`
- `disclaimer_message`

## Zrodla danych

- PAP Biznes RSS dla newsow ESPI
- KNF dla krotkiej sprzedazy
- BiznesRadar dla rekomendacji analitycznych
- Yahoo Finance dla danych rynkowych i czesci danych fundamentalnych
- Google Gemini dla funkcji AI

## Disclaimer

Projekt sluzy do uzytku prywatnego i edukacyjnego. Generowane alerty, analizy i odpowiedzi AI nie stanowia rekomendacji inwestycyjnych.
