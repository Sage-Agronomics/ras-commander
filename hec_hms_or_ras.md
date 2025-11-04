# Flood Process Modeling: Choosing Between HEC-HMS and HEC-RAS

Flood process modeling involves two main components: **hydrological** and **hydraulic** modeling.
Understanding which one to use depends on the available data and the specific goal of the analysis.

---

## HEC-HMS vs HEC-RAS: Roles and Purpose

| Purpose                                                                                      | Model                     | Use it if...                                                                                                                     |
| -------------------------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Hydrological model** – converts rainfall to runoff (streamflow hydrograph)          | **HEC-HMS**         | There are no measured discharge (flow) data, and the goal is to simulate how rainfall turns into runoff at the catchment outlet. |
| **Hydraulic model** – simulates how water moves through rivers and across floodplains | **HEC-RAS (1D/2D)** | The goal is to create maps of flood depth, extent, velocity, and duration for fields or other areas of interest.                 |

---

## Typical Workflow

1. **Rainfall → Flow**HEC-HMS simulates how a rainfall event (e.g., a 1-in-50-year storm) produces a discharge hydrograph at a river outlet.
2. **Flow → Flood Maps**
   The hydrograph from HEC-HMS becomes the input for HEC-RAS, which models how that water spreads over the terrain.

---

## Which Model Is Needed?

### HEC-RAS – Definitely Needed

HEC-RAS is essential for generating:

- Flood depth, duration, and velocity per field
- Inputs for DSSAT to estimate yield impacts
- Flood risk and Average Annual Damage maps

These outputs require a **hydraulic model**, so **HEC-RAS 2D** is the correct choice.

---

### HEC-HMS – Probably Needed

If there are **no observed discharge data** or **design hydrographs** for return periods (e.g., 10-, 50-, 100-year floods),
then it’s necessary to generate them with **HEC-HMS**.

HEC-HMS uses rainfall, soil, and land cover data to calculate how much runoff enters the river system and when.
The resulting hydrographs are then passed to HEC-RAS for flood simulation.

---

## Shortcut Option

If discharge data (e.g., from hydrological services or national databases) are available,
it’s possible to skip HEC-HMS.
In that case, use the observed or statistically derived flow data to create synthetic hydrographs and feed them directly into HEC-RAS.

---

## Rule of Thumb

- Need **flood maps** → use **HEC-RAS**
- Need **flood flows from rainfall** → use **HEC-HMS**
- Need **both** → use **both**, coupled together

---

## Recommended Setup for Agricultural Flood Risk

For agricultural flood risk and DSSAT coupling, the optimal setup includes both models:

1. **Rainfall + Soil + DEM → HEC-HMS**Produces runoff hydrographs based on rainfall and watershed characteristics.
2. **HEC-HMS Hydrograph + High-Resolution DEM + Roughness Map → HEC-RAS 2D**Simulates water flow, flood extent, and duration.
3. **HEC-RAS Output + Field Polygons → Zonal Statistics → DSSAT Input**
   Converts flood depth and duration into stress conditions for yield impact simulation.

---

## Summary

- HEC-RAS is essential for simulating flood depth, duration, and extent.
- HEC-HMS is required if flow data are unavailable or rainfall-runoff processes need to be modeled.
- The two models are most powerful when used together in a **coupled hydrologic–hydraulic framework**.

---

*Next step:*
Assess available data (DEM resolution, discharge records, rainfall datasets) to decide whether HEC-HMS is necessary in the workflow.
