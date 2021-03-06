# The Scripting

**Please Note: Names of certain tables, schemas and companies have been changed for proprietary and privacy reasons. Any aggregations showns in this documentation are kept to 10 units or more for privacy protection.**


In order to get the base Charliecard data, we have to connect to the Postgres server where it is stored. This involves SQL scripting, which can be achieved by utilising functions from two R-packages. In addition, the script makes use of predefined variables, which can be changed according to the user, month of query, and the list of companies for which data is required.

## Packages:

We require the following packages to be installed/loaded for the script to run:
```{r}
library(DBI) #for the SQL queries
library(RPostgres) #for thhe Postgres Server connection
library(DescTools) #for statistical/data wrangling tools
library(tidyverse) #for statistical/data wrangling tools
library(gmapsdistance) #for getting the distances/times through the use of Google's distance API
library(lubridate) #for working with time-related data
library(DT) #for making interactive charts in an html
library(rmarkdown)
```
 
## Defining Ad-hoc Functions:

We created two ad-hoc functions which are used in our script. These are not part of any packages, and thus have to be separately defined.

**Shift Function:**
It is used to shift particular rows 'up' by 'n' count. A negative value will push them down. We use this function while getting a list of distinct journeys from the 'legs' table (which is explained in detail below). Essentially, for every journey, we need information on the first leg's origin, and the last leg's destination. We use the _shift_ function for 'shifting' the cells in the last leg's destination columns.
```{r}
shift <- function(x, n){
  c(x[-(seq(n))], rep(NA, n))
}
```

**Next Weekday Function:**
The GmapsDistance function works only with future date-times. As such, we need to specify an arrival/departure date in the future. This function takes a particular input date, and day-of-the-week code, and gives the next date of that particular day.

| Code (wday) | Day |
|-----:|:----|
| 1 | Sunday |
| 2 | Monday |
| 3 | Tuesday |
| 4 | Wednesday |
| 5 | Thursday |
| 6 | Friday |
| 7 | Saturday |

```{r}
nextweekday <- function(date, wday) { #date = date you want to get next weekday for, wday = day of the week you want the date for.
  date <- as.Date(date)
  diff <- wday - wday(date)
  if( diff <= 0)
    diff <- diff + 7
  return(date + diff)
}
```
 
## Relevant Variables:

