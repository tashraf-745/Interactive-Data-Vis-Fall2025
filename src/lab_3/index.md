---
title: "Lab 3: Mayoral Mystery"
toc: false
---

# **Mayoral Mystery: A Geospatial Analysis of Campaign Performance in New York City**

### **Examining Vote Share and Neighborhood Demographics Across Borough–Community Districts**

This dashboard investigates the spatial and demographic dynamics of the 2024 mayoral campaign by integrating election results with neighborhood income profiles across New York City’s borough–community districts. Through a combination of choropleth mapping and demographic overlays, the analysis highlights where the candidate performed strongly, where support fell short, and how socioeconomic characteristics shaped voting patterns.

By visualizing variation in vote share alongside district-level demographics, this dashboard provides insight into the campaign’s strengths, areas requiring improved outreach, and strategic opportunities for enhancing performance in future election cycles.

---
<!-- Import Data -->
```js
const nyc = await FileAttachment("data/nyc.json").json();
const results = await FileAttachment("data/election_results.csv").csv({ typed: true });
const survey = await FileAttachment("data/survey_responses.csv").csv({ typed: true });
const events = await FileAttachment("data/campaign_events.csv").csv({ typed: true });


```


```js
// The nyc file is saved in data as a topoJSON instead of a geoJSON. Thats primarily for size reasons -- it saves us 3MB of data. For Plot to render it, we have to convert it back to its geoJSON feature collection. 
const districts = topojson.feature(nyc, nyc.objects.districts)

```

## **Vote Share and Income Profile Across NYC Districts**


This interactive map displays all NYC borough–community districts shaded by the candidate’s vote share. Darker blue districts reflect stronger performance, while lighter blue areas indicate weaker support. Each district also contains a green circle representing its income tier—small/light green for low-income, medium green for middle-income, and large/dark green for high-income neighborhoods. By combining color (vote share) with circle size and shade (income category), you can quickly compare how the candidate performed across different socioeconomic contexts. Hover over any district to view exact values, including vote share percentage, income tier, and total votes.



```js

const resultsByDistrict = Object.fromEntries(
  results.map(d => {
    const total_votes = d.votes_candidate + d.votes_opponent;
    const vote_share =
      total_votes > 0 ? d.votes_candidate / total_votes : null;

    return [
      d.boro_cd, 
      {
        ...d,             
        total_votes,
        vote_share
      }
    ];
  })
);


const enrichedDistricts = {
  type: "FeatureCollection",
  features: districts.features.map(f => {
    const cd = +f.properties.BoroCD;        
    const stats = resultsByDistrict[cd];    

    return {
      ...f,
      properties: {
        ...f.properties,  
        ...stats          
      }
    };
  })
};

```



```js

Plot.plot({
  width: 800,
  color: {
    type: "linear",
    scheme: "blues",
    label: "Candidate vote share",
    domain: [0, 1],
    legend: true
  },
  projection: {

    type: "mercator",
    domain: enrichedDistricts
  },
  marks: [

    
    Plot.geo(enrichedDistricts, {
      fill: d => d.properties.vote_share,
      stroke: "#fff",
      strokeWidth: 0.7,
      tip: true,
      title: d => {
        const p = d.properties;
        if (!p || p.vote_share == null) return `District ${p?.BoroCD}\nNo data`;
        return `District ${p.BoroCD}
Income: ${p.income_category}
Vote share: ${(p.vote_share * 100).toFixed(1)}%
Candidate votes: ${p.votes_candidate}
Opponent votes: ${p.votes_opponent}`;
      }
    }),

   
    Plot.dot(
      enrichedDistricts,
      Plot.centroid({
        r: d => {
          const cat = d.properties.income_category;
          if (cat === "Low") return 4;      
          if (cat === "Middle") return 7;   
          if (cat === "High") return 10;    
          return 0;
        },
        fill: d => {
          const cat = d.properties.income_category;
          if (cat === "Low") return "#c7e9c0";   
          if (cat === "Middle") return "#74c476"; 
          if (cat === "High") return "#238b45";   
          return "transparent";
        },
        stroke: "white",
        strokeWidth: 1,
        opacity: 0.95
      })
    )
  ]
})
```

