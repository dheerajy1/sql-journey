This article will demonstrate the data ingestion and exploration for World Development Indicator datasets like World Bank Country & World Bank CO2 emissions (kt) & World Bank GDP & World Bank Life expectancy provided by The World Bank Open Data.

# The World Bank

World Bank Open Data

# 1\. Import libraries

```python
import requests
import pandas as pd
import copy
import pyodbc
from sqlalchemy import create_engine, text
from sqlalchemy.engine import URL
from sqlalchemy.orm import sessionmaker
from sqlalchemy.exc import OperationalError
import seaborn as sns
import matplotlib.pyplot as plt
import os

# Set the display option to show all max column width
pd.set_option('display.max_colwidth', None)

# Set the display option to show all columns
pd.set_option('display.max_columns', None)
```

# 2\. Ingest data from World Bank of ([<mark>World Development Indicators</mark>](https://datacatalog.worldbank.org/search/dataset/0037712/World-Development-Indicators))

* [API Basic Call Structures](https://datahelpdesk.worldbank.org/knowledgebase/articles/898581-api-basic-call-structures)
    

## 2.1 [<mark>World Bank Country</mark>](https://datahelpdesk.worldbank.org/knowledgebase/articles/898590-country-api-queries) data

* Requesting Country Data to list all countries: The following information will appear, when available, in the response when using this country query through the World Bank API:
    
    * 3 letter ISO 3166-1 alpha-3 code
        
    * 2 letter ISO 3166-1 alpha-2 code
        
    * Name
        
    * Region: ID, name and World Bank 2 letter code
        
    * Income Level: ID, name and World Bank 2 letter code
        
    * Lending Type: ID, name and World Bank 2 letter code
        
    * Capital City
        
    * Longitude
        
    * Latitude
        

### 2.1.1 Url Connection setup

```python
wbctry_code_url = (
    f"http://api.worldbank.org/v2/country"
    f"?format=json"
    f"&per_page=50000"
)
```

### 2.1.2 using Get request to ingest from url:

**<mark>Run the below cell only once</mark>**

```python
# Make the HTTP request.
response = requests.get(wbctry_code_url)

# Check the status of the request
if response.status_code == 200:
    raw_data = response.json()
    print("Request was successful.",response.headers['Content-Type'])
else:
    print(f"Request failed with status code: {response.status_code}")
```

* headers
    

```python
response.headers
```

* contents
    

```python
response.content
```

### 2.1.3 Exploring json response

```python
type(raw_data), len(raw_data)
```

```python
raw_data[0]
```

```python
raw_data[1][0]
```

```python
json_wbctry = copy.deepcopy(raw_data[1])
type(json_wbctry)
```

```python
len(json_wbctry)
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717394938478/2e56c1e5-aa11-47e5-8df6-960cde40fe82.png)

```python
json_wbctry[0]
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717394970799/8e031e66-10ad-4e6c-a7e8-5513179260c4.png)


### 2.1.4 Destructuring json response

```python
destrc_json_wbctry = []

for ctry in json_wbctry:
    
    # Destructuring nested objects and sorting the keys
    destrc_json = {f"{k}_{nested_k}": nested_v.strip() for k, v in ctry.items() if isinstance(v, dict) for nested_k, nested_v in v.items()}
    srt_destrc_json = {k: destrc_json[k] for k in sorted(destrc_json.keys())}

    # Adding the main key to the destructured objects
    srt_destrc_json.update({k: v for k, v in ctry.items() if not isinstance(v, dict)})
    destrc_json_wbctry.append(srt_destrc_json)
```

```python
destrc_json_wbctry
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395015163/9695f67b-defd-4ee1-a02d-550cb36f12e0.png)

### 2.1.5 Use pandas DataFrame to convert the JSON data:

```python
wbctry = pd.DataFrame(data=destrc_json_wbctry)[[
    'id',
    'iso2Code',
    'name',
    'region_id',
    'region_iso2code',
    'region_value',
    'adminregion_id',
    'adminregion_iso2code',
    'adminregion_value',
    'incomeLevel_id',
    'incomeLevel_iso2code',
    'incomeLevel_value',
    'lendingType_id',
    'lendingType_iso2code',
    'lendingType_value',
    'capitalCity',
    'longitude',
    'latitude',
]]
wbctry.dtypes, wbctry.shape
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395042020/6c2f73bb-de57-48ef-ac02-2e6469df3722.png)

