# Microsoft Fabric Scalable and Efficient OneLake Hydration: Bronze Layer Curation for Full, Change Tracking, CDC, and Watermark Change Capture Mechanisms  #

TBD

## High-level Architecture

 <img width="736" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/68d6b525-6eaa-4989-bfa2-d50ecae9dcf7">


## High-level Flow

The High level flow  involves the following steps:

- The SQL Tables to be copied and their Full/Incremental copy methodology (Full , Change Capture , CDC , Watermark) are configured in a control table

- Generic Pipelines are created that can ingest incremental data based on the change capture mechanism
  
   -  Pipelines that can copy data for Full load , incremental data copy based on Change Capture , CDC , Watermark are created. 

- Generic Pyspark notebooks that can Upsert and Delete for each incremental data copy mechanism (Change Capture , CDC , Watermark) are created.

- Orchestration pipeline is designed to execute the ingestion and curation notebook for each incremental data copy mechanism (Change Capture , CDC , Watermark)

- Orchestration reads from the control table and for each table to be ingested the corresponding ingestion pipeline and curation notebook are executed.

  -  For example , If the table to be ingested has CDC has the change capture mechanism then CDC Ingestion and Bronze layer curation notebook is executed
  
- The Orchestration pipeline executes the ingestion and curation in a parrallel fashion for each table to ingested


## Prerequisites


