# StockRadar

StockRadar to prywatny radar giełdowy dla GPW i wybranych spółek zagranicznych. System pobiera dane z wielu źródeł, analizuje je modułowo i wysyła alerty do Telegrama.

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
| `FUND_OVERVIEW` | Fundamentalna | BUY / HOLD / WAIT (scoring 0–100) |
| `FUND_SCORE_PIOTROSKI` | Fundamentalna | STRONG (7-9) / WEAK (0-4) |
| `FUND_EARNINGS_DRIFT` | Fundamentalna | POSITIVE DRIFT / NEGATIVE DRIFT |
| `META_CONFLUENCE` | Meta | STRONG_BULLISH / BULLISH / STRONG_BEARISH / BEARISH (MIXED = brak emisji sygnalu) |
| `ALERT_PRICE_CHANGE` | Alert | UP / DOWN |
| `ALERT_PRICE_LEVEL` | Alert | BUY / SELL |
| `ALERT_RECOMMENDATIONS` | Alert | BUY / SELL / HOLD |
| `ALERT_ESPI` | Alert | informacyjny |
| `ALERT_KNF_SHORT` | Alert | SHORT (zmiana pozycji) |
| `ALERT_CALENDAR` | Alert | informacyjny |
| `REPORT_AI_RECOMMENDATIONS` | Raport (AI) | KUP / TRZYMAJ / OMIJAJ |
| `REPORT_AI_DAILY_PICK` | Raport (AI) | pick dnia |
| `REPORT_MORNING_BRIEF` | Raport | informacyjny |

### Telegram

System wysyła alerty na Telegram i może uruchomić interaktywnego bota.

Obsługiwane komendy:

- `/cena TICKER`
- `/analiza TICKER`
- `/wycena TICKER`
- `/arkusz TICKER`
- `/fundamenty TICKER`
- `/run MODULE_NAME` — natychmiastowe uruchomienie wybranego modułu (np. `/run TECH_DIVERGENCE`)

Bot ma blokadę wielokrotnego uruchomienia i zabezpieczenie przed konfliktem `409` przy `getUpdates`.

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

Wykrywa anomalie wolumenowe względem historycznej średniej.

**Wskaźniki:** wolumen bieżący, SMA wolumenu (20 sesji)

| Sygnał | Warunki |
|--------|--------|
| 🟢 BUY (BULLISH) | Wolumen ≥ 300% MA20 AND świeca zielona (Close > Open) |
| 🔴 SELL (BEARISH) | Wolumen ≥ 300% MA20 AND świeca czerwona (Close < Open) |
| ⚪ WAIT | Brak skoku wolumenu |

Próg (domyślnie 300%) konfigurowalny przez `volume_criteria.threshold_pct`.

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

Wykrywa i śledzi luki cenowe na interwale H1.

**Dane:** H1 OHLC, SQLite (historia luk)

| Sygnał | Warunki |
|--------|--------|
| ALERT (GAP UP) | Open > Close poprzedniej świecy o ≥ `min_gap_pct` |
| ALERT (GAP DOWN) | Open < Close poprzedniej świecy o ≤ `-min_gap_pct` |
| 🎯 FILLED | Cena wróciła do strefy luki (luka wypełniona) |

Domyślna minimalna luka: 1.5%, konfigurowalny przez `gap_criteria.min_gap_pct`.

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

### FUND_OVERVIEW

Ocenia kondycję fundamentalną spółki systemem punktowym (0–100+).

**Dane:** Yahoo Finance — cena, C/Z, P/B, ROE, D/E, FCF, wyniki finansowe, historia dywidend, cele analityków

#### Tabela punktowa

| Kryterium | Maks. pkt | Logika |
|-----------|-----------|--------|
| Stopa dywidendy | 20 | ≥ 6%: 20 pkt \| ≥ 3%: 10 pkt |
| Regularne dywidendy (3 lata) | 5 | Bonus: dywidenda > 0 każdego roku |
| C/Z (trailing P/E) | 20 | < 8: 20 \| < 12: 15 \| < 15: 10 \| < 20: 5 |
| DCF upside | 15 | ≥ 30%: 15 \| ≥ 10%: 10 \| > 0: 5 |
| Cel analityków (upside) | 10 | ≥ 20%: 10 \| ≥ 10%: 7 \| > 0: 3 |
| Zysk netto r/r | −5 do +10 | ≥ 10%: 10 \| ≥ 0%: 5 \| spadek: −5 |
| ROE | 10 | ≥ 15%: 10 \| ≥ 10%: 5 |
| Dług/Kapitał (D/E) | 10 | < 50%: 10 \| < 100%: 5 |
| Current Ratio | 5 | ≥ 1.5: 5 \| ≥ 1.0: 2 |
| Kwartalny zysk netto q/q | 5 | ≥ 20%: 5 \| ≥ 0%: 2 \| spadek: 0 |

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

