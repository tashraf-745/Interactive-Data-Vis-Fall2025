---
title: "Lab 2: Subway Staffing"
toc: true
---


<!-- Import Data -->
```js
const incidents = FileAttachment("./data/incidents.csv").csv({ typed: true })
const local_events = FileAttachment("./data/local_events.csv").csv({ typed: true })
const upcoming_events = FileAttachment("./data/upcoming_events.csv").csv({ typed: true })
const ridership = FileAttachment("./data/ridership.csv").csv({ typed: true })
```

<!-- Include current staffing counts from the prompt -->

```js
const currentStaffing = {
  "Times Sq-42 St": 19,
  "Grand Central-42 St": 18,
  "34 St-Penn Station": 15,
  "14 St-Union Sq": 4,
  "Fulton St": 17,
  "42 St-Port Authority": 14,
  "Herald Sq-34 St": 15,
  "Canal St": 4,
  "59 St-Columbus Circle": 6,
  "125 St": 7,
  "96 St": 19,
  "86 St": 19,
  "72 St": 10,
  "66 St-Lincoln Center": 15,
  "50 St": 20,
  "28 St": 13,
  "23 St": 8,
  "Christopher St": 15,
  "Houston St": 18,
  "Spring St": 12,
  "Chambers St": 18,
  "Wall St": 9,
  "Bowling Green": 6,
  "West 4 St-Wash Sq": 4,
  "Astor Pl": 7
}
```
# NYC Subway Staffing Dashboard — Preparing for Summer 2026

This dashboard brings together NYC Transit’s ridership, incident, and event datasets to support smarter staffing decisions ahead of a busy 2026 summer season.  
Using real operational patterns from Summer 2025 and ten years of incident history, this analysis identifies where crowd surges happen, which stations respond fastest to emergencies, and where limited staffing poses the greatest operational risk.  
By combining past behavior with next year’s event calendar, the dashboard highlights the stations that will need the most support—and pinpoints the single location where added staffing will deliver the biggest impact for safety and service reliability.

---

# When Events Move the City: Understanding Summer Ridership Patterns


This view shows how daily subway ridership changed across Summer 2025 and how those changes line up with local events and the July 15 fare increase.  
Use the dropdown to switch between system-wide ridership and a specific station. The blue line shows total entrances + exits, orange bubbles mark event days (scaled by attendance), and the red dashed line marks the fare change to $3.00.

```js
// All stations from ridership data
const stations = Array.from(new Set(ridership.map(d => d.station))).sort();

// Input
const stationSelection = view(
  Inputs.select(["All Stations", ...stations], {
    label: "Station view"
  })
);
```

```js
// Fare change date
const fareChangeDate = new Date("2025-07-15");

// Use all data or just one station based on the dropdown
const ridershipForView =
  stationSelection === "All Stations"
    ? ridership
    : ridership.filter(d => d.station === stationSelection);

// Compute daily total traffic (entrances + exits) for the selected view
const dailyTotals = Array.from(
  ridershipForView.reduce((map, d) => {
    const key = d.date.toISOString().slice(0, 10);
    const total = (d.entrances ?? 0) + (d.exits ?? 0);
    map.set(key, (map.get(key) ?? 0) + total);
    return map;
  }, new Map()),
  ([date, total]) => ({ date: new Date(date), total })
).sort((a, b) => a.date - b.date);

// Events matching the current selection
const eventsForView =
  stationSelection === "All Stations"
    ? local_events
    : local_events.filter(d => d.nearby_station === stationSelection);

// Lookup map: date (YYYY-MM-DD) -> total traffic on that day
const totalsByKey = new Map(
  dailyTotals.map(d => [d.date.toISOString().slice(0, 10), d.total])
);

// Aggregate events by date, attaching traffic, attendance, and names
const eventPoints = Array.from(
  eventsForView.reduce((map, d) => {
    const key = d.date.toISOString().slice(0, 10);
    const existing =
      map.get(key) ||
      {
        date: new Date(key),
        total: totalsByKey.get(key) ?? null,
        attendance: 0,
        count: 0,
        names: []
      };

    existing.attendance += d.estimated_attendance ?? 0;
    existing.count += 1;
    existing.names.push(d.event_name);

    map.set(key, existing);
    return map;
  }, new Map()),
  ([, value]) => value
);

const maxTotal = dailyTotals.length
  ? Math.max(...dailyTotals.map(d => d.total))
  : 0;

const lastDate = dailyTotals.length
  ? dailyTotals[dailyTotals.length - 1].date
  : fareChangeDate;
```

