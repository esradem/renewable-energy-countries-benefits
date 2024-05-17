# Esra's Portfolio Project

# Introduction
## Aim:
To assess the evolution and impact of renewable energy sources on the total primary energy supply from 1960 to 2015.

## Scope:
- Renewable energy sources including hydro (excluding pumped storage), geothermal, solar, wind, tide, and wave.
+ Bioenergy sources such as solid biofuels, biogasoline, biodiesels, other liquid biofuels, biogases, and the renewable fraction of municipal waste.
* Definition and inclusion criteria for biofuels and municipal waste in the context of renewable energy.

## Data Collection
### Sources:
This dataset is collected from Kaggle.
Historical energy statistics from international energy agencies, national governments, and research institutions.
Data on renewable energy production and consumption, categorized by source (hydro, wind, solar, etc.).
Information on the primary energy equivalent conversion for various renewable sources. " Energy derived from solid biofuels, biogasoline, biodiesels, other liquid biofuels, biogases and the renewable fraction of municipal waste are also included. Biofuels are defined as fuels derived directly or indirectly from biomass (material obtained from living or recently living organisms). This includes wood, vegetal waste (including wood waste and crops used for energy production), ethanol, animal materials/wastes and sulphite lyes. Municipal waste comprises wastes produced by the residential, commercial and public service sectors that are collected by local authorities for disposal in a central location for the production of heat and/or power. This indicator is measured in thousand toe (tonne of oil equivalent) as well as in percentage of total primary energy supply."
### Metrics:
Quantity of energy produced by each renewable source, measured in thousand tonne of oil equivalent (ktoe).
'The tonne of oil equivalent (toe) is a unit of energy defined as the amount of energy released by burning one tonne of crude oil. It is approximately 42 gigajoules or 11.630 megawatt-hours.'
The percentage share of each renewable source in the total primary energy supply.
## Methodology
![Method diagram](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/Methodology-01.jpg?raw=true)<br />
### Data Analysis:
- Calculate the annual contribution of each renewable energy source to the TPES.
* Analyze trends in renewable energy usage, identifying periods of significant growth or decline.
+ Further analysis can be conducted; the impact of technological, economic, and policy factors on renewable energy development.
### Tools and Techniques:
 Analyzing the location-based data on renewable energy usage from 1960 to 2015 with a focus on tools like Python, Azure Data Factory, Synapse Analytics, and Power BI.
- Data cleaning and preprocessing.
* Exploratory data analysis (EDA) to understand trends, patterns, and anomalies in the data.
   

# Prerequirement 
+ Kaggle API Token
+ Azure Subscription
+ Azure Data Lake Storage 
+ Azure Data Factory Studio and Synapse Analysis workspace
+ Power BI Desktop
After your data settled to ADLS you need to give necessary permissons to ADF and Synapse to create link services.
[ˆ1] Add you IP to Synapse Workspace through networking tab
[ˆ2] To access ADLS data, the ADF workspace should have the storage blob data contributor role in the ADLS account. This is done inside ADLS > Access Control tab
## Setup
First inside the Azure CLI you need to connect Kaggle and pull the dataset. If you want quicker way you can:
- Download Renewable energy dataset from Kaggle
+ Upload it to Azure Data Lake storage

## Step by step 

CLI Bash commands are:
```
#Install the Kaggle API: If not already installed, you need to install the Kaggle API. This can be done in an Azure virtual machine or Azure Cloud Shell.
pip install kaggle

#Upload kaggle.json to your Azure VM or Cloud Shell, and set it up:
mkdir ~/.kaggle
cp path_to_your_kaggle.json ~/.kaggle/kaggle.json
chmod 600 ~/.kaggle/kaggle.json

# Download dataset from Kaggle
kaggle datasets download -d dataset-owner/dataset-name -p /path/to/download

# Unzip if necessary
unzip /path/to/download/dataset-name.zip -d /path/to/download

# Upload to Azure Blob Storage
az storage blob upload --account-name yourstorageaccount --container-name yourcontainer --file "/path/to/download/dataset.csv" --name "dataset.csv"

# Clean up local files if you don't need them after upload
rm -rf /path/to/download/*
```
- First create external table in the Synapse serverless SQL pool.
```
#1-create external file format
IF NOT EXISTS (SELECT * FROM sys.external_file_formats WHERE name = 'SynapseDelimitedTextFormat') 
	CREATE EXTERNAL FILE FORMAT [SynapseDelimitedTextFormat] 
	WITH ( FORMAT_TYPE = DELIMITEDTEXT ,
	       FORMAT_OPTIONS (
			 FIELD_TERMINATOR = ',',
			 USE_TYPE_DEFAULT = FALSE,
			 FIRST_ROW = 2 
			))
GO

#2-create external data source
IF NOT EXISTS (SELECT * FROM sys.external_data_sources WHERE name = 'renewableenergycontainer_datalakestorageren003_dfs_core_windows_net') 
	CREATE EXTERNAL DATA SOURCE [renewableenergycontainer_datalakestorageren_dfs_core_windows_net] 
	WITH (
		LOCATION = 'abfss://rencontainer@datalakestorageren.dfs.core.windows.net' 
	)
GO

#3  Role assignment to data factory
CREATE USER [datafactory] FROM EXTERNAL PROVIDER
ALTER ROLE db_owner ADD MEMBER [datafactory]

#4-create external data source
CREATE EXTERNAL TABLE dbo.renenergy (
	  Location NVARCHAR(100),
    Indicator NVARCHAR(100),
    Subject NVARCHAR(100),
    Measure NVARCHAR(100),
    Frequency NVARCHAR(100),
    Time NVARCHAR(100),  -- Assuming 'Time' is a year
    Value DECIMAL(15,4),  -- Decimal with 10 digits precision and 2 digits scale
    FlagCodes NVARCHAR(50)
	)
	WITH (
	LOCATION = 'renewable_energy.csv',
	DATA_SOURCE = [container_datalakestorageren_dfs_core_windows_net],
	FILE_FORMAT = [SynapseDelimitedTextFormat]
	)
GO

SELECT TOP 10 * FROM dbo.renenergy
GO
```
+ Next launch the Data Factory and create pipeline
+ From Activities > Move and Transform > Copy data
+ Clean the data and prepare for loading to Synapse for further analytics.
![Azure Data Factory Screen](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/ADF_1.png?raw=true)
+ While uploading the data from Data Lake to Synapse true mapping is very important. Look how is the data type in Data lake csv file and how will be in the synapse data type.<br />
![Azure Data Factory Mapping](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/ADF_2.png?raw=true)<br />
 Here is json file of creating pipeline <br />