```python
wbctry
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395068673/dd234abb-6f6b-4dd6-9602-f8671e98bdd8.png)

### 2.1.6 Transfromations

##### 2.1.6.1 Renaming column

```python
wbctry_filt = wbctry.rename(columns={'id': 'iso2Code', 'iso2Code': 'id'})
```

##### 2.1.6.2 Filtering data

```python
wbctry_filt.loc[wbctry_filt['incomeLevel_value'] != 'Aggregates'].reset_index(drop=True, inplace=True)
```

#### 2.1.6.2 Rearranging

```python
wbctry_filt = wbctry_filt[[
    'id',
    'iso2Code',
    'name',
    'region_id',
    'region_iso2code',
    'region_value',
    'adminregion_id',
    'adminregion_iso2code',
    'adminregion_value',
    'incomeLevel_id',
    'incomeLevel_iso2code',
    'incomeLevel_value',
    'lendingType_id',
    'lendingType_iso2code',
    'lendingType_value',
    'capitalCity',
    'longitude',
    'latitude',
]].copy()
```
![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395831182/fe4fbf01-3fcd-473a-a905-0bf273313e4c.png)


## 2.2 Finding source Id

### 2.2.1 Url Connection setup

```python
wbsrc_code_url = (
    f"https://api.worldbank.org/v2/sources"
    f"?format=json"
    f"&per_page=50000"
)
```

### 2.2.2 using Get request to ingest from url:

**<mark>Run the below cell only once</mark>**

```python
# Make the HTTP request.
response = requests.get(wbsrc_code_url)

# Check the status of the request
if response.status_code == 200:
    raw_data = response.json()
    print("Request was successful.",response.headers['Content-Type'])
else:
    print(f"Request failed with status code: {response.status_code}")
```

```python
response.headers
```

```python
response.content
```

### 2.2.3 Exploring json response

```python
type(raw_data), len(raw_data)
```

```python
raw_data[0]
```

```python
json_wbsrc = copy.deepcopy(raw_data[1])
type(json_wbsrc)
```

```python
json_wbsrc
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395354124/fa17f67f-1b17-4b98-84a0-6907ff63169b.png)

### 2.2.4 Use pandas DataFrame to convert the JSON data:

```python
wbsrc = pd.DataFrame(data=json_wbsrc)
wbsrc.dtypes, wbsrc.shape
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395390903/e1947c1c-8718-4319-90bf-7db1d55cdad0.png)

```python
wbsrc
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395414522/da9e0521-272e-4ea1-aeae-7552127b9061.png)

### 2.2.5 Get Source Id for a Database

```python
wdi_srcid = wbsrc.loc[
    wbsrc['name'].str.contains('World Development Indicators', case=False),
    'id'
].to_string(index=False)
wdi_srcid
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717395543666/bd589d3f-8d2f-49e8-9373-3b534e488bcd.png)

---

## 2.3 [<mark>World Bank CO2 emissions (kt)</mark>](https://datahelpdesk.worldbank.org/knowledgebase/articles/898599-indicator-api-queries)

### 2.3.1 Finding indicator Id

#### 2.3.1.1 Url Connection setup

```python
schkeywrd = 'CO2'

wbindicaco_code_url = (
    f"https://api.worldbank.org/v2"
    f"/sources"
    f"/{wdi_srcid}"
    f"/search"
    f"/{schkeywrd}"
    f"?format=json"
    f"&per_page=20000"
)
```

#### 2.3.1.2 using Get request to ingest from url:

**<mark>Run the below cell only once</mark>**

```python
# Make the HTTP request.
response = requests.get(wbindicaco_code_url)

# Check the status of the request
if response.status_code == 200:
    raw_data = response.json()
    print("Request was successful.",response.headers['Content-Type'])
else:
    print(f"Request failed with status code: {response.status_code}")
```

```python
response.headers
```

```python
response.content
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717396061270/44db1cb4-17ac-4c29-8e71-709f812270bc.png)


#### 2.3.1.3 Exploring json response

```python
type(raw_data), len(raw_data)
```

```python
raw_data
```