```js
Plot.plot({
  height: 420,
  marginLeft: 70,
  marginRight: 20,
  marginBottom: 40,
  grid: true,
  x: {
    label: "Date",
    type: "utc"
  },
  y: {
    label: "↑ Total Traffic"
  },
  marks: [
  
    lastDate &&
      Plot.rectX(
        [{ x1: fareChangeDate, x2: lastDate, y1: 0, y2: maxTotal * 1.05 }],
        {
          x1: "x1",
          x2: "x2",
          y1: "y1",
          y2: "y2",
          fill: "#E3EAF5", 
          opacity: 0.7
        }
      ),

 
    Plot.lineY(dailyTotals, {
      x: "date",
      y: "total",
      stroke: "#0039A6", 
      strokeWidth: 2
    }),

 
    eventPoints.length &&
      Plot.dot(eventPoints, {
        x: "date",
        y: "total",
        r: d => Math.max(3, Math.sqrt(d.attendance) / 40),
        fill: "#FF6319", 
        stroke: "white",
        strokeWidth: 1
      }),

    // Tooltip
    eventPoints.length &&
      Plot.tip(
        eventPoints,
        Plot.pointerX({
          x: "date",
          y: "total",
          title: d => {
            const dateLabel = d.date.toLocaleDateString("en-US", {
              month: "short",
              day: "numeric"
            });
            const names = d.names.join(", ");
            return `${dateLabel}
Events: ${names}
Total attendance: ${d.attendance.toLocaleString()}
Total traffic: ${d.total?.toLocaleString() ?? "N/A"}`;
          }
        })
      ),

    // Red vertical line at fare change
    Plot.ruleX([fareChangeDate], {
      stroke: "#D73027", 
      strokeWidth: 2
    }),

    // “Fare increase” label near the bottom
    Plot.text(
      [{ date: fareChangeDate, y: 0, label: "Fare increase" }],
      {
        x: "date",
        y: "y",
        text: "label",
        dy: -12,
        fill: "#D73027",
        textAnchor: "start"
      }
    )
  ].filter(Boolean)
})
```

Ridership in Summer 2025 shows two clear patterns shaped by price and events.  
- The **July 15 fare increase** caused an immediate, sustained **drop in daily traffic** across the system. Before the increase, ridership was higher and more stable; afterward it settled into a lower range.  
- **Local events** consistently drove strong ridership spikes, regardless of the fare change. Larger events produced larger traffic surges, especially at major hubs.  

Medium-traffic stations showed the biggest post-fare decline, while busy stations remained more resilient. Overall, summer ridership was driven by a mix of **fare sensitivity** and **event-driven demand**.












---

# Speed Matters: Station Performance Through Response Times

This view shows the average customer-assistance response time at each station and how performance compares across the system. Stations are sorted from fastest to slowest. Dot position and color indicate response speed, while the dashed line marks the system average. Hovering reveals each station’s exact response time and difference from the overall baseline.


```js
// Mean response time per station 
const stationMeans = Array.from(
  incidents.reduce((map, d) => {
    const s = d.station;
    const arr = map.get(s) || [];
    arr.push(d.response_time_minutes);
    map.set(s, arr);
    return map;
  }, new Map()),
  ([station, values]) => ({
    station,
    mean: values.reduce((a, b) => a + b, 0) / values.length
  })
).sort((a, b) => b.mean - a.mean);

const minMean = Math.min(...stationMeans.map(d => d.mean));
const maxMean = Math.max(...stationMeans.map(d => d.mean));

const overallMean =
  incidents.reduce((sum, d) => sum + d.response_time_minutes, 0) /
  incidents.length;


```

