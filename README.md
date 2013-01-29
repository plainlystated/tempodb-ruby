# TempoDB Ruby API Client

The TempoDB Ruby API Client makes calls to the [TempoDB API](http://tempo-db.com/api/).  The module is available as a gem.

``
gem install tempodb
``

[![Build Status](https://travis-ci.org/tempodb/tempodb-ruby.png?branch=master)](https://travis-ci.org/tempodb/tempodb-ruby)

# Classes

## Client(key, secret, *host="api.tempo-db.com"*, *port=443*, *secure=true*)
Stores the session information for authenticating and accessing TempoDB. Your api key and secret is required. The Client
also allows you to specify the hostname, port, and protocol (http or https). This is used if you are on a private cluster.
The default hostname and port should work for the standard cluster.

All access to data is made through a client instance.
### Members
* key - api key (string)
* secret - api secret (string)
* hostname - hostname for the cluster (string)
* port - port for the cluster (int)
* secure - protocol to use (true=https, false=http)

## DataPoint(ts, value)
Represents one timestamp/value pair.
### Members
* ts - timestamp (Time)
* value - the datapoint's value (double, long, boolean)

## Series(id, key, *name=""*, *attributes={}*, *tags=[]*)
Represents metadata associated with the series. Each series has a globally unique id that is generated by the system
and a user defined key. The key must be unique among all of your series. Each series may have a set of tags and attributes
that can be used to filter series during bulk reads. Attributes are key/value pairs. Both the key and attribute must be
strings. Tags are keys with no values. Tags must also be strings.
### Members
* id - unique series id (string)
* key - user defined key (string)
* name - human readable name for the series (string)
* attributes - key/value pairs providing metadata for the series (Hash - keys and values are strings)
* tags - (Array of strings)

## DataSet(series, start, stop, *data=[]*, *summary=nil*)
Represents data from a time range of a series. This is essentially a list of DataPoints with some added metadata. This is the object
returned from a query. The DataSet contains series metadata, the start/end times for the queried range, a list of the DataPoints
and a statistics summary table. The Summary table contains statistics for the time range (sum, mean, min, max, count, etc.)
### Members
* series - series metadata (TempoDB::Series)
* start - start time for the queried range (Time)
* stop - end time for the queried range (Time)
* data - datapoints (Array of DataPoints)
* summary - a summary table of statistics for the queried range (TempoDB::Summary)

# Client API

## create_series(*key=nil*)

Creates and returns a series object, optionally with the given key.

### Parameters
* key - key for the series (string)

### Returns
The newly created Series object

### Example

The following example creates two series, one with a given key of "my-custom-key", one with a randomly generated key.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    series1 = client.create_series("my-custom-key")
    series2 = client.create_series()

## get_series(*options={}*)
Gets a list of series objects, optionally filtered by the provided parameters. Series can be filtered by id, key, tag and
attribute.

### Parameters
* ids - an array of ids to include (Array of strings)
* keys - an array of keys to include (Array of strings)
* tags - an array of tags to filter on. These tags are and'd together (Array of strings)
* attributes - a hash of key/value pairs to filter on. These attributes are and'd together. (Hash)

### Returns
An array of Series objects

### Example

The following example returns all series with tags "tag1" and "tag2" and attribute "attr1" equal to "value1".

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    tags = ["tag1", "tag2"]
    attributes = {
        :attr1 => "value1"
    }

    series_list = client.get_series(:tags => tags, :attributes => attributes)

## update_series(series)
Updates a series. The series id is taken from the passed-in series object. Currently, only tags and attributes can be
modified. The easiest way to use this method is through a read-modify-write cycle.
### Parameters
* series - the new series (Series)

### Returns
The updated Series

### Example

The following example reads the list of series with key *test1* (should only be one) and replaces the tags with *tag3*.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    keys = ["test1"]
    series_list = client.get_series(:tags => tags, :attributes => attributes)

    if !series_list.empty?
        series = series_list[0]
        series.tags = ["tag3"]
        client.update_series(series)
    end

## read(start, stop, *options={}*)

Gets an array of DataSets for the specified start/stop times.

### Parameters
* start - start time for the query (Time)
* stop - end time for the query (Time)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* tz - the time zone of the output datapoints
* ids - an array of ids to include (Array of strings)
* keys - an array of keys to include (Array of strings)
* tags - an array of tags to filter on. These tags are and'd together (Array of strings)
* attributes - a hash of key/value pairs to filter on. These attributes are and'd together. (Hash)

The interval parameter allows you to specify a rollup period. For example, "1hour" will roll the data up on the hour using the provided function. The function parameter specifies the folding function
to use while rolling the data up. A rollup is selected automatically if no interval or function is given. The auto rollup interval
is calculated by the total time range (end - start) as follows:

* range <= 2 days - raw data is returned
* range <= 30 days - data is rolled up on the hour
* else - data is rolled up by the day

Rollup intervals are specified by a number and a time period. For example, 1day or 5min. Supported time periods:

* min
* hour
* day
* month
* year

Supported rollup functions:

* sum
* max
* min
* avg or mean
* stddev (standard deviation)
* count

You can also retrieve raw data by specifying "raw" as the interval. The series to query can be filtered using the
remaining parameters.

### Returns
An array of DataSets

### Example

The following example returns an array of series from 2012-01-01 to 2012-01-02 for the series with key "my-custom-key",
with the maximum value for each hour.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    start = Time.utc(2012, 1, 1)
    stop = Time.utc(2012, 1, 2)
    keys = ["my-custom-key"]

    data = client.read(start, stop, :keys => keys, :interval => "1hour", :function => "max")

## read_id(series_id, start, stop, *options={}*)

Gets a DataSet by series id. The id, start, and stop times are required. The same rollup rules apply as for the multi series
read (above).

### Parameters
* series_id - id for the series to read from (string)
* start - start time for the query (Time)
* stop - end time for the query (Time)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* tz - the time zone of the output datapoints

### Returns

A DataSet

### Example

The following example reads data for the series with id "38268c3b231f1266a392931e15e99231" from 2012-01-01 to 2012-02-01 and
returns a minimum datapoint per day.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    start = Time.utc(2012, 1, 1)
    stop = Time.utc(2012, 1, 2)

    data = client.read_id("38268c3b231f1266a392931e15e99231", start, stop, :interval => "1day", :function => "min")

## read_key(series_key, start, stop, *options={}*)

Gets a DataSet by series key. The key, start, and stop times are required. The same rollup rules apply as for the multi series
read (above).

### Parameters
* series_key - key for the series to read from (string)
* start - start time for the query (Time)
* stop - end time for the query (Time)
* interval - the rollup interval (string)
* function - the rollup folding function (string)
* tz - the time zone of the output datapoints

### Returns

A DataSet

### Example

The following example reads data for the series with key "my-custom-key" from 2012-01-01 to 2012-02-01 and
returns a minimum datapoint per day.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    start = Time.utc(2012, 1, 1)
    stop = Time.utc(2012, 1, 2)

    data = client.read_key("my-custom-key", start, stop, :interval => "1day", :function => "min")

## write_id(series_id, data)

Writes datapoints to the specified series. The series id and an array of DataPoints are required.

### Parameters
* series_id - id for the series to write to (string)
* data - the data to write (Array of DataPoints)

### Returns
Nothing

### Example

The following example write three datapoints to the series with id "38268c3b231f1266a392931e15e99231".

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    data = [
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 0, 0), 12.34),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 1, 0), 1.874),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 2, 0), 21.52)
    ]

    client.write_id("38268c3b231f1266a392931e15e99231", data)

