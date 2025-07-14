# ☁️ Automated Weather Forecast Emailer (n8n + Meteomatics)

Get hyperlocal weather forecasts delivered to your inbox every morning using [n8n](https://n8n.io) and the [Meteomatics Weather API](https://www.meteomatics.com/en/api/).  
No backend. No frontend. Just visual low-code automation + JavaScript snippets.

---

## 🧱 What You'll Build

- ✅ Pull high-resolution 48-hour weather data (temperature, wind, precipitation)
- ✅ Parse forecasts into “Today” & “Tomorrow” using JavaScript
- ✅ Generate a clean, emoji-enhanced HTML summary
- ✅ Automatically send the forecast via Gmail
- ✅ 100% customizable, extendable, and reliable

---

## 🚀 Prerequisites

- [n8n](https://n8n.io) 
- Meteomatics Weather API credentials (get a free trial [here](https://www.meteomatics.com/en/api/))
- Gmail account with OAuth2 credentials configured in n8n

---

## 🧩 Workflow Overview

### 1. Schedule Trigger

Run the workflow daily at your preferred time using the **Schedule Trigger** node.

---

### 2. HTTP Request Node

Fetch 48-hour forecast data from Meteomatics:
Use the URL from our URL Creator


Authenticated using basic auth with your Meteomatics username and password.

---

### 3. JavaScript Function: Split Forecast by Day

```js
const data = $json;

const getParam = (param) =>
  data.data.find(d => d.parameter === param)?.coordinates[0]?.dates || [];

const temps = getParam("t_2m:F");
const precip = getParam("precip_5min:in");
const wind = getParam("wind_speed_FL10:mph");

function groupByDate(entries) {
  return entries.reduce((acc, entry) => {
    const date = entry.date.split("T")[0];
    acc[date] = acc[date] || [];
    acc[date].push(entry);
    return acc;
  }, {});
}

const tempByDate = groupByDate(temps);
const precipByDate = groupByDate(precip);
const windByDate = groupByDate(wind);

const sortedDates = Object.keys(tempByDate).sort();
const [todayStr, tomorrowStr] = sortedDates;

function buildDay(dateStr) {
  const t = tempByDate[dateStr] || [];
  const p = precipByDate[dateStr] || [];
  const w = windByDate[dateStr] || [];

  const maxTemp = t.reduce((max, cur) => (cur.value > max.value ? cur : max), t[0]);
  const minTemp = t.reduce((min, cur) => (cur.value < min.value ? cur : min), t[0]);

  const totalPrecip = p.reduce((sum, entry) => sum + (entry.value || 0), 0);
  const precipTimes = p.filter(e => e.value > 0).map(e => e.date);

  const windValues = w.map(e => e.value);
  const avgWind = windValues.reduce((a, b) => a + b, 0) / windValues.length;

  return {
    highTemp: { value: Math.round(maxTemp.value), time: maxTemp.date },
    lowTemp: { value: Math.round(minTemp.value), time: minTemp.date },
    wind: { averageMPH: parseFloat(avgWind.toFixed(1)) },
    precipitation: {
      totalIn: parseFloat(totalPrecip.toFixed(2)),
      times: precipTimes
    },
    forecastByPeriod: {
      morning: summarizeBlock(t, p, 6, 12),
      afternoon: summarizeBlock(t, p, 12, 18),
      evening: summarizeBlock(t, p, 18, 22),
      overnight: summarizeBlock(t, p, 22, 6)
    }
  };
}

function summarizeBlock(tempArr, precipArr, startHr, endHr) {
  const tempBlock = tempArr.filter(t => {
    const hr = new Date(t.date).getHours();
    return startHr < endHr ? hr >= startHr && hr < endHr : hr >= startHr || hr < endHr;
  });

  const avgTemp = tempBlock.length
    ? tempBlock.reduce((sum, t) => sum + t.value, 0) / tempBlock.length
    : null;

  const precipChance = precipArr.filter(p =>


