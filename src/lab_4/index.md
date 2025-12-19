---
title: "Lab 4: Clearwater Crisis"
toc: false
---

# Who Polluted Lake Clearwater? An Evidence-Based Environmental Forensics Analysis

Lake Clearwater has experienced a sudden ecological collapse marked by declining fish populations and deteriorating water quality. Four suspects operate around the lake, each with plausible pollution pathways. Using environmental monitoring data, biological surveys, and documented activity logs, this investigation evaluates which suspect best explains the collapse across space, time, and biological impact.


___



## Where and When Heavy Metal Contamination Exceeds Ecological and Regulatory Thresholds?

<br>
This chart displays weekly heavy metal concentrations (ppb) measured at four monitoring stations over the study period. Two reference lines mark key thresholds: an ecological concern threshold at **20 ppb** and a regulatory limit at **30 ppb**, enabling clear identification of exceedance events.




```js

const waterQuality = await FileAttachment("data/water_quality.csv").csv({ typed: true });
const monitoringStations = await FileAttachment("data/monitoring_stations.csv").csv({ typed: true });
const fishSurveys = await FileAttachment("data/fish_surveys.csv").csv({ typed: true });
const suspectActivities = await FileAttachment("data/suspect_activities.csv").csv({ typed: true });

```


```js
Plot.plot({
  width: 900,
  height: 450,
  marginLeft: 70,
  marginRight: 130,
  x: { label: "Date" },
  y: { label: "Heavy Metals (ppb)" },
  color: { scheme: "Tableau10", legend: true },

  marks: [

    Plot.ruleY([0], { stroke: "#111827", strokeOpacity: 0.35 }),
    Plot.ruleX([Plot.valueof(waterQuality, "date").reduce((a, b) => (a < b ? a : b))], {
      stroke: "#111827",
      strokeOpacity: 0.35
    }),

    
    Plot.areaY(
      waterQuality,
      Plot.groupX(
        { y1: "min", y2: "max" },
        {
          x: "date",
          y: "heavy_metals_ppb",
          fill: "station_id",
          fillOpacity: 0.12
        }
      )
    ),

    
    Plot.line(waterQuality, {
      x: "date",
      y: "heavy_metals_ppb",
      stroke: "station_id",
      strokeWidth: 2
    }),

    
    Plot.dot(waterQuality, {
      x: "date",
      y: "heavy_metals_ppb",
      stroke: "station_id",
      fill: "white",
      r: 2.5
    }),

    
    Plot.dot(
      waterQuality,
      Plot.pointer({
        x: "date",
        y: "heavy_metals_ppb",
        r: 6,
        fill: "transparent",
        tip: {
          format: {
            date: d => d.date.toLocaleDateString(),
            station_id: true,
            station_name: true,
            heavy_metals_ppb: d => `${d.heavy_metals_ppb.toFixed(1)} ppb`,
            status: d =>
              d.heavy_metals_ppb >= 30
                ? "Exceeds regulatory limit (30 ppb)"
                : d.heavy_metals_ppb >= 20
                ? "Exceeds concern threshold (20 ppb)"
                : "Within normal range"
          }
        }
      })
    ),

    
    Plot.ruleY(
      waterQuality,
      Plot.groupZ(
        { y: "mean" },
        {
          z: "station_id",
          y: "heavy_metals_ppb",
          stroke: "station_id",
          strokeOpacity: 0.6,
          strokeDasharray: "3,4"
        }
      )
    ),

    
    Plot.ruleY([20], {
      stroke: "#6b7280",
      strokeDasharray: "4,4",
      strokeWidth: 1.5
    }),
    Plot.ruleY([30], {
      stroke: "#ef4444",
      strokeWidth: 1.5
    }),

    
    Plot.text(
      [
        { y: 20, t: "Concern threshold (20 ppb)" },
        { y: 30, t: "Regulatory limit (30 ppb)" }
      ],
      {
        x: Plot.valueof(waterQuality, "date").reduce((a, b) => (a < b ? a : b)),
        y: "y",
        text: "t",
        dx: 6,
        dy: -6,
        fontSize: 11,
        fill: "#374151"
      }
    ),

    
    Plot.text(
      waterQuality,
      Plot.selectLast({
        x: "date",
        y: "heavy_metals_ppb",
        z: "station_id",
        text: "station_id",
        fill: "station_id",
        dx: 10,
        fontSize: 11
      })
    )
  ]
})
```