- Microsoft Fabric You can enable free trial [here](https://learn.microsoft.com/en-us/fabric/get-started/fabric-trial).
- Access to SQL Server and SQL Databases (Azure based)


## Things to Note 




## Steps


**MetaData SQL Configurations**

-  Create the Artifacts required for the Metadata driven ingestion by follwoing below steps
   
- Below is a sample schema for the configutaion table to store the tables to be ingested

   
         CREATE TABLE [dbo].[ControlTable](
         	[TableID] [int] IDENTITY(1,1) NOT NULL,
         	[SourceTableName] [varchar](255) NULL,
         	[FolderPath] [varchar](800) NULL,
         	[SchemaName] [varchar](50) NULL,
         	[PartitionColumnName] [varchar](255) NULL,
         	[PartitionLowerBound] [varchar](50) NULL,
         	[PartitionUpperBound] [varchar](50) NULL,
         	[LakeHouseName] [varchar](255) NULL,
         	[OneLakePath_or_TableName] [varchar](255) NULL,
         	[CopyMode] [varchar](50) NULL,
         	[DeltaColumnName] [varchar](255) NULL,
         	[IsActive] [varchar](10) NULL,
         	[WaterMarkColumnName] [varchar](100) NULL,
         	[ProcessingPath] [varchar](255) NULL,
         	[LakehouseArtifactID] [varchar](255) NULL
         ) ON [PRIMARY]
         GO
- Below is a sample schema for the log table to store custom pipleine run data

          CREATE TABLE [dbo].[IngestionLog](
        	[Id] [bigint] IDENTITY(1,1) NOT NULL,
        	[ServerName] [varchar](100) NULL,
        	[DbName] [varchar](100) NULL,
        	[TableName] [varchar](100) NULL,
        	[RunStatus] [varchar](50) NULL,
        	[SourceCount] [bigint] NULL,
        	[BronzeCount] [bigint] NULL,
        	[UpdateDate] [datetime] NULL,
        	[ErrorMessage] [varchar](max) NULL,
         CONSTRAINT [PK_IngestionLog] PRIMARY KEY CLUSTERED 
        (
        	[Id] ASC
        )WITH (STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
        GO

-  Below is a sample schema for storing the last changed version for Watermark , CDC , Change Tracking
  
        CREATE TABLE [dbo].[table_store_watermark_value](
        	[ServerName] [varchar](255) NULL,
        	[TableName] [varchar](255) NULL,
        	[WatermarkValue] [datetime] NULL
        ) ON [PRIMARY]
        GO

        CREATE TABLE [dbo].[table_store_ChangeTracking_version](
        	[TableName] [varchar](255) NULL,
        	[SYS_CHANGE_VERSION] [bigint] NULL
        ) ON [PRIMARY]
        GO
        
        CREATE TABLE [dbo].[table_store_CDC_version](
        	[ServerName] [varchar](255) NULL,
        	[TableName] [varchar](255) NULL,
        	[CDCLastRun] [varchar](255) NULL
        ) ON [PRIMARY]
        GO

-  Below are stored procedures that can upsert in to above mentioned tables

      
       CREATE PROCEDURE [dbo].[InsertLog]
       (
           -- Add the parameters for the stored procedure here
            @ServerName   varchar (100) NULL,
       	 @DbName   varchar (100) NULL,
       	 @TableName   varchar (100) NULL,
       	 @RunStatus   varchar (50) NULL,
       	 @SourceCount   bigint  NULL,
       	 @BronzeCount   bigint  NULL,
       	 @error varchar(max)
       )
       AS
       BEGIN
           -- SET NOCOUNT ON added to prevent extra result sets from
           -- interfering with SELECT statements.
           SET NOCOUNT ON
       
       
       
       INSERT INTO  dbo.IngestionLog 
                  ( ServerName 
                  , DbName 
                  , TableName 
                  , RunStatus 
                  , SourceCount 
                  , BronzeCount 
       		   ,ErrorMessage
                  , UpdateDate )
            VALUES
                  (@ServerName,  
                  @DbName,  
                  @TableName,  
                  @RunStatus,
                  @SourceCount,
                  @BronzeCount, 
       		   @error,
                  GETDATE())
       
       
       
       
       END
       
     
       SET ANSI_NULLS ON
       GO
       SET QUOTED_IDENTIFIER ON
       GO
       
       CREATE PROCEDURE [dbo].[Update_CDCLSN_Version]  @serverName varchar(255), @TableName varchar(255), @lsn varchar(50)
       AS
       BEGIN
       
       IF EXISTS(SELECT 1 FROM dbo.table_store_CDC_version WHERE ServerName = @serverName and TableName=@TableName)
            UPDATE dbo.table_store_CDC_version 
            set [CDCLastRun] = @lsn
       	 where ServerName = @serverName and TableName=@TableName
       ELSE
            INSERT INTO dbo.table_store_CDC_version
                 (ServerName, TableName, CDCLastRun)
            VALUES
                 (@serverName,@TableName,@lsn)
       
       
       END
       
     
       SET ANSI_NULLS ON
       GO
       SET QUOTED_IDENTIFIER ON
       GO
       CREATE PROCEDURE [dbo].[Update_ChangeTracking_Version] @CurrentTrackingVersion BIGINT, @TableName varchar(50)
       AS
       BEGIN
       UPDATE table_store_ChangeTracking_version
       SET [SYS_CHANGE_VERSION] = @CurrentTrackingVersion
       WHERE [TableName] = @TableName
       END
       GO
      
       SET ANSI_NULLS ON
       GO
       SET QUOTED_IDENTIFIER ON
       GO
       
       
       CREATE PROCEDURE [dbo].[Update_WaterMark_Value]  @serverName varchar(255), @TableName varchar(255), @watermark datetime
       AS
       BEGIN
       
       IF EXISTS(SELECT 1 FROM dbo.table_store_watermark_value WHERE ServerName = @serverName and TableName=@TableName)
            UPDATE dbo.table_store_watermark_value 
            set [WatermarkValue] = @watermark
       	 where ServerName = @serverName and TableName=@TableName
       ELSE
            INSERT INTO dbo.table_store_watermark_value
                 (ServerName, TableName, WatermarkValue)
            VALUES
                 (@serverName,@TableName,@watermark)
       
       
       END
       
       GO


##### Configure the tables to be ingested 

- Insert the tables to be ingested in the control table
  
- The Copy Mode indicates the nature of copy - CDC , CHANGE TRACKING , Full , WATERMARK are the values to be configured
  
- For Full copy mechanism where count of rows to be copied can be very high , we can enable parrallel data copy using the Partition column names
  -  PartitionColumnName would denote the partition column that the pipeline can use to select in multiple threads
  -  PartitionLowerBound and PartitionUpperBound denote the upper and lower bound for the parallel copy 

- Refer [here](https://learn.microsoft.com/en-us/azure/data-factory/connector-sql-server?tabs=data-factory#parallel-copy-from-sql-database) to understand various parrallel copy partition options  

- In order to configure the Target Lakehouse file location and the target lakehouse in fabric follow below steps
  
-  To retrieve the LakeHouseName , Select any table or file within the fabric lakehouse and click on properties and select the ABFS path until the lakehouse name , the format would be abfss://<workspacename>@<tenanttname>-onelake.dfs.fabric.microsoft.com/<lakehousename>.Lakehouse

      <img width="439" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/0248e8ff-efd7-4d68-8d7a-73968aff4132">

 -  One way to retreive the lakehouse artifact id to enable run time  Lakehouse sink creation , create a data pipeline and drag a copy activity and chose the lakehouse and view code the Json code would provide the artifactId for Lakehouse which needs to configured in LakehouseArtifactID column
   
 
  <img width="628" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/54d50fc5-d1ea-42ee-9db3-1cc89344ad6e">


  <img width="841" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/702b945c-2a30-4064-b443-f6a89a14ecb7">

 - The paths where the files to be landed for processing in one lake should be configured , these paths are relative and can be for example "/tobeprocessed/cdc/"

 - The Primary keys of the tables would be configured in deltacolumnname and for watermark additional watermark column name usually a date column is also needed

 - The control table would be as below after configuration

   <img width="1193" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/29b062d4-364c-498b-86d9-baae9bbc68fa">

##### Building Generic Ingestion Pipelines

#### Note:- The Idea to create individual pipelines to handle Full and other change capture mechanism is to adress seperation of concerns , flexibility in deployment and ease of maintenance 

##### Creating the Generic CDC Load Pipeline

- Browse to your Fabric enabled workspace in Power Bi and switch to Data Factory and create a new pipeline

  <img width="388" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/5d27b4f1-b4a0-4ff0-9483-f6babc7b0cf6">

- Name the Pipeline related to CDC , For example "PL_GENERIC_SQL_CDC_INGEST"
  
- The CDC workflow is as below
 -  Select the From LSN last processed and the current pipeline run date as To LSN
 -  Use the CDC function fn_cdc_get_all_changes_  or fn_cdc_get_net_changes_  to retrieve the delta between the previous run and current run
 -  Copy the delta data into Parquet files and make a transient copy to processing folder
 -  The copy of processing folder will be deleted after curation , this way you keep the actual data landed from Source SQL for logging , auditing , debugging needs
 -  Update the ingestion log and the CDC LSN table to store the latest sucessful LSN value for next run 
  
- Add the parameters required for the pipeline , all these parameters will be read from the control and the CDC version table

          "parameters": {
                   "TableName": {
                       "type": "string"
                   },
                   "SchemaName": {
                       "type": "string"
                   },
                   "LakeHouseName": {
                       "type": "string"
                   },
                   "OneLakePath": {
                       "type": "string"
                   },
                   "LSNStartTime": {
                       "type": "string"
                   },
                   "LSNEndTime": {
                       "type": "string"
                   },
                   "ProcessingPath": {
                       "type": "string"
                   },
                   "UniqueID": {
                       "type": "string"
                   }
        
- Create lookup activity to retreive the last run LSN from LSN Table


  <img width="512" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/687b16bc-5bea-4ff7-ac97-cb42ef0fb4ef">

        Select * from dbo.table_store_CDC_version where TableName='@{concat(pipeline().parameters.SchemaName,'.', pipeline().parameters.TableName)}'


  - Using a Lookup activity to retrieve the count of changed rows between last run and current run

                 @concat('DECLARE @begin_time datetime, @end_time datetime, @from_lsn binary(10), @to_lsn binary(10) ; 
             SET @begin_time = ''',activity('LookupLSNCDC').output.firstRow.CDCLastRun,''';
             SET @end_time = ''',pipeline().parameters.LSNEndTime,''';
             SET @from_lsn = sys.fn_cdc_map_time_to_lsn(''smallest greater than or equal'', @begin_time);
             SET @to_lsn = sys.fn_cdc_map_time_to_lsn(''largest less than or equal'', @end_time);
             SELECT count(1) changecount FROM cdc.fn_cdc_get_net_changes_',pipeline().parameters.SchemaName,'_',pipeline().parameters.TableName,'(@from_lsn, @to_lsn,   ''all'')')
  
- Add a if condiiton and only if there are count of records to ingest then we will have to copy the records


  <img width="781" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/9bc65640-b1a5-4813-b7ba-13e16b502572">


        @greater(int(activity('Get CDC Change Count').output.firstRow.changecount),0)


- Within the True section of the If activity , Copying of the delta data and upadting the logs and LSN values happen

  <img width="784" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c51ba082-710b-404a-8706-371c91258f37">

- In the copy activity source , establish connection to SQL server and use below query to pull incremental  data from the CDC functions
  
 <img width="751" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c32cd554-1be6-4e16-902a-35ccad3e59f9">

          @concat('DECLARE @begin_time datetime, @end_time datetime, @from_lsn binary(10), @to_lsn binary(10) ; 
         SET @begin_time = ''',activity('LookupLSNCDC').output.firstRow.CDCLastRun,''';
         SET @end_time = ''',pipeline().parameters.LSNEndTime,''';
         SET @from_lsn = sys.fn_cdc_map_time_to_lsn(''smallest greater than or equal'', @begin_time);
         SET @to_lsn = sys.fn_cdc_map_time_to_lsn(''largest less than or equal'', @end_time);
         SELECT * FROM cdc.fn_cdc_get_all_changes_',pipeline().parameters.SchemaName,'_',pipeline().parameters.TableName,'(@from_lsn, @to_lsn, ''all'')')


  - In the destination of the Copy Activity Parameterize the LakeHouse object ID and the file path to be written

    <img width="800" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/de7bb4b0-851e-4ea5-8352-6e6dd29a8e81">

              @pipeline().parameters.LakeHouseName
                @concat(pipeline().parameters.OneLakePath,pipeline().parameters.TableName,'/','year=',formatDateTime(utcnow(),'yyyy'),'/','month=',formatDateTime(utcnow(),'MM'),'/','day=',formatDateTime(utcnow(),'dd'),'/',pipeline().parameters.UniqueID)
          



     
