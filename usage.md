# RAS Commander Application Guide for Agricultural Flood Modeling

## Summary

RAS Commander is a Python automation library designed to programmatically control HEC-RAS 6.x and extract results from HDF5 output files. Within an integrated flood modeling framework, RAS Commander addresses the **hydraulic modeling stage**—the computationally intensive step that generates flood inundation maps for multiple return-period scenarios. It eliminates manual interaction with the HEC-RAS graphical interface and enables batch processing of numerous flood simulations.

---

## 1. Core Role: Hydraulic Modeling Automation

A flood modeling framework requires running HEC-RAS 2D simulations multiple times:

- Once for each return period (10-year, 50-year, 100-year flood events)
- Each with a different flow hydrograph produced by the hydrological model (HEC-HMS or equivalent)
- Each producing distinct flood depth, duration, and velocity maps

RAS Commander automates this repetitive, data-intensive process through a Python application programming interface (API).

---

## 2. Integration Points in the Modeling Chain

### 2.1 Input Requirements for Hydraulic Modeling Stage

| Input | Source | Description |
|-------|--------|-------------|
| Flow hydrograph | Hydrological model (HEC-HMS) | Time-series discharge data in m³/s |
| Digital Elevation Model | GEE or LiDAR | High-resolution terrain (1–3 m recommended for agricultural areas) |
| Manning's 'n' roughness map | Land use/land cover classification | Spatial field defining surface resistance to flow |
| HEC-RAS 2D project file | Manual configuration or existing workflow | Pre-configured project with computational mesh and domain |

### 2.2 RAS Commander Functional Components

#### Project and Plan Manipulation

- Load existing HEC-RAS project files (.prj)
- Modify boundary conditions (replace upstream flow boundary with new hydrograph for each return period)
- Update steady/unsteady flow files to reflect different precipitation scenarios
- Save modified project configuration without opening graphical user interface

#### Batch Computation Execution

- Execute HEC-RAS computational engine for each scenario programmatically via `RasCmdr.compute_plan()`
- Run simulations in parallel for multiple return periods using `RasCmdr.compute_parallel()`
- Monitor computation progress and capture error states programmatically

#### Results Data Extraction

- Access HDF5 output files produced by HEC-RAS
- Extract maximum water surface elevation rasters for each grid cell
- Extract inundation depth rasters (water surface elevation minus DEM elevation)
- Extract flow velocity rasters
- Query results at specific time steps for inundation duration analysis
- Convert extracted data to GeoTIFF or other GIS-compatible formats

### 2.3 Output Products

- Raster maps of flood depth for each return period (m)
- Raster maps of flow velocity for each return period (m/s)
- Raster maps of inundation extent for each return period (binary inundation state)
- Temporal data for duration analysis if multi-timestep extraction is performed

---

## 3. Operational Workflow

### 3.1 Framework Integration Sequence

```
┌─────────────────────────────────────────────────────────┐
│ Stage 1: Geospatial Data Preparation                    │
│ Input: GEE, soil maps, satellite imagery                │
│ Output: DEM, LULC, soil hydraulic properties            │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ Stage 2: Hydrological Modeling (HEC-HMS/SWAT/etc.)     │
│ Input: Rainfall (10, 50, 100-year return periods)      │
│ Output: Flow hydrographs (Q vs. time)                   │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │  RAS COMMANDER STAGE  │
         │  Hydraulic Modeling   │
         │  (HEC-RAS 2D)         │
         └───────────┬───────────┘
                     │
      Output: Flood depth, velocity, duration rasters
                     │
┌────────────────────▼────────────────────────────────────┐
│ Stage 3: Zonal Statistics & Field-Level Aggregation    │
│ Input: Flood rasters, farm field polygons               │
│ Output: Field_ID | Flood_Depth | Flood_Duration table  │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ Stage 4: Crop Impact Simulation (DSSAT)                │
│ Input: Baseline yields, flood stress parameters        │
│ Output: Yield loss per field and return period         │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ Stage 5: Risk Assessment & Economic Analysis           │
│ Input: Yield losses, market prices                      │
│ Output: Risk matrix, Average Annual Damage (AAD)        │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Pseudocode Implementation

```python
# Load HEC-RAS project (configured once with DEM, domain, computational mesh)
ras_project = RasCmdr.load_project("agricultural_area_model.prj")