While heavy metal levels at the East, North, and
 South stations remain relatively stable and largely below both thresholds, the West station exhibits repeated and pronounced spikes that exceed the concern threshold and, on several occasions, surpass the regulatory limit. These exceedances are temporally clustered rather than isolated, suggesting a localized and recurring source of contamination affecting the West station specifically.<br>


### How many weeks did each station exceed the ecological concern threshold (≥20 ppb)?


```js
const exceedanceSummary = ["East", "North", "South", "West"].map(station => {
  const stationData = waterQuality.filter(d => d.station_id === station);

  return {
    station,
    exceed_20: stationData.filter(d => d.heavy_metals_ppb >= 20).length,
    exceed_30: stationData.filter(d => d.heavy_metals_ppb >= 30).length
  };
});
```

```js 
Plot.plot({
  width: 600,
  height: 300,
  marginLeft: 70,
  marginBottom: 50,
  x: {
    label: "Monitoring Station",
    domain: ["East", "North", "South", "West"]
  },
  y: {
    label: "Number of weeks ≥ 20 ppb",
    grid: true
  },
  color: {
    scheme: "Tableau10",
    legend: false
  },

  marks: [
    Plot.barY(exceedanceSummary, {
      x: "station",
      y: "exceed_20",
      fill: "station",
      tip: {
        format: {
          station: true,
          exceed_20: d => `${d.exceed_20} weeks ≥ 20 ppb`
        }
      }
    }),

    
    Plot.text(exceedanceSummary, {
      x: "station",
      y: "exceed_20",
      text: d => d.exceed_20,
      dy: -6,
      fontSize: 11,
      fill: "#111827"
    })
  ]
})
```

Ecological concern from heavy metal contamination is concentrated at the West monitoring station, which experienced more than twice as many exceedance events as all other stations combined, indicating a localized and recurring pollution source.

<br>

## Ecological Impact of Heavy Metal Pollution on Trout Health 


### Linking water quality conditions to biological stress across Lake Clearwater’s monitoring stations

<br>

This chart examines the relationship between **heavy metal concentrations (ppb)** and **average trout weight (g)** across monitoring stationst. Each point represents a quarterly trout survey, with circle color indicating **dissolved oxygen levels (mg/L)**, an additional marker of aquatic stress. The vertical dashed line marks the ecological concern threshold for heavy metals at **20 ppb**. By encoding dissolved oxygen levels alongside heavy metal concentrations, the visualization distinguishes between isolated chemical exposure and compounded ecological stress. 

```js
const fish = fishSurveys.map(d => ({
  ...d,
  date: new Date(d.date)
}));

const water = waterQuality.map(d => ({
  ...d,
  date: new Date(d.date)
}));
```

```js
const trout = fish.filter(d => d.species === "Trout");
```

```js
const troutWithWater = trout.map(f => {
  const prior = water
    .filter(w =>
      w.station_id === f.station_id &&
      w.date <= f.date
    )
    .sort((a, b) => b.date - a.date)[0];

  return {
    ...f,
    heavy_metals_ppb: prior?.heavy_metals_ppb ?? null,
    dissolved_oxygen_mg_per_L: prior?.dissolved_oxygen_mg_per_L ?? null
  };
});
```

```js
const chart = Plot.plot({
  width: 900,
  height: 420,
  grid: true,

  x: {
    label: "Heavy metals (ppb)",
    domain: [0, 80]
  },

  y: {
    label: "Average trout weight (g)"
  },

  color: {
    legend: true,
    label: "Dissolved oxygen (mg/L)",
    scheme: "blues",
    reverse: true
  },

  facet: {
    data: troutWithWater,
    x: "station_id"
  },

  marks: [
    Plot.dot(troutWithWater, {
      x: "heavy_metals_ppb",
      y: "avg_weight_g",
      fill: "dissolved_oxygen_mg_per_L",
      r: 5,
      opacity: 0.85,
      tip: true
    }),

    
    Plot.ruleX([20], {
      strokeDasharray: "4,4",
      stroke: "#ef4444"
    })
  ]
});

display(chart);

```
Across the East, North, and South stations, heavy metal levels remain largely below the threshold, and trout weights stay relatively high and stable, indicating healthy biological conditions. In contrast, the West station exhibits a distinct pattern of elevated heavy metal concentrations combined with lower dissolved oxygen and substantially reduced trout weights. This convergence of chemical stressors and biological response suggests that pollution-sensitive trout are experiencing the greatest ecological impact at the West station.

