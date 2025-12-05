# Healthcare Associated Infections in Hospitals by Income Trends - DSCI511 Final Project Group 5
## By Stefanie Jackson, Jonas Ventimiglia and Jin Ting Zhao

# Summary:
The dataset created in this project includes U.S. hospitals along with their Healthcare-Associated Infection (HAI) scores and related measures, geographic coordinates (latitude and longitude), and county-level poverty rates. It was created to support analysis of how hospital infection outcomes relate to community socioeconomic factors. The dataset was created from acquiring data from 3 different sources: Centers for Medicaid & Medicare Services (CMS), Cencus Data, and the Geospatial Data

The purpose of this dataset is to help identify patterns in infection disparities—for example, whether hospitals serving lower-income communities tend to have higher HAI scores—while also highlighting areas where additional infection prevention resources may be needed. It enables quality improvement teams to benchmark infection performance against hospitals serving similar patient populations, rather than only well-resourced facilities, and supports geographic visualization of infection burden across counties or census tracts to inform local public health efforts.

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

