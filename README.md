# Dockless Open Data

This guide will help you convert [MDS](https://github.com/OpenMobilityFoundation/mobility-data-specification) trip data to anonymized open data, useful for city governments.

## Points to Consider

- Cities need to be transparent with the kinds of data cities and private companies collect on residents.
- Data sharing is required by open records laws, local policy, state law, and federal law, with exceptions for personally identifiable information, trade secrets of companies, and sensitive data.
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
  `Fuzzed` tinyint(4) DEFAULT '0',
  `StartLat` float DEFAULT NULL,
  `StartLon` float DEFAULT NULL,
  `EndLat` float DEFAULT NULL,
  `EndLon` float DEFAULT NULL,
  PRIMARY KEY (`TripID`),
  KEY `idx_DocklessOpenData_StartLat` (`StartLat`),
  KEY `idx_DocklessOpenData_StartLon` (`StartLon`),
  KEY `idx_DocklessOpenData_EndLat` (`EndLat`),
  KEY `idx_DocklessOpenData_EndLon` (`EndLon`),
  KEY `idx_DocklessOpenData_Fuzzed` (`Fuzzed`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

Note the use of *varchars*, because not all company MDS feeds have reliable/complete data in the right format.  

The last 5 columns are used just for anonymizing the data later, in step 4 below.

**Note no trip line/polyline data is being stored, and no provider information**

When inserting from MDS into *DocklessOpenData*, use the following SQL for formatting values:

```
TripID = insert(insert(insert(insert(md5(sha2(source.OriginalTripID, '256')),9,1,'-'),14,1,'-'),19,1,'-'),24,1,'-') 
-- creates a new trip UUID from the orginal. This is a one-way function based on the source Trip UUID.  
-- There may be a better way to do this, but we wanted to not just generate a new random UUID 
-- and instead wanted it to be reproducable based on source data.

-- round start and end locations to 3 decimal places: about 2 city blocks, 
-- depending on your location on earth and city size
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

### 4 Anonymize start and end points with few trips

If there are not many trips between a starting location even after binning to 3 decimal places in step 2, then we anonymize further to protect privacy of individual riders.  This common practice is called "k-anonymity".

In our case, we look for O/D pairs of less than 5.  That is, where there are less than 5 trips made between any combination of 2 aggregated start and end trip areas across the city.  If there are, then we randomly move those points in a larger radius from the original location.  The radius here is about 0.3km in a random direction, which is a generalization method. 

In the final data there is no way to know which trips have been anonymized in this way, and which trips are only aggreggated to the block level without further anonymization.

To do this with only SQL, we use the column called 'Fuzzed' which tracks what O/D pairs need to be fuzzed, and which ones have then been fuzzed with a stored procedure.  There are also 4 columns for lat/lon start/end coordinates, that are the original values before fuzzing. 


```
-- 1 clear fuzz list 
update mobility.DocklessOpenData set Fuzzed = 0;

-- 2 reset initial/fill new lat/lons into fuzzed latitude/longitude columns
Update DocklessOpenData o
Set 
    o.StartLatitude = o.StartLat, o.StartLongitude = o.StartLon, o.EndLatitude = o.EndLat, o.EndLongitude = o.EndLon;
    
-- 3 update Fuzzed for OD pairs of 4 or less. 

update mobility.DocklessOpenData d set d.Fuzzed = 1 where d.TripID in 
(SELECT * FROM (
SELECT o.TripID as TripID
FROM mobility.DocklessOpenData o
group by o.StartLat, o.StartLon, o.EndLat, o.EndLon
having count(o.TripID) <= 4
 order by count(o.TripID) desc 
 ) tblTmp)
;
```

For the final step, we need to run a stored procedure.  You can create that procedure one time with this code:

```
DELIMITER $$
CREATE DEFINER=`root`@`localhost` PROCEDURE `FuzzOpenData`()
BEGIN

DECLARE randA FLOAT DEFAULT 0;
DECLARE randB FLOAT DEFAULT 0;

DECLARE n INT DEFAULT 0;
DECLARE i INT DEFAULT 0;

SELECT COUNT(*) FROM DocklessOpenData where Fuzzed = 1 INTO n;

SET i=0;
WHILE i<n DO

	SET @randA = rand();
	SET @randB = rand();

	update DocklessOpenData 

	set 
		StartLatitude = truncate(@randB * .004 * cos( 2*pi() * @randA / @randB ) + StartLat, 3),
		StartLongitude = truncate(@randB * .005 * sin( 2*pi() * @randA / @randB ) + StartLon, 3),
		EndLatitude = truncate(@randB * .004 * cos( 2*pi() * @randA / @randB ) + EndLat, 3),
		EndLongitude = truncate(@randB * .005 * sin( 2*pi() * @randA / @randB ) + EndLon, 3),
		
		Fuzzed = 2
 
	where Fuzzed = 1
	limit 1;
    
    SET i = i + 1;
    
END WHILE;

END$$
DELIMITER ;

```

Once this procedure is created, you can just run the procedure with this SQL.


```
-- 4 update to be fuzzed values with random 3rd digit in a circle
call `FuzzOpenData`();
```

## Final format of Open Data

Export and post in CSV format **just the following fields** from the open data table. Note we are not including the last 5 fields that were used in the k-anonymity step 4 above.

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

See a sample CSV file of this data in this repo: [DocklessOpenData-Sample-Aug2019-Louisville.csv](https://github.com/louisvillemetro-innovation/dockless-open-data/blob/master/DocklessOpenData-Sample-Aug2019-Louisville.csv)

## Cities with Dockless Trip Open Data

1. [Louisville, KY](https://data.louisvilleky.gov/dataset/dockless-vehicles) (3 decimal places lat/lon, 15 min time increments, outliers cleaned, k-anonymity of <5 O/D pairs fuzzed more)
1. [Austin, TX](https://data.austintexas.gov/Transportation-and-Mobility/Shared-Micromobility-Vehicle-Trips/7d8e-dm7r) (aligned to census tracts, outliers cleaned)
1. [Minneapolis, MN](http://opendata.minneapolismn.gov/datasets/motorized-foot-scooter-trips-2018) (aligned to line segments, outliers cleaned)

## References

- [Harvard's Civic Analytics Network](https://datasmart.ash.harvard.edu/news/article/civic-analytics-network-dockless-mobility-open-letter)
- [NACTO's guidance](https://nacto.org/wp-content/uploads/2018/07/NACTO-Shared-Active-Transportation-Guidelines.pdf)