```
{
    "name": "pipeline1",
    "properties": {
        "activities": [
            {
                "name": "Copy Dataset",
                "type": "Copy",
                "dependsOn": [],
                "policy": {
                    "timeout": "0.12:00:00",
                    "retry": 0,
                    "retryIntervalInSeconds": 30,
                    "secureOutput": false,
                    "secureInput": false
                },
                "userProperties": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource",
                        "storeSettings": {
                            "type": "AzureBlobFSReadSettings",
                            "recursive": true,
                            "enablePartitionDiscovery": false
                        },
                        "formatSettings": {
                            "type": "DelimitedTextReadSettings"
                        }
                    },
                    "sink": {
                        "type": "SqlDWSink",
                        "allowPolyBase": true,
                        "polyBaseSettings": {
                            "rejectValue": 0,
                            "rejectType": "value",
                            "useTypeDefault": true
                        }
                    },
                    "enableStaging": true,
                    "stagingSettings": {
                        "linkedServiceName": {
                            "referenceName": "AzureDataLakeStorage1",
                            "type": "LinkedServiceReference"
                        },
                        "path": "renewableenergycontainer"
                    },
                    "translator": {
                        "type": "TabularTranslator",
                        "mappings": [
                            {
                                "source": {
                                    "name": "LOCATION",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "Location",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "INDICATOR",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "Indicator",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "SUBJECT",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "Subject",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "MEASURE",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "Measure",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "FREQUENCY",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "Frequency",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "TIME",
                                    "type": "String",
                                    "format": "yyyy"
                                },
                                "sink": {
                                    "name": "Time",
                                    "type": "String"
                                }
                            },
                            {
                                "source": {
                                    "name": "Value",
                                    "type": "Decimal"
                                },
                                "sink": {
                                    "name": "Value",
                                    "type": "Decimal"
                                }
                            },
                            {
                                "source": {
                                    "name": "Flag Codes",
                                    "type": "String"
                                },
                                "sink": {
                                    "name": "FlagCodes",
                                    "type": "String"
                                }
                            }
                        ]
                    }
                },
                "inputs": [
                    {
                        "referenceName": "csvrenenrfile",
                        "type": "DatasetReference"
                    }
                ],
                "outputs": [
                    {
                        "referenceName": "AzureSynapseAnalyticst",
                        "type": "DatasetReference"
                    }
                ]
            },
           
            
        ],
    

    },
    "type": "Microsoft.DataFactory/factories/pipelines"
}
```
+ When you sink to linked service Azure Synapse Analytics, it is ready to publish all the changes.
+ You can open Synapse and in the SQL pool and query your table to see if everything is alright.<br />
 For analysis in Synapse there is 2 query to see how countries benefited from the renewable resources.
### Top 15 Country Renewable Enerygy Production from 1960 to 2015:
```
SELECT TOP 15 location, Time, SUM(Value) AS total_renewable_energy
FROM dbo.renenergy
GROUP BY location, Time
ORDER BY total_renewable_energy DESC;
```
### Top 20 Country Renewable Energy Production at 2015
```

SELECT TOP 20 location,
              SUM(value) AS total_renewable_energy
FROM dbo.renenergy
WHERE dbo.renenergy.Time = '"2015"'
GROUP BY Location
ORDER BY total_renewable_energy DESC;
```


+ After creating those table you can export them and upload to Power BI and visualize it. 
+ In order dynamic visualization you can create a link service to Power BI directly from Azure Synapse Analytics.
![configuring power bi](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/connecting_server.png?raw=true)
![configuring power bi](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/selecting_table.png?raw=true)
Inside the Power BI you need to connect to SQL database using your login information and crenditials. <br />
Then you can choose which table you want to visualize> choose the columns. If you want to edit the specific columns or exclude it as I did in the renewable energy production over time analysing per country, you can do it by clicking the Transform data.<br />
When you select all the column you want and you can even get query from Synapse by get data button. All of the visualization in your Dashboear in Power BI you can export it as a pdf.<br />
![power bi export](https://github.com/esradem/portfolio/blob/main/Renewable_Energy_Resources_Country_Benefits/Images/analyticsof%20renewableenergy.jpg?raw=true)


# End