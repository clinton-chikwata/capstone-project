# Immigration Data Process
### Data Engineering Capstone Project

#### Project Scope
This projects aims to enrich the US I94 immigration data with further data such as demographics and temperature data to have a wider basis for analysis on the immigration data.

#### Describe and Gather Data

The following datasets are include:

1. I94 Immigration Data: This data comes from the US National Tourism and Trade Office. A data dictionary is included in the workspace. This is where the data comes from. There's a sample file so you can take a look at the data in csv format before reading it all in. You do not have to use the entire dataset, just use what you need to accomplish the goal you set at the beginning of the project.
2. World Temperature Data: This dataset came from Kaggle. 
3. U.S. City Demographic Data: This data comes from OpenSoft.
4. Airport Code Table: This is a simple table of airport codes and corresponding cities.

Immigration Data
You can access the immigration data in a folder with the following path:
> ../../data/18-83510-I94-Data-2016/. There's a file for each month of the year. 
> 
An example file name is i94_apr16_sub.sas7bdat. 
Each file has a three-letter abbreviation for the month name. 
So a full file path for June would look like this: ../../data/18-83510-I94-Data-2016/i94_jun16_sub.sas7bdat. 
Below is what it would look like to import this file into pandas. 
Note: these files are large, so you'll have to think about how to process and aggregate them efficiently.

fname = '../../data/18-83510-I94-Data-2016/i94_apr16_sub.sas7bdat'
> df = pd.read_sas(fname, 'sas7bdat', encoding="ISO-8859-1")

The most important decision for modeling with this data is thinking about the level of aggregation. Do you want to aggregate by airport by month? Or by city by year? This level of aggregation will influence how you join the data with other datasets. There isn't a right answer, it all depends on what you want your final dataset to look like.

Temperature Data
You can access the temperature data in a folder with the following path: ../../data2/. 
There's just one file in that folder, called GlobalLandTemperaturesByCity.csv.
Below is how you would read the file into a pandas dataframe.

> fname = '../../data2/GlobalLandTemperaturesByCity.csv'
df = pd.read_csv(fname)

### Explore and Assess the Data
#### Explore the Data

##### I94 Immigration Data
Initial schema after loaded to spark dataframe
```
root
|-- cicid: double (nullable = true)
|-- i94yr: double (nullable = true)
|-- i94mon: double (nullable = true)
|-- i94cit: double (nullable = true)
|-- i94res: double (nullable = true)
|-- i94port: string (nullable = true)
|-- arrdate: double (nullable = true)
|-- i94mode: double (nullable = true)
|-- i94addr: string (nullable = true)
|-- depdate: double (nullable = true)
|-- i94bir: double (nullable = true)
|-- i94visa: double (nullable = true)
|-- count: double (nullable = true)
|-- dtadfile: string (nullable = true)
|-- visapost: string (nullable = true)
|-- occup: string (nullable = true)
|-- entdepa: string (nullable = true)
|-- entdepd: string (nullable = true)
|-- entdepu: string (nullable = true)
|-- matflag: string (nullable = true)
|-- biryear: double (nullable = true)
|-- dtaddto: string (nullable = true)
|-- gender: string (nullable = true)
|-- insnum: string (nullable = true)
|-- airline: string (nullable = true)
|-- admnum: double (nullable = true)
|-- fltno: string (nullable = true)
|-- visatype: string (nullable = true)
```

The following changes a made
- Removed rows with invalid  i94ports, i94cit and i94res values
- dropped `['i94mon', 'entdepd', 'insnum', 'entdepu', 'matflag', 'entdepa', 'count', 'i94yr']` columns because they dont have clear description or cantain too many null values
- renamed columns: `i94bir` to `age`, `i94cit` to `coc`, `i94res` to `cor`, `i94port` to `port_code`, `i94addr` to`landing_state`, `visapost` to `visa_issued_in`, `cicid` to `id`
- changed the date format on `arrdate`, `depdate`, `dtadfile`, `dtaddto`
- changed `i94mode` and `i94visa` column values from numbers to descriptive texts

##### World Temperature Data
- removed non-US data
- dropped columns: `['AverageTemperatureUncertainty', 'Latitude', 'Longitude', 'Country']`

##### U.S. City Demographic Data
- created a separate table `general` and `race` to remove redundancy

##### Airport Codes
- removed non-US data
- removed rows wit null `iata_codes`
- split `coordinates` column to separate `latitude` and `longitude` columns
- created a `state_code` column from `iso_region`
- dropped this columns: `['coordinates', 'iso_country', 'continent', 'iso_region']`

##### I94 SAS Labels Descriptions
- from this file I created the following tables: `countries.csv`, `us_ports.csv`, `us_states.csv`

### Data Model Definition
#### Conceptual Data Model
For storing the data in database I proposed the following Star Schema.
##### Dimension Tables
```
dim_us_airports
    id
    type
    name
    elevation_ft
    municipality
    gps_code
    iata_code
    local_code
    latitude
    longitude
    state_code

dim_demographics_general
    city
    state_code
    median_age
    male_population
    female_population
    total_population
    number_of_veterans
    foreign_born
    average_household_size

dim_demographics_race
    city
    state_code
    race
    count
    
dim_us_temperatures
    dt
    avg_temp
    city
 
dim_countries
    country_code
    country_name

dim_us_ports
    municipality
    port_code
    state_code
    
dim_us_states
    state_code
    state_name
```

##### Fact Table
```
fact_immigrations
    id
    coc
    cor
    port_code
    age
    visa_issued_in
    occup
    biryear
    gender
    airline
    admnum
    fltno
    visatype
    arrival_dt
    departure_dt
    added_to_i94
    allowed_until
    arrival_mode
    visit_purpose
```

#### Mapping Out Data Pipelines
1. Create tables by executing `create_tables.py`.
2. Pre-process the raw source with `use_spark_to_create_fact.ipynb` and `use_pandas_to_create_dimensions.ipynb`
3. Insert data to database tables with the `etl.py`

### Complete Project Write Up
#### Choice of tools and technologies
I used the `pandas` framework the preprocess small tables. On the huge immigration table I preferred to use `spark` as a distrubuted computation framework.

My choice of database was the postgreSQL because it can handle more lazy queries than for example a noSQL Cassandra.

### Future stages
#### If the data was increased by 100x.
Use Spark to process the data (not just the main table) efficiently in a distributed way e.g. with EMR. In case we recognize that we need a write-heavy operation, I would suggest using a Cassandra database instead of PostgreSQL.

#### If the pipelines were run on a daily basis by 7am.
Use Airflow and create a DAG that performs the logic of the described pipeline. If executing the DAG fails, I recommend to automatically send emails to the engineering team using Airflow's builtin feature, so they can fix potential issues soon.

#### If the database needed to be accessed by 100+ people.
Use RedShift to have the data stored in a way that it can efficiently be accessed by many people. 
