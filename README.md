# ☁️ Automated Weather Forecast Emailer (n8n + Meteomatics)

Get hyperlocal weather forecasts delivered to your inbox every morning using [n8n](https://n8n.io) and the [Meteomatics Weather API](https://www.meteomatics.com/en/api/).
No backend. No frontend. Just visual low-code automation + JavaScript snippets.

---

## 🧱 What You'll Build

* ✅ Pull high-resolution 48-hour weather data (temperature, wind, precipitation)
* ✅ Parse forecasts into “Today” & “Tomorrow” using JavaScript
* ✅ Generate a clean, emoji-enhanced HTML summary
* ✅ Automatically send the forecast via Gmail
* ✅ 100% customizable, extendable, and reliable

---

## 🚀 Prerequisites

* [n8n](https://n8n.io) installed and running
* Meteomatics Weather API credentials (get a free trial [here](https://www.meteomatics.com/en/api/))
* Gmail account with OAuth2 credentials configured in n8n

---

## 🧹 Workflow Overview

### 1. Schedule Trigger

Run the workflow daily at your preferred time using the **Schedule Trigger** node.

---

### 2. HTTP Request Node

Fetch 48-hour forecast data from Meteomatics:

```
GET https://api.meteomatics.com/2025-07-13T00:00:00.000-04:00--2025-07-15T00:00:00.000-04:00:PT15M/t_2m:F,precip_5min:in,wind_speed_FL10:mph/33.7544657,-84.3898151/json?model=mix
```

Authenticated using basic auth with your Meteomatics username and password.

---

### 3. JavaScript Function: Split Forecast by Day

```
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

  const precipChance = precipArr.filter(p => {
    const hr = new Date(p.date).getHours();
    return (startHr < endHr ? hr >= startHr && hr < endHr : hr >= startHr || hr < endHr) && p.value > 0;
  }).length > 0 ? 5 : 0;

  return {
    temp: avgTemp ? Math.round(avgTemp) : null,
    summary: "Clear",
    precipChance
  };
}

return [{
  json: {
    location: "Atlanta, GA",
    today: buildDay(todayStr),
    tomorrow: buildDay(tomorrowStr)
  }
}];
```

---

### 4. JavaScript Function: Generate HTML Forecast Summary

```
const forecast = $json;

function formatTime(iso) {
  const d = new Date(iso);
  const hr = d.getHours();
  const min = d.getMinutes().toString().padStart(2, '0');
  const ampm = hr >= 12 ? 'PM' : 'AM';
  const hr12 = hr % 12 || 12;
  return `${hr12}:${min} ${ampm}`;
}

function formatDate(iso) {
  return new Date(iso).toLocaleDateString('en-US', {
    weekday: 'long',
    month: 'short',
    day: 'numeric'
  });
}

function buildDayBlock(label, data) {
  return [
    `🎯 **${forecast.location}** — **${label}** (${formatDate(data.highTemp.time)})`,
    `🔺 High: ${data.highTemp.value}°F at ${formatTime(data.highTemp.time)}`,
    `🔻 Low: ${data.lowTemp.value}°F at ${formatTime(data.lowTemp.time)}`,
    `💨 Avg Wind: ${data.wind.averageMPH} mph`,
    `🌧️ Precip: ${data.precipitation.totalIn}"` + 
      (data.precipitation.totalIn > 0 ? ` at ${data.precipitation.times.map(formatTime).join(', ')}` : ' (None)'),
    ``,
    `🌅 Morning: ${data.forecastByPeriod.morning.temp}° — ${data.forecastByPeriod.morning.summary}`,
    `☀️ Afternoon: ${data.forecastByPeriod.afternoon.temp}° — ${data.forecastByPeriod.afternoon.summary}`,
    `🌇 Evening: ${data.forecastByPeriod.evening.temp}° — ${data.forecastByPeriod.evening.summary}`,
    `🌙 Overnight: ${data.forecastByPeriod.overnight.temp}° — ${data.forecastByPeriod.overnight.summary}`
  ];
}

const htmlLines = [
  ...buildDayBlock("Today", forecast.today),
  ``,
  ...buildDayBlock("Tomorrow", forecast.tomorrow)
];

const summaryHtml = htmlLines.join('<br>');

return [{ json: { summary: summaryHtml } }];
```

---

### 5. Gmail Node

* **Email Type**: `HTML`
* **Message Body**:

```
{{ $json.summary }}
```

---

## 💬 Feedback or Questions?

Feel free to [open an issue](https://github.com/your-username/your-repo/issues) or contact the Meteomatics API team directly.

---

## 🏷 Tags

`#weatherapi` `#meteomatics` `#n8n` `#lowcode` `#automation` `#gmailworkflow` `#forecast` `#html` `#javascript`
