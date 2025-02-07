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

And here is the resulting table, that the SQL statement is returning:

![SQL result table](/assets/img/2021-08-19-result-table.jpeg)

This is a simple select – in line 2 the function JSON_TABLE is invoked – an it 
receives two „parameters“ – the first is the JSON document or expression, and 
the second „parameter“ is a JSON path expression and column definition. 
In fact the whole function call to JSON_TABLE ends in line 34 with the closing 
parentheses, just before the semicolon.

Now to the first parameter of JSON_TABLE – the JSON document or expression. 
It’s received from a call to the QSYS2.HTTP_GET function. QSYS2.HTTP_GET has also 
two parameters – the first is the URL and the second are the HTTP headers. 
We only need the URL.

The URL of the webservice basically looks like this:

http://api.openweathermap.org/data/2.5/forecast?zip={ZIP},{Country}&appid={API-Key}

The „zip“ parameter could look like „zip=91257,DE“ which is the city of Pegnitz 
in Germany. You could also add „units=metric“ to receive the data in metric units. 
And last but not least you need „appid=<…your.API.key…>“ which you get after you 
registered at the website.

Of course you can use „http://…“ and „https://…“ – QSYS2.HTTP_GET can handle both 
protocols. But I had some problems using https on my test server pub400.com – 
so I changed the code to http. When using https and the server uses a self signed 
certificate (and not an „officially“ signed one) you may have to add the servers 
certificate to the local key store.

But now to the second parameter of JSON_TABLE – the JSON path expression and column 
definitions. Basically the first JSON path expression matches the „root“ of the 
document.

Here a shortened example of that JSON document, that we receive from the webservice.
The „$“ sign matches the current or selected structure in the JSON document – in 
this case the first structure level of curly braces { … }. The term „lax“ means 
„don’t take everything so seriously“ or more technically, that several types of 
structural error in the document are automatically resolved (Link).

The following „column“ keyword means „take the JSON key-value pairs of this level, 
and create column definitions. As with every column definition we need a name and 
a SQL data type. The special is the „path“ keyword with the following JSON path 
expression. This expression is always relative to the path expression before the 
column keyword.

The nested keyword means, that we have a nested structure with keys – often these 
are arrays with multiple occurrences of structures like the „list“ in our example. 
It works a bit like a JOIN – the occurrences of the nested structure are „joined“ 
with the values on the same level – you can see this in our example.

The „description“ column is selected from an array of values – as we only need the 
first entry, we can select it with an „[0]“ expression. But we always start with 
„lax $ …“ – that means, that we start in this case on the level of „list“.

The rest of the column definitions is pretty much self explaining. But just before 
the end we read „error on error“ – these keywords are following the column 
definitions, and it means, that the function will fail, if there is any error. 
If we do not specify this, we would receive a <NULL> value if an error occurs.

I hope this little example helps to get you started with the JSON_TABLE function.
