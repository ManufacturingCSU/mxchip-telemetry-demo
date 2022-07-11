# mxchip-telemetry-demo
The purpose of this repository is to be able to deploy the MX Chip demonstration with a cold, warm and hot path scenario.

### Setup ###

Please clone this repository to a local directory on your computer using the following URL: `https://github.com/ManufacturingCSU/mxchip-telemetry-demo`

In order to add all the components above, we will need to create a new resource group within Azure and add the following components:
* IoT Hub
* Azure Storage Account
* SQL Server
* SQL Database
* Service Bus
* 2 API Connections
* Logic App
* Stream Analytics

#### Creating the Resource Group ####

To get started, lets create the resource group using the commands below.  Using Powershell, run these commands:

```javascript
Connect-AzAccount

New-AzResourceGroup -Name <insert your resource group name> -Location <insert your location> -Tag @{Workload="MXChip Telemetry Demo"}
```

You can use the following code to get a listing of the Azure regions that you can use for the code above:
```javascript
az account list-locations -o table
```

#### Deploying the Azure Assets within the Resource Group (First Pass) ####

Next we need to deploy the following resources within the Resource Group:
* IoT Hub
* Azure Storage Account
* SQL Server
* SQL Database
* Azure Function
* Service Bus
* 2 API Connections


This will be done using the *00-azuredeploy_parameters.json* and *01-azuredeploy.json* ARM template files in the arm folder of this repository.  After you have cloned this directory, there are two steps to follow...
1. Open the *00-azuredeploy_paramaters.json* file and add the values needed for any section with  _insert value here_
    1. objectid - This is found under your user profile within Azure Active Directory setting within the Azure portal 
    2. tenantid - This is found under the Azure Active Directory setting within the Azure portal