# Define return periods to model
return_periods = [10, 50, 100]  # years

# For each return period scenario
for return_period in return_periods:
    
    # Step 1: Obtain hydrograph from hydrological model
    hydrograph_data = run_hydrological_model(
        return_period=return_period,
        model_type="HEC_HMS"
    )
    
    # Step 2: Update HEC-RAS boundary condition with new hydrograph
    ras_project.update_unsteady_flow_boundary(
        location="upstream_boundary",
        hydrograph=hydrograph_data,
        scenario_name=f"scenario_{return_period}yr"
    )
    
    # Step 3: Execute HEC-RAS 2D simulation
    RasCmdr.compute_plan(
        project=ras_project,
        plan_name=f"plan_{return_period}yr",
        output_path=f"./results/{return_period}yr/"
    )
    
    # Step 4: Extract depth and velocity rasters from HDF5
    depth_raster = RasCmdr.extract_maximum_depth_raster(
        hdf5_file=f"./results/{return_period}yr/Results.hdf5",
        output_format="GeoTIFF",
        crs="EPSG:4326"
    )
    
    velocity_raster = RasCmdr.extract_maximum_velocity_raster(
        hdf5_file=f"./results/{return_period}yr/Results.hdf5",
        output_format="GeoTIFF",
        crs="EPSG:4326"
    )
    
    # Step 5: Perform zonal statistics for field-level aggregation
    field_statistics_table = compute_zonal_statistics(
        depth_raster=depth_raster,
        velocity_raster=velocity_raster,
        field_polygons="farm_fields.shp",
        statistics=["mean", "max", "std"]
    )
    
    # Step 6: Pass flood stress parameters to DSSAT
    for field_record in field_statistics_table:
        field_id = field_record["field_id"]
        max_inundation_depth = field_record["depth_max"]
        inundation_duration = field_record["duration_days"]
        
        # Run baseline DSSAT (no flood stress)
        baseline_yield = run_dssat(
            field_id=field_id,
            scenario="baseline",
            flood_stress=False
        )
        
        # Run stressed DSSAT (with flood condition)
        stressed_yield = run_dssat(
            field_id=field_id,
            scenario=f"flooded_{return_period}yr",
            flood_stress=True,
            inundation_depth=max_inundation_depth,
            inundation_duration_days=inundation_duration
        )
        
        # Calculate yield loss
        yield_loss_absolute = baseline_yield - stressed_yield
        yield_loss_percent = (yield_loss_absolute / baseline_yield) * 100
        
        # Store results
        risk_matrix.append({
            "field_id": field_id,
            "return_period": return_period,
            "baseline_yield_t_ha": baseline_yield,
            "stressed_yield_t_ha": stressed_yield,
            "yield_loss_percent": yield_loss_percent,
            "inundation_depth_m": max_inundation_depth,
            "inundation_duration_days": inundation_duration
        })

