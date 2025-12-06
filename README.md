# Healthcare Associated Infections in Hospitals by Income Trends - DSCI511 Final Project Group 5
By Stefanie Jackson, Jonas Ventimiglia and Jin Ting Zhao

## Table of Contents

- [Summary](#summary)
- [Data Sources](#data-sources)
- [Imports/Installations](#importsinstallations)
- [Constants](#constants)
- [Acquiring the CMS Data](#acquiring-the-cms-data)
- [Preprocessing the CMS Data](#preprocessing-the-cms-data)
- [Acquiring the Census Data](#acquiring-the-census-data)
- [Acquiring the Geocoding Data](#acquiring-the-geocoding-data)
- [Final Preprocessing Step](#final-preprocessing-step)
- [Data Dictionary](#data-dictionary)
- [How to Recreate the Project](#how-to-recreate-the-project)


# Summary:
The dataset for the project includes U.S. hospitals with their Healthcare-Associated Infection (HAI) scores and related measures, geographic coordinates (latitude and longitude), and county-level poverty rates. It was created to support analysis of how hospital infection outcomes relate to community socioeconomic factors. The dataset was created from acquiring data from 3 different sources: Centers for Medicaid & Medicare Services (CMS), Census Data, and the Geospatial Data

The purpose of this dataset is to help identify patterns in infection disparities. For example, whether hospitals serving lower-income communities tend to have higher HAI scores—while also highlighting areas where additional infection prevention resources may be needed. It enables quality improvement teams to benchmark infection performance against hospitals serving similar patient populations, rather than only well-resourced facilities, and supports geographic visualization of infection burden across counties or census tracts to inform local public health efforts.

Dataset is located here - [**Final Dataset CMS_Census_Geodata_Merged.csv**](https://github.com/jintingzhao2/dsci511-project/blob/main/Final%20Dataset%20CMS_Census_Geodata_Merged.csv).


# Data Sources:
[Centers for Medicaid & Medicare Services (CMS) Data:](https://data.cms.gov/provider-data/api/1/datastore/query/77hc-ibv8/0)
- Facility ID, Measure ID, Measure Name, Benchmark Status, Numeric Scores
- Hospital Location & ZIP Code

[Census Data:](https://api.census.gov/data/2023/acs/acs5/subject)
- ACS 5-year Subject Table
- Median Household Income
- Poverty Percentage
- Tract ID, County, and State FIPS

[Geospatial Data:](https://geocoding.geo.census.gov/geocoder/geographies/address)
- Longitude and Latitude
- Tract ID
- Address, City, State

# Imports
```python
import requests
import pandas as pd
from concurrent.futures import ThreadPoolExecutor
```
The project requires multiple packages to be installed/imported to create the final dataset. The `requests` and `pandas` packages need to be installed since they are not built-in Pythonp ackages. The `requests` was imported to allow making API calls from our sources. `pandas` was imported to allow us to clean and view the data. In our code, `concurrent.futures` was imported to use `ThreadPoolExecutor`. This helped make concurrent API requests to CMS to quickly retrieve all the data since the maxiumun amount of rows per call is 1500. Concurrency allowed us do multiple API calls at the same time to allow the code to run faster and efficently. 

# Constants 
```python
CMS_BASE_URL = "https://data.cms.gov/provider-data/api/1/datastore/query/77hc-ibv8/0"
PAGE_SIZE = 1500 
CENSUS_API_KEY = # Since this repository is public, we have removed it for privacy issues
CENSUS_BASE_URL = "https://api.census.gov/data/2023/acs/acs5/subject"
GEOCODING_BASE_URL = "https://geocoding.geo.census.gov/geocoder/geographies/address"
```
For organization, the constants had their own code block. The constants were base URLs for the APIs to allow for better reuse. The PAGE_SIZE has the value 1500 due to the CMS API limitation. 

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
This first chunk of code is a function that fetches a single page from the CMS API. It builds the full request by combining CMS_BASE_URL with several query parameters that tell the API what to return and how to format the data. After assembling the full URL string, the function prints it for observability purposes. Then it uses `pandas.read_csv` to fetch the CSV data directly from the URL (this does a `GET` request behind the scence) and loads it into a Pandas DataFrame. Then the function returns the DataFrame.

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
To find out how many API calls needed to be done for the CMS data due to the 1500 row limitation, we wanted to find out the size of the CMS dataset. This allows us to make batch calls concurrently instead of using a for-loop to make one request at a time. After running this code, the result was 172,476 as the total number of rows.

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
This code saves the CMS data acquired into a csv file and shows the first few rows of the CMS data that was acquired. The next step is to clean up the CMS data.


# Preprocessing the CMS Data
```python
def filter_for_pa(df: pd.DataFrame) -> pd.DataFrame:
  return df.query("State == 'PA'")
```

We wanted to look at only hospitals in Pennsylvania (PA).


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
In the original CMS data, each row was a different measure name, so we weren't able to use the score as a column, which would make data analysis difficult. For example, one row would be representing number of days while another could be number of observed cases. In order to make the data more usable, we pivoted the the measure names from rows into columns to allow analysts to view each hospital, measure id, and their scores side by side. In addition, we renamed the Measure ID names to simplify and abbreviate it more since some of the names were very long.

# Acquiring the Census Data

```python
def get_census_data() -> pd.DataFrame:
  # Defining query parameters
  params = {
      "get": "NAME,S1901_C01_012E,S1701_C03_001E",  # Median Income, Poverty %
      "for": "tract:*",                             # All tracts
      "in": "state:42 county:*",                    # PA = FIPS 42, all counties in PA
      "key": CENSUS_API_KEY
  }

  # Making requests
  response = requests.get(CENSUS_BASE_URL, params=params)
  data = response.json()

  # Converting to DataFrame
  return pd.DataFrame(data[1:], columns=data[0])

census_df = get_census_data()
census_df = (
    census_df.rename(columns={
      "S1901_C01_012E": "median_income",
      "S1701_C03_001E": "poverty_percentage",
  }).drop(columns=["NAME", "state", "county"])
)
```
We call the Census API and feed the requests some parameters. These parameters allow us to select columns we want, such as median income and poverty %. It also allows us to filter only PA data. Additionally this API required an API key, which should always remain private, but for our project it will be public.

# Acquiring the Geocoding Data
```python
hospital_addresses = list(
    cms_transformed_df[["address", "city", "state"]].drop_duplicates()
    .itertuples(index=False, name=None)
)

def create_geocoding_url(street: str, city: str, state: str) -> str:
  print(street, city, state)
  return f"{GEOCODING_BASE_URL}?street={street}&city={city}&state={state}&benchmark=Public_AR_Current&vintage=Current_Current&layers=10&format=json"

with ThreadPoolExecutor(max_workers=10) as executor:
  responses = list(executor.map(
      lambda hospital_address: requests.get(
          create_geocoding_url(street=hospital_address[0], city=hospital_address[1], state=hospital_address[2])
      ), hospital_addresses
  ))

pa_geodata_data = [response.json() for response in responses]

pa_geodata = []
for data in pa_geodata_data:
  if data["result"]["addressMatches"]:
    pa_geodata.append({
        "city": data["result"]["input"]["address"]["city"],
        "street": data["result"]["input"]["address"]["street"],
        "state": data["result"]["input"]["address"]["state"],
        "x-coordinates": data["result"]["addressMatches"][0]["coordinates"]["x"],
        "y-coordinates": data["result"]["addressMatches"][0]["coordinates"]["y"],
        "tract": data["result"]["addressMatches"][0]["geographies"]["Census Block Groups"][0]["TRACT"]
    })

pa_geodata_df = pd.DataFrame(pa_geodata)
```

In order to map CMS data to Census data, we have to get the tract ID for CMS. We used the Geocoding API that given the street, city, and state, it will return a bunch of information, zip code, tract ID, x and y coordinates. To get the tract IDs for the hospitals we selected only the address, city, and state columns from CMS dataset and turned the dataframe into a list and deduped the values. Then we created the `create_geocoding_url` which takes in that information and calls the API once for those parameters. However for our use case we want to get that information for all the addresses, so we make concurrent API requests to speed things up. Then we process the responses, only get the tract IDs from the viable addresses, and finally convert it to a pandas dataframe.

# Final Preprocessing Step
```python
census_and_geodata_df = census_df.merge(pa_geodata_df, on="tract")
final_df = (
    cms_transformed_df
    .rename(columns={"address": "street"})
    .merge(
        census_and_geodata_df,
        on=["street", "city", "state"]
    )
)
```
For our final transformation, we have to merge all three data sources together. We apply an inner merge for Census with Geocoding on `tract`. Then we apply an inner merge for CMS with Census and Geocoding on the address.

# Data Dictionary
| Column Name | Description | Data Type |
|-------------|-------------|-----------|
| facility_id | Unique identifier for the hospital facility (CMS) | String / Integer |
| facility_name | Name of the hospital facility | String |
| street | Street address of the hospital | String |
| city | City or town where the hospital is located | String |
| state | State abbreviation | String |
| zip_code | ZIP Code of the hospital | String / Integer |
| county | County or parish of the facility | String |
| telephone_number | Hospital contact phone number | String |
| start_date | Start date of the reporting period | String (Date) |
| end_date | End date of the reporting period | String (Date) |
| cauti_sir | SIR for Catheter-Associated Urinary Tract Infections | Float |
| cauti_lcl | Lower confidence limit for CAUTI SIR | Float |
| cauti_catheter_days | Number of urinary catheter days | Integer / Float |
| cauti_observed | Observed CAUTI cases | Integer |
| cauti_predicted | Predicted CAUTI cases | Float |
| cauti_ucl | Upper confidence limit for CAUTI SIR | Float |
| clabsi_sir | SIR for Central Line-Associated Bloodstream Infections | Float |
| clabsi_lcl | Lower confidence limit for CLABSI SIR | Float |
| clabsi_observed | Observed CLABSI cases | Integer |
| clabsi_predicted | Predicted CLABSI cases | Float |
| clabsi_ucl | Upper confidence limit for CLABSI SIR | Float |
| clabsi_device_days | Number of device days for CLABSI | Integer / Float |
| cdiff_sir | SIR for C. difficile infections | Float |
| cdiff_lcl | Lower confidence limit for C. diff SIR | Float |
| cdiff_observed | Observed C. diff cases | Integer |
| cdiff_patient_days | Patient-days for C. diff measure | Integer / Float |
| cdiff_predicted | Predicted C. diff cases | Float |
| cdiff_ucl | Upper confidence limit for C. diff SIR | Float |
| mrsa_sir | SIR for MRSA bacteremia | Float |
| mrsa_lcl | Lower confidence limit for MRSA SIR | Float |
| mrsa_observed | Observed MRSA cases | Integer |
| mrsa_patient_days | Patient-days for MRSA measure | Integer / Float |
| mrsa_predicted | Predicted MRSA cases | Float |
| mrsa_ucl | Upper confidence limit for MRSA SIR | Float |
| ssi_hyst_sir | SIR for SSI – Abdominal Hysterectomy | Float |
| ssi_hyst_lcl | Lower confidence limit for SSI (hysterectomy) SIR | Float |
| ssi_hyst_procedures | Number of abdominal hysterectomy procedures | Integer |
| ssi_hyst_observed | Observed SSI cases (hysterectomy) | Integer |
| ssi_hyst_predicted | Predicted SSI cases for hysterectomy | Float |
| ssi_hyst_ucl | Upper confidence limit for SSI (hysterectomy) SIR | Float |
| ssi_colon_sir | SIR for SSI – Colon Surgery | Float |
| ssi_colon_lcl | Lower confidence limit for SSI (colon surgery) SIR | Float |
| ssi_colon_procedures | Number of colon surgery procedures | Integer |
| ssi_colon_observed | Observed SSI cases (colon surgery) | Integer |
| ssi_colon_predicted | Predicted SSI cases for colon surgery | Float |
| ssi_colon_ucl | Upper confidence limit for SSI (colon surgery) SIR | Float |
| median_income | Median household income for census tract | Integer / Float |
| poverty_percentage | Percent of population below poverty line | Float |
| tract | Census tract identifier | String / Integer |
| x-coordinates | Longitude of hospital (geocoded) | Float |
| y-coordinates | Latitude of hospital (geocoded) | Float |


# How to Recreate the Project 
1. Run these commands in the terminal
    ```bash
    > python3.13 -m venv venv
    > source venv/bin/activate
    > pip install -r requirements.txt
    ```
2. Then copy the Jupyter Notebook to Google Colab or run `jupyter notebook`.