1. Warstwa 1 (decyzja TAK/NIE): kategorie fundamentalne, np. Piotroski / PEAD / FUND_OVERVIEW / FUND_AI_FAIR_VALUE.
2. Warstwa 2 (timing): momentum i wejście techniczne, np. RS / ADX / Bollinger / Support Bounce.

Sygnał końcowy jest liczony jako ważona kompozycja kategorii (domyślnie FUND 50%, MOMENTUM 30%, TECH_ENTRY 20%).
Warstwa 1 pełni rolę bramki: jeśli FUND nie potwierdza kierunku, sygnał jest blokowany.

**Dane:** tabela `trade_signals` w SQLite (ostatnie N dni)

| Sygnał | Warunki |
|--------|--------|
| 🟢🟢 STRONG BULLISH | Composite bullish, mocny wynik i brak istotnej kontrstrony |
| 🟢 BULLISH | Composite bullish, przekroczony próg |
| 🔴🔴 STRONG BEARISH | Composite bearish, mocny wynik i brak istotnej kontrstrony |
| 🔴 BEARISH | Composite bearish, przekroczony próg |
| ⚪ BRAK SYGNAŁU | Wynik poniżej progu lub blokada przez Layer 1 |

Klasyfikacja sygnałów:
- **Bycze:** BUY, STRONG BUY, STRONG (Piotroski), OUTPERFORM (RS), POSITIVE_DRIFT (PEAD), STRONG_UP (ADX), HOLD
- **Niedźwiedzie:** SELL, WEAK (Piotroski), UNDERPERFORM (RS), NEGATIVE_DRIFT (PEAD), STRONG_DOWN (ADX)

Konfiguracja (nowa): `confluence_criteria.lookback_days`, `module_weights`, `module_category_map`, `category_weights`, `layer1_categories`, `layer1_min_score`, `composite_threshold`.

Fallback: jeśli nie zdefiniujesz mapowania kategorii (`module_category_map` + `category_weights`), moduł przechodzi do starszego trybu sumowania wag (`min_signals`).

#### Jak interpretować komunikaty na konsoli (META_CONFLUENCE)

- `🟢 BULLISH` / `🟢🟢 STRONG_BULLISH` = praktyczny odpowiednik kierunku BUY po agregacji sygnałów (nie jest to literalny sygnał `BUY`, tylko sygnał meta).
- `🔴 BEARISH` / `🔴🔴 STRONG_BEARISH` = praktyczny odpowiednik kierunku SELL/AVOID po agregacji.
- `⚪ BELOW THRESHOLD` = composite nie przekroczył `composite_threshold`; traktuj jako brak przewagi (`NO-TRADE`).
- `🚫 L1 GATE` = warstwa FUND (Layer 1) zablokowała sygnał mimo wskazań warstwy technicznej.

Przykład:

- `Bullish blocked — FUND: +0.00 ≤ 0.00` oznacza, że FUND nie potwierdził kierunku long przy `layer1_min_score: 0.0`.
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

Pobiera i klasyfikuje rekomendacje analityczne z BiznesRadar.

**Dane:** BiznesRadar (cache 1h), cele cenowe

Klasyfikacja rekomendacji:

| Sygnał | Słowa kluczowe |
|--------|---------------|
| 🟢 BUY | KUPUJ, AKUMULUJ, PRZEWAŻAJ, BUY, OVERWEIGHT |
| 🔴 SELL | SPRZEDAJ, REDUKUJ, NIEDOWAŻAJ, SELL, UNDERWEIGHT |
| 🟡 HOLD | Pozostałe |

Alerty Telegram tylko dla rekomendacji z dzisiaj lub wczoraj.

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

Generuje jeden "pick dnia" — najlepszą spółkę z listy.

**Model:** Google Gemini (raz dziennie, o skonfigurowanej godzinie)

