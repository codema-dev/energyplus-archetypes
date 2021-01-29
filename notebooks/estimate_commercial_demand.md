---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.9.1
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Commercial Modelling for Dublin using the VO Dataset

```python
cd ..
```

```python
import pandas as pd
import geopandas as gpd
import glob
from pathlib import Path
import numpy as np
import pygeos
import rtree
import matplotlib.pyplot as plt
```

```python
df = pd.concat(map(pd.read_csv, glob.glob('data/commercial/*.csv')))
```

```python
df['Area'] = df['Area'].fillna(0)
```

### Need to define area limits to correct for VO input errors

```python
df_area = df[(df['Area'] > 5) & (df['Area'] <= 50000 )]
```

```python
df_ext = df_area.drop_duplicates(subset=['Property Number', 'Area'])
```

```python
df_out = df_ext.groupby(by="Property Number", as_index=True)["Area"].sum().rename("property_total_area").to_frame().reset_index()
```

```python
df_merge = pd.merge(df, df_out, on="Property Number")
```

```python
df_final = df_merge.drop_duplicates(subset=['Property Number'])
```

```python
df_final["Uses_Clean"] = df_final['Uses'].str.replace(r', -', '')
```

### Cibse benchmarks provide references values per floor area

```python
benchmarks = pd.read_csv("data/commercial/benchmark_use_links.csv")
```

```python
bench_linked = pd.merge(df_final, benchmarks, left_on="Uses_Clean", right_on="Uses")
```

```python
bench_linked["ff_demand_kwh"] = bench_linked["property_total_area"] * bench_linked["typical_fossil_fuel_y"]
```

```python
bench_linked["elec_demand_kwh"] = bench_linked["property_total_area"] * bench_linked["typical_electricity_y"]
```

```python
bench_linked["esb_actual_kwh"] = bench_linked["property_total_area"] * (0.06*3600)
```

```python
bench_linked = gpd.GeoDataFrame(bench_linked, geometry=gpd.points_from_xy(bench_linked[" X ITM"], bench_linked[" Y ITM"]))
```

```python
bench_linked = bench_linked.set_crs(epsg = "2157")
```

```python
bench_linked = bench_linked.to_crs(epsg = "4326")
```

```python
postcode = gpd.read_parquet("data/spatial/dublin_postcodes.parquet")
```

```python
small_areas = gpd.read_parquet("data/spatial/small_area_geometries_2016.parquet")
```

```python
elec_pcode = gpd.sjoin(bench_linked, postcode, op="within")
```

```python
elec_sa = gpd.sjoin(bench_linked, small_areas, op="within")
```

```python
largest_consumer = elec_sa["elec_demand_kwh"].sort_values()
```

```python
pcode_demand_elec = elec_pcode.groupby("postcodes")["elec_demand_kwh"].sum().rename("cibse_postcode_elec_demand_kwh").reset_index()
```

```python
pcode_demand_esb = elec_pcode.groupby("postcodes")["esb_actual_kwh"].sum().rename("esb_postcode_elec_demand_kwh").reset_index()
```

```python
pcode_demand_ff = elec_pcode.groupby("postcodes")["ff_demand_kwh"].sum().rename("postcode_ff_demand_kwh").reset_index()
```

```python
pcode_demand_elec = pd.merge(pcode_demand_elec, postcode, on="postcodes")
```

```python
pcode_demand_elec = gpd.GeoDataFrame(pcode_demand_elec)
```

```python
pcode_demand_esb = pd.merge(pcode_demand_esb, postcode, on="postcodes")
```

```python
pcode_demand_esb = gpd.GeoDataFrame(pcode_demand_esb)
```

```python
pcode_demand_ff = pd.merge(pcode_demand_ff, postcode, on="postcodes")
```

```python
pcode_demand_ff = gpd.GeoDataFrame(pcode_demand_ff)
```

```python
sa_demand_elec = elec_sa.groupby("small_area")["elec_demand_kwh"].sum().rename("sa_elec_demand_kwh").reset_index()
```

```python
sa_demand_esb = elec_sa.groupby("small_area")["esb_actual_kwh"].sum().rename("sa_esb_demand_kwh").reset_index()
```

```python
sa_demand_ff = elec_sa.groupby("small_area")["ff_demand_kwh"].sum().rename("sa_ff_demand_kwh").reset_index()
```

```python
sa_demand_elec = pd.merge(sa_demand_elec, small_areas, on="small_area")
```

```python
sa_demand_elec = gpd.GeoDataFrame(sa_demand_elec)
```

```python
sa_demand_elec
```

```python
sa_demand_esb = pd.merge(sa_demand_esb, small_areas, on="small_area")
```

```python
sa_demand_esb = gpd.GeoDataFrame(sa_demand_esb)
```

```python
sa_demand_ff = pd.merge(sa_demand_ff, small_areas, on="small_area")
```

