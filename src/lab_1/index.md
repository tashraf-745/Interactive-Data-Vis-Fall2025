---
title: "Lab 1: Passing Pollinators"
toc: true
---


# Pollinator Activity Analysis 

This project utilizes observational data to identify key trends in bee behavior, flower productivity, and environmental factors. The insights are crucial for **optimizing farm management** and **enhancing pollination efforts**.

***

## Data Analysis Summary

The data reveals distinct patterns across pollinator characteristics, weather preferences, and flower nectar production.

## 1. Body Mass and Wing Span Distribution of Each Pollinator Species

The analysis confirms the expected size classifications:
* **Honeybees** (Small Bees) are tightly clustered around 0.10 body mass and 0.20 wing span.
* **Bumblebees** (Medium Bees) occupy the range of 0.20$–$0.30 and $30$–$40.
* **Carpenter Bees** (Large Bees) are the largest, with masses concentrated between 0.35 and $0.55 and wing spans of 40$–$50.

The distinct clusters (visualized below) indicate consistent species identification and provide a clear baseline for studying how size influences foraging and flight efficiency.

```js
const pollinator_activity_data = await FileAttachment("pollinator_activity_data.csv").csv({ typed: true });
```

```js
Plot.plot({
  height: 500,
  title: "Pollinator Species: Body Mass vs. Wing Span",
  subtitle: "Visualizing distinct physical clusters for Honeybee, Bumblebee, and Carpenter Bee.",
  marginTop: 30,
  grid: true,
  color: {
    domain: ["Honeybee", "Bumblebee", "Carpenter Bee"],
    range: ["#F8C471", "#9B59B6", "#27AE60"], 
    legend: true,
    label: "Pollinator Species"
  },
  x: {
    label: "Average Body Mass (g)",
    domain: [0, 0.6],
    nice: true
  },
  y: {
    label: "Average Wing Span (mm)",
    domain: [15, 50],
    nice: true
  },
  marks: [
    Plot.dot(pollinator_activity_data, {
      x: "avg_body_mass_g",
      y: "avg_wing_span_mm",
      fill: "pollinator_species",
      stroke: "pollinator_species",
      r: 5,
      opacity: 0.7
    }),
    Plot.frame()
  ]
})
```


## 2. Ideal Weather Conditions for Pollination Visits

This analysis examines how **temperature**, **humidity**, and **wind speed** influence pollinator activity under varying weather conditions.  

* Pollinator visits **increase steadily with temperature**, peaking under **sunny** conditions. Warmer weather enhances foraging activity, likely due to better nectar flow and higher bee mobility.

* Visits **peak at moderate humidity (65–75%)**, especially in **partly cloudy** weather. Both very dry and very humid conditions reduce activity, suggesting optimal foraging occurs in balanced moisture levels.

* Pollinator visits **drop sharply as wind speed rises**, with all weather types showing reduced activity beyond moderate breezes. Calm conditions clearly support more stable and efficient pollination.



```js
Plot.plot({
  title: `Pollinator Visits by ${conditionX.label}`,
  subtitle: `Average visit count variation across different weather conditions by ${conditionX.label.toLowerCase()}.`,
  height: 500,
  grid: true,
  color: {
    legend: true,
    label: "Weather Condition",
    domain: ["Sunny", "Partly Cloudy", "Cloudy"],
    range: ["#F1948A", "#F5B041", "#5DADE2"]
  },
  x: {
    label: conditionX.label,
    nice: true
  },
  y: {
    label: "Average Visit Count",
    nice: true,
    zero: true
  },
  marks: [
    Plot.lineY(
      pollinator_activity_data,
      Plot.binX(
        { y: "mean" },
        {
          x: d => d[conditionX.value],
          y: "visit_count",
          stroke: "weather_condition",
          curve: "natural",
          thresholds: 10
        }
      )
    ),
    Plot.ruleY([0]),
    Plot.frame()
  ]
})
```
```js
const envOptions = [
  { label: "Temperature", value: "temperature" },
  { label: "Humidity", value: "humidity" },
  { label: "Wind Speed", value: "wind_speed" }
];

const conditionX = view(Inputs.select(envOptions, {
  label: "Select Environmental Variable:",
  value: envOptions[0],
  format: d => d.label
}));

```




## 3. Mean Nectar Production by Flower Species

This analysis identifies which **flower species** produce the most nectar on average.

Among the observed flowers, **Sunflowers** show the **highest nectar production**, indicating their strong potential to attract and sustain pollinator visits.  

**Coneflowers** and **Lavender** follow closely, each contributing consistently to nectar availability across the study plots.  


```js
Plot.plot({
  title: "Average Nectar Production by Flower Species",
  subtitle: "Identifying which flower produces the most nectar on average.",
  height: 500,
  marginLeft: 100,
  grid: true,
  x: {
    label: "Average Nectar Production (μL)",
    nice: true
  },
  y: {
    label: "Flower Species"
  },
  color: {
    legend: false,
    range: ["#58D68D", "#AF7AC5", "#F4D03F"]
  },
  marks: [
    Plot.barX(
      pollinator_activity_data,
      Plot.groupY(
        {x: "mean"}, // aggregate mean nectar production
        {
          y: "flower_species",
          x: "nectar_production",
          fill: "flower_species"
        }
      )
    ),
    Plot.ruleX([0])
  ]
})

```