- Analizuje wszystkie tickery z listy i wybiera jeden
- Działa w trybie `api` (Gemini) lub `prompt` (gotowy prompt na Telegram)
- Wynik cachowany — nie odpala się drugi raz tego samego dnia

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
```

Obsługiwane pola zależne od modułu:

- `keywords`
- `isin`
- `buy_alert`
- `sell_alert`
- `buy_alert_action` — akcja po osiągnięciu poziomu buy (`once`, `once_daily`, `adjust:...`)
- `sell_alert_action` — akcja po osiągnięciu poziomu sell (`once`, `once_daily`, `adjust:...`)
- `alert_threshold`

## Konfiguracja analizatorów

### Luki cenowe (TECH_GAPS)

```yaml
gap_criteria:
  min_gap_pct: 1.5  # Minimalna wielkość luki (%)
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
  layer1_min_score: 0.0
  composite_threshold: 0.2
  category_weights:
    FUND: 0.50
    MOMENTUM: 0.30
    TECH_ENTRY: 0.20
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
python stock_radar.py
```

Tryb harmonogramu:

```bash
python stock_radar.py --schedule
```

Tryb harmonogramu z wymuszeniem ignorowania okien czasowych:

```bash
python stock_radar.py --schedule --ignore-schedule
```

Analiza tylko wybranych spolek:

```bash
python stock_radar.py --ticker XTB.PL,CDR.PL
```

Uruchomienie tylko wybranych modulow:

```bash
python stock_radar.py --modules FEED_ESPI,FEED_KNF_SHORT,FEED_RECOMMENDATIONS
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
- `backfill gaps`: po podaniu `--backfill-gaps` aplikacja wykona backfill i zakonczy proces
- `backtest trade_signals`: po podaniu `--backtest-trade-signals` aplikacja uruchomi backtest i zakonczy proces

## Wszystkie argumenty CLI

- `--schedule` - uruchamia petle harmonogramu
- `--ignore-schedule` - ignoruje `active_hours` i uruchamia aktywne moduly niezaleznie od okien czasu
- `--silent` - tryb cichy: brak logow konsolowych i brak powiadomien Telegram; sygnaly i dane dalej trafiaja do SQLite
- `--no-session` - resetuje i pomija trwały stan z `session_state.db`
- `--ticker <T1,T2,...>` - ogranicza analizę do wybranych tickerow (np. `PKO.PL,MSFT.US`)
- `--modules <M1,M2,...>` - wlacza tylko podane moduly (pozostale sa tymczasowo wylaczane)
- `--backfill-gaps` - uruchamia backfill historii luk cenowych (`TECH_GAPS`)
- `--backfill-period <PERIOD>` - okres backfillu, np. `3mo`, `6mo`, `1y` (domyslnie `1y`)
- `--backtest-trade-signals` - uruchamia backtest na tabeli `trade_signals`
- `--backtest-horizons <D1,D2,...>` - globalne horyzonty oceny w dniach, np. `1,7,30,90`; gdy parametr nie jest podany, backtest bierze `backtest_horizons` z konfiguracji modułu
- `--backtest-dedup-days <D>` - okno deduplikacji sygnalow dla pary `(ticker, module, signal)`; domyslnie `7`, `0` wylacza deduplikacje
- `--backtest-success-threshold <P>` - prog sukcesu w procentach; transakcja jest liczona jako trafiona, gdy `directional_return_pct > P`; domyslnie `1.0`
- `--backtest-min-confidence <C>` - filtruje backtest do sygnalow z `confidence_score >= C`
- `--backtest-from <YYYY-MM-DD>` - data poczatkowa filtrowania sygnalow
- `--backtest-to <YYYY-MM-DD>` - data koncowa filtrowania sygnalow
- `--backtest-modules <M1,M2,...>` - filtr po polu `module` w `trade_signals`
- `--backtest-signals <S1,S2,...>` - filtr po polu `signal` (np. `BUY,SELL`)
- `--backtest-export <PATH.csv>` - eksport CSV wynikow backtestu, rankingow modulow i raportow confidence buckets
- `--backtest-ai-analysis` - po zakonczeniu backtestu wysyla wyniki do AI (Gemini/OpenAI) w celu analizy; w trybie `prompt` dostarcza gotowy prompt na Telegram

## Dostepne moduly (`AnalysisModule`)

