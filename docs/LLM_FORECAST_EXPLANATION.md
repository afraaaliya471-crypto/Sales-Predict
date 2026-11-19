# LLM-Based Forecast Explanation

## 🎯 Overview

The system now uses a **hybrid approach** for forecast explanations:
1. **Statistical models** (Prophet, ETS) generate numerical forecasts
2. **LLM analyzes the data** and generates natural language explanations

## 🔄 How It Works

```
User Query: "forecast sales next 6 months"
     ↓
Prophet Model Runs
  • Generates forecast: [Sep: 2100, Oct: 1838, Nov: 5200, ...]
  • Historical data: [Sep: 1500, Oct: 1600, Nov: 4800, ...]
  • YoY changes: [+40%, +15%, +8%, ...]
     ↓
Data Packaged for LLM
  • Model accuracy (WAPE)
  • Historical performance summary
  • Forecast values
  • YoY comparisons
  • Peak/trough months
     ↓
LLM Receives Prompt:
  "Here is the forecast data:
   - Model: Prophet
   - Accuracy: 117% WAPE
   - Historical: Sep 1500, Oct 1600, Nov 4800...
   - Forecast: Sep 2100 (+40%), Oct 1838 (+15%)...
   
   Analyze this and explain."
     ↓
LLM Generates Insights:
  - "Prophet selected due to best fit on monthly WAPE..."
  - "Strongest month was Nov 2025 (5,258 units)..."
  - "Recent YoY trend shows -29% avg decline..."
     ↓
User sees forecast chart + LLM explanation
```

**The LLM doesn't need to "read data beforehand" — it reads it FROM THE PROMPT!**

## 📊 Example Prompt to LLM

```
You are a sales forecasting analyst. Analyze the following forecast data and provide clear, factual insights.

**FORECAST DATA:**
- Model: Prophet
- Selection Reason: best fit (monthly WAPE, last 112 days)
- Accuracy (WAPE): 117.0%
- Forecast Periods: 8

**HISTORICAL PERFORMANCE (Prior Year):**
  Sep 2025: 1500 units
  Oct 2025: 1600 units
  Nov 2025: 4800 units
  ...
- Average: 2226 units/month
- Strongest: Nov 2025 (4800 units)
- Weakest: Jun 2025 (0 units)

**FORECAST (This Year):**
  Sep 2026: 2100 units (YoY: +40%)
  Oct 2026: 1838 units (YoY: +15%)
  Nov 2026: 5200 units (YoY: +8%)
  ...
- Average: 2500 units/month
- Overall Change: +12.3% vs prior year

**YOUR TASK:**
Write a concise, factual analysis in 3-5 bullet points:
1. Why this forecast method was selected
2. Key trends (strongest/weakest months, YoY patterns)
3. Notable changes vs last year (be specific with numbers)

**RULES:**
- Be factual and specific (cite numbers and months)
- Don't invent business reasons (e.g., don't say "due to holiday shopping" unless you know)
- Focus on patterns in the DATA, not external causes
- Keep each point to 1-2 sentences
```

## 🎛️ Configuration

### Enable/Disable LLM Explanations

In `query_engine.py`:

```python
chart_data = build_demand_planning_view(
    historical,
    forecast,
    window["start"],
    window["end"],
    use_llm_explanation=True,   # ← Set to False for Python templates
    llm_model="deepseek-r1:7b", # ← Choose which LLM to use
)
```

### Model Selection

- **`deepseek-r1:7b`** (default): Best for reasoning and analysis
- **`qwen2.5:7b`**: Faster, but less analytical depth
- **`llama3.2:3b`**: Lightweight fallback

## ✅ Benefits

### vs Pure Python Templates:
- ✓ **More natural language** (not rigid templates)
- ✓ **Adapts to data patterns** dynamically
- ✓ **Can identify nuances** humans might spot

### vs Pure LLM Forecasting:
- ✓ **No hallucination** (Prophet/ETS generate actual numbers)
- ✓ **Auditable** (statistical models are deterministic)
- ✓ **Explainable** (we know why the model made its choice)

## 🛡️ Safety Features

1. **Fallback**: If LLM fails, automatically uses Python templates
2. **Validation**: LLM insights are parsed and sanitized
3. **No business assumptions**: LLM is instructed to only analyze DATA patterns
4. **Temperature 0.3**: Lower temperature for factual, less creative responses

## 📝 Example Output

**Before (Python Template):**
```
- Method: Prophet — fitted trend + yearly/weekly seasonality on daily sold units.
- Selection: best fit (monthly WAPE, last 112 days); holdout WAPE 117%.
- In history, strongest month was Nov 2025 (5,258 units); weakest was Jun 2025 (0 units).
```

**After (LLM Analysis):**
```
- Prophet was selected for its superior fit on monthly WAPE validation, capturing 
  both trend and seasonality in the historical data with 117% WAPE accuracy.
- Historical data shows Nov 2025 as the peak month (5,258 units), while Jun 2025 
  had zero sales, indicating strong seasonal variation.
- Year-over-year trends reveal a recent 29% average decline, with only 2 of the 
  last 6 months showing growth compared to the prior year.
```

## 🧪 Testing

Test with any forecast query:
```
User: "forecast sales next 6 months"
User: "predict December 2026 sales"
User: "forecast top 3 items for next quarter"
```

The system will:
1. Generate forecast using Prophet/ETS
2. Pass data to LLM
3. Display chart + LLM insights

## 🔧 Implementation Files

- `src/core/forecast_service.py`:
  - `explain_forecast_with_llm()` — new LLM-based explanation
  - `explain_forecast_basis()` — legacy Python templates (fallback)
  - `build_demand_planning_view()` — updated to support both modes

- `src/core/query_engine.py`:
  - Updated calls to `build_demand_planning_view()`

## 🚀 Future Improvements

- [ ] Cache LLM explanations for identical forecast patterns
- [ ] Add user preference for explanation style (technical vs business-friendly)
- [ ] A/B test LLM explanations vs Python templates for user satisfaction
- [ ] Experiment with embedding-based retrieval of similar past forecasts

---

**Key Insight:** The LLM doesn't forecast — it *explains* the forecast that statistical models already generated. This gives us the best of both worlds: **statistical rigor + natural language fluency**.