<br>

## Spatial Proximity of Suspects to the West Monitoring Station
<br>

This chart compares the distance of each suspect location to the West monitoring station, measured in meters. 
```js
const west = monitoringStations.find(d => d.station_id === "West");
```

```js
const proximity = [
  { suspect: "ChemTech Manufacturing", distance_m: west.distance_to_chemtech_m },
  { suspect: "Riverside Farm", distance_m: west.distance_to_farm_m },
  { suspect: "Lakeview Resort", distance_m: west.distance_to_resort_m },
  { suspect: "Clearwater Fishing Lodge", distance_m: west.distance_to_lodge_m }
];
```

```js
proximity.sort((a, b) => a.distance_m - b.distance_m);
```

```js
const chart = Plot.plot({
  width: 600,
  height: 220,
  marginLeft: 200,

  x: {
    label: "Distance to West station (meters)"
  },

  y: {
    label: "Suspect"
  },

  marks: [
    Plot.dot(proximity, {
      x: "distance_m",
      y: "suspect",
      r: 7,
      fill: "#2563eb",
      tip: true
    }),

    Plot.ruleX([0])
  ]
});

display(chart);
```
ChemTech Manufacturing is situated substantially closer to the West station than any other suspect, while Clearwater Fishing Lodge, Lakeview Resort, and Riverside Farm are located at progressively greater distances.

<br>

## Comparing documented activity windows with exceedance events across suspects
<br>

This timeline visualization overlays each suspect’s documented activity periods with observed **heavy metal exceedance events (>20 ppb)** across the study period. Colored horizontal bands represent the timing and duration of specific activities, while pink markers indicate weeks when heavy metal concentrations exceeded the ecological concern threshold.

```js

const STATION = "West";
const HM_THRESH = 20;
const DO_THRESH = 5;


const intensityOpacity = new Map([
  ["Low", 0.20],
  ["Medium", 0.40],
  ["High", 0.60]
]);


const wq = waterQuality
  .filter(d => d.station_id === STATION)
  .map(d => ({ ...d, date: new Date(d.date) }));


const acts = suspectActivities.map(d => {
  const start = new Date(d.date);
  const dur = d.duration_days ?? 1;
  const end = new Date(start.getTime() + dur * 86400000);
  return {
    ...d,
    start,
    end,
    opacity: intensityOpacity.get(d.intensity) ?? 0.30
  };
});


const spikes = wq
  .map(d => {
    const hm = d.heavy_metals_ppb > HM_THRESH;
    const od = d.dissolved_oxygen_mg_per_L < DO_THRESH;
    const spike_type =
      hm && od ? "Both" :
      hm ? "Heavy metals > 20" :
      od ? "DO < 5" :
      null;
    return { ...d, spike_type };
  })
  .filter(d => d.spike_type);


const chart = Plot.plot({
  width: 980,
  height: 380,
  marginLeft: 210,
  x: { label: "Date" },
  y: { label: "Suspect" },

  color: { legend: true, label: "Spike type" },

  marks: [

    Plot.rect(acts, {
      x1: "start",
      x2: "end",
      y: "suspect",
      inset: 3,
      fill: "activity_type",
      fillOpacity: d => d.opacity,
      tip: true
    }),

 
    Plot.dot(
      spikes.flatMap(s => acts.map(a => ({
        suspect: a.suspect,
        date: s.date,
        spike_type: s.spike_type,
        heavy_metals_ppb: s.heavy_metals_ppb,
        dissolved_oxygen_mg_per_L: s.dissolved_oxygen_mg_per_L
      }))),
      {
        x: "date",
        y: "suspect",
        fill: "spike_type",
        r: 3.5,
        tip: {
          format: {
            spike_type: true,
            date: d => d.date.toLocaleDateString(),
            heavy_metals_ppb: d => d.heavy_metals_ppb.toFixed(1),
            dissolved_oxygen_mg_per_L: d => d.dissolved_oxygen_mg_per_L.toFixed(2)
          }
        }
      }
    )
  ]
});

display(chart);
```
ChemTech Manufacturing shows the strongest temporal alignment, with multiple exceedance events occurring during or immediately adjacent to its operational and maintenance phases. In contrast, Clearwater Fishing Lodge, Lakeview Resort, and Riverside Farm display more diffuse or intermittent overlap, with pollution spikes often occurring outside their discrete activity windows. This pattern suggests that while some exceedances coincide incidentally with other suspects’ activities, only ChemTech demonstrates repeated and sustained temporal overlap consistent with a recurring pollution source.


