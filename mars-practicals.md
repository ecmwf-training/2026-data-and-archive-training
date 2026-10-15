# MARS practicals

*Source: [Confluence UDOC page 362414703](https://confluence.ecmwf.int/spaces/UDOC/pages/362414703/MS+computing+training+courses) — version 33, last updated 2026-10-12. Expandable "solution" boxes on the page are shown here as **Solution** blocks.*

---

Open a terminal and login to the ECS login node following practicals covered on Monday (Teleport or VDI) cd to HPCPERM folder:

```bash
ssh ecs-login
cd $SCRATCH
```

On ECS/HPC "mars" command is installed as part of MARS client and can be called without loading any module. The solutions below provide valid MARS requests which can be passed to the mars command as:

```bash
mars < myrequestfile
# or
mars myrequestfile
```

## Contents

1. [Basic MARS usage](#1-basic-mars-usage)
   1. [Simple retrieves](#11-simple-retrieves)
   2. [Post-processing](#12-post-processing)
2. [Advanced MARS usage](#2-advanced-mars-usage)
   1. [Efficiency](#21-efficiency)
   2. [List](#22-list)
   3. [Multi-target](#23-multi-target)
   4. [Ensemble forecasts](#24-ensemble-forecasts)
   5. [Read, Write, Compute](#25-read-write-compute)

---

# 1. Basic MARS usage

## 1.1. Simple retrieves

### 1.1.1. Exercise

Retrieve 1000, 850, 700 and 500 hPa temperatures from the 08th of October 2025 12 UTC Analysis [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=an&year=2025&month=oct&levtype=pl)

**Solution**

```
retrieve,
    class    = od,
    stream   = oper,
    type     = an,
    expver   = 1,
    date     = 20261010,
    time     = 12,
    param    = 130.128,
    levtype  = pressure level,
    levelist = 1000/850/700/500,
    target   = "t.grb"
```

Check the request output. Any "ERROR" or "WARN" messages? If not, check if the target file exists:

```bash
ls t.grb
```

On Thursday afternoon, you will learn more about [ecCodes](https://confluence.ecmwf.int/display/ECC/ecCodes+Home). For now, this is how you can use it to check content of your GRIB file:

```bash
module load ecmwf-toolbox
grib_ls t.grb
```

How many messages does the GRIB file contain? Does it match the number of expected fields retrieved from MARS?

**Solution**

```
grib_ls t.grb

edition      centre       typeOfLevel  level        dataDate     stepRange    dataType     shortName    packingType  gridType
1            ecmf         isobaricInhPa  1000         20261010     0            an           t            spectral_complex  sh
1            ecmf         isobaricInhPa  850          20261010     0            an           t            spectral_complex  sh
1            ecmf         isobaricInhPa  700          20261010     0            an           t            spectral_complex  sh
1            ecmf         isobaricInhPa  500          20261010     0            an           t            spectral_complex  sh

4 of 4 messages in t.grb      <-- (highlighted)
4 of 4 total messages in 1 files
```

### 1.1.2. Exercise

We run the first MARS request on a login node of ECS. Because of the limitations imposed on interactive sessions (e.g., memory, cpu time) it is better to use batch jobs to run MARS request which can be large with long runtime. Yesterday, you learned how to run fractional jobs. Let's use it know to execute next MARS request. The following template script can be used for this purpose:

> If the exercises are done on hpc-login instead of ecs-login, you need to change line `#SBATCH --qos=ef` with `#SBATCH --qos=nf`.

**mars_job.sh**

```bash
#!/bin/bash

#SBATCH --job-name=mars_job
#SBATCH --qos=ef
###SBATCH --time=1-01:00:00
#SBATCH --output=mars.%j.out
#SBATCH --error=mars.%j.out

# For larger requests downloading data from tapes
# extend walltime "--time" to 1 day or longer

set -xe

# Enable multi target option:
export MARS_MULTITARGET_STRICT_FORMAT=1

export SCRATCH
export PERM

mars <<EOF
#{paste your MARS request here}
EOF
```

More examples can be found on: [HPC2020: example Slurm serial batch job scripts for ECS](https://confluence.ecmwf.int/display/UDOC/HPC2020%3A+example+Slurm+serial+batch+job+scripts+for+ECS)

Use the template above to retrieve wind components valid for 5 days ago at 12:00 and 18:00 from forecast made 7 days ago at 12 UTC at all 137 model levels [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=fc&year=2025&month=oct&levtype=ml)

**Solution**

```
retrieve,
    class    = od,
    stream   = oper,
    type     = forecast,
    expver   = 1,
    param    = 131/132,
    levtype  = model level,
    levelist = 1/to/137,
    date     = -7,
    time     = 12,
    step     = 48/54,
    target   = "$SCRATCH/wind.grb"
```

Submit your MARS job with:

```bash
ecsbatch mars_job.sh
```

You can check the job status with:

```bash
watch ecsqueue -u $USER
```

Once the MARS job is complete, please check the job's output file, read MARS messages, check for any ERROR or WARNING messages. Read "INFO" messages, which action has taken longest time:

**Solution**

```
mars - INFO   - 20231029.184331 - Transfering 1798291592 bytes
mars - INFO   - 20231029.184333 - Deriving U and V from vorticity and divergence     <-- (highlighted)
mars - INFO   - 20231029.184404 - 548 fields retrieved from 'marsod'
```

From now on, we will use **mars_job.sh** template to run all MARS requests as batch jobs.

### 1.1.3. Exercise

Use the same batch job template (mars_job.sh) from the example 2) to run MARS request retrieving the 1000 hPa Geopotential from two weeks old 12 UTC forecast ([> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=fc&year=2025&month=oct&levtype=pl&date=2025-10-01&time=12:00:00)) steps: 12 hour to 240 hour with 12 hour intervals:

**Solution**

```
retrieve,
    class = od,
    stream = oper,
    type = fc,
    expver = 1,
    date = -14,
    time = 12,
    step = 12/to/240/by/12,
    levtype = pl,
    level = 1000,
    param = 129.128,
    target = "$SCRATCH/z1000_c.grb"
```

### 1.1.4. Exercise

Fix errors in request:

```
retrieve,
    class = od,
    date = -7,
    expver = 1,
    levelist = 500/700/1000,
    levtype = pl,
    param = 130.128,
    step = 83/to/93/by/1,
    stream = oper,
    time = 12:00:00,
    type = fc,
    target = "$SCRATCH/temp.pl.grb"
```

This request returns some errors:

```
mars - ERROR  - 20231030.070854 - Last error is -43
mars - ERROR  - 20231030.070854 - Expected 33, got 27.
mars - ERROR  - 20231030.070854 - Request failed
```

Retrieved number of fields is lower than requested. Use [MARS catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=fc&year=2023&month=oct&levtype=pl&date=2023-10-24&time=12:00:00) to see which of requested values are not available in MARS. You can use ecCodes to list content of resulting GRIB file and compare it to request:

```bash
grib_ls temp.pl.grb
```

Fix the MARS request by removing non-existing values from the request and re-run it again.

**Solution**

After step 90, steps are archived on 3 hours. Steps 91 and 92 do not exist in MARS.

```
retrieve,
    class = od,
    date = -7,
    expver = 1,
    levelist = 500/700/1000,
    levtype = pl,
    param = 130.128,
    step = 83/84/85/86/87/88/89/90/93,
    stream = oper,
    time = 12:00:00,
    type = fc,
    target = "$SCRATCH/temp.pl.grb"
```

## 1.2. Post-processing

### 1.2.1. Exercise

Retrieve the 1000, 500 and 300 hPa temperatures valid for day before yesterday at noon from the 12 UTC forecast of four days ago on an N48 Gaussian Grid [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?stream=oper&levtype=pl&time=12%3A00%3A00&expver=1&month=oct&year=2025&date=2023-10-11&type=fc&class=od)

**Solution**

```
retrieve,
    class    = od,
    stream   = oper,
    expver   = 1,
    type     = forecast,
    param    = 130.128,
    date     = -4,
    time     = 12,
    step     = 48,
    grid     = N48,
    levelist = 1000/500/300,
    levtype  = pl,
    target   = "$SCRATCH/t.gg.grb"
```

Retrieve the same on a latitude/longitude grid, 1 by 1 degrees. [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?stream=oper&levtype=pl&time=12%3A00%3A00&expver=1&month=oct&year=2025&date=2023-10-11&type=fc&class=od)

**Solution**

```
retrieve,
    class = od,
    stream = oper,
    expver = 1,
    type = forecast,
    param = 130.128,
    date = -4,
    time = 12,
    step = 48,
    grid = 1/1,
    levelist = 1000/500/300,
    levtype = pl,
    target = "$SCRATCH/t.ll.grb"
```

Retrieve European domain on the same latitude-longitude grid as in the previous retrieval. Compare the size of the resulting file with the previous one. [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?stream=oper&levtype=pl&time=12%3A00%3A00&expver=1&month=oct&year=2025&date=2023-10-11&type=fc&class=od)

**Solution**

```
retrieve,
    class = od,
    stream = oper,
    expver = 1,
    type = forecast,
    param = 130.128,
    date = -4,
    time = 12,
    step = 48,
    grid = 1/1,
    levelist = 1000/500/300,
    levtype = pl,
    area = e,
    target = "$SCRATCH/t.ll.europe.grb"
```

Use grib_ls to print grid type for each of the three GRIB files download in above requests. Is the resulting grid expected?

**Solution**

```
module load ecmwf-toolbox

grib_ls t.gg.grb
edition      centre       typeOfLevel  level        dataDate     stepRange    dataType     shortName    packingType  gridType
1            ecmf         isobaricInhPa  1000         20261008     48           fc           t            grid_simple  reduced_gg

grib_ls t.ll.grb
edition      centre       typeOfLevel  level        dataDate     stepRange    dataType     shortName    packingType  gridType
1            ecmf         isobaricInhPa  1000         20261008     48           fc           t            grid_simple  regular_ll
```

How many Latitude and Longitude points are found in the European domain with 1 * 1 degrees resolution?

**Solution**

```
grib_dump -p Ni,Nj t.europe.grb
...
  Ni = 73;
  Nj = 41;
...
```

### 1.2.2. Exercise

Retrieve surface winds for previous week's 24 hour forecast on the European area converting to a 0.57 by 0.57 degrees latitude/longitude grid. [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=an&year=2025&month=oct&levtype=sfc)
What happens to the conversion?

**Solution**

```
retrieve,
    class=od,
    date=-7,
    expver=1,
    levtype=sfc,
    param=165.128/166.128,
    step=24,
    stream=oper,
    time=12:00:00,
    type=fc,
    area = europe,                  # As the area is not compatible with the grid,
    grid = 0.57/0.57,               # MARS will expand the area to make it fit.
    target = "$SCRATCH/10u_10v.grb"
```

With grib_dump you will see that the area in **10u_10v.grb** and previously downloaded file from European area: **t.ll.europe.grb** are not identical.

---

# 2. Advanced MARS usage

## 2.1. Efficiency

In one of previous' requests output find a line looking like this:

```
mars - INFO   - ............... - The efficiency of your requests in the last 12 hours is
```

What does it say, what is efficiency of your MARS usage?

This score is based on number of times you access the same data tape during 12 hours. If you access the same tape more than once, the score will be lower than 100%. More accesses to the same tape - lower the score is. Conclusion: **One should download all required data from a single tape file in one go to avoid queueing for the same tape multiple times!**

Question: You need to download selection of surface parameters from operational analysis at 12 UTC, every day for two consecutive months September and October 2021. How many requests are you going to run?

Tip: Use [MARS catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=an&year=2021&month=sep&levtype=sfc) to see which analysis are stored together in a tape file.

**Solution**

An entire month of operational analysis for selected level type is stored in one tape file. Consequently, all needed analysis for one month should be download in one request to avoid queuing for the same tape multiple times. Two months are download in two requests, one for each month.

## 2.2. List

Find the request size of deterministic forecast for a historical date without downloading the data:

```
list,
    class=od,
    date=2022-07-01,
    expver=1,
    levtype=pv,
    levelist=1500/2000,
    param=54.128/129.128/131.128/132.128/133.128,
    step=1/to/96/by/1,
    stream=oper,
    time=12,
    type=fc
```

Since this only query MARS database without downloading the data and running interpolation, the request can be run interactively on the login node:

```bash
mars myrequestfile
```

instead of using batch system.

What is the total size of requested data?

**Solution**

```
Grand Total:
============
Entries       : 92
Total         : 305,955,384 (291.782 Mbytes)
mars - INFO   - 20231029.195448 - Request performed on database 'marsod'
mars - INFO   - 20231029.195448 - Memory used: 31.71 Mbyte(s)
mars - INFO   - 20231029.195448 - No errors reported
```

To find which portion of requested data is available online vs on tape, print the request "cost":

```
list,
    output=cost,
    class=od,
    date=2022-07-01,
    expver=1,
    levtype=pv,
    levelist=1500/2000,
    param=54.128/129.128/131.128/132.128/133.128,
    step=1/to/96/by/1,
    stream=oper,
    time=12,
    type=fc
```

Is the data available on-line or on tapes only?

**Solution**

```
....
size=8496119040;
number_of_fields=920;
online_size=0;
off_line_size=8496119040;
number_of_tape_files=1;
number_of_disk_files=0;
number_of_online_fields=0;
number_of_offline_fields=920;
number_of_tapes=1;      <-- (highlighted)
...
```

Run the same request for 7 days:

```
list,
    output=cost,
    class=od,
    date=2022-07-01/to/2022-07-07,
    expver=1,
    levtype=pv,
    levelist=1500/2000,
    param=54.128/129.128/131.128/132.128/133.128,
    step=1/to/96/by/1,
    stream=oper,
    time=12,
    type=fc
```

On how many tapes is the data distributed now?

**Solution**

```
...
number_of_tapes=2;      <-- (highlighted)
...
```

How about entire month of such data:

```
list,
    output=cost,
    class=od,
    date=2022-07-01/to/2022-07-31,
    expver=1,
    levtype=pv,
    levelist=1500/2000,
    param=54.128/129.128/131.128/132.128/133.128,
    step=1/to/96/by/1,
    stream=oper,
    time=12,
    type=fc
```

**Solution**

```
number_of_tapes=2;
```

**If you need to download this data for entire January 2020, how would you proceed:**

- **Running one request for entire month (pros / cons)**
- **Running 31 daily requests (pros / cons)**

**?**

## 2.3. Multi-target

*(This section is empty on the Confluence page.)*

## 2.4. Ensemble forecasts

### 2.4.1. Exercise

Retrieve the 2 metre temperature and 2 metre dew point temperature fields of the first 10 members of a week old 12 UTC ensemble prediction (type: perturbed forecast) for 2, 3 and 4 days in advance [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?stream=enfo&levtype=sfc&time=12%3A00%3A00&expver=1&month=oct&year=2023&date=2023-10-20&type=pf&class=od)

**Solution**

```
retrieve,
    class=od,
    date=-7,
    expver=1,
    levtype=sfc,
    number=1/to/10,
    param=167.128/168.128,
    step=48/to/96/by/24,
    stream=enfo,
    time=12,
    type=pf,
    target = "$SCRATCH/ensemble.[type].{shortName}.grb"
```

### 2.4.2. Exercise

Retrieve the same for the deterministic forecast. [> see in MARS Catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=enfo&expver=1&type=cf&year=2023&month=oct&levtype=sfc&date=2023-10-20&time=12:00:00)

**Solution**

```
retrieve,
    class=od,
    date=-7,
    expver=1,
    levtype=sfc,
    param=167.128/168.128,
    step=48/to/96/by/24,
    stream=oper,
    time=12,
    type=fc,
    target = "$SCRATCH/deterministic.[type].{shortName}.grb"
```

Note that this time we used the MARS keyword "type" and the ecCodes key "shortName" in the name of the target file. How many target files did the requests create?

**Solution**

2 target files

## 2.5. Read, Write, Compute

Retrieve U and V wind components at 10 metre from 10 day old analysis and calculate wind speed [> see in MARS catalogue](https://apps.ecmwf.int/mars-catalogue/?class=od&stream=oper&expver=1&type=an&year=2023&month=oct&levtype=sfc&date=2023-10-11&time=00:00:00)

```
retrieve,
    class = od,
    expver = 1,
    stream = oper,
    type = analysis,
    date = -10,
    time = 12,
    levtype = sfc,
    param = 165.128, # u
    grid = 1/1,
    fieldset = u
retrieve,
    param = 166.128, # v
    fieldset = v
compute,
    formula = "sqrt(u*u + v*v)",
    fieldset = speed
write,
    fieldset = speed,
    target = "windspeed"
```

Additional exercise:

This request will read previously created GRIB file, crop the European area from global domain and save it as another GRIB file:

```
read,
    source = "windspeed",
    grid = 1/1,
    area = europe,
    target = "europe_windspeed"
```

To convert previously created GRIB file to netCDF using [ecCodes](https://confluence.ecmwf.int/display/ECC/ecCodes+Home) in a terminal on the ECS or HPC:

```bash
>module load ecmwf-toolbox
>grib_to_netcdf -o windspeed.nc windspeed

grib_to_netcdf: Version 2.30.2
grib_to_netcdf: Processing input file 'windspeed'.
grib_to_netcdf: Found 1 GRIB field in 2 files.
grib_to_netcdf: Ignoring key(s): method, type, stream, refdate, hdate
grib_to_netcdf: Creating netCDF file 'windspeed.nc'
grib_to_netcdf: NetCDF library version: 4.9.1 of Feb  9 2023 13:54:09 $
grib_to_netcdf: Creating large (64 bit) file format.
grib_to_netcdf: Defining variable 'u10'.
grib_to_netcdf: Done.
```