```js
Plot.plot({
  height: 520,
  marginLeft: 160,
  marginRight: 40,
  marginBottom: 40,
  x: {
    label: "Average response time (minutes)",
    domain: [0, maxMean + 2]
  },
  y: {
    label: null,
    domain: stationMeans.map(d => d.station) 
  },
  color: {
    type: "linear",
    domain: [minMean, maxMean],
    range: ["#9EC9FF", "#0039A6"] 
  },
  marks: [

    Plot.ruleX(stationMeans, {
      x1: 0,
      x2: "mean",
      y: "station",
      stroke: "#d0d7e5"
    }),


    Plot.dot(stationMeans, {
      x: "mean",
      y: "station",
      r: 7,
      fill: "mean",
      stroke: "#0039A6",
      strokeWidth: 1
    }),

    // System average reference line
    Plot.ruleX([overallMean], {
      stroke: "#0039A6",
      strokeDasharray: "4,2",
      strokeWidth: 1.5
    }),

    // Tooltip 
    Plot.tip(
      stationMeans,
      Plot.pointerX({
        x: "mean",
        y: "station",
        title: d => {
          const diff = d.mean - overallMean;
          const sign = diff > 0 ? "+" : "";
          return `${d.station}
Avg response time: ${d.mean.toFixed(1)} min
vs system average: ${sign}${diff.toFixed(1)} min`;
        }
      })
    )
  ]
})

```

- **Most stations outperform the system average**, clustering **between 5–9 minutes**, indicating strong response efficiency across much of the network.

- A handful of stations, particularly those **above ~17–20 minutes**, show significantly slower response times, suggesting operational bottlenecks or staffing challenges.

- Faster-performing stations are distributed across the city, meaning **high responsiveness is not limited to a single borough or line**.

- Slower stations stand out clearly on the right, making them ideal priority targets for **staffing adjustments, training, or incident-process improvements**.


---










# Anticipating the Surge: Stations Facing the Greatest 2026 Pressure

This chart ranks stations by **Staffing Pressure Index**, expected 2026 event attendance divided by current staffing. Bars show relative strain: longer bars = greater pressure. Top three stations are highlighted. Hover to view exact attendance, staffing, and pressure values. This helps quickly identify where reinforcement is most urgent.

```js
// 2026 event load per station 
const eventLoadByStation = Array.from(
  upcoming_events.reduce((map, d) => {
    const s = d.nearby_station;
    const current = map.get(s) ?? 0;
    map.set(s, current + (d.expected_attendance ?? 0));
    return map;
  }, new Map()),
  ([station, totalAttendance]) => ({ station, totalAttendance })
);

// Combine with current staffing and compute a pressure index
// Pressure index = total expected attendance / current staff
const stationPressure = eventLoadByStation
  .map(d => {
    const staff = currentStaffing[d.station] ?? 0;
    const pressure =
      staff > 0 ? d.totalAttendance / staff : d.totalAttendance; 
    return { ...d, staff, pressure };
  })
  .sort((a, b) => b.pressure - a.pressure); 

// Tag top 3 stations
stationPressure.forEach((d, i) => {
  d.rank = i + 1;
  d.isTop3 = i < 3;
});

// Convenience max for axis padding
const maxPressure = Math.max(...stationPressure.map(d => d.pressure));
```