```python
raw_data.keys()
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397350113/88432bad-17a1-43a7-8421-6ec328bc56e7.png)

```python
type(raw_data['source']), len(raw_data['source'])
```

```python
raw_data['source'][0]['concept'][0]['variable']
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397453699/0148d9c1-8c0c-4a98-804c-c6bf7db604cb.png)

```python
json_wbindicaco = copy.deepcopy(raw_data['source'][0]['concept'][0]['variable'])
type(json_wbindicaco)
```

```python
json_wbindicaco[0]
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397480850/28ab91a3-ef21-44b1-b588-ed8dc05b8f5d.png)

#### 2.3.1.4 Destructuring json response

```python
destrc_json_wbindicaco = []

for indica in json_wbindicaco:
    destrc_json = {}

    for k, v in indica.items():
        if isinstance(v, list):
            for i, nested_v in enumerate(v):
                for nested_k, nested_val in nested_v.items():
                    destrc_json[f"{k}_{i}_{nested_k}"] = nested_val
        else:
            destrc_json[k] = v

    destrc_json_wbindicaco.append(destrc_json)
```

```python
destrc_json_wbindicaco
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397537520/facbaead-03be-48f5-9d30-5e5769493b51.png)



#### 2.3.1.5 Use pandas DataFrame to convert the JSON data:

```python
wbindicaco = pd.DataFrame(data=destrc_json_wbindicaco)
wbindicaco.dtypes, wbindicaco.shape
```

#### 2.3.1.6 changing the data type

```python
wbindicaco['metatype_1_value'] = wbindicaco['metatype_1_value'].astype(str)
```

```python
wbindicaco
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397625721/08572be4-e005-4bbc-9d06-80ecb67cc6a1.png)

#### 2.3.1.7 Get indicator Id

```python
coindicaid = wbindicaco.loc[
    wbindicaco['metatype_1_value'].str.contains(r'CO2 emissions \(kt\)', case=False),
    'id'
].to_string(index=False)
coindicaid
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397703643/2dc1d248-f3a1-4e1a-9ffd-11dddf39187a.png)

### 2.3.2 CO2 emissions (kt) data

#### 2.3.2.1 Url Connection setup

```python
wbcoemi_code_url = (
    f"http://api.worldbank.org/v2"
    f"/country"
    f"/all"
    f"/indicator"
    f"/{coindicaid}"
    f"?format=json"
    f"&per_page=20000"
)
```

#### 2.3.2.2 using Get request to ingest from url:

```python
# Make the HTTP request.
response = requests.get(wbcoemi_code_url)

# Check the status of the request
if response.status_code == 200:
    raw_data = response.json()
    print("Request was successful.",response.headers['Content-Type'])
else:
    print(f"Request failed with status code: {response.status_code}")
```

```python
response.headers
```

```python
response.content
```

#### 2.3.2.3 Exploring json response

```python
type(raw_data), len(raw_data)
```

```python
raw_data[0]
```

```python
json_wbcoemi = copy.deepcopy(raw_data[1])
type(json_wbcoemi)
```

```python
json_wbcoemi[0]
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397808996/2235576a-43cc-4303-96b0-d546d46eaf41.png)

#### 2.3.2.4 Destructuring json response

```python
destrc_json_wbcoemi = []

for coemi in json_wbcoemi:
    destrc_json = {}
    
    for k, v in coemi.items():
        if isinstance(v, dict):
            for nested_k, nested_v in v.items():
                destrc_json[f'{k}_{nested_k}'] = nested_v
        else:
            destrc_json[k] = v
    destrc_json_wbcoemi.append(destrc_json)
```

```python
destrc_json_wbcoemi
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397859121/dcf7eced-2240-4820-ae39-a0900544c90c.png)

#### 2.3.2.5 Use pandas DataFrame to convert the JSON data:

```python
wbcoemi = pd.DataFrame(data=destrc_json_wbcoemi)
wbcoemi.dtypes, wbcoemi.shape
```

```python
wbcoemi
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397895573/444598c0-ebce-4af1-b4d4-1518533827d0.png)

#### 2.3.2.6 Transformations

##### 2.3.2.6.1 Renaming column

```python
wbcoemi_trnsfmtn = wbcoemi.rename(columns={'date': 'year'}).add_prefix('co2_') #, 'value': 'co2_value'
```

##### 2.3.2.6.2 Sorting

```python
wbcoemi_trnsfmtn.sort_values(['co2_country_value', 'co2_year'], ascending=[True, True], inplace=True)
wbcoemi_trnsfmtn.reset_index(drop=True, inplace=True)
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717397966682/84a1640f-ddfd-4bea-a98e-75a639b734f3.png)