**Which Neighborhoods Contributed Most?**
The candidate’s strongest support came from high-income districts, particularly in Manhattan and northwest Brooklyn, where large dark-green circles align with the darkest blue vote-share regions. Middle-income neighborhoods showed mixed performance, with some pockets of solid support and others—especially in eastern Queens and southern Brooklyn—indicating weaker engagement. Low-income districts across the Bronx, central Brooklyn, and southern Staten Island consistently showed the lowest vote share, suggesting gaps in outreach, message resonance, and presence. Several middle- and high-income districts also underperformed expectations, pointing to localized engagement issues. Overall, the candidate excelled in higher-income areas but underperformed in lower-income regions, indicating that future campaign strategy should prioritize outreach and tailored messaging to low- and middle-income communities.

---



## **Policy Issue Alignment by Income Group**

This visualization compares average agreement scores (1–5) for each major policy issue—Affordable Housing, Childcare Support, Police Reform, Public Transit, and Small Business Tax—across three income groups: Low, Middle, and High. Each dot represents the mean alignment score for a given issue-income combination, with color indicating income category.




```js
const resultsLookup = new Map(
  results.map(r => [r.boro_cd, r])
);

const surveyWithIncome = survey.map(d => {
  const res = resultsLookup.get(d.boro_cd) || {};
  return {
    ...d,
    income_category: res.income_category ?? "Unknown", 
    turnout_rate: res.turnout_rate,                    
    boro_cd: d.boro_cd
  };
});
```




```js
const issues = surveyWithIncome.flatMap(d => [
  {
    boro_cd: d.boro_cd,
    income_category: d.income_category,
    issue: "Affordable housing",
    alignment: d.affordable_housing_alignment
  },
  {
    boro_cd: d.boro_cd,
    income_category: d.income_category,
    issue: "Public transit",
    alignment: d.public_transit_alignment
  },
  {
    boro_cd: d.boro_cd,
    income_category: d.income_category,
    issue: "Childcare support",
    alignment: d.childcare_support_alignment
  },
  {
    boro_cd: d.boro_cd,
    income_category: d.income_category,
    issue: "Small business tax",
    alignment: d.small_business_tax_alignment
  },
  {
    boro_cd: d.boro_cd,
    income_category: d.income_category,
    issue: "Police reform",
    alignment: d.police_reform_alignment
  }
]);


const issuesLowMid = issues.filter(d =>
  d.income_category === "Low" || d.income_category === "Middle"
);
```




```js

const issuesClean = issues.filter(
  d =>
    d.income_category != null &&
    ["Low", "Middle", "High"].includes(d.income_category) &&
    d.alignment != null
);


const issueSummary = Object.values(
  issuesClean.reduce((acc, d) => {
    const key = `${d.issue}|${d.income_category}`;
    if (!acc[key]) {
      acc[key] = {
        issue: d.issue,
        income_category: d.income_category,
        sum: 0,
        n: 0
      };
    }
    acc[key].sum += d.alignment;
    acc[key].n += 1;
    return acc;
  }, {})
).map(d => ({
  issue: d.issue,
  income_category: d.income_category,
  avg_alignment: d.sum / d.n
}));
```


```js

Plot.plot({
  width: 800,
  height: 420,
  marginLeft: 60,
  marginBottom: 70,

  x: {
    label: "Policy issue",
    tickRotate: -25
  },
  y: {
    label: "Average agreement (1–5)",
    domain: [1, 5],
    grid: true
  },
  color: {
    label: "Income group",
    legend: true,
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#74c476", "#238b45"] 
  },

  marks: [
    
    Plot.ruleY([1, 2, 3, 4, 5], { stroke: "#eee" }),


    Plot.dot(issueSummary, {
      x: "issue",
      y: "avg_alignment",
      fill: "income_category",
      r: 8,
      stroke: "white",
      strokeWidth: 1.5,
      tip: true,
      title: d =>
        `${d.issue}
Income: ${d.income_category}
Average agreement: ${d.avg_alignment.toFixed(2)}`
    })
  ]
})
```


The chart reveals a clear income-based divide in policy alignment: **low-income voters** show the strongest agreement with the candidate’s platform—particularly on **affordable housing, childcare support, and public transit**—indicating these issues form the core of the candidate’s natural base. **Middle-income voters** are also supportive, though slightly less enthusiastic, suggesting they remain persuadable with messages emphasizing practical family and mobility benefits. In contrast, **high-income voters** consistently show low alignment across all issues, signaling minimal resonance with the candidate’s current agenda and a lower return on investment for targeted persuasion.



---



## **Estimated Attendance Across Event Types by Income Category**


This chart illustrates how estimated attendance varies across event types and income groups, highlighting which formats attract broader or more diverse participation.