```js
Plot.plot({
  height: 480,
  marginLeft: 170,
  marginRight: 40,
  marginBottom: 40,
  x: {
    label: "Staffing pressure index (expected 2026 attendees per staff)",
    domain: [0, maxPressure * 1.1]
  },
  y: {
    label: null,
    domain: stationPressure.map(d => d.station) // sorted worst → best
  },
  marks: [
    // Bars: overall staffing pressure
    Plot.barX(stationPressure, {
      x: "pressure",
      y: "station",
      fill: d => (d.isTop3 ? "#0039A6" : "#C7D8F5"), 
      inset: 0.5
    }),

    // Rank labels for the top 3 stations
    Plot.text(
      stationPressure.filter(d => d.isTop3),
      {
        x: d => d.pressure + maxPressure * 0.03,
        y: "station",
        text: d => `#${d.rank}`,
        fill: "#0039A6",
        fontWeight: "bold",
        dy: 3
      }
    ),

    // Tooltip 
    Plot.tip(
      stationPressure,
      Plot.pointerX({
        x: "pressure",
        y: "station",
        title: d =>
          `${d.station}
Expected 2026 event attendance: ${d.totalAttendance.toLocaleString()}
Current staff: ${d.staff}
Pressure index: ${d.pressure.toFixed(1)} attendees per staff`
      })
    )
  ]
})
```
The Staffing Pressure Index reveals a sharp imbalance in how 2026 event traffic will impact the subway system. **Canal St, 34 St–Penn Station, and 23 St face exceptionally high pressure, with far more expected attendees per staff member than any other location—indicating urgent need for reinforcement**. A second tier of stations, including Chambers St, West 4 St–Wash Sq, and Port Authority, show moderate but still concerning strain during peak event periods. Lower-pressure stations remain stable and could serve as a staffing reservoir. Overall, the analysis highlights where targeted staffing adjustments will deliver the greatest operational impact for Summer 2026.

---

# The Top Priority: Where Added Staff Will Make the Biggest Difference

To identify the single station where added staffing will make the biggest impact, three operational dimensions were analyzed together. All three metrics were normalized to a common 0–1 scale so they contribute equally to the final priority score. This allows event pressure, incident risk, and response times, each measured on different scales, to be combined fairly into a single ranking.

```js

const stationsAll = Object.keys(currentStaffing);

// Event pressure: expected 2026 attendance per staff member
const eventAttendanceMap = upcoming_events.reduce((map, d) => {
  const s = d.nearby_station;
  const current = map.get(s) ?? 0;
  map.set(s, current + (d.expected_attendance ?? 0));
  return map;
}, new Map());

// Incident risk: weighted by severity (low=1, medium=2, high=3)
const severityWeight = { low: 1, medium: 2, high: 3 };

const incidentRiskMap = incidents.reduce((map, d) => {
  const s = d.station;
  const w = severityWeight[d.severity] ?? 1;
  const current = map.get(s) ?? 0;
  map.set(s, current + w);
  return map;
}, new Map());

// Response time: average per station
const responseStatsMap = incidents.reduce((map, d) => {
  const s = d.station;
  const entry = map.get(s) || { sum: 0, count: 0 };
  entry.sum += d.response_time_minutes ?? 0;
  entry.count += 1;
  map.set(s, entry);
  return map;
}, new Map());

// Combine into one object per station
let stationScores = stationsAll.map(station => {
  const staff = currentStaffing[station] ?? 0;

  const totalAttendance = eventAttendanceMap.get(station) ?? 0;
  const eventPressure =
    staff > 0 ? totalAttendance / staff : totalAttendance; 

  const risk = incidentRiskMap.get(station) ?? 0;

  const stats = responseStatsMap.get(station) ?? { sum: 0, count: 0 };
  const avgResponse =
    stats.count > 0 ? stats.sum / stats.count : NaN;

  return {
    station,
    staff,
    totalAttendance,
    eventPressure,
    risk,
    avgResponse
  };
});

// Normalize helper 

function addNormalized(field, normField) {
  const vals = stationScores
    .map(d => d[field])
    .filter(v => Number.isFinite(v));
  const min = Math.min(...vals);
  const max = Math.max(...vals);
  const range = max - min || 1;

  stationScores.forEach(d => {
    const v = d[field];
    d[normField] = Number.isFinite(v) ? (v - min) / range : 0;
  });
}

// Normalize each metric to 0–1
addNormalized("eventPressure", "eventPressureNorm");
addNormalized("risk", "riskNorm");
addNormalized("avgResponse", "responseNorm");

// Composite priority score 

stationScores.forEach(d => {
  d.priorityScore =
    0.5 * d.eventPressureNorm +
    0.3 * d.riskNorm +
    0.2 * d.responseNorm;
});

// Sort by priority 
stationScores = stationScores.sort((a, b) => b.priorityScore - a.priorityScore);
stationScores.forEach((d, i) => {
  d.rank = i + 1;
  d.isTop = i === 0;
});

const topStation = stationScores[0].station;

// Diagnostic dot-matrix data 

const diagnosticData = [];

