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
```python
CMS_BASE_URL = "https://data.cms.gov/provider-data/api/1/datastore/query/77hc-ibv8/0"
PAGE_SIZE = 1500 

CENSUS_API_KEY = #this is the API key, which can be requested from the CENSUS_BASE_URL. Since this repository is public, we have removed it for privacy issues
CENSUS_BASE_URL = "https://api.census.gov/data/2023/acs/acs5/subject"

GEOCODING_BASE_URL = "https://geocoding.geo.census.gov/geocoder/geographies/address"
```
The constants had their own section in the code to allow a more organized code. The link to the 3 websites we used to create the dataset had their own variable created so it would cause less confusion when reading the code. The PAGE_SIZE has the value 1500 due to the CMS API limitation. 

# Acquiring the CMS Data
```python
def fetch_page(offset: int) -> pd.DataFrame:
    """Fetch one page of data from CMS API."""
    url = (
        f"{CMS_BASE_URL}?offset={offset}&"
         "count=true&"
         "results=true&"
         "schema=true&"
         "keys=true&"
         "format=csv&"
         "rowIds=false"
    )
    print(f"Calling {url}")
    return pd.read_csv(url)
```
This first chunk of code is a function that fetches a single page or batch of data from the CMS API since the CMS dataset is stored in pages on the website. It builds the full request by combining CMS_BASE_URL with several query parameters that tell the API to return results. After assembling the full URL string, the function prints it for debuggings so we can see which page is being called. Then it uses pandas.read_csv to fetch the CSV data directly from the URL andload it into a Pandas DataFrame. Then the code returns the DataFrame so the caller can work with the retrived page of API results.

```python
def get_total_number_of_rows() -> int:
  initial_url = (
      f"{CMS_BASE_URL}?offset=0&count=true&results=true&schema=false&keys=true"
      "&format=json&rowIds=false"
  )
  response = requests.get(initial_url)
  response.raise_for_status()
  data = response.json()

  total_count = data["count"]
  print(f"Total number of rows: {total_count}")
  return total_count

total_number_of_rows_for_cms_data = get_total_number_of_rows()
```
To find out how many API calls needed to be done for the CMS data due to the 1500 row limitation, we wanted to find out the size of the CMS dataset. After runnining this code, the result was 172476 as the total number of rows.

```python
offsets = list(range(0, total_number_of_rows_for_cms_data, PAGE_SIZE))
print(f"Fetching {len(offsets)} pages concurrently...")
```
After finding out the total number of rows from the CMS dataset, we needed to find out how many times we needed to make the API request from the CMS API. This code was created to find out that number. The output result was 115 times. 

```python
def fetch_all(offsets, max_workers=10):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(fetch_page, offsets))
    return pd.concat(results, ignore_index=True)

cms_df = fetch_all(offsets)
```
This code block makes concurrent API requests to get all of the CMS data.

```python
print(cms_df.shape)
cms_df.to_csv("cms-data.csv", index=False)
cms_df.head()
```
This code saves the CMS data acquired into a csv file and shows the first fewrows of the CMS data that was acquired. 