```js

Plot.plot({
  width: 800,
  marginLeft: 60,
  marginBottom: 90,
  x: {
    label: "Event type",
    tickRotate: -30
  },
  y: {
    label: "Estimated attendance",
    grid: true
  },
  color: {
    label: "Income category",
    legend: true,
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#74c476", "#238b45"] 
  },
  marks: [
    Plot.barY(
      events,
      Plot.groupX(
        { y: "sum" },                
        {
          x: "event_type",
          y: "estimated_attendance",
          fill: "income_category",
          tip: true                   
        }
      )
    ),
    Plot.ruleY([0])
  ]
})
```







```js

const resultsByDistrict = new Map(
  results.map(r => [r.boro_cd, r])
);

const surveyMerged = survey.map(d => {
  const r = resultsByDistrict.get(d.boro_cd) || {};
  const vc = r.votes_candidate ?? 0;
  const vo = r.votes_opponent ?? 0;
  const total = vc + vo;

  let voted_for = null;
  if (total > 0) {
    voted_for = vc >= vo ? "Candidate" : "Opponent"; 
  }

  return {
    ...d,
    boro_cd: d.boro_cd,
    age_group: d.age_group,        
    income_category: r.income_category,
    turnout_rate: r.turnout_rate,
    voted_for
  };
}).filter(d => d.voted_for != null && d.age_group != null);

const issuesLong = surveyMerged.flatMap(d => [
  {
    age_group: d.age_group,
    voted_for: d.voted_for,
    issue: "Affordable housing",
    alignment: d.affordable_housing_alignment
  },
  {
    age_group: d.age_group,
    voted_for: d.voted_for,
    issue: "Public transit",
    alignment: d.public_transit_alignment
  },
  {
    age_group: d.age_group,
    voted_for: d.voted_for,
    issue: "Childcare support",
    alignment: d.childcare_support_alignment
  },
  {
    age_group: d.age_group,
    voted_for: d.voted_for,
    issue: "Small business tax",
    alignment: d.small_business_tax_alignment
  },
  {
    age_group: d.age_group,
    voted_for: d.voted_for,
    issue: "Police reform",
    alignment: d.police_reform_alignment
  }
]).filter(d => d.alignment != null);
```



```js
const voteChoiceChart = Plot.plot({
  width: 800,
  marginLeft: 80,
  marginBottom: 40,
  x: {
    label: "Number of respondents"
  },
  y: {
    label: "Age group"
  },
  color: {
    label: "Voted for",
    legend: true,
    domain: ["Candidate", "Opponent"],
    range: ["#3182bd", "#b0b0b0"] 
  },
  marks: [
    Plot.barX(
      surveyMerged,
      Plot.groupY(
        { x: "count" },
        {
          y: "age_group",
          fill: "voted_for",
          tip: true,
          title: d =>
            `Age: ${d.age_group}
Voted for: ${d.voted_for}`
        }
      )
    ),
    Plot.ruleX([0])
  ]
});
```


This chart compares the estimated attendance of different campaign event types across income groups. Across all events, **low-income attendees consistently make up the largest share**, reflecting stronger engagement among lower-income communities. **Community Meetings, Rallies, and Roundtables** draw the highest overall turnout, with particularly strong participation from **middle- and high-income groups** at these events. In contrast, **Canvassing Kickoffs and Town Halls** show more modest attendance from higher-income participants. The distribution suggests that while broad engagement exists, **policy-focused and interactive events attract a more diverse income mix**, making them especially valuable for reaching across demographic segments.


---




## **Strategic Priority Analysis: Identifying High-Impact Voter Groups and Neighborhoods**

This scatterplot compares **candidate vote share** (x-axis) with **turnout rate** (y-axis) across districts, colored by **income category**. The dotted lines represent the **citywide average turnout** and **average support**, creating four strategic quadrants. This visualization highlights where the campaign has the greatest opportunity to expand its base, improve turnout, and refine outreach efforts.






```js
// Count number of events per district
const eventsAgg = Object.values(
  events.reduce((acc, d) => {
    const cd = d.boro_cd;
    if (!acc[cd]) acc[cd] = { boro_cd: cd, event_count: 0 };
    acc[cd].event_count += 1;
    return acc;
  }, {})
);
```






```js
const eventLookup = new Map(
  eventsAgg.map(d => [d.boro_cd, d.event_count])
);
```








