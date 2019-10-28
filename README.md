# Dockless Open Data

This guide will help you convert [MDS](https://github.com/CityOfLosAngeles/mobility-data-specification) trip data to anonymized open data, useful for city governments.

## Points to Consider

- Cities need to be transparent with the kinds of data cities and private companies collect on residents.
- Data sharing is required by local policy, state law, and federal law, with exceptions for personally identifiable information, trade secrets of companies, and sensitive data.
- Cities need to balance transparency requirements and open records laws with privacy best practices.

## Data Processing

These are the steps to processing from MDS to open data.

### 1 Obtain secure access to an MDS feed

Using your city's [Dockless Vehicle Policy](https://data.louisvilleky.gov/dataset/dockless-vehicles/resource/541f050d-b868-428e-9601-c48a04eba17c) data sharing and enforcement requirements, obtain authentication which each operator's MDS feed for your city.

### 2 Ingest a subset of MDS data into a database table

Ingestion method from MDS is left as an exercise for the reader.  Code for this should be added here at a later date.

This is the table structure for the *DocklessOpenData* open data table:

```
CREATE TABLE `DocklessOpenData` (
  `TripID` varchar(50) NOT NULL,
  `StartDate` varchar(20) DEFAULT NULL,
  `StartTime` varchar(20) DEFAULT NULL,
  `EndDate` varchar(20) DEFAULT NULL,
  `EndTime` varchar(20) DEFAULT NULL,
  `TripDuration` float DEFAULT NULL,
  `TripDistance` float DEFAULT NULL,
  `StartLatitude` float DEFAULT NULL,
  `StartLongitude` float DEFAULT NULL,
  `EndLatitude` float DEFAULT NULL,
  `EndLongitude` float DEFAULT NULL,
  `DayOfWeek` varchar(45) DEFAULT NULL,
  `HourNum` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`TripID`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

Note the use of *varchars*, because not all company MDS feeds have reliable/complete data.  

**Note no trip line/polyline data is being stored.**

When inserting from MDS into *DocklessOpenData*, use the following SQL as a guide:

```
TripID = insert(insert(insert(insert(md5(sha2(source.OriginalTripID, '256')),9,1,'-'),14,1,'-'),19,1,'-'),24,1,'-') 
-- creates a new trip UUID from the orginal. This is a one-way function based on the source Trip UUID.  There may be a better way to do this, but we wanted to not just generate a new random UUID and instead wanted it to be reproducable based on source data.

-- round start and end locations to 3 decimal places: about 2 city blocks, depending on your location on earth and city size
StartLatitude = ROUND(source.OriginalStartLatitude, 3)
StartLongitude = ROUND(source.OriginalStartLongitude, 3)
EndLatitude = ROUND(source.OriginalEndLatitude, 3)
EndLongitude = ROUND(source.OriginalEndLongitude, 3)

StartDate = STR_TO_DATE(source.OriginalStartDateTime, '%Y-%m-%d')

EndDate = STR_TO_DATE(source.OriginalEndDateTime,   '%Y-%m-%d')

StartTime = LEFT(SEC_TO_TIME(FLOOR((TIME_TO_SEC(source.OriginalStartDateTime) + 450) / 900) * 900), 5) 
-- bins to 15 minute increments

EndTime = LEFT(SEC_TO_TIME(FLOOR((TIME_TO_SEC(source.OriginalEndDateTime)   + 450) / 900) * 900), 5) 
-- bins to 15 minute increments

TripDuration = Round( ( UNIX_TIMESTAMP(source.OriginalEndDateTime) - UNIX_TIMESTAMP(source.OriginalStartDateTime) ) /60 )
-- rounded to nearest minute
```

### 3 Run scripts to convert and clean open data

This removes distance outliers and populates day of week and hour of day fields.

```
Update
	DocklessOpenData o
Set 
	o.DayOfWeek = dayofweek(o.StartDate),
	o.HourNum = left(o.StartTime, 2),
	o.TripDistance = -1 where o.TripDistance < 0,
	o.TripDistance = 100 where o.TripDistance > 100;
```

## Final format of Open Data

Export and post in CSV format the fields from the open data table.

- **TripID** - a unique ID created by city
- **StartDate** - in YYYY-MM-DD format
- **StartTime** - rounded to the nearest 15 minutes in HH:MM format
- **EndDate** - in YYYY-MM-DD format
- **EndTime** - rounded to the nearest 15 minutes in HH:MM format
- **TripDuration** - duration of the trip minutes
- **TripDistance** - distance of trip in miles - company provided data
- **StartLatitude** - rounded to nearest 3 decimal places (about 
- **StartLongitude** - rounded to nearest 3 decimal places
- **EndLatitude** - rounded to nearest 3 decimal places
- **EndLongitude** - rounded to nearest 3 decimal places
- **DayOfWeek** - 1-7 based on date, 1 = Sunday through 7 = Saturday, useful for analysis
- **HourNum** - the hour part of the time from 0-24 of the StartTime, useful for analysis

## Cities with Dockless Trip Open Data

1. [Louisville, KY](https://data.louisvilleky.gov/dataset/dockless-vehicles) (3 decimal places lat/lon, 15 min time increments, outliers cleaned)
1. [Austin, TX](https://data.austintexas.gov/Transportation-and-Mobility/Shared-Micromobility-Vehicle-Trips/7d8e-dm7r) (aligned to census tracts, outliers cleaned)
1. [Minneapolic, MN](http://opendata.minneapolismn.gov/datasets/motorized-foot-scooter-trips-2018) (aligned to line segments, outliers cleaned)

## References

- [Harvard's Civic Analytics Network](https://datasmart.ash.harvard.edu/news/article/civic-analytics-network-dockless-mobility-open-letter)
- [NACTO's guidance](https://nacto.org/wp-content/uploads/2018/07/NACTO-Shared-Active-Transportation-Guidelines.pdf)
