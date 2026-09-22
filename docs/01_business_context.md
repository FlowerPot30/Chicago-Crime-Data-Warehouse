# 📋 Business Context

## 🎯 Business Problem

Law enforcement and city planning teams lack a consistency updated, analysis-ready view of crime incidents across Chicago. Two specific gaps this project address:

1. **Inefficient resource allocation** - There is no reliable way to see which areas, times, and crime types require more attention, making it hard to plan patrol and resource deployment effectively.
2. **No visibility into arrest status changes over time** - a case initially marked "no arrest" may later result in an arrest, but this progression is not tracked, so arrest-rate trends and investigation lag cannot be measured accurately.

## ❓Business Questions

**Spatial & Temporal Patterns**
- Which crime types occur most frequently in each district / community area?
- What time patterns (weekday vs. weekend, monthly trends) show the highest crime volume, and in which areas?
- Which beats/wards show a consistent increase in crime over the last N month?
- Are domestic-related incidents concentrated in specific area?
- Which hours of the day see the highest crime volume, and does this pattern differ by crime type (eg. burglary concentrated overnight vs. theft during the day)?
- Do certain districts show a distinct hour-of-day crime pattern that differs from the citywide average, useful for planning patrol shift timing?

**Arrest Tracking**
- How does the arrest rate change over time (quarter over quarter, year over year)?
- What percentage of initially unarrested cases result in an arrest later, and how long does that typically take?
- Which crime types have the highest rate of delayed arrests?

> Time-to-arrest are only reliable for status changes detected after pipeline go-live; earlier changes are not observable from the source API.

## 🔑 Key Metrics (Maybe change later...)

- Arrest Rate (by quarter, by crime type)
- Crime Count (by district, by time period)
- Case Resolution Lag (average time from incident to arrest)