## write_key(series_key, data)
Writes datapoints to the specified series. The series key and an array of DataPoints are required. Note: a series will be created
if the provided key does not exist.

### Parameters
* series_key - key for the series to write to (string)
* data - the data to write (Array of DataPoints)

### Returns
Nothing

### Example

The following example write three datapoints to the series with key "my-custom-key".

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    data = [
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 0, 0), 12.34),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 1, 0), 1.874),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 2, 0), 21.52)
    ]

    client.write_key("my-custom-key", data)

## write_bulk(ts, data)
Writes values to multiple series for a particular timestamp. This function takes a timestamp and a parameter called data, which is an
array of hashes containing the series id or key and the value. For example:

    data = [
        { :id => '01868c1a2aaf416ea6cd8edd65e7a4b8', :v => 4.164 },
        { :id =>'38268c3b231f1266a392931e15e99231', :v => 73.13 },
        { :key => 'your-custom-key', :v => 55.423 },
        { :key => 'foo', :v => 324.991 }
    ]

### Parameters
* ts - a timestamp of the data points (Time)
* data - an array of dictionaries containing an id or key and the value

### Returns
Nothing

### Example

The following example writes datapoints to four separate series at the same timestamp.

    require 'tempodb'

    client = TempoDB::Client("api-key", "api-secret")

    ts = Time.utc(2012, 1, 8, 1, 21)
    data = [
        { :id => '01868c1a2aaf416ea6cd8edd65e7a4b8', :v => 4.164 },
        { :id =>'38268c3b231f1266a392931e15e99231', :v => 73.13 },
        { :key => 'your-custom-key', :v => 55.423 },
        { :key => 'foo', :v => 324.991 }
    ]

    client.write_bulk(ts, data)