2. Open powershell and navigate to the arm directory of this cloned repository and run the following command:
```javascript
$parameters = @{
    'Name' = 'MXChip Telemetry Deployment1'
    'ResourceGroupName' = '<insert your resource group name from above>'
    'TemplateFile'      = '01-azuredeploy.json'
    'TemplateParameterFile' = '00-azuredeploy_parameters.json'
    'Verbose' = $true
}

New-AzResourceGroupDeployment @parameters
```
Navigate to the [Azure Portal](https://pocrtal.azure.com) to view your resource group and the resources within it.

#### Add Telemetry Consumer Group to the IoT Hub ####

NOTE: This must be done before you run the additional azuredeploy.json script.

1. Go to your IoT HUb in the Azure portal and click on the Built-in endpoints blade
2. Add _telemetry_ as a new Consumer Group.
3. Click Save if needed

#### Update the API Connections ####

Even though we created the API Connections, we still 

#### Updating the SQL Server and IoT SQL Database ####

Modify the Firewall settings

1. Navigate to your SQL Server and under the Netowrks blade:
   a. Add your IP address
   b. Set _Allow Azure services and resources to access this server_ to Yes.
   c. Save your firewall settings and proceed to the next step. 
2. Navigate to the IoT SQL Database and click on the Query editor (preview) blade and run the following SQL script to create the Telemetry table. 

```sql
/****** Object:  Table [dbo].[Telemetry]    Script Date: 7/10/2022 3:33:35 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [dbo].[Telemetry](
	[MessageID] [bigint] IDENTITY(1,1) NOT NULL,
	[Device] [varchar](100) NOT NULL,
	[WindowEnd] [datetime] NOT NULL,
	[Celsius] [float] NULL,
	[Fahrenheit] [float] NULL,
	[Humidity] [float] NULL,
	[Pressure] [float] NULL,
	[AccelerometerX] [float] NULL,
	[AccelerometerY] [float] NULL,
	[AccelerometerZ] [float] NULL,
	[AnomalyScore] [float] NULL,
	[Isanomaly] [bigint] NULL,
	[ReasonCode] [varchar](100) NULL,
	[ReasonCodeDescription] [varchar](max) NULL,
 CONSTRAINT [PK_Telemetry_MessageID] PRIMARY KEY CLUSTERED 
(
	[MessageID] ASC
)WITH (STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
GO

```

#### Deploying the Azure Assets within the Resource Group (Second Pass) ####

After the steps above have been completed, we need to deploy the following resources within the Resource Group:
* Stream Analytics
* Logic App

This will be done using the *00-azuredeploy_parameters.json* and *02-azuredeploy.json* ARM template files in the arm folder of this repository.  After you have cloned this directory, perform the following...
1. Open powershell and navigate to the arm directory of this cloned repository and run the following command:
```javascript
$parameters = @{
    'Name' = 'MXChip Telemetry Deployment1'
    'ResourceGroupName' = '<insert your resource group name from above>'
    'TemplateFile'      = '02-azuredeploy.json'
    'TemplateParameterFile' = '00-azuredeploy_parameters.json'
    'Verbose' = $true
}

New-AzResourceGroupDeployment @parameters
```
### Post Deployment Steps ###

#### Verify/Update the Stream Analytics Jobs ####

1. From within the Portal open up the Stream Analytics Job and Test the connections below.  
    a. Input - iothub:  If this fails change the value under the _shared access policy name_ and then change it back to _iothubowner_ and click save.
    b. Output - servicebusqueue:  If this fails change the value under the _authentication mode_ and then change it back to _Connection string_ and click save.
2. Within the query section, add the code below and Save the query.  
    _Make sure to add your phone number under the Service Bus section in the script._  

```sql

--MX Chip Telemetry Data
WITH Telemetry 
AS 

(   
    SELECT    
        Stream.IoTHub.ConnectionDeviceId as Device
        ,0 as simulated    
        ,System.TimeStamp AS WindowEnd    
        ,AVG(Stream.humidity) as humidity    
        ,AVG(Stream.temp) as celsius    
        ,AVG(((Stream.temp*1.8)+32)) as fahrenheit    
        ,AVG(Stream.pressure) as pressure    
        ,AVG(Stream.magnetometerX) as magnetometerX    
        ,AVG(Stream.magnetometerY) as magnetometerY    
        ,AVG(Stream.magnetometerZ) as magnetometerZ    
        ,AVG(Stream.accelerometerX) as accelerometerX    
        ,AVG(Stream.accelerometerY) as accelerometerY    
        ,AVG(Stream.accelerometerZ) as accelerometerZ    
        ,AVG(Stream.gyroscopeX) as gyroscopeX    
        ,AVG(Stream.gyroscopeY) as gyroscopeY    
        ,AVG(Stream.gyroscopeZ) as gyroscopeZ
        ,CASE
            WHEN AVG(Stream.accelerometerZ) < 0 THEN 100
            ELSE 0
        END AS anomalyscore
	    ,CASE
            WHEN AVG(Stream.accelerometerZ) < 0 THEN 1
            ELSE 0
        END AS isanomaly
        
    FROM iothub Stream TIMESTAMP BY IoTHub.EnqueuedTime
    GROUP BY Stream.IoTHub.ConnectionDeviceId, TumblingWindow(second, 15)
)

-- Send summarized data to SQL
SELECT 
      Device
    , CAST(System.TimeStamp as datetime) AS WindowEnd
    , CAST(celsius as float) AS Celsius
	, CAST(fahrenheit as float) as Fahrenheit
	, CAST(humidity as float) as Humidity
	, CAST(pressure as float) as Pressure
    , CAST(accelerometerx as float) as AccelerometerX
	, CAST(accelerometery as float) as AccelerometerY
	, CAST(accelerometerz as float) as AccelerometerZ
	, CAST(anomalyscore as float) as anomalyscore
	, CAST(isanomaly as bigint) as isanomaly

INTO sql
FROM Telemetry

/*
-- Pass Anomalies into the Function 
SELECT     
      deviceid AS Device 
    , windowend AS Time
    ,'Accelerometer Z' AS ReadingType    
    , ROUND(AccelerometerZ, 1) as reading 
    , isanomaly AS abn
INTO function
FROM Telemetry
WHERE isanomaly = 1
*/

-- Send Alerts to Service Bus Message Queue for Sending a Text Alert
SELECT 
    Device as device
    ,'<insert your mobile phone number here>' as phone
    ,'Accelerometer Z' AS readingtype
    ,ROUND(AccelerometerZ, 1) as reading
	,windowend as time
	,isanomaly 

INTO servicebusqueue
FROM Telemetry
WHERE isanomaly = 1

```
3. Start the Stream Analytics Job 

#### Configure the MXChip ####

1.	Within the Azure Portal, go to the IoT HUb and click on the Devices Blade and click + Add New Device.   
2.	Name the device _AZ3166_ and click Save
3.	Next, we need to capture our Device Connection string.  Click on your device within the Devices listing to open its properties and copy the Primary Connection String.  Paste this into Notepad for later use.
4.	Plug in the MX Chip into your computer's USB port, it should show as a new drive.
5.  Using File Explorer, copy the _AZ3166-IoT-Central-2.1.2.bin_ file from the mxchip directory of this repo to the root of your device.
6.	When the MXChip reboots, you should see a Wifi SID on the display.  Disconnect your PC from your current Wifi and connect to that SID
7.  Once connected, go to http://192.168.0.1/start make sure to copy your <b>Connection string – primary key</b> from Notepad into the <b>Device connection string</b> field and select all checkboxes listed.<br>&nbsp;<br>
<img src="https://jq25qg.dm2302.livefilestore.com/y4mG_faJylzWG9bcN4fiO_DNQ0-FXxda3W9K3l1lxmp2uOzJ-drp2zUo5HPXJpriI9Lv3JBSv9btZjJDYz4KfyoUn97E_oTjugqa8qkTsMDi-T3YPiJHzddg8IB-GG0p5BNpUyEmsZKCdKJ72Ijx-w77BSBVJYXA0G_ctKrg2J30TvBuHq3CwBWvCyKUCdFQvi1UTfN8RGq5ANOlWjfHaEXiw?width=660&height=387&cropmode=none" width="660" height="387" /><br>&nbsp;<br>
Click the <b>Configure Device</b> button and click the reset button on the MX Chip.<br>&nbsp;<br>
NOTE:  If you do not get the Device connection string field in the window above, try doing a hard reset on the MX CHIP by holding down both the A & B buttons for a few seconds.

8.	Now you should see telemetry coming from your custom named AZ3166.  You can view this by downloading <a href="https://docs.microsoft.com/en-us/azure/iot-fundamentals/howto-use-iot-explorer" target="_blank">Azure IoT Explorer</a>.

### Verify the Setup ###

### Test the Setup ###
Now we are ready to test the data flow.  To do so, perform the following steps to generate a data flow from the *Azure Subscription Details-SAS* Power BI Report through Data Factory to the AzureSubscriptions SQL database that will be consumed by the *ACR Totals by Subscription* Power BI Report that we will configure in a subsequent section:
1. Run the *Azure Subscription Details-SAS* Power BI Report against the account you are tracking.  To do so, change the TPID to your account in the filter pane.
    1. Click on the *Monthly Export for ACR* tab and click on the elipses button in the upper right corner in the table and select *Exprt Data*.  Save your data in a CSV format locally.
2. Navigate to your storage account within the [Azure Portal](https://portal.azure.com) and open the *ACR* container.  We need to upload the file we just exported from the *Azure Subscription Details-SAS* Power BI Report.  We must create the *monthlyacr_new* folder under the acr container and upload your CSV file within that *monthlyacr_new* folder in order for the trigger within data factory to execute the pipelines listed above. 
3. After the file is copied up to the *monthlyacr_new* folder, go to the Monitor blade within Data Factory and you should see the pipeline triggered.  It typically takes about 6-10 minutes for both pipelines to execute.  The following is what the pipelines accomplish when finished:
    1. Run Monthly_ACR Data Flow - Populates the *MonthlyACR* table in the AzureSubscriptions database.  
    <B>NOTE:  This does not check for duplicates so I typically filter by month and add data for the previous month on the 10th of the current month.</B> 
    2. ACR Pipeline Stored Procedure - This executes the *spUpsertSubscriptions stored procedure* which populates the *Subscriptions* table.  It checks for new subscriptions in the MonthlyACR table and if one exists, it adds them to this table.
4. Once the pipelines are complete, you can query the AzureSubscriptions database with the following queries:
```javascript
    SELECT * FROM [dbo].[MonthlyACR] order by FiscalMonth DESC, ACR DESC
```
```javascript
    SELECT * FROM [dbo].[Subscriptions]
```

## ACR Totals by Subscription ##
Lastly we are going to configure the Power BI Report and PowerApp to point to the AzureSubscriptions SQL database.

This report and PowerApp allow you to supplement consumption data with the following fields:
* Customer Subsidiary
* Customer Initiative
* Customer Initiative Owner
* Microsoft Lead

### PowerApp Configuration ###
1. We will start by downloading the PowerApp configuration file from the internal Teams Site.  Click <a href="https://microsoft.sharepoint.com/:u:/t/ScottTestSite/ESd1W-6txCNJmB2XU6FQ6bQBntMfnNaeb5Wsy4xS0Y040Q?e=hrkQP6" target="_blank">here</a> to download.
2. Go to the [PowerApps Portal](https://powerapps.microsoft.com) and click on Apps in the left nav bar.
3. Click _Import Canvas App_ from the toolbar and upload the file.
4. On the Import package screen set the following and click _Import_.
    1. _Customer Subscription INFO_: Via the ACTION control, set the IMPORT SETUP to be _create as new_ and click the Import button below.
    2. _Related Resources - SQL Server Connection_: Via the ACTION control, create a new SQL Server Connection pointing to the AzureSubscriptions database on your newly created SQL Server and select it so the new connection shows under IMPORT SETUP.

![picture alt](/images/PowerApps-ImportPackage.png "PowerApps")

5. Once finished your Power App should be live, you can test it within PowerApps.  It is time to move onto the Power BI report.

### Power BI Report Configuration ###

1. Download the report from the internal Teams site by clicking <a href="https://microsoft.sharepoint.com/:u:/t/ScottTestSite/EcbI60nSdTVOsKfAQ1WNdqMBDdBRXdtHRTiOVgUVE4e1-g?e=GvxqQC" target="_blank">here</a>.
2. Open the report with <a href="https://powerbi.microsoft.com/en-us/desktop/" target="_blank">Power BI Desktop</a> and from the toolbar, click the _Transform data_ drop down and select _Data source settings_. 
3. You are presented with the Data source settings popup window.  Click change source in the bottom left and change the Server value to your SQL Server name found on the overview tab of your SQL Server within the [Azure Portal](https://portal.azure.com).  When finished click OK.
4. Back at the Data source settings popup window click Close and on the Power BI main screen click Apply Changes.
5. At the Native Database Query popup window click Run.
6. At the SQL server database window click Database along the left and enter the serverAdminUsername and serverAdminPassword values you inputted in the *01-azuredeploy_parameters.json* file and click OK.  Your report should refresh with your data.
7. Go to the Data Entry tab in the report and click on the Insert tab in the toolbar and select Power Apps.
    1. Add the following fields to the PowerApps Data control in the Visualizations pane
        1. Subscription ID
        2. Subscription Name
        3. Subsidiary
    2. Click _Choose app_ in the Power App control and select the newly created _Customer Subscription Info_ canvas app and click Add.
    3. Resize and test the Power App.  Your values will show in the Power BI report once you refresh the dataset and the report.
8. Save your Power BI Report.
9. You can publish your report to the Power BI Web service using these [instructions](https://docs.microsoft.com/en-us/power-bi/create-reports/desktop-upload-desktop-files).
