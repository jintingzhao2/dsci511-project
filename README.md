# Healthcare Associated Infections in Hospitals by Income Trends - DSCI511 Final Project Group 5
## By Stefanie Jackson, Jonas Ventimiglia and Jin Ting Zhao

# Summary:
The dataset created in this project includes U.S. hospitals along with their Healthcare-Associated Infection (HAI) scores and related measures, geographic coordinates (latitude and longitude), and county-level poverty rates. It was created to support analysis of how hospital infection outcomes relate to community socioeconomic factors. The dataset was created from acquiring data from 3 different sources: Centers for Medicaid & Medicare Services (CMS), Cencus Data, and the Geospatial Data

The purpose of this dataset is to help identify patterns in infection disparities—for example, whether hospitals serving lower-income communities tend to have higher HAI scores—while also highlighting areas where additional infection prevention resources may be needed. It enables quality improvement teams to benchmark infection performance against hospitals serving similar patient populations, rather than only well-resourced facilities, and supports geographic visualization of infection burden across counties or census tracts to inform local public health efforts.

No de-identification is needed since data is publicy available. The created dataset is added to this [repository](https://github.com/jintingzhao2/dsci511-project) for distrubition for public use. The created dataset is named "Final Dataset CMS_Census_Geodata_Merged.csv".


# Source of Data:
[Centers for Medicaid & Medicare Services (CMS) Data:](https://geocoding.geo.census.gov/geocoder/geographies/address)
- Facility ID, measure ID, measure name, benchmark status, numeric scores
- Hospital location & ZIP code

[Census Data:](https://api.census.gov/data/2023/acs/acs5/subject)
- ACS 5-year Subject Table
- Median household income
- Poverty percentage
- Tract, county, and state FIPS

[Geospatial Data:](https://geocoding.geo.census.gov/geocoder/geographies/address)
- Longitude and Latitude

# Imports/Installations
```python
import requests
import pandas as pd
from concurrent.futures import ThreadPoolExecutor
```
The project requires multiple packages to be installed/imported to create the final dataset. The "requests" and "pandas" packages need to be installation since they are not built into Python. The package "requests" was imported to allow making API calls from our sources. The "pandas" package was imported to allow us to clean and view the data in a cleaner way. In our code, another package "concurrent.futures" was imported in case someone running the code was running a version of Python older than 3.0. The ThreadPoolExecutor was imported from the package to help us make concurrent API requests to get the CMS data since the maxiumun amount of rows we can query per call at once is 1500, allowing us do multiple API calls at the same time to allow the code to run faster and efficently. 

# Constants 




