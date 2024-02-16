# Real-time analytics with KQL databases** **and Eventstreams

## Add a new KQL database and ingest data

Navigate to the "Fabric for Power BI Professionals" workspace and click the "New" from the ribbon. If you are using the Real-Time Analytics persona, select "KQL Database", otherwise select "More options" and select the "KQL Database" tile.

Name the database "MyKusto".

Once it is created, add some sample data. Click on the "Sample Data" tile. and select "Automotive operations analytics". This is a truncated database of New York taxi operations over a two year period, and the largest provided sample data set. This will take a few moments to load.

Open the "Automotive" table to view the schema.

Click the "Explore your data" button in the upper right hand corner. A window will open with some sample queries.

Replace "YOUR_TABLE_HERE" with "Automotive" for the first two queries.

Highlight rows 8 and 9, and click the "Run" button. You will see a random selection of 100 records from the table.

Highlight rows 12 and 13 and click the "Run" button. You should see that there are over 2 million records in the table.

## Connect to the database with the ADX client

The data explorer is handy, but if you're working extensively with KQL, you will likely want to use a purpose built client like Azure Data Studio or Visual Studio Code or the Azure Data Explorer client.

Close the "Explore your data" window, and select the "MyKusto" database from the Explorer pane on the left.

In the "Database details" section, select "Copy URI" to the right of "Query URI".

Open a new browser tab and paste the URI into the address bar. In this browser based editor, you can work with multiple queries in different tabs.

Add the following query that retrieves a number of metric by day and borough:

```PlainText
Automotive
| summarize Fares = sum(fare_amount), Passengers = sum(passenger_count), distance = sum(trip_distance) by Day = bin(pickup_datetime,1d), pickup_boroname
```

Click on the "Run" button to run the query, and note the time required to run in the results window. It should be a little over 1 second.

## Build a materialized view

While running a summarized query against 3 million rows of data is good, more complex queries with elements like joins, etc, and larger amounts of data can present a challenge. If these query patterns are common, it is a good idea to build a materialized view.

While still in Azure Data Explorer, add a new tab and add in the following code:

```PlainText
.create async materialized-view with(backfill = true) FareSummary on table Automotive { 
Automotive
| summarize Fares = sum(fare_amount), Passengers = sum(passenger_count), distance = sum(trip_distance) by Day = bin(pickup_datetime,1d), pickup_boroname
}
```

The async directive tells Fabric to process this command in the background, which is necessary to use the backfill=true option. Backfill tells Fabric to include all current data in the materialized view - without it, the view. will only contain new data. Click on the "Run" button in the ribbon.

Wait a few seconds and then refresh the database in the left pane. You will see a new node appear named "Materialized Views". Open the node and open the "FareSummary" Materialized view to see the schema.

Open a new tab, and simply enter the name of the view, FareSummary. Click on the "Run" button. It should run in about half the time that the original query ran. Also note the number of rows returned.

## Build a Power BI report

Close the Azure Data Explorer tab and return to the MyKusto database. Click on the "Refresh" button in the ribbon.

Open the "Tables" node in the Explorer pane, and select Automotive.

From the ribbon, click on "Build Power BI report".

Close the filter pane, and add bar chart to the canvas.

Add "pickup_boroname" to the Y-axis, and "total_amount to the X-axis

Select File-Save and name the report NYCCabs. Select the "Fabric for Power BI Professionals" workspace and click "Continue".

Navigate back to the "Fabric for Power BI Professionals" workspace. Note that there is both a report and a semantic model created. KQL data is not stored natively in OneLake (more on that below), so the semantic model is a Direct Query (not Direct Lake) connection to the KQL data.

We need to allow model editing in a browser. Click on "Workspace settings" from the ribbon (this may be under an ellipsis).

Open the "Power BI" blade and select "General". check the bottom checkbox to allow model editing.

Navigate back to the workspace and click on the "NYCCabs" semantic model.

Click the "Open data model" button in the ribbon.

From the ribbon, click the "New measure" button.

Enter the following in the editor:

```PlainText
Trips = COUNTROWS('Kusto Query Result')
```

Navigate back to the workspace, and click on the NYCCabs report

Click "Edit" in the ribbon. Add a new card visual to the report, and add the new "Trips" measure to its Fields.