<br>


## Following the Evidence: Who Left the Ecological Footprint?

<br>
This evidence matrix summarizes how each suspect aligns across three independent analytical dimensions: **spatial proximity to the West monitoring station**, **temporal overlap with pollution spike events**, and **biological impact as reflected in trout population decline**. Green cells indicate strong alignment, yellow cells indicate partial or ambiguous alignment, and gray cells indicate little to no supporting evidence.


```js
const evidence = [
  { suspect: "ChemTech Manufacturing", dimension: "Spatial (West proximity)", score: 1 },
  { suspect: "ChemTech Manufacturing", dimension: "Temporal (spike overlap)", score: 1 },
  { suspect: "ChemTech Manufacturing", dimension: "Biological (trout decline)", score: 1 },

  { suspect: "Clearwater Fishing Lodge", dimension: "Spatial (West proximity)", score: 0 },
  { suspect: "Clearwater Fishing Lodge", dimension: "Temporal (spike overlap)", score: 0.5 },
  { suspect: "Clearwater Fishing Lodge", dimension: "Biological (trout decline)", score: 0 },

  { suspect: "Lakeview Resort", dimension: "Spatial (West proximity)", score: 0 },
  { suspect: "Lakeview Resort", dimension: "Temporal (spike overlap)", score: 0 },
  { suspect: "Lakeview Resort", dimension: "Biological (trout decline)", score: 0 },

  { suspect: "Riverside Farm", dimension: "Spatial (West proximity)", score: 0.5 },
  { suspect: "Riverside Farm", dimension: "Temporal (spike overlap)", score: 0 },
  { suspect: "Riverside Farm", dimension: "Biological (trout decline)", score: 0 }
];
```

```js
const chart = Plot.plot({
  width: 720,
  height: 300,
  marginLeft: 180,

  x: {
    label: "Evidence dimension",
    domain: [
      "Spatial (West proximity)",
      "Temporal (spike overlap)",
      "Biological (trout decline)"
    ]
  },

  y: {
    label: "Suspect"
  },

  color: {
    legend: true,
    label: "Evidence strength",
    domain: [0, 0.5, 1],
    range: ["#e5e7eb", "#facc15", "#16a34a"]
  },

  marks: [
    Plot.cell(evidence, {
      x: "dimension",
      y: "suspect",
      fill: "score",
      inset: 3,
      tip: true
    }),

    Plot.text(evidence, {
      x: "dimension",
      y: "suspect",
      text: d => d.score === 1 ? "✓" : d.score === 0.5 ? "△" : "",
      fontSize: 14
    })
  ]
});

display(chart);
```

Lake Clearwater's ecological collapse is most plausibly linked to ChemTech Manufacturing's operations, given the strong temporal alignment between their activity periods and heavy metal pollution spikes at the West monitoring station. The localized nature of contamination, coupled with significant declines in pollution-sensitive trout populations in the same area, supports the conclusion that ChemTech's discharges are the primary driver of the lake's environmental degradation. Further investigation and regulatory scrutiny of ChemTech's practices are warranted to mitigate ongoing harm and restore Lake Clearwater's ecological health.