## increment_id(series_id, data)
Increments the value of the specified series at the given timestamp. The value of the datapoint is the amount to increment. This is similar to a write. However the value is incremented by the datapoint value
instead of overwritten. Values are incremented atomically, so this is useful for counting events. The series id and an array of DataPoints are required.

### Parameters
* series_id - id for the series to increment (string)
* data - the data to write (Array of DataPoints)

### Returns
Nothing

### Example

The following increments three datapoints of the series with id "38268c3b231f1266a392931e15e99231".

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    data = [
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 0, 0), 1),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 1, 0), 2),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 2, 0), 1)
    ]

    client.increment_id("38268c3b231f1266a392931e15e99231", data)

## increment_key(series_key, data)
Increments the value of the specified series at the given timestamp. The value of the datapoint is the amount to increment. This is similar to a write. However the value is incremented by the datapoint value
instead of overwritten. Values are incremented atomically, so this is useful for counting events. The series key and an array of DataPoints are required. Note: a series will be created
if the provided key does not exist.

### Parameters
* series_key - key for the series to increment (string)
* data - the data to write (Array of DataPoints)

### Returns
Nothing

### Example

The following example increments three datapoints of the series with key "my-custom-key".

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    data = [
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 0, 0), 1),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 1, 0), 2),
        TempoDB::DataPoint.new(Time.utc(2012, 1, 1, 1, 2, 0), 4)
    ]

    client.increment_key("my-custom-key", data)

## increment_bulk(ts, data)
Increments values of multiple series for a particular timestamp. This function takes a timestamp and a parameter called data, which is an
array of hashes containing the series id or key and the value. For example:

    data = [
        { :id => '01868c1a2aaf416ea6cd8edd65e7a4b8', :v => 4 },
        { :id =>'38268c3b231f1266a392931e15e99231', :v => 2 },
        { :key => 'your-custom-key', :v => 1 },
        { :key => 'foo', :v => 1 }
    ]

### Parameters
* ts - a timestamp of the data points (Time)
* data - an array of dictionaries containing an id or key and the value

### Returns
Nothing

### Example

The following example increments datapoints of four separate series at the same timestamp.

    require 'tempodb'

    client = TempoDB::Client("api-key", "api-secret")

    ts = Time.utc(2012, 1, 8, 1, 21)
    data = [
        { :id => '01868c1a2aaf416ea6cd8edd65e7a4b8', :v => 4 },
        { :id =>'38268c3b231f1266a392931e15e99231', :v => 2 },
        { :key => 'your-custom-key', :v => 1 },
        { :key => 'foo', :v => 1 }
    ]

    client.increment_bulk(ts, data)

## delete_id(series_id, start, stop, *options={}*)

Deletes a range of datapoints from a series specified by id. The id, start, and stop times are required. Delete a single datapoint
by setting the start and stop times to the same time.

### Parameters
* series_id - id for the series to read from (string)
* start - start time for the query (Time)
* stop - end time for the query (Time)
* options - (unused for now)

### Returns

Nothing

### Example

The following example deletes data for the series with id "38268c3b231f1266a392931e15e99231" from 2012-01-01 to 2012-02-01.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    start = Time.utc(2012, 1, 1)
    stop = Time.utc(2012, 1, 2)

    client.delete_id("38268c3b231f1266a392931e15e99231", start, stop)

## delete_key(series_key, start, stop, *options={}*)

Deletes a range of datapoints from a series referenced by series key. The key, start, and stop times are required. Delete a single datapoint
by setting the start and stop times to the same time.

### Parameters
* series_key - key for the series to read from (string)
* start - start time for the query (Time)
* stop - end time for the query (Time)
* options - (unused for now)

### Returns

Nothing

### Example

The following example deletes data for the series with key "my-custom-key" from 2012-01-01 to 2012-02-01.

    require 'tempodb'

    client = TempoDB::Client.new("api-key", "api-secret")

    start = Time.utc(2012, 1, 1)
    stop = Time.utc(2012, 1, 2)

    client.delete_key("my-custom-key", start, stop)
