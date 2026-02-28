## `JSON_TABLE` re-visited

In my blog post 

https://www.ibm.com/docs/en/i/7.6.0?topic=functions-json-table

https://goessner.net/articles/JsonPath/

## Consuming a web service with SQL on IBM i

Consuming web services on IBM i was never easier – but in fact, it’s nothing new, 
the DB2 SQL functions are available at least since Version 7.2.

With Version 7.4 TR5 and Version 7.3 TR11 IBM has released new HTTP functions, which 
now reside in QSYS2 and which are much faster than the HTTP functions in SYSTOOLS. 

For our example, I will use a service from openweathermap.org, to retrieve a 5 day 
weather forecast for a given ZIP and country code. To get startet, you have to 
register and get your own API key – since we won’t gather large amounts of data, 
the free tier is OK.

Let’s start with the code:

```sql
select *
from json_table(
  qsys2.http_get('http://api.openweathermap.org/data/2.5/forecast?zip={...ZIP...},{...COUNTRY...}&amp;units=metric&amp;appid={...API-KEY...}')
  ,
  'lax $' columns (
    nested '$.list' columns (
      description varchar(255) path 'lax $.weather[0].description',
      cloud_percent numeric(3, 0) path 'lax $.clouds.all',
      temp numeric(5, 2) path 'lax $.main.temp',
      temp_feel numeric(5, 2) path 'lax $.main.feels_like',
      temp_min numeric(5, 2) path 'lax $.main.temp_min',
      temp_max numeric(5, 2) path 'lax $.main.temp_max',
      pressure numeric(5, 0) path 'lax $.main.pressure',
      humidity numeric(3, 0) path 'lax $.main.humidity',
      visibility numeric(7, 0) path 'lax $.visibility',
      wind_speed numeric(3, 1) path 'lax $.wind.speed',
      wind_degr numeric(3, 0) path 'lax $.wind.deg',
      rain1h numeric(5, 0) path 'lax $.rain.1h',
      rain3h numeric(5, 0) path 'lax $.rain.3h',
      snow1h numeric(5, 0) path 'lax $.snow.1h',
      snow3h numeric(5, 0) path 'lax $.snow.3h',
      probability numeric(3, 2) path 'lax $.pop',
      reported bigint path 'lax $.dt'
    ),
    sunrise bigint path 'lax $.city.sunrise',
    sunset bigint path 'lax $.city.sunset',
    utc_offset bigint path 'lax $.city.timezone',
    lat numeric(6, 2) path 'lax $.city.coord.lat',
    lon numeric(6, 2) path 'lax $.city.coord.lon',
    country char(2) path 'lax $.city.country',
    city varchar(255) path 'lax $.city.name'
  )
  error on error
);
```

And here is the resulting table, that the SQL statement i
