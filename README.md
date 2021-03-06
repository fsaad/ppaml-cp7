# PPAML Challenge Problem 7, phase 3

The task in this phase of CP7 is to predict [seasonal rates](https://en.wikipedia.org/wiki/Flu_season) of Influenza-Like Illness (_ILI_ or 'flu') in 60 distinct sub-populations of the continental US, ranging in size from the entire country to individual counties.

In addition to historical ILI rate data for each population, three different kinds of covariate data, representing flu-related tweets, vaccination claims, and weather, are provided for use in solutions.
In a _simulated forecast_ experiment, all four kinds of variables will be made available to solutions, one week of data at time, over the 32-week target season.

### Contents:

- Data sets
    - Population data
    - ILI rates
    - Tweet counts
    - Vaccinations
    - Weather
- Problem statement
- Evaluation protocol
    - Evaluator script

---

## Data sets

Data is provided as a single JSON file containing constants that describe the target populations, and as a set of CSV files that contain time-series variable data for some of the populations.
Each time series has weekly data following the [MMWR week calendar](epiweek.py).
The official CDC flu season runs from MMWR week 40 through week 20 of the following year.

Each week is encoded as a fixed-point number: for example, the last week in 2015 is represented as "2015.52".
In this document, "week _w_ + integer _k_" means the week that begins _k_ weeks after _w_:
for example, 2015.50 + 10 = 2016.08.


### Population data

The populations form a geographic containment hierarchy:

- The United States is the root
- [HHS Regions 1 – 10](https://www.hhs.gov/about/agencies/iea/regional-offices/index.html) divide the US at the lowest resolution
- An HHS region is composed of several states, identified with two-letter postal codes
- A state contains counties, each of which has a [four-digit FIPS code](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standard_state_code)

For states and counties, [`demographics.json`](data/demographics.json) provides population count estimates and other demographic information from the US Census Bureau, under the `"data"` key.
For counties, a list of FIPS codes of adjacent counties is also included.
FIPS codes of the counties or states that constitute each population can be found using the hierarchy under the `"indices"` key. 


### ILI rates

The CDC has a voluntary flu surveillance program called ILINet.
Each week, participating clinics submit counts of patients diagnosed with ILI to their state health department, along with their total patient counts.
These counts form the basis for the published rates for each HHS region.
Some state health departments also publish their weekly rates directly.

Filename | populations | source | first week | last week | notes
-------- | ----------- | ------ | ---------- | --------- | -----
<nobr>[`USA-flu.csv`](data/USA-flu.csv)</nobr> | Continental United States and 10 HHS Regions | CDC ILINet | 1997.40 | 2015.29 | off-season data missing from early years
<nobr>[`MS-flu.csv`](data/MS-flu.csv)</nobr> | Mississippi and 9 [Public Health Districts](http://msdh.ms.gov/msdhsite/_static/resources/3468.pdf) | MS Department of Health | 2012.48 | 2015.20 | no off-season data
<nobr>[`NC-flu.csv`](data/NC-flu.csv)</nobr> | North Carolina | NC Division of Public Health | 2001.40 | 2015.20 | includes diagnosed and total patient counts; no off-season data
<nobr>[`NJ-flu.csv`](data/NJ-flu.csv)</nobr> | New Jersey and 21 counties | NJ Department of Health | 2005.39 | 2015.20 | includes reported rates from long-term care facilities (`.ltc`), schools (`.sch`), and emergency clinics (`.emr`) in each county; off-season data only for 2009
<nobr>[`RI-flu.csv`](data/RI-flu.csv)</nobr> | Rhode Island | RI Department of Health | 2013.40 | 2015.20 | includes rates for five age ranges; no off-season data
<nobr>[`TN-flu.csv`](data/TN-flu.csv)</nobr> | Tennessee and  13 [Health Regions](https://tn.gov/health/topic/localdepartments) | TN Department of Health | 2009.32 | 2015.29 | six regions are individual counties; includes off-season data
<nobr>[`TX-flu.csv`](data/TX-flu.csv)</nobr> | Texas | TX Department of State Health Services | 2005.40 | 2015.29 | includes counts for four or five patient age ranges; includes off-season data starting in 2009

Some state health departments publish additional data which may be useful.
The column headers in each CSV are of the form `[POP].[VAR]`, with each `VAR` described in the table below.

Variable name | populations | meaning
------------- | ----------- | -------
`%ILI` | MS, NC, RI, TN, TX, USA | percentage of patients diagnosed with ILI
`#ILI` | NC, TX | number of patients diagnosed with ILI
`#patients` | NC, TX | total number of patients
`#sites` | TX | number of clinics reporting
`ltc` | NJ | percentage of ILI patients in long-term care facilities 
`sch` | NJ | percentage of ILI patients in schools 
`emr` | NJ | percentage of ILI patients in hospital emergency departments
<nobr>`age[H]-[L]`</nobr> | RI, TX | percentage of ILI patients between ages _H_ and _L_, inclusive <br> **note:** The TX health dept reported ages in four bins before week 2009.40, and five bins afterward. For those later weeks, column `TX.age25-64*` contains data for ages 25 – 49.


### Tweet counts

Geo-located tweets which included the words 'flu' or 'influenza' during a four-year period were aggregated to populations and MMWR weeks and counted to form a social media data set.

To support adjustments for the differences between the Twitter user base and the general US population, a small table of demographic information is provided.
The column labeled `% 2016 (US)` shows percentages of Twitter users in various demographic categories among _all_ US adults, while other columns show percentages among US adult _internet users_.

Filenames | source 
--------- | ------
<nobr>`[POP]-tweets.csv`</nobr> | [GNIP Historical PowerTrack](https://gnip.com/historical/historical-powertrack/)
[`twitter-demographics.csv`](data/twitter-demographics.csv) | Pew Research Social Media Updates [2016](http://www.pewinternet.org/2016/11/11/social-media-update-2016/) and [2014](http://www.pewinternet.org/2015/01/09/social-media-update-2014/)

Variable name | meaning
------------- | -------
`tc` | tweet counts


### Medicare vaccination claims

Medicare recipients are eligible for subsidized flu vaccinations.
The National Vaccine Program Office tracks the total number of eligible recipients for each county and flu season, for all ages as those 65 and older.
The NVPO records the vaccinated percentage of those eligible on a weekly basis.
These percentages are cumulative and thus non-decreasing over a flu season.

Filenames | source 
--------- | ------
<nobr>`[POP]-vaccinations.csv`</nobr> | [US Department of Health & Human Services National Vaccine Program Office](https://www.hhs.gov/nvpo/about/resources/interactive-mapping-tool-flu-vaccination-claims-by-week/)

Variable name | meaning
------------- | -------
`all` | total number of eligible recipients
`allV%` | percentage of eligible recipients vaccinated
`65+` | number of eligible recipients age 65 and over
`65+V%` | percentage of eligible recipients age 65+ vaccinated


### Weather data

In temperate climates like the continental US, flu epidemics are much more prevalent during cold weather.
To encourage teams to explore this correlation, aggregated weather data are provided for each MMWR week and each population.

Filenames | source 
--------- | ------
<nobr>`[POP]-weather.csv`</nobr> | [National Oceanic and Atmospheric Administration _GHCN-Daily_](https://www.ncdc.noaa.gov/oa/climate/ghcn-daily/)

Variable name | meaning
------------- | -------
`Tmax` | mean daily high temperature, in degrees Celsius
`Tmin` | mean daily low temperature, in degrees Celsius
`prcp` | mean daily percipitation, in millimeters

---

## Problem statement

Denoting ILI rate data for population _p_ and week _w_ as _I<sub>pw</sub>_, and similarly for all covariates _C_, a _forecaster_ for week _n_ extending _m_ weeks forward can be described as a function

 _F<sub>p,n,m</sub>_ : { _I<sub>p(w-2)</sub>_ , _C<sub>pw</sub>_ | _w_ ≤ _n_ ;∀ _p_ } → { _I<sub>pw</sub>_ | _n_ ≤ _w_ ≤ _n_ + _m_ }

Given data sets _I_ and _C_, as described above, solutions will produce a set of forecasts <nobr>{ _F<sub>p,n,m_</sub>(_I_, _C_) }</nobr> where

Parameter | value
--------- | -----
_p_ | teams may choose any of the 60 populations, but _must include_ <br> **HHS Region 4, TN state, and Knox County** (TN.D10 = FIPS 47093)
_n_ | each of the weeks 2015.40 ... 2016.20
_m_ | 0 ... remaining weeks in target season

Solutions may use both _I_ and _C_ data from any of the given populations to predict ILI rates for a specific population _p_. 
We are interested in how prediction accuracy for a given forecast period improves throughout the season, as new data is made available.

Each forecast will be evaluated against ground truth data <nobr>{ _I<sub>pw</sub>_ | _n_ ≤ _w_ ≤ _n_ + _m_ }</nobr> and assigned a sum of squared errors (SSE) score _s_ for each successive forecast period (week _n_, weeks _n_ through _n_ + 1, ... , weeks _n_ through _n_ + _m_).

Note that if _m_ = 0 then the problem is a simulated _nowcast_ rather than a forecast: the goal is to infer "current" ILI rates in the target populations, as of week _n_, from both current and historical covariate data as well as historical ILI data up to two weeks previous.

![Example forecast](example-forecast.png)

In chart above, _n_ = 2014.40, _m_ = 10, and _p_ = HHS Region 4.
Data points from three of the eight covariates are shown in warm colors.
(The * after their names indicates that they have been multiplied by scalars to fit on the chart.)
While _I_ and _C_ data prior to week 2013.32 are not shown, they are available to the forecaster function.
For evaluation, solutions will target the flu season beginning in week 2015.40, but teams are encouraged to test their solutions on data from previous seasons.

---

## Evaluation protocol

In concrete terms, ILI rates _I<sub>p(w-2)</sub>_ and covariates _C<sub>pw</sub>_ for all 60 populations _p_ and weeks _w_, 2015.20 < _w_ ≤ _n_ ≤ 2016.20, will be represented in a single file named `week-`_n_`.txt`.
This file will consist of concatenated CSV data of the same format as those provided, preceded by filenames, and followed by blank lines.
The basic idea is that each line of data could be appended to the appropriate CSV to continue the time series.
An [example file](data/week-2014.42.txt) is provided with data from the 2014–2015 season, covering weeks 2014.21 through 2014.42.
Because the CDC and health departments only publish ILI rates after a two-week delay, while data from other sources are available sooner, `week-`_n_`.txt` will contain data from the start of the season up to and including week _n_ for tweets, weather, and vaccination variables, but **only** up to week _n_ - 2 for ILI variables.

For each evaluation week _n_, 2015.40 ≤ _n_ ≤ 2016.20, solutions should read the `week-`_n_`.txt` file (as well as the contents of the present `data` directory) and produce a similar file `forecast-`_n_`.txt`, containing only lines with forecast ILI rates for weeks _n_ through _n_ + _m_ for each target population `[POP]-flu.csv`.

Note that the first evaluation data file, `week-2015.40.txt`, will include off-season baseline data for the 20 weeks 2015.21 through 2015.40, for all populations and covariates where this off-season data is available.
It will contain at most 18 weeks of data for ILI variables in the populations (TN, TX, and USA) which report off-season ILI rates, and less in the other populations, which do not.
The second evaluation file, `week-2015.41.txt`, will have covariate data for 21 weeks (and ILI data for 19); the third for 22 weeks, and so on.

A solution should present a command line interface in the form of a shell script with arguments:

`run.sh [CONFIG-FILE] [DATA-DIR] [WEEK-FILE]` so that a command like <br>
`$ run.sh solution.conf ../data/ week-2015.40.txt` writes the file `forecast-2015.40.txt`.

The configuration file should include some representation of forecast length _m_ and populations _p_.
At minimum, the output forecast file must include row _n_ + 1 for
column `R04.%ILI` of file `USA-flu.csv` and
columns `TN.%ILI`, `D10.%ILI` of file `TN-flu.csv`.
Solutions may save intermediate results between program runs to avoid recomputing models for each evaluation week.

Teams are encouraged (but not required) to submit solutions via GitHub, by forking this repository and adding program code outside the `data` directory.

### Evaluator script

A [Python script](Evaluator/evaluate.py) is included to calculate SSE scores of forecasts.
It should run under both Python 2 and 3.
If passed the `-p` flag, it can chart the forecast and ground-truth data together, using the standard `matplotlib` plotting package.

It requires a target and reference file, both containing CSV data over the same range of weeks.
By passing `-c [COLUMN]`, a specific column can be selected by name or (zero-based) index.
If `-c` is omitted, the variable of interest is assumed to be in column 1.