```python
sa_demand_ff = gpd.GeoDataFrame(sa_demand_ff)
```

```python
sa_demand_total = pd.merge(sa_demand_ff, sa_demand_elec, on="small_area")
```

```python
sa_demand_total = pd.merge(sa_demand_esb, sa_demand_total, on="small_area")
```

### Cibse Energy is the sum of the Elec & FF values

```python
sa_demand_total["sa_energy_demand_kwh"] = sa_demand_total["sa_ff_demand_kwh"] + sa_demand_total["sa_elec_demand_kwh"]
```

### Values in kWh are annual so divisible by 9760 to relate to kW

```python
sa_demand_total["sa_elec_demand_kw"] = sa_demand_total["sa_elec_demand_kwh"] / 8760
```

```python
sa_demand_final = sa_demand_total[["small_area", "sa_energy_demand_kwh", "sa_elec_demand_kwh", "sa_elec_demand_kw", "sa_esb_demand_kwh", "geometry_x"]]
```

```python
sa_demand_final = gpd.GeoDataFrame(sa_demand_final, geometry="geometry_x")
```

```python
pcode_demand_total = pd.merge(pcode_demand_ff, pcode_demand_elec, on="postcodes")
```

```python
pcode_demand_total = pd.merge(pcode_demand_esb, pcode_demand_total, on="postcodes")
```

```python
pcode_demand_total["postcode_energy_demand_kwh"] = pcode_demand_total["postcode_ff_demand_kwh"] + pcode_demand_total["cibse_postcode_elec_demand_kwh"]
```

```python
pcode_demand_final = pcode_demand_total[["postcodes", "postcode_energy_demand_kwh", "cibse_postcode_elec_demand_kwh", "esb_postcode_elec_demand_kwh", "geometry_x"]]
```

```python
pcode_demand_final = gpd.GeoDataFrame(pcode_demand_final, geometry="geometry_x")
```

```python
pcode_demand_final.plot(column="postcode_energy_demand_kwh", legend=True, legend_kwds={'label': "Commercial Energy Demand by Postcode (kWh)"})
```

```python
pcode_demand_final.plot(column="cibse_postcode_elec_demand_kwh", legend=True, legend_kwds={'label': "Commercial Elec Demand by Postcode (kWh)"})
```

```python
pcode_demand_final.to_csv("data/outputs/commercial_postcode_demands.csv")
```

```python
sa_demand_final.plot(column="sa_energy_demand_kwh", legend=True, legend_kwds={'label': "Commercial Energy Demand by Small Area (kWh)"})
```

```python
sa_demand_final.plot(column="sa_elec_demand_kwh", legend=True, legend_kwds={'label': "Commercial Elec Demand by Small Area (kWh)"})
```

```python
sa_demand_final.to_csv("data/outputs/commercial_sa_demands.csv")
```

## Calculating Peak Demands from SME Profiles

```python
peak = pd.read_csv("data/roughwork/cer_smart_meter/construction.csv")
```

```python
peak = pd.DataFrame(peak)
```

```python
import datetime as dt
```

```python
peak["datetime"] = pd.to_datetime(peak["datetime"])
```

```python
peak_df = peak[(peak["datetime"].dt.hour>=8) & (peak["datetime"].dt.hour<=20)]
```

```python
peak_df.groupby([peak_df["datetime"].dt.hour, peak_df["datetime"].dt.minute])["demand"].max()
```

```python
peak_df["demand"].max()
```

### Wrangling the VO dataset

```python
df_zero_area = df_final.loc[df_final["Area"] == 0]
```

```python
df_zero_area["Uses"].value_counts()
```

```python
df[df["Category"].str.contains("HOSPITALITY", na=False)]
```

```python
df_test = df_final[(df_final['Uses'].str.contains("HOTEL")) & (df_final['property_total_area'] <= 10000 )]
```

```python
df_final["Floor Use"].value_counts()
```

```python
largest_props = df_final["property_total_area"].sort_values()
```

```python
use_group = df_final.groupby("Uses")["property_total_area"].sum().sort_values()
```

```python
print(largest_props.to_string())
```

```python
df_final.loc[72632]
```

```python
df_final.loc[df_final["Uses"] == "APART / HOTEL, -"]
```

```python
df.loc[df["Property Number"] == 447026.0]
```

```python
print(largest_consumer.to_string())
```

```python
elec_sa.loc[23224]
```

```python
consumer_groups = elec_sa.groupby("Uses_Clean")["elec_demand_kwh"].sum().sort_values()
```

```python
print(consumer_groups.to_string())
```

```python
sa_demand_final["sa_elec_demand_kw"].sort_values()
```

```python
sa_demand_final.iloc[2031]
```

```python
elec_sa.loc[elec_sa["small_area"] == "268132008"]
```

```python

```