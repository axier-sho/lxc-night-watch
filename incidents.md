# Incidents

One entry per low-sunlight episode. An episode opens on the first reading below
450 W/m², is updated by every further low reading, and closes on the first
reading back at or above 450 W/m².

Format:

```
## Incident <n> — <open | recovered>
- Started: <UTC timestamp>
- Opening reading: <wPerM2> W/m2 (<condition>)
- Lowest reading: <wPerM2> W/m2
- Low readings observed: <count>
- Recovered: <UTC timestamp, or "not yet">
- Recovery reading: <wPerM2> W/m2 (<condition>)
```
