# Healthcare Associated Infections in Hospitals by Income Trends - DSCI511 Final Project Group 5
## By Stefanie Jackson, Jonas Ventimiglia and Jin Ting Zhao

# Summary:
The dataset created in this project includes U.S. hospitals along with their Healthcare-Associated Infection (HAI) scores and related measures, geographic coordinates (latitude and longitude), and county-level poverty rates. It was created to support analysis of how hospital infection outcomes relate to community socioeconomic factors. The dataset was created from acquiring data from 3 different sources: Centers for Medicaid & Medicare Services (CMS), Cencus Data, and the Geospatial Data

The purpose of this dataset is to help identify patterns in infection disparities—for example, whether hospitals serving lower-income communities tend to have higher HAI scores—while also highlighting areas where additional infection prevention resources may be needed. It enables quality improvement teams to benchmark infection performance against hospitals serving similar patient populations, rather than only well-resourced facilities, and supports geographic visualization of infection burden across counties or census tracts to inform local public health efforts.

No de-identification is needed since data is publicy available. The created dataset is added to this [repository](https://github.com/jintingzhao2/dsci511-project) for distrubition for public use. The created dataset is named "Final Dataset CMS_Census_Geodata_Merged.csv".


# Source of Data:
[Centers for Medicaid & Medicare Services (CMS) Data:](https://data.cms.gov/provider-data/api/1/datastore/query/77hc-ibv8/0)
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
This code saves the CMS data acquired into a csv file and shows the first fewrows of the CMS data that was acquired. The next step is to clean up the CMS data.


# Preprocessing the CMS Data
```python
def filter_for_pa(df: pd.DataFrame) -> pd.DataFrame:
  return df.query("State == 'PA'")
```

Since we wanted to look at only the hospitals in the state of Pennsylvania (PA), we filtered the CMS to contain hospitals in PA only.


```python
def pivot_by_measure_name(df: pd.DataFrame) -> pd.DataFrame:
  return (
      df.pivot(
        index=[
            "Facility ID", "Facility Name", "Address", "City/Town", "State",
            "ZIP Code", "County/Parish", "Telephone Number", "Start Date",
            "End Date"
        ],
        columns="Measure Name",
        values="Score",
    ).reset_index()
  )
```
In the original CMS data, each row was a measure name  To make the CMS dataset easier to understand, we pivoted the the measure names from rows into columns. 


```python
def rename_cms_columns(df: pd.DataFrame) -> pd.DataFrame:
  return df.rename(columns={
    "Facility ID": "facility_id",
    "Facility Name": "facility_name",
    "Address": "address",
    "City/Town": "city",
    "State": "state",
    "ZIP Code": "zip_code",
    "County/Parish": "county",
    "Telephone Number": "telephone_number",
    "Start Date": "start_date",
    "End Date": "end_date",

    # CAUTI (Catheter Associated UTI)
    "Catheter Associated Urinary Tract Infections (ICU + select Wards)": "cauti_sir",
    "Catheter Associated Urinary Tract Infections (ICU + select Wards): Lower Confidence Limit": "cauti_lcl",
    "Catheter Associated Urinary Tract Infections (ICU + select Wards): Number of Urinary Catheter Days": "cauti_catheter_days",
    "Catheter Associated Urinary Tract Infections (ICU + select Wards): Observed Cases": "cauti_observed",
    "Catheter Associated Urinary Tract Infections (ICU + select Wards): Predicted Cases": "cauti_predicted",
    "Catheter Associated Urinary Tract Infections (ICU + select Wards): Upper Confidence Limit": "cauti_ucl",

    # CLABSI
    "Central Line Associated Bloodstream Infection (ICU + select Wards)": "clabsi_sir",
    "Central Line Associated Bloodstream Infection (ICU + select Wards): Lower Confidence Limit": "clabsi_lcl",
    "Central Line Associated Bloodstream Infection (ICU + select Wards): Observed Cases": "clabsi_observed",
    "Central Line Associated Bloodstream Infection (ICU + select Wards): Predicted Cases": "clabsi_predicted",
    "Central Line Associated Bloodstream Infection (ICU + select Wards): Upper Confidence Limit": "clabsi_ucl",
    "Central Line Associated Bloodstream Infection: Number of Device Days": "clabsi_device_days",

    # C.Diff
    "Clostridium Difficile (C.Diff)": "cdiff_sir",
    "Clostridium Difficile (C.Diff): Lower Confidence Limit": "cdiff_lcl",
    "Clostridium Difficile (C.Diff): Observed Cases": "cdiff_observed",
    "Clostridium Difficile (C.Diff): Patient Days": "cdiff_patient_days",
    "Clostridium Difficile (C.Diff): Predicted Cases": "cdiff_predicted",
    "Clostridium Difficile (C.Diff): Upper Confidence Limit": "cdiff_ucl",

    # MRSA Bacteremia
    "MRSA Bacteremia": "mrsa_sir",
    "MRSA Bacteremia: Lower Confidence Limit": "mrsa_lcl",
    "MRSA Bacteremia: Observed Cases": "mrsa_observed",
    "MRSA Bacteremia: Patient Days": "mrsa_patient_days",
    "MRSA Bacteremia: Predicted Cases": "mrsa_predicted",
    "MRSA Bacteremia: Upper Confidence Limit": "mrsa_ucl",

    # SSI – Abdominal Hysterectomy
    "SSI - Abdominal Hysterectomy": "ssi_hyst_sir",
    "SSI - Abdominal Hysterectomy: Lower Confidence Limit": "ssi_hyst_lcl",
    "SSI - Abdominal Hysterectomy: Number of Procedures": "ssi_hyst_procedures",
    "SSI - Abdominal Hysterectomy: Observed Cases": "ssi_hyst_observed",
    "SSI - Abdominal Hysterectomy: Predicted Cases": "ssi_hyst_predicted",
    "SSI - Abdominal Hysterectomy: Upper Confidence Limit": "ssi_hyst_ucl",

    # SSI – Colon Surgery
    "SSI - Colon Surgery": "ssi_colon_sir",
    "SSI - Colon Surgery: Lower Confidence Limit": "ssi_colon_lcl",
    "SSI - Colon Surgery: Number of Procedures": "ssi_colon_procedures",
    "SSI - Colon Surgery: Observed Cases": "ssi_colon_observed",
    "SSI - Colon Surgery: Predicted Cases": "ssi_colon_predicted",
    "SSI - Colon Surgery: Upper Confidence Limit": "ssi_colon_ucl",
})

cms_transformed_df = (
    cms_df.pipe(filter_for_pa)
    .pipe(pivot_by_measure_name)
    .pipe(rename_cms_columns)
    .dropna()
)
```





