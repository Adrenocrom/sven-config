---
name: weather_api_wttr
description: Free weather API using wttr.in — no API key required, supports JSON output
  with current conditions and 3-day forecasts. Includes usage examples and data structure
  documentation.
tags:
- weather
- api
- wttr
- free-api
- curl
- python
created_at: '2026-07-12T18:59:41.009310+00:00'
---

# Weather API: wttr.in

## Overview
**wttr.in** is a free, open-source weather service that requires **no API key**. It provides current conditions and multi-day forecasts via simple HTTP requests. Handles ~100M queries/day with high reliability.

**Primary domain**: `https://wttr.in`  
**Fallback domain**: `https://wttr.is` (fully equivalent)

---

## Quick Start

### Current Weather (Plain Text)
```bash
curl wttr.in/London
curl wttr.in  # auto-detects location via IP
```

### JSON Output (Recommended for Programs)
```bash
curl 'wttr.in/London?format=j1'
# or
curl 'wttr.is/New+York?format=j2'  # j2 = smaller payload (no hourly data)
```

---

## URL Structure

```
https://wttr.in/{LOCATION}?{OPTIONS}
```

### Location Formats
| Format | Example | Notes |
|--------|---------|-------|
| City name | `London` | Works for most cities |
| Spaces | `Salt+Lake+City` or `Salt Lake City` | URL-encode spaces as `+` |
| Airport code | `muc` (Munich) | 3-letter IATA codes |
| Coordinates | `51.508,-0.121` | Lat,lon format |
| IP-based | `@github.com` or just `wttr.in` | Uses geolocation |

### Options
| Option | Description | Example |
|--------|-------------|---------|
| `format=j1` | Full JSON (current + 3-day forecast with hourly data) | `?format=j1` |
| `format=j2` | Compact JSON (no hourly data) | `?format=j2` |
| `format=v2` | Data-rich terminal output (experimental) | `?format=v2` |
| `u` / `m` / `M` | Unit system: USCS / Metric / Metric with m/s wind | `?u`, `?m`, `?M` |
| `lang=xx` | Output language code | `?lang=de` (German) |
| `T` | Plain text (no ANSI colors) | `?T` |

**Multiple options**: Combine without delimiters for single-letter opts, use `&` for long opts:
```bash
curl 'wttr.in/Amsterdam?m2&lang=nl'
```

---

## JSON Response Structure (`format=j1`)

### Top-Level Keys
- `current_condition[]` — Current weather (array of 1 item)
- `nearest_area[]` — Geolocated area info
- `request[]` — Query details
- `weather[]` — Array of 3 daily forecasts

### Current Condition Fields
```json
{
  "temp_C": "27",           // Temperature in Celsius
  "temp_F": "81",           // Temperature in Fahrenheit
  "FeelsLikeC": "26",       // Feels like temperature (°C)
  "humidity": "34",         // Humidity (%)
  "weatherDesc": [{"value": "Sunny"}],  // Weather description
  "windspeedKmph": "24",    // Wind speed (km/h)
  "winddir16Point": "ENE",  // Wind direction (16-point compass)
  "pressure": "1023",       // Pressure (hPa)
  "visibility": "10",       // Visibility (km)
  "uvIndex": "1"            // UV index (0-12+)
}
```

### Daily Forecast Fields (`weather[]`)
```json
{
  "date": "2026-07-12",
  "maxtempC": "29",         // Max temperature (°C)
  "mintempC": "18",         // Min temperature (°C)
  "avgtempC": "23",         // Average temperature (°C)
  "sunHour": "17.0",        // Sun hours
  "uvIndex": "7",           // Max UV index
  "astronomy": [{            // Sunrise/sunset/moon data
    "sunrise": "04:57 AM",
    "sunset": "09:14 PM",
    "moon_phase": "Waning Crescent"
  }],
  "hourly": [...]           // 8 time slots (0000, 0300, ..., 2100)
}
```

### Hourly Forecast Fields
- `tempC`, `tempF` — Temperature
- `FeelsLikeC`, `FeelsLikeF` — Feels like
- `humidity` — Humidity (%)
- `weatherDesc` — Condition description
- `chanceofrain` — Probability of rain (%)
- `precipMM` — Precipitation (mm)
- `winddir16Point`, `windspeedKmph` — Wind info

---

## Usage Examples

### Get Weather for Multiple Cities
```bash
curl 'wttr.in/London:Paris:Berlin?format=j2'
# or using brace expansion in bash:
curl -s 'wttr.in/{London,Paris,Berlin}?format=j2'
```

### One-Line Output (for status bars)
```bash
curl wttr.in/Nuremberg?format=3
# Output: "Nuremberg: 🌦 +11⁰C"

# Custom format with %-notation:
curl 'wttr.in/London?format="%l:+%c+%t\n"'
# %l = location, %c = condition emoji, %t = temperature
```

### Force Metric Units (for US users)
```bash
curl wttr.in/New+York?m
```

### Get Moon Phase
```bash
curl wttr.in/Moon
# Output: 🌖 (current phase)

# For specific date:
curl 'wttr.in/Moon@2024-12-25'
```

---

## Python Usage Example

```python
import requests

def get_weather(city: str, units: str = "m") -> dict:
    """Fetch current weather for a city using wttr.in."""
    url = f"https://wttr.in/{city}?format=j1&{units}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

# Usage
weather = get_weather("London")
current = weather["current_condition"][0]
print(f"London: {current['temp_C']}°C, {current['humidity']}% humidity")
```

---

## Limitations & Caveats

1. **Rate limiting**: No official docs, but heavy usage may trigger blocks. Use `wttr.is` as fallback.
2. **Data source**: Uses WorldWeatherOnline (WWO) backend — verify critical data against primary sources.
3. **Forecast accuracy**: 3-day forecasts are reasonable; beyond that, use dedicated services.
4. **No historical data**: Only current + forecast. For historical weather, use Open-Meteo or Weather Underground.
5. **JSON format changes**: The `weatherCode` field uses WWO's enumeration — check [constants.py](https://github.com/chubin/wttr.in/blob/master/lib/constants.py) for mappings.

---

## Alternatives

| Service | Key Required | Best For |
|---------|--------------|----------|
| **wttr.in** (this skill) | No | Quick lookups, no-auth scripts |
| [Open-Meteo](https://open-meteo.com) | No (non-commercial) | Historical data, high-res forecasts |
| OpenWeatherMap | Yes (free tier: 1000 calls/day) | Production apps needing SLA |

---

## Sources
- Official docs: https://wttr.in/:help
- GitHub repo: https://github.com/chubin/wttr.in
- JSON format reference: https://github.com/chubin/wttr.in#json-output

