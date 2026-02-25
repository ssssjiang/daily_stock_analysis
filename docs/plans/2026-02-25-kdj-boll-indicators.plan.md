# Plan: Add KDJ and Bollinger Bands Indicators

**Status:** DEFERRED (put on hold, pending future scheduling)
**Date:** 2026-02-25
**Author:** Discussion between user and AI assistant

---

## Background

The current system provides three technical indicators: MA (Moving Average), MACD, and RSI.
This plan proposes adding KDJ and Bollinger Bands as two additional classic indicators
to provide more dimensional analysis signals.

---

## Current Architecture

The full indicator pipeline:

```
Calculation Layer (src/stock_analyzer.py)
  _calculate_mas() / _calculate_macd() / _calculate_rsi()
     ↓
Analysis Layer (src/stock_analyzer.py)
  _analyze_trend() / _analyze_macd() / _analyze_rsi()
     ↓
Result Structure: TrendAnalysisResult (src/stock_analyzer.py)
     ↓
Context Integration: pipeline.py::_enhance_context()
     ↓
AI Analysis: analyzer.py + Report Output: notification.py
```

New indicators must be integrated into this complete pipeline.

---

## Indicator Specifications

### KDJ

- **Formula:** Based on high/low/close prices, compute RSV first, then smooth via EWM to get K and D, finally `J = 3K - 2D`
- **Standard Parameters:** RSV window = 9, K/D smoothing factor = 1/3
- **Signal Semantics:**
  - J < 20: Oversold zone (potential buy opportunity)
  - J > 80: Overbought zone (potential sell risk)
  - K crosses above D (golden cross): Buy signal
  - K crosses below D (death cross): Sell signal
- **Minimum data requirement:** ~20 candles (9 + warm-up)

### Bollinger Bands (BOLL)

- **Formula:** `MID = MA20`, `UPPER = MA20 + 2σ`, `LOWER = MA20 - 2σ`
- **Standard Parameters:** MA period = 20, standard deviation multiplier = 2
- **Signal Semantics:**
  - Price touching/breaking upper band: Overbought or strong breakout
  - Price touching/breaking lower band: Oversold or weak breakdown
  - Bandwidth squeeze (narrow band): Low volatility, signals imminent trend change
  - `%B` metric (0~1 range): Relative position of price within the Bollinger Bands
- **Minimum data requirement:** 20 candles

---

## Proposed Change Scope (Minimal)

| Change Point | File | Content |
|---|---|---|
| Calculation layer | `src/stock_analyzer.py` | Add `_calculate_kdj()` and `_calculate_boll()` |
| Analysis layer | `src/stock_analyzer.py` | Add `_analyze_kdj()` and `_analyze_boll()` |
| Result structure | `src/stock_analyzer.py` | Add KDJ/BOLL fields to `TrendAnalysisResult` |
| Call integration | `src/stock_analyzer.py` | Call new methods in `analyze()` main flow |
| Context integration | `src/core/pipeline.py` | Pass new fields in `_enhance_context()` |
| Class constants | `src/stock_analyzer.py` | Add KDJ/BOLL parameter constants |

**Not affected:** `analyzer.py` (AI will automatically use new fields from context), `notification.py` (report format unchanged for now), Agent tool layer (existing `analyze_trend_tool` will pass through new fields).

---

## Open Design Decisions (Pending Confirmation)

The following questions were raised during the initial discussion and need answers before implementation begins:

### Q1: Should KDJ parameters be configurable?
- **Option A (tentative preference):** Hardcode as class constants (`KDJ_PERIOD = 9`). Simple and direct; industry-standard values rarely need adjustment.
- **Option B:** Expose via `.env` / `Config`. Flexible but adds configuration complexity.

### Q2: How to define the Bollinger Band "squeeze" threshold?
- When `band_width_pct` (bandwidth / midline) falls below a threshold, consider it a squeeze.
- Should this be hardcoded (e.g., 5%) or configurable?

### Q3: Field granularity in `TrendAnalysisResult`?
- **Option A (recommended):** Store key numeric values (`kdj_k`, `kdj_d`, `kdj_j`, `boll_upper`, `boll_mid`, `boll_lower`, `boll_bandwidth`) + enum status (`kdj_status`, `boll_status`), consistent with existing MACD/RSI style.
- **Option B:** Store only enum status. Saves fields but reduces information; AI cannot perform numeric reasoning.

### Q4: Should `notification.py` report be updated simultaneously?
- **Option A:** Do not modify report format in this PR; let AI use new fields freely.
- **Option B:** Explicitly display KDJ/Bollinger Bands values in the technical analysis section of the report.

### Q5: Unit tests?
- Recommendation: Write one pytest each for `_calculate_kdj()` and `_calculate_boll()`, validating edge cases (insufficient data, all-up, all-down candles, etc.).

---

## Implementation Steps (when resumed)

1. Confirm answers to the 5 design decisions above
2. Add `_calculate_kdj()` to `src/stock_analyzer.py`
3. Add `_calculate_boll()` to `src/stock_analyzer.py`
4. Extend `TrendAnalysisResult` with new fields
5. Add `_analyze_kdj()` and `_analyze_boll()` signal analysis methods
6. Integrate new methods into `analyze()` main flow
7. Update `_enhance_context()` in `pipeline.py`
8. Write unit tests
9. Run `./test.sh syntax` and `flake8` to validate
10. Update `README.md` and `docs/CHANGELOG.md`
