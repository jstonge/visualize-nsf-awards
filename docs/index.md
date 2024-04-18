---
toc: false
sql:
  data: ./data/nsf_awards.parquet
---


<div class="hero">
  <h1>Exploring NSF grants</h1>
  <h2>A small exploration of the data available from <a href="https://www.nsf.gov/awardsearch/">https://www.nsf.gov/awardsearch/</a>.</h2>
</div>

## Timeseries

```js
const select_cum = Inputs.select(uniqState, {multiple: 23, value: ["California", "Vermont"]})
const chosen_state = Generators.input(select_cum);
```

Lets first look at the data overtime.

<div class="card">${
    resize((width) => 
    Plot.plot({
      width: 1200,
      marginLeft: 120,
      y: {grid: true},
      color: {legend: true},
      marks: [
        Plot.ruleY([0]),
        Plot.lineY(money_overtime, {x: "Call_Year", y: "tot", stroke: "Grant_Type"}),
        Plot.dotY(money_overtime, {x: "Call_Year", y: "tot", stroke: "Grant_Type", tip: true})
      ]
    })
    )}
</div>
<div class="custom-grid grid-cols-2">
  <div class="card card-a" style="max-width: 230px;">
    <h2>Choose state<h2/><br>
     ${select_cum}
  </div>
  <div class="card card-b">
      ${resize((width) => 
        Plot.plot({
          width: 800,
          title: "Cumulative sum",
          color: {legend: true},
          y: {type: "log", grid: true},
          marks: [
            Plot.ruleY([0]),
            Plot.lineY([...cum_money_overtime].filter(d => !chosen_state.includes(d["Institution_StateName"])), {
              x: "Call_Year", y: "tot", z: "Institution_StateName", 
              stroke: 'black',
              strokeOpacity: 0.1
              }),
            Plot.lineY([...cum_money_overtime].filter(d => chosen_state.includes(d["Institution_StateName"])), {
              x: "Call_Year", y: "tot", z: "Institution_StateName", 
              stroke: "Institution_StateName",
              strokeOpacity: 1, tip: true,
              })
          ]
      })
      )}
  </div>
</div>

### Who gets the money for a given year?

```js
const select = view(Inputs.select(uniq_GrantType, {label: "Select grant type", value: "Standard Grant"}))
const range = view(Inputs.range([1959, 2023], {label: "Year", step: 1, value: 2000}))
const toggleLog = view(Inputs.toggle({label: "Do Log", value: true}))
const select1 = view(Inputs.select(["Institution_StateName", "Host_Institution(s)", "Researcher(s)", "Domain"], {label: "Level of analysis"}))
```

<div class="grid grid-cols-2" style="grid-auto-rows: 604px;">
  <div class="card">${
    resize((width) => Plot.plot({
    title: "Who earns the most overall 💰?",
    marginLeft: 220,
    width,
    x: {label: "NSF contribution($)", grid: true,  type: toggleLog ? "log" : "linear"},
    y: {label: null},
    marks: [
      Plot.ruleX([0]),
      Plot.axisY({label: null, lineWidth: 20}),
      Plot.tickX(
        specificGrant,
        Plot.groupY(
          {x: "sum"},
          {x: "NSF_contribution", y: select1,
          stroke: "grey", strokeWidth: 4, sort: {y: "-x", limit: top_n}}
        )
      )
    ]
  }))
  }</div>
  <div class="card">${
    resize((width) => Plot.plot({
    title: "Average money in red, median in green",
    marginLeft: 220,
    width,
    x: {label: "NSF contribution($)", grid: true, type: toggleLog ? "log" : "linear"},
    marks: [
      Plot.ruleX([0]),
      Plot.axisY({label: null, lineWidth: 20}),
      Plot.tickX(
        specificGrant,
        Plot.groupY(
          {x: "mean"},
          {x: "NSF_contribution", y: select1, stroke: "red", strokeWidth: 4, sort: { y: "-x", limit: top_n }}
        )),
      Plot.tickX(
        specificGrant,
        { x: "NSF_contribution", y: select1, strokeOpacity: .3, tip: true, title: d => `Title: ${d.Project_Title}` }
      ),
      Plot.tickX(
        specificGrant,
        Plot.groupY(
          {x: "median"},
          {x: "NSF_contribution", y: select1, stroke: "lightgreen", strokeWidth: 4, sort: {y: "-x", limit: top_n}}
        )
      )
    ]
  }))
  }</div>
</div>


<div class="card" style="padding: 0;">
  ${Inputs.table(search, {rows: 16})}
</div>


```js
const search = view(Inputs.search(specificGrant, {
  placeholder: "Search anything…",
  columns: [
    "Project_Title", 
    'Researcher(s)', 
    "Domain", 
    'NSF_contribution', 
    'Host_Institution(s)', 
    "Institution_CityName",
    'Institution_StateName' 
  ],
  header: {
    Institution_CityName: "City",
  }
  }));
```

Remember right now we are only seeing **${select}** for the year **${range}**.


```sql id=specificGrant
SELECT EU_contribution::DOUBLE as NSF_contribution, * 
FROM data WHERE Grant_Type=${select} AND Call_Year=${range}
```
```js
const uniqState = [...cum_money_overtime].map(d=>d.Institution_StateName).filter(onlyUnique)
```


```sql id=cum_money_overtime
SELECT Call_Year, Institution_StateName, SUM(EU_contribution::DOUBLE) as tot 
FROM data
WHERE Call_Year > 1959 AND Call_Year < 2024
GROUP BY Call_Year, Institution_StateName
ORDER BY CAll_Year
```

```sql id=money_overtime
SELECT Call_Year, Grant_Type, SUM(EU_contribution::DOUBLE) as tot 
FROM data
WHERE Call_Year > 1959 AND Call_Year < 2024
GROUP BY Call_Year, Grant_Type 
ORDER BY CAll_Year
```

```js
function onlyUnique(value, index, array) {
  return array.indexOf(value) === index;
}
```
```js
const uniq_GrantType = [...money_overtime].map(d=>d.Grant_Type).filter(onlyUnique)
const wrap = 40
const top_n = 24
// const top_n = view(Inputs.range([30, 100], {label: "Top N", step: 1, value: 30}))
```

<style>

.custom-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr); /* Defines a 4-column layout */
  gap: 10px; /* Spacing between cards */
}

/* Specific sizes for each card */
.card-a { grid-column: span 1; } /* Card A takes up 2 columns */
.card-b { grid-column: span 3; } /* Card B takes up 1 column */
.card-c { grid-column: span 2; } /* Card B takes up 1 column */

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 2rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 300;
  line-height: 1.2;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
