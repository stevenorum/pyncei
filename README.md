pyncei
======

This module provides tools to request data from the [Climate Data Online
webservices](http://www.ncdc.noaa.gov/cdo-web/webservices/v2#gettingStarted
provided by NOAA's National Centers for Environmental information (formerly
the National Center for Climate Data). Install with:

```
pip install pyncei
```

Getting started
---------------

To use the NCEI webservices, you'll need a token. The token is a 32-character
string; users can request one [here]([https://www.ncdc.noaa.gov/cdo-web/token).
Pass the token to pyncei.NCEIReader() to get started:

```python
from pyncei.NCEIReader import NCEIReader

token = 'AnExampleTokenFromTheNCEIWebsite'
ncei = NCEIReader(token)
```

NCEIReader includes functions corresponding to each of the
[endpoints](http://www.ncdc.noaa.gov/cdo-web/webservices/v2#gettingStarted)
described on the CDO website. Query parameters specified by CDO can be
passed as arguments:

```python
ncei.get_stations(location='FIPS:11')
ncei.get_data(dataset='GHCND',
              station=['COOP:010957'],
              datatype=['TMIN','TMAX'],
              startdate='2015-03-01',
              enddate='2016-03-01')
```

The table below provides some information about the different endpoints.
More information about query parameters for each endpoint is available at the
NCO website.

| NCO Endpoint         | NCO Query Parameter | Argument         |
| :------------------- | :------------------ | :--------------- |
| [datasets]           | datasetid           | dataset          |
| [datacategories]     | datacategoryid      | datacategory     |
| [datatypes]          | datatypeid          | datatype         |
| [locationcategories] | locationcategoryid  | locationcategory |
| [locations]          | locationid          | location         |
| [stations]           | stationid           | station          |
| [data]               | --                  | --               |

[datasets]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#datasets
[datacategories]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#datacategories
[datatypes]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#datatypes
[locationcategories]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#locationcategories
[locations]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#locations
[stations]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#stations
[data]: http://www.ncdc.noaa.gov/cdo-web/webservices/v2#data

Note that id fields used by CDO have been renamed here. For example, datasetid
has been renamed dataset, and locationid has been renamed location. Unlike
CDO, which accepts only ids, NCEIReader will accept either ids or name strings.
If names are used, NCEIReader attempts to map the name strings to valid id
using NCEIReader.map_term(), called manually here:

```python
ncei.map_term('District of Columbia', 'locations')
('FIPS:11', True)
```

When the mapping function fails to find an exact match, it throws an exception
containing a list of similar values that can be used to refine the original
query. You can also search the available terms for each endpoint using
NCEIReader.find_in_endpoint() and NCEI.find_all():

```python
ncei.find_in_endpoint('District of Columbia', 'locations')
['FIPS:11 => District of Columbia',
 'FIPS:11001 => District of Columbia County, DC']

ncei.find_all('temperature')
[('datacategories', 'ANNTEMP', 'Annual Temperature'),
 ('datacategories', 'AUTEMP', 'Autumn Temperature'),...]
```

You can search by city, state, zip code, data type, etc. If the search term is
None, NCEIReader.find_in_endpoint() will list ALL available ids for the given
endpoint.

The NCEI.find_all() function searches across all endpoints for a given term
and can be useful in locating a specific dataset or data type if you have
no idea what's available or where to look.

The mapping functions uses a set of .csv files included with the package. These
files can be updated using the NCEIReader.refresh_lookup() function:

```python
ncei.refresh_lookups()
```

Queries are cached for one day by default. Users can change this behavior
using the cache parameter when initializing an NCEIReader object. This
parameter specifies the number of seconds pages should persist in the cache;
a value of zero disables the cache entirely.

Example: Return data from a station
-----------------------------------

```python
import csv
from datetime import date

from pyncei.NCEIReader import NCEIReader


# Initialize NCEIReader object using your token string
ncei = NCEIReader('AnExampleTokenFromTheNCEIWebsite')
ncei.debug = True  # this flag produces verbose output

# Set the parameters you're looking for. You can use ncei.find_all() or
# ncei_find_in_endpoint() to search the available parameters if you don't
# know what to use.
mindate = '1966-01-01'  # either yyyy-mm-dd or a datetime object
maxdate = '2015-12-31'
datatypes = ['TMIN', 'TMAX']
dataset = 'GHCND'

# You can manually verify parameters if you're so inclined
for datatype in datatypes:
    ncei.map_name(datatype, 'datatypes')

# Get all DC stations operating between mindate and maxdate. The date
# parameters in station queries are a little odd. According to the docs,
# queries will return stations with data from on/before the enddate and
# on/after the startdate. If both parameters are included, the result set
# seems to include all stations that EITHER have data from on/before the
# startdate OR have data on/after the enddate.
stations = ncei.get_stations(location='District of Columbia',
                             dataset=dataset,
                             datatype=datatypes,
                             enddate=mindate)

# Filter out stations no longer operating using maxdate
stations = [station for station in stations
            if station['maxdate'] >= maxdate]

# Find the station with the best data coverage in the result set
stations.sort(key=lambda s:s['datacoverage'], reverse=True)
station = stations[0]
minyear = int(station['mindate'][:4])

# Get temperature data for the the lifetime of the station. Note that for the
# data endpoint, you can't request more than one year's worth of data at a
# time.
year = date.today().year - 1
results = []
while year >= minyear:
    results.extend(ncei.get_data(dataset=dataset,
                                 station=station['id'],
                                 datatype=datatypes,
                                 startdate=date(year, 1, 1),
                                 enddate=date(year, 12, 31)))
    year -= 1

fn = station['id'].replace(':', '') + '.csv'
with open(fn, 'wb') as f:
    writer = csv.writer(f, delimiter=',', quotechar='"')
    keys = ('date', 'datatype', 'value')
    writer.writerow(keys)
    for row in results:
        row['date'] = row['date'].split('T')[0].replace('-', '')
        writer.writerow([row[key] for key in keys])

```
