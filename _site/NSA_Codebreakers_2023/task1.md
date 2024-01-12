# TASK 1
- [Where to start](#where-to-start)
- [Analyzing the Database](#analyzing-the-database)

### Task Description
> The US Coast Guard (USCG) recorded an unregistered signal over 30 nautical miles away from the continental US (OCONUS). NSA is contacted to see if we have a record of a similar signal in our databases. The Coast guard provides a copy of the signal data. Your job is to provide the USCG any colluding records from NSA databases that could hint at the object’s location. Per instructions from the USCG, to raise the likelihood of discovering the source of the signal, they need multiple corresponding entries from the NSA database whose geographic coordinates are within 1/100th of a degree. Additionally, record timestamps should be no greater than 10 minutes apart.

### Files Given
USCG.log (A file holding metadata for the signal)
database.db (a sql database with a list of signals from the NSAs collection)

### Where to Start
The easiest place to look is the log. Running `file USCG.log` shows that the file is a simple JSON file. Once we run `cat USCG.log` or just open it up in a file editor we see this:
```json
{
    "coordinates": [
        {
            "latitude": "28.52255",
            "longitude": "-91.63558"
        }
    ],
    "timestamp": "02/08/2023, 04:12:40"
}
```
From this file, we can see a couple ways to filter the database. Between trying to filter by dates or floats, I will go for floats any day. The next step is to find data in the database to filter

### Analyzing the Database
I'm working on linux, so I downloaded the [sqlite package](https://sqlite.org/2023/sqlite-tools-linux-x64-3440200.zip) to get the binary. After running the binary, a sqlite shell will pop up. From here there are two type of commands that can be run: sql querries and dot commands. SQL queries work like normal, but dot commands are sqlite-specific commands to interface with data sources. To get the database info into the sqlite instace, run `.open database.db` if you're in the folder where you downloaded database.db. The next step from here is to figure out what the datbase looks like. To do this, there are two main commands to look at. `.tables` will print out all of the tables holding data. From here the collumn names can be found by running `.schema [tableName]`

From this enumeration, we get four tables (audio_object, event, location, timestamp), but the most relavent ones are location and timestamp. We could search by either the location or date, but I personally think that location is easier. To scan through location, I used this SQL query: `SELECT * FROM location WHERE ABS(latitude-28.52) < 0.03;`

Breaking down this query, we start with `SELECT *` which selects all the available columns from the table. Next, `FROM location` means that we're pulling from the location table. Lastly, `WHERE ABS(latitude-28.52) < 0.03;` means that only entries whose latitude value fits \|latitude - 28.52\| < 0.03 will be shown. this is the best way to do floats because we can get how far away the latitude value is from the value we want and then filter to make sure the distance is less than 0.03. I chose 0.03 because the task specifies a variation of 0.01, but I wanted to make sure rounding wouldn't cause a problem. This query then shows two locations:

874\|28.5293\|-91.63676\|0 m
969\|28.53213\|-91.64326\|0 m

The first collumn is the location ID. The second collumn is the latitude. The third collumn is the longitude, The last collumn is the elevation. Both of these fit, but I also decided to double check the results by doing a similar query with the longitude: `SELECT * FROM location WHERE ABS(longitude+91.63) < 0.03;`. That resulted in the same location, so I knew I was set. The last thing to get is the event ID and the timestamps. To get the event ID, run `SELECT * FROM event WHERE location_id=874;` and `SELECT * FROM event WHERE location_id=969;`. From these, we can see that the event ID and the timestamp ID are the same as the location ID. To tripple check, I looked at the timestamp with `sqlite> SELECT * FROM timestamp WHERE id=969 OR id=874;` and saw that they were in the 10 minute range

**ANSWER**:874 969