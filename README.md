# Overview

[eland](https://github.com/elastic/eland) is Python Elasticsearch client for exploring and analyzing data residing in Elasticsearch with a familiar Pandas-compatible API.

Where possible the package uses existing Python APIs and data structures to make it easy to switch between numpy, pandas, scikit-learn to their Elasticsearch powered equivalents. In general, the data resides in Elasticsearch and not in memory, which allows Eland to access large datasets stored in Elasticsearch.Elasticsearch.

# Installation

> Eland can be installed from PyPI via pip:


```
pip install eland
````

> JupyterLab is a web-based interactive development environment for Jupyter notebooks, code, and data. JupyterLab is flexible: configure and arrange the user interface to support a wide range of workflows in data science, scientific computing, and machine learning. JupyterLab is extensible and modular: write plugins that add new components and integrate with existing ones.

JupyterLab can be installed using pip

```
pip install jupyterlab
````
> The Jupyter Notebook is an open-source web application that allows you to create and share documents that contain live code, equations, visualizations and narrative text. Uses include: data cleaning and transformation, numerical simulation, statistical modeling, data visualization, machine learning, and much more.

Jupyter Notebook  can be installed using pip

```
pip install notebook
````

To run the notebook, run the following command at the Terminal (Mac/Linux) or Command Prompt (Windows):

```
jupyter notebook
````

# Online Retail Analysis

The data used in this article is derived from a dataset referenced in [kaggle](https://www.kaggle.com/carrie1/ecommerce-data). This dataset was randomized. items and other data are fabricated and any resemblance to real data is coincidental.

Download [invoices.csv](./invoices.7z) and let's get started.

```python
import eland as ed
import pandas as pd
import matplotlib.pyplot as plt

# import elasticsearch-py client
from elasticsearch import Elasticsearch

# Function for pretty-printing JSON
def json(raw):
    import json
    print(json.dumps(raw, indent=2, sort_keys=True))
```

Let’s quickly go over the libraries I’ve imported:

- Eland — to load the data from file or elasticsearch as an eland data frame and analyze the data. 

- Pandas — to load the data file as a Pandas data frame and analyze the data.

- From Matplotlib I’ve imported pyplot in order to plot graphs of the data

Let's create an elasticsearch client using [python offcial client](https://elasticsearch-py.readthedocs.io/)

```python
# Connect to an Elasticsearch instance
# here we use the official Elastic Python client
# check it on https://github.com/elastic/elasticsearch-py
es = Elasticsearch(
  ['http://localhost:9200'],
  http_auth=("es_kbn", "changeme")
)
# print the connection object info (same as visiting http://localhost:9200)
# make sure your elasticsearch node/cluster respond to requests
json(es.info())
```

Here we will load our dataset from a csv file into a pandas dataframe

```python
# Load the dataset from the local csv file of call logs
pd_df = pd.read_csv("/home/telcos-ecs/eland_es_analytics/invoices.csv", sep=';', encoding = 'unicode_escape')
pd_df.info()
```

We can see tha type of the DataFrame returned is `pandas.core.frame.DataFrame`

```
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 541909 entries, 0 to 541908
Data columns (total 13 columns):
 #   Column             Non-Null Count   Dtype  
---  ------             --------------   -----  
 0   invoice_id         541909 non-null  object 
 1   item_id            541909 non-null  int64  
 2   item_model         541909 non-null  object 
 3   item_name          541909 non-null  object 
 4   item_brand         541909 non-null  object 
 5   item_vendor        541909 non-null  object 
 6   order_qty          541909 non-null  int64  
 7   invoice_date       541909 non-null  object 
 8   unit_price         541909 non-null  float64
 9   customer_id        541909 non-null  int64  
 10  country_name       541909 non-null  object 
 11  country_latitude   541909 non-null  float64
 12  country_longitude  541909 non-null  float64
dtypes: float64(3), int64(3), object(7)
memory usage: 53.7+ MB
```

Let's apply some tranformations to our dataset before indexing into elasticsearch

```python
#converting the type of Invoice Date Field from string to datetime.
pd_df['invoice_date'] = pd.to_datetime(pd_df['invoice_date'])

# Arrange prices for phones
pd_df['unit_price'] = pd_df['unit_price'] * 10.00

# Rename the columns to be snake_case
pd_df.columns = [x.lower().replace(" ", "_") for x in pd_df.columns]

# Combine the 'latitude' and 'longitude' columns into one column 'location' for 'geo_point'
pd_df["country_location"] = pd_df[["country_latitude", "country_longitude"]].apply(lambda x: ",".join(str(item) for item in x), axis=1)

# Drop the old columns in favor of 'location'
pd_df.drop(["country_latitude", "country_longitude"], axis=1, inplace=True)

pd_df.info()
```

```
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 541909 entries, 0 to 541908
Data columns (total 12 columns):
 #   Column            Non-Null Count   Dtype         
---  ------            --------------   -----         
 0   invoice_id        541909 non-null  object        
 1   item_id           541909 non-null  int64         
 2   item_model        541909 non-null  object        
 3   item_name         541909 non-null  object        
 4   item_brand        541909 non-null  object        
 5   item_vendor       541909 non-null  object        
 6   order_qty         541909 non-null  int64         
 7   invoice_date      541909 non-null  datetime64[ns]
 8   unit_price        541909 non-null  float64       
 9   customer_id       541909 non-null  int64         
 10  country_name      541909 non-null  object        
 11  country_location  541909 non-null  object        
dtypes: datetime64[ns](1), float64(1), int64(3), object(7)
memory usage: 49.6+ MB
```

Let's load the dataframe into elasticsearch using eland

```python
# Load the data into elasticsearch
ed_df = ed.pandas_to_eland(
    pd_df=pd_df,
    es_client=es,

    # Where the data will live in Elasticsearch
    es_dest_index="es-invoices",

    # Type overrides for certain columns, this can be used to customize index mapping before ingest
    es_type_overrides={
        "invoice_id": "keyword",
        "item_id": "keyword",
        "item_model": "keyword",
        "item_name": "keyword",     
        "item_brand": "keyword",
        "item_vendor": "keyword",   
        "order_qty": "integer",
        "invoice_date": "date",
        "unit_price": "float",  
        "customer_id": "keyword",
        "country_name": "keyword",
        "country_location": "geo_point"  
    },

    # If the index already exists what should we do?
    es_if_exists="replace",

    # Wait for data to be indexed before returning
    es_refresh=True,
)
ed_df.info()
```

We can see tha type of the DataFrame returned is `eland.dataframe.DataFrame`

```
<class 'eland.dataframe.DataFrame'>
Index: 541909 entries, 1500 to 541908
Data columns (total 12 columns):
 #   Column            Non-Null Count   Dtype         
---  ------            --------------   -----         
 0   country_location  541909 non-null  object        
 1   country_name      541909 non-null  object        
 2   customer_id       541909 non-null  object        
 3   invoice_date      541909 non-null  datetime64[ns]
 4   invoice_id        541909 non-null  object        
 5   item_brand        541909 non-null  object        
 6   item_id           541909 non-null  object        
 7   item_model        541909 non-null  object        
 8   item_name         541909 non-null  object        
 9   item_vendor       541909 non-null  object        
 10  order_qty         541909 non-null  int64         
 11  unit_price        541909 non-null  float64       
dtypes: datetime64[ns](1), float64(1), int64(1), object(9)
memory usage: 64.0 bytes
```

> Note that the data resides in Elasticsearch and not in memory, which allows Eland to access large datasets stored in Elasticsearch

We can get an eland data frame by reading directly the csv file and load to elasticsearch using eland

To get started, let’s create an `eland.DataFrame` by reading a csv file. This creates and populates the `es-customers` index in the local Elasticsearch cluster.

```python
ed_df = ed.read_csv("/home/telcos-ecs/eland_es_analytics/invoices.csv",
                es_client=es,

    # Where the data will live in Elasticsearch
    es_dest_index="es-invoices",

    # Type overrides for certain columns, this can be used to customize index mapping before ingest
    es_type_overrides={
        "invoice_id": "keyword",
        "item_id": "keyword",
        "item_model": "keyword",
        "item_name": "keyword",     
        "item_brand": "keyword",
        "item_vendor": "keyword",   
        "order_qty": "integer",
        "invoice_date": "date",
        "unit_price": "float",  
        "customer_id": "keyword",
        "country_name": "keyword",
        "country_location": "geo_point"  
    },

    # If the index already exists what should we do?
    es_if_exists="replace",

    # Wait for data to be indexed before returning
    es_refresh=True,
)
ed_df.info()
```

```
<class 'eland.dataframe.DataFrame'>
Index: 541909 entries, 1500 to 541908
Data columns (total 12 columns):
 #   Column            Non-Null Count   Dtype         
---  ------            --------------   -----         
 0   country_location  541909 non-null  object        
 1   country_name      541909 non-null  object        
 2   customer_id       541909 non-null  object        
 3   invoice_date      541909 non-null  datetime64[ns]
 4   invoice_id        541909 non-null  object        
 5   item_brand        541909 non-null  object        
 6   item_id           541909 non-null  object        
 7   item_model        541909 non-null  object        
 8   item_name         541909 non-null  object        
 9   item_vendor       541909 non-null  object        
 10  order_qty         541909 non-null  int64         
 11  unit_price        541909 non-null  float64       
dtypes: datetime64[ns](1), float64(1), int64(1), object(9)
memory usage: 64.0 bytes
```

Here we see that the "_id" field was used to index our data frame.

```python
ed_df.index.es_index_field
```

Next, we can check which field from elasticsearch are available to our eland data frame. columns is available as a parameter when instantiating the data frame which allows one to choose only a subset of fields from your index to be included in the data frame. Since we didn’t set this parameter, we have access to all fields.


```python
ed_df.columns

Index(['country_location', 'country_name', 'customer_id', 'invoice_date',
       'invoice_id', 'item_brand', 'item_id', 'item_model', 'item_name',
       'item_vendor', 'order_qty', 'unit_price'],
      dtype='object')
```

Go to Kibana `Dev Console` and check our index `es-invoices`

```
GET es-invoices/_search?track_total_hits=true&size=1
```

The response should be like

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 541909,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "es-invoices",
        "_id" : "138000",
        "_score" : 1.0,
        "_source" : {
          "invoice_id" : "553464",
          "item_id" : 35310308,
          "item_model" : "SM-J110H/DS",
          "item_name" : "Samsung SM-J110H/DS",
          "item_brand" : "Samsung",
          "item_vendor" : "Samsung Korea",
          "order_qty" : 12,
          "invoice_date" : "2019-05-17T11:07:00",
          "unit_price" : 16.5,
          "customer_id" : 16218,
          "country_name" : "Morocco",
          "country_location" : "31.791702,-7.09262"
        }
      }
    ]
  }
}
```

### Boolean Indexing
we also allow you to filter the data frame using boolean indexing. Under the hood, a boolean index maps to a terms query that is then passed to elasticsearch to filter the index.

```python
# the construction of a boolean vector maps directly to an elasticsearch query
print(ed_df['country_name'] == 'Morocco')
ed_df[(ed_df['country_name'] == 'Morocco')].head(5)
```
```
{'term': {'country_name': 'Morocco'}}
```