# Generate final risk assessment output
output_risk_table(risk_matrix, output_format="GeoPackage")
calculate_average_annual_damage(risk_matrix)
```

---

## 4. Critical Advantages for Agricultural Flood Modeling

### 4.1 Scalability for Multiple Scenarios

A complete framework requires 3–4 distinct HEC-RAS runs (one per return period). Manual simulation via graphical interface would be time-consuming and subject to operator error. RAS Commander eliminates this constraint and enables parallel execution, reducing total wall-clock time significantly.

### 4.2 Reproducibility and Version Control

Since RAS Commander operates through Python scripts, all modeling decisions—boundary conditions, parameter selections, computational settings—are maintained in version-controlled code. This enables:

- Transparent model calibration through comparison of simulated hydrographs with observed river gauge data
- Sensitivity analysis to test effects of DEM resolution, Manning's 'n' parameterization, and mesh density on results
- Complete audit trails for agricultural risk assessments used in insurance or policy decisions

### 4.3 Data Extraction Automation

HEC-RAS outputs results in HDF5 format. RAS Commander provides direct programmatic access to these results without requiring manual file export through the graphical interface. This is essential when hundreds of individual farm field polygons require flood statistics computation and subsequent DSSAT processing.

### 4.4 Integration with Python Data Ecosystem

RAS Commander operates within Python, enabling seamless integration with:

- Geopandas and Rasterio for spatial data handling and zonal statistics computation
- NumPy and SciPy for numerical analysis and scientific computing
- Existing DSSAT controller scripts and crop model interfaces
- Google Earth Engine Python API for automated geospatial data acquisition

### 4.5 Reduction of Manual Workflow Steps

Without RAS Commander, the workflow requires:

1. Run HEC-HMS to generate hydrograph
2. Manually open HEC-RAS project file
3. Manually edit boundary condition files
4. Manually run computation
5. Manually export results to GIS format
6. Manually import into GIS software
7. Manually perform zonal statistics

RAS Commander consolidates steps 2–7 into automated code, reducing errors and improving efficiency.

---

## 5. Implementation Considerations

### 5.1 Prerequisites

#### Software Requirements

- HEC-RAS 6.x installed locally on computation system
- Python 3.7 or higher
- RAS Commander library installed via package manager or source installation

#### Data and Configuration Requirements

- Baseline HEC-RAS 2D project file pre-configured with:
  - High-resolution DEM (1–3 m resolution; acquired from LiDAR, USGS 3DEP, or equivalent high-accuracy sources)
  - Computational mesh defining the simulation domain with appropriate cell resolution
  - Manning's 'n' roughness field spatially parameterized by land use/land cover classification
  - Boundary conditions defined (upstream flow boundary, downstream stage boundary if applicable)
  - HDF5 output format specified in computation parameters

#### Computational Resources

- Sufficient disk space for storing HDF5 output files (potentially multi-gigabyte for high-resolution 2D simulations)
- Multi-core processor for parallel simulation execution
- 8 GB or greater RAM depending on mesh resolution and domain size

### 5.2 Model Calibration Workflow

Before production deployment, the framework requires calibration against historical observations:

1. Identify at least one (preferably multiple) historical flood events in the study area
2. Obtain historical rainfall data for those events
3. Run the hydrological model (HEC-HMS) with historical rainfall as input
4. Use RAS Commander to run HEC-RAS 2D with the resulting hydrograph
5. Compare simulated flood extent/depth with observed data (from field reports, satellite imagery, or gauge networks)
6. Adjust hydrological model parameters (particularly infiltration rates tied to pedotransfer functions) until simulated hydrographs match observed river gauge discharge data
7. Validate that hydraulic model outputs (depth, extent) align with observed flood extent

This calibration process is essential for model credibility in agricultural risk assessment and insurance applications.

### 5.3 DEM Resolution Considerations

A critical parameter for hydraulic modeling accuracy is DEM resolution. Within the framework context:

- **Hydrological modeling stage (HEC-HMS)**: 30 m resolution DEM from GEE is acceptable, as this stage operates at watershed scale
- **Hydraulic modeling stage (HEC-RAS 2D)**: Requires higher resolution. Agricultural-scale flood modeling is highly sensitive to micro-topography (ditches, farm roads, berms, culverts). A 30 m DEM will "smooth out" these features. Recommended minimum resolution: 1–3 m from LiDAR or USGS 3DEP

This resolution mismatch between hydrological and hydraulic models should be documented during framework design.

### 5.4 Limitations and Scope

#### RAS Commander Capabilities

- Automates HEC-RAS execution and result extraction
- Supports batch processing and parallel computation
- Provides programmatic access to HDF5 results

#### RAS Commander Limitations

- Does not include hydrological modeling functionality; HEC-HMS must be run separately
- Does not compute pedotransfer functions or soil hydraulic properties; PTF calculations must occur upstream
- Does not directly interface with DSSAT; zonal statistics and DSSAT parameterization must be handled by separate framework components
- Does not perform model calibration automatically; calibration workflow must be designed and executed by the operator

---

## 6. Data Flow and Dependencies

### 6.1 Input Data Specifications

| Data Type | Format | Source | Resolution/Units | Required |
|-----------|--------|--------|-------------------|----------|
| DEM | GeoTIFF | LiDAR, USGS 3DEP | 1–3 m (hydraulic), 30 m (hydrologic) | Yes |
| Land Use/Land Cover | GeoTIFF | Sentinel-2, Landsat | 10–30 m | Yes |
| Hydrograph | CSV or text time-series | HEC-HMS output | Q (m³/s) vs. time (hrs) | Yes |
| Manning's 'n' | GeoTIFF (raster field) | Derived from LULC | 10–30 m | Yes |
| Boundary conditions | HEC-RAS .u or .plan files | Manual configuration | Varies | Yes |
| Computational mesh | Embedded in .prj file | HEC-RAS mesh generator | Cell size (m) | Yes |

### 6.2 Output Data Specifications

| Output | Format | Use | Units |
|--------|--------|-----|-------|
| Flood depth raster | GeoTIFF | Zonal statistics, visualization | m |
| Flow velocity raster | GeoTIFF | Erosion assessment, visualization | m/s |
| Inundation extent raster | GeoTIFF or shapefile | Field overlap analysis | Binary |
| Field-level statistics table | CSV or GeoPackage | DSSAT input preparation | Varies |

---

## 7. Performance and Scalability

### 7.1 Computational Requirements by Scenario

| Aspect | Single Return Period | Four Return Periods | Notes |
|--------|---------------------|-------------------|-------|
| Sequential execution time | 30–120 min | 2–8 hours | Depends on mesh resolution and domain size |
| Parallel execution time | 30–120 min | 40–150 min | Using `compute_parallel()` with 4 cores |
| Disk storage (HDF5 output) | 0.5–2 GB | 2–8 GB | High-resolution, fine mesh requires more storage |
| Field-level processing time | 5–15 min | 20–60 min | Dependent on number of fields |

### 7.2 Optimization Strategies

- Execute RAS Commander simulations in parallel for different return periods on multi-core systems
- Pre-compute Manning's 'n' roughness field and DEM validation before running framework
- Use appropriate computational mesh resolution balancing accuracy and computation time
- Store results in efficient formats (HDF5 for intermediate results, GeoTIFF for final outputs)

---

## 8. Summary: RAS Commander Role in Flood Modeling Framework

| Framework Stage | Requirement | RAS Commander Solution | Status |
|---|---|---|---|
| Hydraulic modeling execution | Run 3–4 HEC-RAS scenarios with different hydrographs | Programmatic project modification, multi-scenario batch computation | Primary Function |
| Multi-scenario efficiency | Execute simulations rapidly and efficiently | Native parallel computation support via `compute_parallel()` | Native Support |
| Results accessibility | Extract flood depth, velocity, duration from HDF5 | Direct HDF5 parsing, GIS format conversion | Core Capability |
| Framework reproducibility | Version-control all modeling decisions | Python code captures project setup, parameters, boundary conditions | Enabled |
| System integration | Connect hydrological model (input) to DSSAT (output) | Python API enables data pipelines with upstream/downstream components | Supported |
| Model calibration | Enable iterative parameter testing against observations | Rapid scenario generation supports sensitivity analysis and tuning | Supported |
| Automation | Reduce manual operator intervention | Eliminates GUI-dependent steps | Primary Benefit |

---

## 9. Conclusion

RAS Commander serves as the automation engine for the hydraulic modeling stage within an integrated agricultural flood risk framework. By programmatically controlling HEC-RAS, extracting results from HDF5 files, and enabling parallel execution of multiple return-period scenarios, it transforms hydraulic modeling from a manual, time-intensive process into a reproducible, efficient component of an automated framework. When integrated with upstream hydrological modeling (HEC-HMS), soil parameterization (pedotransfer functions), downstream zonal statistics, and crop impact modeling (DSSAT), RAS Commander enables the quantitative assessment of field-level agricultural flood risk across multiple probability scenarios.
