# Offshore Wind Maintenance Access Simulator

## Purpose
Build a small simulator that measures how weather limits maintenance access to an offshore. With the target of moving a farm from X km to Y km offshore cut availability by Z points in my model, and a more capable vessel recovered about W of them.

## Literature Review

[Notes] (./docs/notes.md) 

Various papers were survyed to gather the required data to build this simulation.

## Question

How does turbine availability change as an offshore wind farm moves further from shore, and how much of the loss does a more capable access vessel recover?

### Hypothesis

1. Further the distance of the wind farm, will result in less availability for accessing the wind farm, given the vessel used and size of crew.
2. Weather will be the largest contribution to accessibility, due to wave height and its effect on transit time, and vessel compatibility.
3. Energy loss will be greater on days most least accessible, due to high wind enegry output.
4. Seasons will align on accessibility trends by year  

## Model

### Independant

- Distance from port
- Vessel Type
- Turbine Type

### Dependant

- Availability
- Energy Loss
- Waiting time to repair

### Controlled 

- farm size
- turbine
- failure rates
- weather record
- shift length

| Part | What it does | Starting values |
| --- | --- | --- |
| Farm | N identical turbines, each either running or failed | 50 turbines, 8 MW each |
| Power | Converts hourly wind speed to output using a power curve | Cut-in, rated and cut-out speeds from a published reference turbine |
| Failures | Each running turbine fails at random at a fixed annual rate, split into categories | Minor repair, major repair; rates and repair hours from the literature |
| Weather | Hourly wind speed and significant wave height at the farm location | One North Sea grid point, 20+ years |
| Vessel | Can sail and transfer crew only while wave height stays under its limit | 1.5 m for a crew transfer vessel |
| Travel | Transit time each way equals distance divided by vessel speed | About 20 knots; verify |
| Shift | Crew work a fixed day from the port, including transit | 12 hours |
| Repair queue | Failed turbines wait their turn; oldest failure first | One vessel, one crew |

## Plan

### Experiment

Eight scenarios: four distances crossed with two vessels, everything else held fixed.

| Variable | Values |
| --- | --- |
| Distance from port | 20, 50, 100, 150 km |
| Vessel | Standard (1.5 m limit), capable (higher limit, value to be sourced) |
| Held fixed | Farm size, turbine, failure rates, weather location, shift length |

For each scenario, run the simulation many times with different random failure seeds over the same weather record, then report the mean and the spread.

### Build phases

Four phases, each ending in something that works and can be shown by itself. Durations assume full-time effort and are estimates.

| Phase | Build | Done when | Estimate |
| --- | --- | --- | --- |
| 1. Weather windows | Load the weather record; find every window where waves stay under a limit for at least N hours | You can state the share of days per month with a usable window, and a chart shows the seasonal pattern | 3 to 4 days |
| 2. One turbine | Random failures, one vessel, the access rule, repair carry-over | A single-turbine run gives an availability figure that passes the hand-calculated sanity check | 4 to 5 days |
| 3. Full farm | N turbines, repair queue, repeated runs with seeds, mean and spread | A 50-turbine scenario runs many times in reasonable time and results are reproducible from a seed | 4 to 5 days |
| 4. Experiment | The eight-scenario sweep, charts, write-up, README | The headline sentence is filled in with real numbers and the limitations are written down | 4 to 5 days |

## Assumptions and limitations

These are the starting simplifications.

| Assumption | Why it is a simplification | Likely effect on results |
| --- | --- | --- |
| Wave height alone decides access | Real limits also depend on wave period, direction, wind, current and daylight | Overstates accessibility |
| Perfect weather forecast | Real crews plan on uncertain forecasts and sometimes turn back | Overstates accessibility |
| One vessel, one crew | Real farms run several vessels and prioritise jobs | Understates availability |
| Failures independent of weather and age | Storms and ageing change failure rates | Unknown; probably understates winter downtime |
| Constant failure rate | Published work shows rates change over a turbine's life | Averages out over a long run |
| No spare-part or crane-vessel delays | Major repairs can wait weeks for a jack-up vessel | Overstates availability for major failures |
| Reanalysis weather at one grid point | Modelled, coarse-resolution data smooths short rough spells | Slightly overstates accessibility |
| No wake effects between turbines | Downwind turbines generate less | Overstates energy, not availability |