Save the report and navigate back to the workspace.

## Materialize KQL data in the lakehouse

As mentioned above, KQL does not store its data in OneLake natively. However, it can be mirrored into OneLake in a read only manner for integration with other systems.

Open the "MyKusto" KQL database. Click the pencil icon beside "OneLake availability".

Set the value to "Active" and click "Done".

If the value still shows as inactive, refresh the screen.

Select the "Automotive" database. If "OneLake availability shows as "Inactive, activate it. 

Navigate back to the workspace, and open the "My_Lakehouse" lakehouse.

From the ribbon, click on "Get data", and select "New shortcut".

Select the OneLake tile, and then select the "MyKusto" KQL database.** **Click "Next".

Check the box next to the "Automotive" table. Click "Next".

Click "Create", and then "Close".

From the lakehouse, select "Automotive". Click on the ellipsis to the right of "Automotive" and select "View files". These are the delta-parquet files behind the lakehouse table.

## Create an event stream and output to KQL and Lakehouse

Navigate to the "Fabric for Power BI Professionals" workspace and click the "New" from the ribbon. If you are using the Real-Time Analytics persona, select "Eventstream", otherwise select "More options" and select the "Eventstream" tile.

Name your eventstream "BikesFeed".

Click on "New source" and select "Sample data".

Name the source "Rental_Bikes" and select "Bicycles (Reflex compatible)" from the dropdown menu. Click the "Add" button. 

Click on the "BikesFeed" tile in the middle and observe the Data preview.

Click the "Data insights" tab to ensure that data is flowing.

Click on "New destination" and select "KQL Database".

Select "Event processing before ingestion", and the following options. When done, click "Add and configure".

Destination name: BikesSummary

Workspace: Fabric for Power BI Professionals

KQL Database: MyKusto

Input data format: Json

Select the "Create New" link under "Destination table". Name the new table "BikesSummary".

Click the "Open event processor" button.

Select the line between the two boxes and delete it.

From the ribbon, press the "Operations" button, and select "Group by".

Connect the "BikesFeed" source to the Group by action, and that action to the "KQL Database 1" destination.

Click on the "Group by 1" action. Use the Average aggregation for the "No_Bikes" field, and create another Average for the No_Empty_Docks" field. 

Scroll down and select the "Neighbourhood" field to group the aggregations. Leave the time window as "Tumbling", and set the duration to 1 minute. Click the "Done" button in the "Group by" pane, and then "Done" in the editor window.

Finally, click "Add" in the "KQL Database" pane. The destination table will be created, and data will begin flowing to it

Click the "+" button to the right of the "BikesFeed" node. Select "Lakehouse" and fill in the following values:

Destination name: BikesLive

Workspace: Fabric for Power BI Professionals

Lakehouse: My_Lakehouse

Input data format: Json

Select **the "**Create New" link under "Delta table". Name the new table "BikesLive".

We will not process these events, so click on the "Add" button.

## Build a Power BI report from the lakehouse

Navigate back to the "Fabric for Power BI Professionals" workspace, and open the "MyLakehouse" lakehouse. Click the "BikesLive" table to see a preview of the data.

Change from "Lakehouse" to "SQL analytics endpoint" with the button in the upper right of the ribbon. Select the "BikesLive" table again.

Click the "New measure" button in the ribbon. Add the following measure:

```PlainText
Latest (UTC) = MAX(BikesLive[EventProcessedUtcTime])
```

Select the "Model" tab at the bottom of the Explorer pane. Find the "BikesLive" table and select the "Latest (UTC)" measure.

Change the value for "Format" to "Custom" and add the following Custom format in the box below:

```PlainText
yyyy-MM-dd yyyy-MM-dd hh:mm:ss
```

Click the "New report" button on the ribbon.

Open the "BikesLive" table in the "Data" pane on the right. 

Add a new Card visual to the report canvas, and add the "Latest (UTC) measure to it.

Add a table to the canvas, and add the following fields: EventProcessedUTCTime, Neighbourhood, No_Bikes, No_Empty_Docks

Change the Aggregation for No_Bikes and No_Empty_Docks to Average by using the dropdowns in the Visualizations pane.

Refresh the report from the upper right corner of the ribbon. Note that the latest readings have changed.

Save the report, and name it "Bikes report in real time".