## 2.4 [<mark>World Bank GDP</mark>](https://datahelpdesk.worldbank.org/knowledgebase/articles/898599-indicator-api-queries) data

### 2.4.1 Finding indicator Id

#### 2.4.1.1 Url Connection setup

```python
schkeywrd = 'GDP'

wbindicagdp_code_url = (
    f"https://api.worldbank.org/v2"
    f"/sources"
    f"/{wdi_srcid}"
    f"/search"
    f"/{schkeywrd}"
    f"?format=json"
    f"&per_page=20000"
)
```

#### 2.4.1.2 using Get request to ingest from url:

**<mark>Run the below cell only once</mark>**

```python
# Make the HTTP request.
response = requests.get(wbindicagdp_code_url)

# Check the status of the request
if response.status_code == 200:
    raw_data = response.json()
    print("Request was successful.",response.headers['Content-Type'])
else:
    print(f"Request failed with status code: {response.status_code}")
```

```python
response.headers
```

```python
response.content
```

#### 2.4.1.3 Exploring json response

```python
type(raw_data), len(raw_data)
```

```python
raw_data
```

```python
raw_data.keys()
```

```python
type(raw_data['source']), len(raw_data['source'])
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717398600685/4555d46b-2fed-4f2c-9a3a-941f663e670b.png)

```python
raw_data['source'][0]['concept'][0]['variable']
```

```python
json_wbindicagdp = copy.deepcopy(raw_data['source'][0]['concept'][0]['variable'])
type(json_wbindicagdp)
```

```python
json_wbindicagdp[0]
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717398761184/89d53e2a-7b15-46ab-9501-7a3da7e62d8c.png)

#### 2.4.1.4 Destructuring json response

```python
destrc_json_wbindicagdp = []

for indica in json_wbindicagdp:
    destrc_json = {}

    for k, v in indica.items():
        if isinstance(v, list):
            for i, nested_v in enumerate(v):
                for nested_k, nested_val in nested_v.items():
                    destrc_json[f"{k}_{i}_{nested_k}"] = nested_val
        else:
            destrc_json[k] = v

    destrc_json_wbindicagdp.append(destrc_json)
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717398819836/b1510b0c-c25d-419e-8280-e28752ab4b70.png)


#### 2.4.1.5 Use pandas DataFrame to convert the JSON data:

```python
wbindicagdp = pd.DataFrame(data=destrc_json_wbindicagdp)
wbindicagdp.dtypes, wbindicagdp.shape
```

#### 2.4.1.6 changing the data type

```python
wbindicagdp['metatype_0_value'] = wbindicagdp['metatype_0_value'].astype(str)
```

```python
wbindicagdp
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717398879722/3729d715-6e09-47e1-b7f8-0a88dfd346df.png)

#### 2.4.1.7 Get indicator Id

```python
gdpindicaid = wbindicagdp.loc[
    wbindicagdp['metatype_0_value'].str.contains(r'GDP \(current US\$\)', case=False),
    'id'
].to_string(index=False)
gdpindicaid
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717398940826/28b6a578-d002-4e3c-b719-0c2230eb859f.png)


# Conclusion

Learning Objectives,

- Python & Pandas: Import libraries and use Pandas for data manipulation and analysis.

- Destructuring json response
- pandas DataFrame to convert the JSON data
- Transfromations: 
    - Renaming column, 
    - Filtering data,
    - Rearranging
- Finding source Id
- using Get request to ingest from url
- Exploring json response


# Source: Self \[[Link](https://databank.worldbank.org/reports.aspx?source=2&series=NY.GDP.MKTP.CD&country=)\], \[[Link](https://datahelpdesk.worldbank.org/knowledgebase/topics/125589-developer-information)\]

# Author: Dheeraj. Yss

# Connect with me:

* [My Twitter](https://twitter.com/yssdheeraj)
    
* [My LinkedIn](https://www.linkedin.com/in/dheerajy1/)
    
* [My GitHub](https://github.com/dheerajy1)