```js
const strategyData = results
  .map(d => {
    const totalVotes = d.votes_candidate + d.votes_opponent;
    if (!totalVotes || d.turnout_rate == null) return null;

    
    const turnout =
      d.turnout_rate > 1 ? d.turnout_rate / 100 : d.turnout_rate;

    return {
      boro_cd: d.boro_cd,
      income_category: d.income_category,
      turnout_rate: turnout,                  
      vote_share: d.votes_candidate / totalVotes, 
      event_count: eventLookup.get(d.boro_cd) ?? 0
    };
  })
  .filter(Boolean);

```




```js
const avgTurnout = d3.mean(strategyData, d => d.turnout_rate);
const avgSupport = d3.mean(strategyData, d => d.vote_share);

```



```js
Plot.plot({
  width: 800,
  height: 480,
  marginLeft: 60,
  marginBottom: 50,

  x: {
    label: "Candidate vote share",
    domain: [0, 1],
    tickFormat: d => `${Math.round(d * 100)}%`
  },

  y: {
    label: "Turnout rate",
    domain: [0, 1],
    grid: true,
    tickFormat: d => `${Math.round(d * 100)}%`
  },

  color: {
    label: "Income category",
    legend: true,
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#74c476", "#238b45"]
  },

  marks: [
    Plot.dot(strategyData, {
      x: "vote_share",
      y: "turnout_rate",
      fill: "income_category",
      r: d => 3 + d.event_count,
      opacity: 0.8,
      tip: true,
      title: d =>
        `District ${d.boro_cd}
Income: ${d.income_category}
Vote share: ${(d.vote_share * 100).toFixed(1)}%
Turnout: ${(d.turnout_rate * 100).toFixed(1)}%
Campaign events: ${d.event_count}`
    }),
    Plot.ruleX([avgSupport], { strokeDasharray: "4,4" }),
    Plot.ruleY([avgTurnout], { strokeDasharray: "4,4" })
  ]
})
```





### **Key Insights**

1. **High-support, low-turnout districts (Bottom-right quadrant) offer the strongest opportunity for impact.**  
   Many **low-income districts** show **above-average support (60%–75%) but below-average turnout (35%–50%)**, indicating voters who *already like the candidate but are not consistently showing up*. These neighborhoods should be the **top priority** for GOTV efforts, door-to-door field operations, and community-based mobilization.

2. **Middle-income districts with moderate support but weak turnout are persuasive opportunities.**  
   Middle-income areas clustered near **average support and low turnout** represent communities that could shift meaningfully with **targeted messaging**—especially around issues they ranked highly such as **public transit, childcare, and housing affordability**. A moderate investment here can produce measurable gains.

3. **High-income districts are not strategic targets for persuasion or turnout mobilization.**  
   High-income districts cluster in the **top-left quadrant**: **extremely high turnout (75%–90%) but low support (20%–35%)**. These voters are already highly engaged but not aligned with the candidate’s platform. Persuasion costs are high and expected returns are minimal, so field resources should be allocated elsewhere.

4. **Event-based outreach appears insufficient in low-turnout supportive districts.**  
   Many supportive districts with low turnout also had **fewer campaign events**, suggesting a direct opportunity: prioritizing **field presence**, **community forums**, and **localized engagement** could convert strong issue alignment into turnout gains.

### **Strategic Recommendations for the Next Campaign**

- **Prioritize low-income, high-support districts** where turnout is under 50%.  
  These neighborhoods offer the **highest return on investment**: mobilizing existing supporters yields immediate and measurable gains.

- **Deploy targeted persuasion messaging in middle-income communities.**  
  Focus on issues where these voters already show moderate agreement—**public transit**, **childcare support**, and **housing policy**. Tailor communications to emphasize stability, affordability, and family benefits.

- **Minimize effort in high-income districts.**  
  Outreach here provides little benefit; these voters have high turnout but are not ideologically aligned. Campaign resources should not be spent attempting to convert them.

- **Integrate survey insights into field strategy.**  
  Strong alignment on **housing, transit, and childcare** among low- and middle-income voters indicates clear messaging priorities. These themes should be emphasized in canvassing scripts, digital ads, and stump speeches targeted to these communities.

### **Overall Conclusion**

The data clearly shows that the candidate’s most powerful growth opportunity lies in **mobilizing supportive but historically low-turnout neighborhoods—primarily low-income districts—and strengthening resonance among middle-income voters with targeted issue framing**. By concentrating field resources where support is already strong and strategically refining outreach where persuasion is plausible, the next campaign can produce significant gains without expanding its geographic footprint.