for (const d of stationScores) {
  diagnosticData.push(
    {
      station: d.station,
      metric: "2026 Event Pressure",
      value: d.eventPressureNorm,
      raw: d.eventPressure,
      priorityScore: d.priorityScore,
      isTop: d.isTop
    },
    {
      station: d.station,
      metric: "Historical Incident Risk",
      value: d.riskNorm,
      raw: d.risk,
      priorityScore: d.priorityScore,
      isTop: d.isTop
    },
    {
      station: d.station,
      metric: "Response Time",
      value: d.responseNorm,
      raw: d.avgResponse,
      priorityScore: d.priorityScore,
      isTop: d.isTop
    },
    {
      station: d.station,
      metric: "Overall Priority",
      value: d.priorityScore, 
      raw: d.priorityScore,
      priorityScore: d.priorityScore,
      isTop: d.isTop
    }
  );
}

// Order stations by final priority score for the y-axis
const stationPriorityOrder = stationScores.map(d => d.station);

// Metric order for the x-axis
const metricOrder = [
  "2026 Event Pressure",
  "Historical Incident Risk",
  "Response Time",
  "Overall Priority"
];
```

```js
Plot.plot({
  height: 520,
  marginLeft: 180,
  marginRight: 40,
  marginBottom: 40,
  x: {
    domain: metricOrder,
    label: null
  },
  y: {
    domain: stationPriorityOrder,
    label: null
  },
  r: { range: [3, 16] },
  marks: [
    // Light row separators
    Plot.ruleY(stationPriorityOrder, {
      x1: metricOrder[0],
      x2: metricOrder[metricOrder.length - 1],
      stroke: "#eef2ff"
    }),

    // Main dot matrix
    Plot.dot(diagnosticData, {
      x: "metric",
      y: "station",
      r: d =>
        d.metric === "Overall Priority"
          ? 8 + d.value * 8
          : 4 + d.value * 6,
      fill: d =>
        d.isTop && d.metric === "Overall Priority"
          ? "#0039A6" // top station, overall
          : d.metric === "Overall Priority"
          ? "#4A74C9" // other stations, overall
          : "#AFC4F5", // sub-metrics
      stroke: d =>
        d.isTop && d.metric === "Overall Priority" ? "#001B44" : "white",
      opacity: d => (d.isTop || d.metric === "Overall Priority" ? 1 : 0.9)
    }),

    // Label the top-priority station 
    Plot.text(
      diagnosticData.filter(
        d => d.isTop && d.metric === "Overall Priority"
      ),
      {
        x: "metric",
        y: "station",
        dx: -40,
        dy: 4,
        text: d => `Top priority: ${d.station}`,
        fill: "#F4A259",
        fontWeight: "bold",
        textAnchor: "start"
      }
    ),

    // Tooltip 
    Plot.tip(
      diagnosticData,
      Plot.pointerX({
        x: "metric",
        y: "station",
        title: d => {
          const pct = Math.round(d.value * 100);
          const header = `${d.station} – ${d.metric}`;
          let detail;

          if (d.metric === "2026 Event Pressure") {
            detail = `Event pressure: ${d.raw.toFixed(
              1
            )} expected attendees per staff`;
          } else if (d.metric === "Historical Incident Risk") {
            detail = `Weighted incident score: ${d.raw.toFixed(1)}`;
          } else if (d.metric === "Response Time") {
            detail = `Average response time: ${d.raw.toFixed(1)} minutes`;
          } else {
            detail = `Overall staffing priority score: ${d.raw.toFixed(2)}`;
          }

          return `${header}
Normalized score: ${pct}/100
${detail}`;
        }
      })
    )
  ]
})
```

The analysis shows that Canal St stands out as the station most in need of additional staffing for Summer 2026. It has the highest event-driven pressure rate, meaning the number of expected attendees per staff member far exceeds other stations. Canal St also carries a significant historical incident burden, indicating persistent operational demands, and its response times trend slower, suggesting current staffing levels may already be stretched. When all three factors are weighted and combined, Canal St emerges as the clear top-priority station—indicating that adding staff here will produce the greatest improvement in system readiness, safety, and passenger experience during the upcoming event season.