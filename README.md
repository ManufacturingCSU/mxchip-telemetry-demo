# mxchip-telemetry-demo
The purpose of this repository is to be able to deploy the MX Chip demonstration with a cold, warm and hot path scenario.

### Setup ###

Please clone this repository to a local directory on your computer using the following URL: `https://github.com/ManufacturingCSU/mxchip-telemetry-demo`

In order to add all the components above, we will need to create a new resource group within Azure and add the following components:
* IoT Hub
* Stream Analytics
* Azure Storage Account
* SQL Server
* SQL Database
* Azure Function
* Service Bus
* Logic App

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

#### Deploying the Azure Assets within the Resource Group ####

Next we need to deploy the following resources within the Resource Group:
* IoT Hub
* Stream Analytics
* Azure Storage Account
* SQL Server
* SQL Database
* Azure Function
* Service Bus
* Logic App

This will be done using the *01-azuredeploy.json* and *01-azuredeploy_parameters.json* ARM template files in the arm folder of this repository.  After you have cloned this directory, there are two steps to follow...
1. Open the *01-azuredeploy_paramaters.json* file and add the values needed for any section with  _insert value here_
    1. objectid - This is found under your user profile within Azure Active Directory setting within the Azure portal 
    2. tenantid - This is found under the Azure Active Directory setting within the Azure portal
2. Open powershell and navigate to the arm directory of this cloned repository and run the following command:
```javascript
$parameters = @{
    'Name' = 'MXChip Telemetry Deployment'
    'ResourceGroupName' = '<insert your resource group name from above>'
    'TemplateFile'      = '01-azuredeploy.json'
    'TemplateParameterFile' = '01-azuredeploy_parameters.json'
    'Verbose' = $true
}

New-AzResourceGroupDeployment @parameters
```
Navigate to the [Azure Portal](https://pocrtal.azure.com) to view your resource group and the resources within it.

#### Creating the tables and views within the SQL Database ####

Navigate to your SQL Server and under _the Firewalls and Virutal Netowrks_ blade:
1. Add your IP address
2. Set _Allow Azure services and resources to access this server_ to Yes.
Save your firewall settings and proceed to the next step. 

Using [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15), connect to the *IoT* SQL Database on your newly created server and run all cells in the *IoT Database* notebook in the notebook directory of this repository.

Once completed, your *AzureSubscriptions* Database should have the following:
* Telemetry table




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