**API Key:** We need the API key as provided by Google for its Distance calculation function. You can get one by checking [Google's Cloud Computing Services](<https://cloud.google.com/maps-platform/?apis=routes>).

```{r}
set.api.key = ("API_KEY_FROM_GOOGLE")
```

**Enquiry Year and month:** The year for which the Perq data is being queried for: In this example we are querying data for September 2018.
```{r}
yyyy<- paste0("2018")
mm <- paste0("09")
```

**Company List:** The IDs of the companies for which data is being queried:
```{r}
companylist <- paste0("1110,3574,11078,2456,1136,1783,1762,1801,1858,1840,1968,3480,1025,3171,1904,1709,3199")
```

**Number of Working Days:** The number of working days for the month of query:
```{r}
w_days <- 19
```

**Date and Time of function call:** We use the date for the upcoming Thursday, which is considered a typical workday, and an arrival time of 9:00am. For the month of September in 2018, around 68% of all trips happened during peak hours, and as such, we can apply the same time-stamp for off-peak trips.
```{r}
funct_calldate <- nextweekday(Sys.Date(), 5)
funct_calltime <- "09:00:00"
```

**Variables required for the SQL server Connection:**

1. SSL Certificates:
These are your personal SSL certificate and key used while connecting to the Postgres Server. You will specify the path to the files stored on your personal work computer.

```{r}
cl_cert <- paste0("PATH_TO_YOUR_POSTGRES_CREDENTIALS_/adeodhar.crt")
cl_key <- paste0("PATH_TO_YOUR_POSTGRES_CREDENTIALS_/adeodhar.key")
```

2. Connection to the Postgres server:

**dbConnect:**

This function is used to define a connection to the Postgres server. It takes the following arguments:

a.	**drv:** Driver used to define which kind of SQL server to connect to. In the case of the MBTA's research server, we use ‘Postgres()’

b.	**user:** It is your username, which is used on the Postgres server. For example, John Smith’s username is ‘jsmith’. However, we use a ‘askForPassword’ function. A window will popup, wherein you will enter your username.

c.	**dbname:** It is the database name

d.	**host:** The url of the host server

e.	**port:** 5432

f.	**sslcert:** The individual user’s personal SSL Certificate used for connecting to the Research Server. It will be in the format: ‘jsmith.crt’. The path is absolute, and will change according where it is stored on the user’s system.

g.	**sslkey:** The individual user’s personal SSL Key used for connecting to the Research Server. It will be in the format: ‘jsmith.key’. The path is absolute, and will change according where it is stored on the user’s system.

```{r}
con <- dbConnect(
Postgres(), 
user = askForPassword("Database User Name"),
dbname = 'dbname',
host = 'host.server.org',
port = 5432,
sslcert = cl_cert,
sslkey = cl_key
)

```

We thus define a variable **con** as the connection to the research server. We will use it in the following function when running SQL queries.

We have now thus defined our variables which will be used throughout the script. We do not need to define any other variables, and just run the script for the desired output.

 
## SQL Queries:

We use our last three variables, that is the SSL Certificate, SSL Key, and the Postgres server connection to run our SQL queries from within R:

### Access to Schema/Tables:

There are a few schemas and tables we need access to which are on the Research Server:

| Schema | Table |
|-----:|:----|:----|
| **perq** | emissions_bymode |
|  | companynamelist |
|  | cr_reference |
| **card_info** | cardorders |
|  | companyorders |
| **odx** | odx_yyyymm |
|  | rides_yyyymm |
| **gtfs** | stops |

### Emissions By Mode:

This query gets the table in which emissions (in pounds of CO<sub>2</sub>) per passenger mile by vehicle mode are stored.
```{r}
emissions_bymode <- dbGetQuery(con, "SELECT * 
           FROM perq.emissions_bymode")
```

### Corporate Names List:

This query gets the names of companies with their company IDs. Most proceeding queries have data only by the company ID, and we join this table to get the name of the company for easy identification.
```{r}
corporate_list <- dbGetQuery(con, paste0("SELECT idcompany, companyname 
FROM perq.companynamelist
WHERE idcompany IN (",companylist,")
GROUP BY idcompany, companyname
ORDER BY companyname"))
corporate_list$idcompany <- as.numeric(corporate_list$idcompany)
```

### Commuter Rail Pass Data:

Commuter rail pass data is worked separately due to the nature of the data we have. The only definitive data we have on users of Commuter Rail is the number of passes issued by Zone to each company per month. As such, calculation for Commuter Rail are done separately from that of Subway and Bus.

**Passes Issued by Company**

```{r}
cr_bycompany <- dbGetQuery(con, paste0("SELECT 
b.IdCompany
,cr.productname AS zone
,cr.count AS count
FROM (SELECT idcompany FROM card_info.cardorders
		WHERE ordermonth =",yyyy,mm, " AND
		(idcompany IN (",companylist,"))
		GROUP BY idcompany) b
LEFT JOIN card_info.companyorders cr
	ON cr.idcompany = b.idcompany
	WHERE month_0 = '",yyyy,"-",mm,"-01' AND
category_0 = 'Commuter Rail'"))
cr_bycompany <- merge(cr_bycompany, corporate_list, by = "idcompany")
cr_bycompany$zone <- gsub("Commuter Rail Zone ", "", cr_bycompany$zone)
cr_bycompany$zone <- gsub("A Pass", "A", cr_bycompany$zone)
```

**Commuter Rail Distance (in miles) and Time Taken (in minutes)**

This data is by zone. The distances and times are the averages of all the stations within that zone to their corresponding station in Downtown (North or South Station, depending on what line they are)

```{r}
cr_reference <- dbGetQuery(con, "SELECT * 
           FROM perq.cr_reference")
```

### ODX Perq CharlieCard Data:

The major part of this script is based on the origins and destinations inferred from taps by the ODX Inference Model. Due to the nature of the inference model which infers destinations based on previous journeys, not all taps register a destinations, for example, for atypical one-off journeys. However, all taps are indicative of undertaken journeys. We therefore get a table of all recorded taps, whether they have inferred destinations or not. We use this count to scale up from the list of taps for which there are inferred destinations. This takes into assumption that the distance and time taken for these trips is normally distributed over the network.

All card level data is completely anonymized, and no individual can be identified with any of the card level information available at this aggregation level.

**All Registered Taps**

```{r}
taps <- dbGetQuery(con, paste0("SELECT * FROM(
	SELECT b.SerialNumber
		,b.IdCompany
		,o.*
		FROM (SELECT * FROM card_info.cardorders
		WHERE ordermonth =",yyyy,mm, " AND
		(idcompany IN (",companylist,"))) b
	LEFT JOIN encrypt.anon_cardnum h ON b.SerialNumber = h.ticketserial
	LEFT JOIN odx.odx_",yyyy,mm," o ON h.hashed = o.card
	WHERE faretransactionkey > 0
	--WHERE origin IS NOT NULL
) r"))

taps <- merge(taps, corporate_list, by = "idcompany")
taps$uqtripid <- paste(taps$idcompany, 
                             taps$serialnumber, 
                             day(taps$tap_time), 
                             taps$journey_sequence, 
                             sep = "_")

taps$mode <- if_else((as.numeric(taps$vehicle) < 3000 | as.numeric(taps$vehicle) > 3999), "bus",
                     if_else((as.numeric(taps$vehicle) > 2999 & as.numeric(taps$vehicle) < 4000), "subway",
                             "subway"))

taps$mode[is.na(taps$mode)] <- "subway"
```

**Journey Legs**

The only way we can calculate distances and times for OD pairs is if we have both origins and destinations for a given OD pair. For the month of September 2018, the average inferred destination rate hovered around the 65% mark for all dates. This means, ODX could determine the destination for around 65% of taps. We therefore query to get a table of these inferred destinations. There are two types of mode or vehicle changes possible:

1. Behind the farebox (eg Red to Orange at Downtown Crossing)

These are represented in the 'rides' column. These represent a vehicle change behind the faregates, and are inferred by the ODX model.

2. Crossing the farebox (eg Changing modes or buses)

These are represented in the 'stages' column. These indicate a tap at a farebox or faregate.

Here too, all card level data is completely anonymized, and no individual can be identified with any of the card level information available at this aggregation level.

```{r}
legs <- dbGetQuery(con, paste0("SELECT 
concat(r.IdCompany,'_',r.SerialNumber,'_',extract(day from on_time),'_', journey_sequence) AS uqtripid
,r.IdCompany
,r.SerialNumber
,journey_sequence
,stage
,ride
,concat(stage,ride)::int AS journey_leg
,concat(on_stop, '_', off_stop) AS uq_od_id
,on_stop AS o_stop_code
,ga.stop_name AS o_stop_name
,ga.stop_lat AS o_lat
,ga.stop_lon AS o_lon
,on_time AS origin_time
,off_stop AS d_stop_code
,gb.stop_name AS d_stop_name
,gb.stop_lat AS d_lat
,gb.stop_lon AS d_lon
,vehicle
FROM(
SELECT 
b.SerialNumber
,b.IdProduct
,b.IdCompany
,o.*
FROM (SELECT * FROM card_info.cardorders
WHERE ordermonth = ",yyyy,mm," AND
(idcompany IN (",companylist,"))) b
LEFT JOIN encrypt.anon_cardnum h ON b.SerialNumber = h.ticketserial
LEFT JOIN odx.rides_",yyyy,mm," o ON h.hashed = o.card) r 
LEFT JOIN gtfs.stops ga ON r.on_stop = ga.stop_code
LEFT JOIN gtfs.stops gb ON r.off_stop = gb.stop_code
WHERE
ga.valid_start_date = '2018-09-06'
AND gb.valid_end_date = '2018-09-06'
ORDER BY uqtripid, journey_leg"))
legs <- merge(legs, corporate_list, by = "idcompany")

legs$mode <- if_else(str_detect(string = legs$vehicle, pattern = "^HR"), "subway",
                     if_else((as.numeric(legs$vehicle) > 2999 & as.numeric(legs$vehicle) < 4000), "subway",
                             "bus"))

# creating a unique OD pair ID - concatenation of the Origin latlong and Destination latlong. 
# Thsi is because the gmapsdistance creates a unique trip id based on these variables. It becomes easier to join tables later on:
legs$uqlatlongid <- paste(legs$o_lat, 
                          legs$o_lon, 
                          legs$d_lat, 
                          legs$d_lon,
                          sep = "+")
```

## Intermediate Tables and Functions:

Based on the data pulled with these SQL queries, we now work with it to ultimately generate our final output table.

### Scaling Factor:

As mentioned in the section above, not all registered taps have an inferred destination. Thus with the assumption that these uninferred destinations are normally distributed across the system, we develop a scaling factor tailored to each company by mode.

The scaling factor is based on taps. This is the most definitive knowledge we have which is not inferred. We summarise the counts of taps by mode for every company.

We divide the total taps by the taps with an inferred destination. This will give us the scaling factor with which we will scale up our distance, time and emissions.

We also create a scalefactor table for cars. As one journey may contain more than one mode, we take the mean of the scalefactors for each company by mode. 

```{r}
taps$uq_tap_id <- paste(taps$idcompany, 
                        taps$serialnumber, 
                        day(taps$tap_time), 
                        taps$journey_sequence,
                        taps$stage,
                        sep = "_")

taps$count <- rep(1,nrow(taps))

total_taps <- taps %>% 
              dplyr::group_by(uq_tap_id) %>% #groups by the unique tap ID.
              slice(c(1)) %>% #takes only one of the multiple rows per tap. Since the vast majority of taps are by mode
              group_by(idcompany, companyname, mode) %>% #selects the idcompany, companyname and mode columns, and groups the data acc to them
              summarise(total_tapcount = sum(count)) # summarises the counts by these three variables.



legs$uq_tap_id <- paste(legs$idcompany, 
                        legs$serialnumber, 
                        day(legs$origin_time), 
                        legs$journey_sequence,
                        legs$stage,
                        sep = "_")

legs$count <- rep(1,nrow(legs))

odx_taps <- legs %>% 
            dplyr::group_by(uq_tap_id) %>%  #groups by the unique tap ID.
            slice(c(1)) %>% #takes only one of the multiple rows per tap. Since the vast majority of taps are by mode
            group_by(idcompany, companyname, mode) %>% #selects the idcompany, companyname and mode columns, and groups the data acc to them
            summarise(odx_tapcount = sum(count)) # summarises the counts by these three variables.

scalefactor_tbl <- merge(odx_taps,
                         total_taps,
                         by = c("idcompany",
                                "companyname",
                                "mode")
                         )

scalefactor_tbl$scalefactor <- scalefactor_tbl$total_tapcount/scalefactor_tbl$odx_tapcount

scalefactor_cars <- scalefactor_tbl %>% 
                    group_by(idcompany,companyname) %>% 
                    summarise(scalefactor = mean(scalefactor))
```

#### Scale Factor Table for Transit:
```{r}
scalefactor_tbl
```


#### Scale Factor Table for Cars:
```{r}
scalefactor_cars
```

### Distinct Journeys:

We already have a table where each row is indicative of a 'leg' in a journey. Each leg is a combination of the 'stage' and 'ride' columns. However, we are also calculating the equivalent distances, times, and emissions for cars for each of those journeys. Cars do not care for changes in legs of journeys, and only need to consider the absolute origin and destination for each journey. As such, we will need to create a table where each row corresponds to a journey. We can get this by filtering the 'legs' table by single leg journeys, and multi-leg journeys. We then format the multi-leg journeys so that the origin of the first leg, and the destination of the last leg are in the same row.

```{r}
# filter out only multi-leg journeys, and get the first and last rows for each unique trip ID ie uqtripid. 
# Thus we get only two rows per journey:
multi_leg <- legs %>%
             dplyr::group_by(uqtripid) %>% # grouping all rows by their unique trip ID, ie uqtripid
             slice(c(1, n())) %>% # taking only the first and last leg
             distinct() %>% # making sure they are distinct
             add_count() %>% # adding a count of number of rows with same uqtripid
             filter(n>1) # keeping only those rows with a count of more than 1

# shift all destination information 'up' by one row. Thus the first row for each uniquetripID will have the absolute ends for each multitrip journey

multi_leg <- cbind(data.frame(multi_leg[,c(1:13)],stringsAsFactors = F),
                   data.frame(data.frame(sapply(multi_leg[,c(14:17)],shift,n=1)) %>% 
                                mutate("d_stop_code" = as.numeric(as.character(d_stop_code)),
                                       "d_stop_name" = as.character(d_stop_name),
                                       "d_lat" = as.numeric(as.character(d_lat)),
                                       "d_lon" = as.numeric(as.character(d_lon)))), 
                   multi_leg[,c(18:21)],stringsAsFactors = F)

# remove the redundant second rows for every uqtripid.
multi_leg <- multi_leg %>%
             dplyr::group_by(uqtripid) %>% # grouping all rows by their unique trip ID, ie uqtripid
             slice(1) %>% # keeping only the first row, which now contains the first origin and last destination.
             distinct() %>% 
             mutate(d_lat = as.numeric(d_lat), d_lon = as.numeric(d_lon)) # making the lat-long columns numeric. For some reason, they turn into factors.

# filter out all single-leg journeys, so we can bind the multi-leg journeys to create desired table.
single_leg <- legs %>%
              dplyr::group_by(uqtripid) %>%  # grouping all rows by their unique trip ID, ie uqtripid
              slice(c(1, n())) %>% # taking only the first and last rows of each unique trip ID.
              distinct() %>% 
              add_count() %>% # adding a count of number of rows with the same uqtripid
              filter(n==1) # keeping only those rows with a count equal to 1

# binding the now distinct single rows per journey together. We also remove redundant columns such as mode, vehicle type
distinctjourneys <- bind_rows(multi_leg, single_leg) # merging these two table. As their column names and types are the same, we can do an bind_rows
distinctjourneys <- distinctjourneys[,c(1:3,8:17,19,21)] # keeping only those rows which are useful for the gmapsdistance function.
```

We will have to update the unique OD pair ID column as well as the unique LAT-LONG pair column to reflect the changes in the now condensed multi-leg journeys.

```{r}
distinctjourneys$uq_od_id <- paste(distinctjourneys$o_stop_code, distinctjourneys$d_stop_code, sep = "_")
distinctjourneys$uqlatlongid <- paste(distinctjourneys$o_lat, 
                                      distinctjourneys$o_lon, 
                                      distinctjourneys$d_lat, 
                                      distinctjourneys$d_lon,
                                      sep = "+")
```

### Function: GmapsDistance

This is the main function with which we calculate the distances and times required to travel between our given OD pairs depending on what mode is specified. Here, we define and explain the variables and elements useful for this script. For a detailed documentation, check the [CRAN page](<https://cran.r-project.org/web/packages/gmapsdistance/gmapsdistance.pdf>)

The gmapsdistance function takes the following arguments:

**origin**

This is a string containing the origin. If a lat-long pair is supplied (as is in our case) the latitude and longitude must be a concatenation and separated by a "+" sign. For example:

```{r}
o_lat = 42.40577
o_lon = -71.14246

origin = "42.40577+-71.14246"
```

**destination**

Similarly for the destination, it is a string containing the destination. If a lat-long pair is supplied (as is in our case) the latitude and longitude must be a concatenation and separated by a "+" sign. For example:

```{r}
d_lat = 42.32316
d_lon = -71.16619

destination = "42.32316+-71.16619"
```

**combinations**

This describes how the pairing of OD pairs will happen while using the function. Since our OD pairs are on the same row, we select "pairwise". To calculate between all origins and destination in the dataframe, we can use "all". However, we do not require this.

```{r}
combinations = "pairwise"
```

**mode**

This describes what mode of travel is to be used while calculating the distance and time. This is similar to how you can choose in the [Google Maps](<https://www.google.com/maps/preview>) interface. Options available are: "bicycling", "walking", "transit" or "driving"

We use this function twice: once for driving, and then for transit.

```{r}
mode = "transit"

OR

mode = "driving"
```

**arr_date**

This is the date of arrival at destination (in the YYYY-MM-DD format, for example 20th July 2017 would be written as 2017-07-20). We provide the date defined in the beginning as `funct_calldate`

**arr_time**

This is the time of arrival at destination (in the HH:MM:SS format, for example 2:31:45 pm would be written as 14:31:45). We provide the time defined in the beginning as `funct_calltime`

**key**

This is the Google API key defined in the begining. It is called by using the `get.api.key()` function.

Before running our function, we have to create two tables: One containing all the unique OD pair combinations for transit, and the other created for driving. We must note the following:

1. The transit table will reflect the number of unique OD pairs by legs. Each OD pair is equivalent to either a leg of a journey, or the whole journey itself, if the journey was a single-leg journey.

2. The driving table will reflect the number of unique OD pairs by journeys. Each OD pair is equivalent to only a journey. There may be an overlap with the transit table for single-leg journeys.

**Transit Table**
```{r}
transittable <- aggregate(uqtripid~uq_od_id+o_stop_code+o_stop_name+o_lat+o_lon+d_stop_code+d_stop_name+d_lat+d_lon+uqlatlongid, 
                          length, 
                          data = legs)
```

**Driving Table**
```{r}
drivingtable <- aggregate(uqtripid~uq_od_id+o_stop_code+o_stop_name+o_lat+o_lon+d_stop_code+d_stop_name+d_lat+d_lon+uqlatlongid, 
                          length, 
                          data = distinctjourneys)
```

**Running the gmapsdistance function**

```{r}
tr_matrix <- gmapsdistance(
  origin = paste(transittable$o_lat, transittable$o_lon, sep = "+"),
  destination = paste(transittable$d_lat, transittable$d_lon, sep = "+"),
  combinations = "pairwise",
  mode = "transit",
  arr_date = as.character(funct_calldate),
  arr_time = as.character(funct_calltime),
  key = get.api.key()
)

dr_matrix <- gmapsdistance(
  origin = paste(drivingtable$o_lat, drivingtable$o_lon, sep = "+"),
  destination = paste(drivingtable$d_lat, drivingtable$d_lon, sep = "+"),
  combinations = "pairwise",
  mode = "driving",
  arr_date = as.character(funct_calldate),
  arr_time = as.character(funct_calltime),
  key = get.api.key()
)
```

The function gives distances in meters and duration in seconds. As such, we will convert them to miles and minutes respectively.

```{r}
tr_tbl <- data.frame(tr_matrix)
tr_tbl <- data.frame("minutes" = tr_tbl$Time.Time/60, # converting time to minutes from seconds
                     "miles" = tr_tbl$Distance.Distance*0.000621371, # converting distance to miles from meters
                     "uqlatlongid" = paste(tr_tbl$Time.or, tr_tbl$Time.de, sep = "+")) # we create a unique identifier tag of the o_latlong and d_latlong, on which we can merge later.

write.csv(tr_tbl, "files/tr_tbl.csv") # we save the output files to a system location.

dr_tbl <- data.frame(dr_matrix)
dr_tbl <- data.frame("minutes" = dr_tbl$Time.Time/60, # converting time to minutes from seconds
                     "miles" = dr_tbl$Distance.Distance*0.000621371, # converting distance to miles from meters
                     "uqlatlongid" = paste(dr_tbl$Time.or, dr_tbl$Time.de, sep = "+")) # we create a unique identifier tag of the o_latlong and d_latlong, on which we can merge later.

write.csv(dr_tbl, "files/dr_tbl.csv") # we save the output files to a system location.
```

The following tables are run using a sample of 2000 randomly picked OD pairs. As such, they will not reflect the true count, and should be treated as such.

**Transit Table**

We now merge the distances table with the legs table based on the `uqlatlongid`. This way, based on the corresponding mode of the OD pair, we get the leg's distance, and time.
```{r}
bus_subway <- merge(legs, tr_tbl[,2:4], by = "uqlatlongid")
```

We now add the emissions column to the bus_subway table. The emissions table will merge based on the mode, and multiply the distance with the emissions factor.

```{r}
bus_subway$trip_emissions <- ifelse(str_detect(bus_subway$mode, pattern = "subway"),
                                    bus_subway$miles*emissions_bymode$co2_subway, # if the trip is made by rapid transit
                                    bus_subway$miles*emissions_bymode$co2_bus) # if the trip is made by bus
bus_subway <- distinct(bus_subway)
```

We also write this bus_subway table to use for other analysis, for example in Tableau:
```{r}
write.csv(bus_subway, "files/bus_subway.csv")
```


And finally the output table for the number of trips by mode for every company along with the total minutes, miles, and emissions in pounds of $CO2$ totaled by month:

```{r}
bus_subway <- bus_subway %>% distinct() %>% drop_na() %>% 
  select(idcompany, companyname, mode, count, minutes, miles, trip_emissions) %>% 
  group_by(idcompany, companyname, mode) %>% 
  summarise(trip_count = sum(count), 
            minutes = sum(minutes),
            miles = sum(miles),
            trip_emissions = sum(trip_emissions))

bus_subway <- data.frame("idcompany" = bus_subway$idcompany,
                         "companyname" = bus_subway$companyname,
                         "mode" = bus_subway$mode,
                         "trip_count" = floor(bus_subway$trip_count*scalefactor_tbl$scalefactor),
                         "minutes" = bus_subway$minutes*scalefactor_tbl$scalefactor,
                         "miles" = bus_subway$miles*scalefactor_tbl$scalefactor,
                         "trip_emissions" = bus_subway$trip_emissions*scalefactor_tbl$scalefactor)


```

**Driving Table**

We now merge the distances table with the distinctjourneys table based on the `uqlatlongid`. This way, based on the corresponding mode of the OD pair, we get the journey's distance, and time.
```{r}
car <- merge(distinctjourneys, dr_tbl[,2:4], by = "uqlatlongid")
car$count <- rep(1,nrow(car))
```

We now add the emissions column to the car table. The emissions table will merge based on the mode, and multiply the distance with the emissions factor.

```{r}
car$mode <- rep("car", nrow(car))
car$trip_emissions <- car$miles*emissions_bymode$co2_car
car <- distinct(car)
```

We also write this car table to use for other analysis, for example in Tableau:
```{r}
write.csv(car, "files/car_dists.csv")
```


And finally the output table for equivalent car trips for subway and bus trips: 

```{r}
car <- car %>% distinct() %>% drop_na() %>% 
  select(idcompany, companyname, mode, count, minutes, miles, trip_emissions) %>%
  group_by(idcompany, companyname, mode) %>% 
  summarise(trip_count = sum(count), 
            minutes = sum(minutes),
            miles = sum(miles),
            trip_emissions = sum(trip_emissions))

car <- data.frame("idcompany" = car$idcompany,
                  "companyname" = car$companyname,
                  "mode" = car$mode,
                  "trip_count" = floor(car$trip_count*scalefactor_cars$scalefactor),
                  "minutes" = car$minutes*scalefactor_cars$scalefactor,
                  "miles" = car$miles*scalefactor_cars$scalefactor,
                  "trip_emissions" = car$trip_emissions*scalefactor_cars$scalefactor)
```

### Journey Emissions data for Commuter Rail

As the commuter rail data is stored separately, and is not calculated through ODX (As we do not currently employ AFC for commuter rail), we have to make assumptions based on pass counts and working days.

Therefore, we multiply all fields which will be enumerated by the number of working days in the given month and 2 (to account for to/fro trips)
```{r}
commrail <- cr_bycompany %>% merge(cr_reference, by = "zone")
```

We separate the commuter rail table by numbers for commuter rail and numbers for driving. This is so, we can later merge the driving tables for both commuter rail with that of the corresponding driving table for bus and subway.

```{r}
cr_tr <- data.frame("idcompany" = commrail$idcompany,
                    "companyname" = commrail$companyname,
                    "mode" = "commrail",
                    "trip_count" = commrail$count*w_days*2,
                    "minutes" = commrail$cr_min_tr*commrail$count*w_days*2,
                    "miles" = commrail$cr_miles_tr*commrail$count*w_days*2,
                    "trip_emissions" = commrail$cr_miles_tr*commrail$count*w_days*2*emissions_bymode$co2_commrail)

cr_tr <- cr_tr %>% distinct() %>% drop_na() %>% 
                    select(idcompany, companyname, mode, trip_count, minutes, miles, trip_emissions) %>%
                    group_by(idcompany, companyname, mode) %>% 
                    summarise(trip_count = sum(trip_count), 
                              minutes = sum(minutes),
                              miles = sum(miles),
                              trip_emissions = sum(trip_emissions))
```


```{r}
cr_dr <- data.frame("idcompany" = commrail$idcompany,
                    "companyname" = commrail$companyname,
                    "mode" = "car",
                    "trip_count" = commrail$count*w_days*2,
                    "minutes" = commrail$cr_min_dr*commrail$count*w_days*2,
                    "miles" = commrail$cr_miles_dr*commrail$count*w_days*2,
                    "trip_emissions" = commrail$cr_miles_dr*commrail$count*w_days*2*emissions_bymode$co2_car)

cr_dr <- cr_dr %>% distinct() %>% drop_na() %>% 
                    select(idcompany, companyname, mode, trip_count, minutes, miles, trip_emissions) %>%
                    group_by(idcompany, companyname, mode) %>% 
                    summarise(trip_count = sum(trip_count), 
                              minutes = sum(minutes),
                              miles = sum(miles),
                              trip_emissions = sum(trip_emissions))
```
```{r, echo=FALSE, layout="l-body-outset"}
paged_table(cr_dr)
```

Now that we have calculations for all four modes of transport ready, we merge the tables to get the final output:
```{r}
transit_tbl <- bind_rows(bus_subway, cr_tr)

cars_tbl <- data.frame("idcompany" = car$idcompany,
                       "companyname" = car$companyname,
                       "mode" = car$mode,
                       "trip_count" = car$trip_count+cr_dr$trip_count,
                       "minutes" = car$minutes+cr_dr$minutes,
                       "miles" = car$miles+cr_dr$miles,
                       "trip_emissions" = car$trip_emissions+cr_dr$trip_emissions)

company_emissions_tbl <- bind_rows(transit_tbl, cars_tbl)
```

# Final Output Table

The final output table contains information on the number of trips done by employees of a given company by mode type, and the time it took (in minutes), distance travelled (in miles) and pounds of CO<sub>2</sub> emitted. All figures are for a monthly basis, with this example for the month of September 2018.