- `REPORT_AI_DAILY_PICK`
- `REPORT_AI_RECOMMENDATIONS`
- `ALERT_PRICE_CHANGE`
- `ALERT_PRICE_LEVEL`
- `FEED_CALENDAR`
- `FEED_ESPI`
- `FEED_KNF_SHORT`
- `FEED_RECOMMENDATIONS`
- `FUND_OVERVIEW`
- `FUND_EARNINGS_DRIFT`
- `FUND_SCORE_PIOTROSKI`
- `META_CONFLUENCE`
- `REPORT_MORNING_BRIEF`
- `TECH_ADX`
- `TECH_CANDLESTICK`
- `TECH_DIVERGENCE`
- `TECH_GAPS`
- `TECH_INDICATORS`
- `TECH_MA_CROSSOVER`
- `TECH_REL_STRENGTH`
- `TECH_SUPPORT_BOUNCE`
- `TECH_VOLUME`

Dla `--modules` dzialaja tez legacy aliasy (stare nazwy): `ESPI`, `PRICE_ALERTS`, `TECHNICAL`, `FUNDAMENTAL`, `AI_PICK`, `VOLUME_SPIKES`, `CANDLESTICK_PATTERNS`, `PRICE_GAPS`, `DIVERGENCES`, `MA_CROSSOVERS`, `SUPPORT_BOUNCES`, `CALENDAR_EVENTS`, `KNF_SHORTS`, `RECOMMENDATIONS`, `MORNING_BRIEF` i ich warianty w liczbie pojedynczej.

## Przyklady CLI

```bash
# harmonogram tylko dla wybranych modulow
python stock_radar.py --schedule --modules ESPI,KNF_SHORTS,RECOMMENDATIONS

# cichy run pod task/service - bez konsoli i Telegrama
python stock_radar.py --schedule --silent

# jednorazowy run dla konkretnej listy spolek
python stock_radar.py --ticker XTB.PL,PKO.PL --modules TECH_INDICATORS,ALERT_PRICE_CHANGE

# backfill luk cenowych dla 6 miesiecy
python stock_radar.py --backfill-gaps --backfill-period 6mo --ticker CDR.PL,PKO.PL

# backtest sygnalow BUY/SELL z eksportem CSV
python stock_radar.py --backtest-trade-signals --backtest-horizons 1,7,30,90 --backtest-signals BUY,SELL --backtest-export backtest_results.csv

# backtest tylko dla mocniejszych sygnalow, z deduplikacja i progiem sukcesu 2%
python stock_radar.py --backtest-trade-signals --backtest-min-confidence 60 --backtest-dedup-days 7 --backtest-success-threshold 2 --backtest-export backtest_results.csv

# backtest z analiza AI wynikow (tryb api = odpowiedz z modelu; tryb prompt = gotowy prompt na Telegram)
python stock_radar.py --backtest-trade-signals --backtest-export backtest_results.csv --backtest-ai-analysis
```

## Zasady backtestu trade_signals

- Backtest domyslnie ocenia tylko moduly z `module_role: signal`; alerty i raporty sa pomijane.
- Jezeli nie podasz `--backtest-horizons`, system bierze horyzonty z `modules.<NAZWA>.backtest_horizons`.
- Domyslne horyzonty sa rozdzielone per typ modułu: fundamentalne maja zwykle `30,60,90,180`, a techniczne `1,7,14,30`.
- Deduplikacja działa w ruchomym oknie czasu dla tego samego `(ticker, module, signal)` i zostawia pierwszy sygnal z okna.
- Gdy dla docelowego horyzontu brakuje ceny wyjscia (np. sygnal jest zbyt swiezy), backtest uzupelnia exit ostatnia dostepna cena po dacie wejscia zamiast pomijac rekord.
- Sukces sygnalu liczony jest po `directional_return_pct`, nie po surowym zwrocie; domyslnie potrzeba wyniku powyzej `1.0%`.
- Confidence score moze sluzyc jednoczesnie do filtrowania sygnalow i do raportowania bucketow skutecznosci.

Pliki CSV po eksporcie backtestu:

- `backtest_results.csv` - wszystkie transakcje wynikowe (`outcomes`)
- `backtest_results_summary.csv` - podsumowanie skutecznosci per horyzont
- `backtest_results_by_module.csv` - statystyki modułów per horyzont
- `backtest_results_module_ranking.csv` - ranking ogolny skutecznosci modułów
- `backtest_results_module_ranking_by_horizon.csv` - ranking modułów osobno dla kazdego horyzontu
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
