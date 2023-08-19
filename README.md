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

 -  One way to retreive the lakehouse artifcat id to enable run time  Lakehouse sink creation , create a data pipeline and drag a copy activity and chose the lakehouse and view code the Json code would provide the artifactId for Lakehouse which needs to configured in LakehouseArtifactID column
   
 
  <img width="628" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/54d50fc5-d1ea-42ee-9db3-1cc89344ad6e">


  <img width="841" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/702b945c-2a30-4064-b443-f6a89a14ecb7">

 - The paths where the files to be landed for processing in one lake should be configured , these paths are relative and can be for example "/tobeprocessed/cdc/"

 - The control table would be as below after configuration

   <img width="1193" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/29b062d4-364c-498b-86d9-baae9bbc68fa">

##### Building Generic Ingestion Pipelines

#### Note:- The Idea to create individual pipelines to handle Full and other change capture mechanism is to adress seperation of concerns , flexibility in deployment and ease of maintenance 

##### Creating the Generic CDC Load Pipeline

- Browse to your Fabric enabled workspace in Power Bi and switch to Data Factory and create a new pipeline

  <img width="388" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/5d27b4f1-b4a0-4ff0-9483-f6babc7b0cf6">

- Name the Pipeline related to CDC , For example "PL_GENERIC_SQL_CDC_INGEST"
  
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
        
- d
  
  
- We have below  functions
  -  Get current time for a location using pytz
  -  Basic Arithmetic calculator
  -  Return top 1 search result from bing search (modify as needed to return more)
  -  Use Pandas dataframe to filter data from Car sales dataset
  -  Get Current stock price using the freemium developer api
  -  Get Current weather  using the freemium developer api
  
*The functions provided in this sample are modified to return simple strings from Json results without robust exception handling , Please ensure to thoroughly validate and test the freemium sample API before integrating it into your application. This includes checking for error handling, response validation, and understanding any potential limitations or restrictions and more* 

       def get_current_time(location):
             try:
                 # Get the timezone for the city
                 timezone = pytz.timezone(location)
         
                 # Get the current time in the timezone
                 now = datetime.now(timezone)
                 current_time = now.strftime("%I:%M:%S %p")
         
                 return current_time
             except:
                 return "Sorry, I couldn't find the timezone for that location."      
             
             
         
         
         def calculator(num1, num2, operator):
             if operator == '+':
                 return str(num1 + num2)
             elif operator == '-':
                 return str(num1 - num2)
             elif operator == '*':
                 return str(num1 * num2)
             elif operator == '/':
                 return str(num1 / num2)
             elif operator == '**':
                 return str(num1 ** num2)
             elif operator == 'sqrt':
                 return str(math.sqrt(num1))
             else:
                 return "Invalid operator"
             
         
         
         def get_bing_search_results(query):
             mkt = 'en-US'
             params = { 'q': query, 'mkt': mkt ,"count":1, "answerCount":1 ,"textDecorations": True, "textFormat": "HTML","responseFilter":"webpages" }
             headers = { 'Ocp-Apim-Subscription-Key': subscription_key }
             try:
                 response = requests.get(search_url, headers=headers, params=params)
                 response.raise_for_status()
                 data= (response.json())
                 #data = json.loads(json_data)
                 name = data['webPages']['value'][0]['name']  
                 url = data['webPages']['value'][0]['url']  
                 return name + ' ' + url
             except Exception as ex:
                 raise ex
             
         
         
         def get_auto_sales_data(carbrand):
             try:
                 
                 # Read the CSV file , file name as UsAutoSales2022.csv 
                 data = pd.read_csv('UsAutoSales2022.csv')
         
                 # Filter data for the given index
                 data_filtered = data[data['Brand'] == carbrand]
         
                 
         
                 # Convert the DataFrame into a dictionary
                 hist_dict = data_filtered.to_dict()
         
                 for key, value_dict in hist_dict.items():
                     hist_dict[key] = {k: v for k, v in value_dict.items()}
         
                 return json.dumps(hist_dict)
             except:
                 return "Sorry, I couldn't find the stock price."
             
         
         def get_stockprice(symbol):
             try:
                 url = vantage_url
                 
                 querystring = {"function": "GLOBAL_QUOTE", "symbol": "{}", "datatype": "json"}  
                 querystring["symbol"] = symbol  
                 headers = {
                 "X-RapidAPI-Key": vantage_key,
                 "X-RapidAPI-Host": "get from the devloper api site"}
                 response = requests.get(url, headers=headers, params=querystring)
                 #print(response.json())
                 return('Price for '+ response.json()['Global Quote']['01. symbol'] + ' is ' + response.json()['Global Quote']['05. price'])
                 #data = json.loads(response.json()) 
                 
             except:
                 return "Sorry, I couldn't find the stock price."
             
         
         def getweather(location):
             try:
                 url = weather_url
                 querystring = {"location":"{}","format":"json","u":"f"}
                 querystring["location"] = location 
                 headers = {
         	"X-RapidAPI-Key": weather_api_key,
         	"X-RapidAPI-Host": "get from the devloper api site"
          }
                 response = requests.get(url, headers=headers, params=querystring)
                 data=response.json()
                 location = data['location']['city']  
                 temperature = data['current_observation']['condition']['temperature']  
                 result = f"The weather in {location} is {temperature}F"  
                 return(result)  
                 #return (response.json())
             except:
                 return "Sorry, I couldn't find the weather for that location."
    
##### Integrating Azure Open AI Function call 

- Use code below the integrate Azure Open AI function calling in our Azure Function
  *As mentioned above refer [here](https://github.com/Azure-Samples/openai/blob/main/Basic_Samples/Functions/working_with_functions.ipynb) to understand the function calling  logic*

          functions = [
                  {
                      "name": "get_current_time",
                      "description": "Get the current time in a given location",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "location": {
                                  "type": "string",
                                  "description": "The location name. The pytz is used to get the timezone for that location. Location names should be in a format like America/New_York, Asia/Bangkok, Europe/London",
                              }
                          },
                          "required": ["location"],
                      },
                  },
                  {
                      "name": "get_auto_sales_data",
                      "description": "Get the sales  data for a given car brand",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "carbrand": {
                                  "type": "string",
                                  "description": "The car brand name. Car brand names are like Honda , Toyota , Ford" },
                          },
                          "required": ["carbrand"],
                      },    
                  },
                  {
                      "name": "calculator",
                      "description": "A simple calculator used to perform basic arithmetic operations",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "num1": {"type": "number"},
                              "num2": {"type": "number"},
                              "operator": {"type": "string", "enum": ["+", "-", "*", "/", "**", "sqrt"]},
                          },
                          "required": ["num1", "num2", "operator"],
                      },
                  },
                  {
                      "name": "get_stockprice",
                      "description": "Retrieve the stock price for a given stock symbol",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "symbol":  {
                              "type": "string",
                              "description": "Stock symbol, for example MSFT for Microsoft , AAPL for Apple"
                              }
                          },
                          "required": ["symbol"],
                      },
                  },
                  {
                      "name": "getweather",
                      "description": "Retrieve the weather  for a given location",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "location":  {
                              "type": "string",
                              "description": "location of a city, for example London , LA for Los Angeles"
                              }
                          },
                          "required": ["location"],
                      },
                  },
                  {
                      "name": "get_bing_search_results",
                      "description": "Retrieve the web search results from bing api",
                      "parameters": {
                          "type": "object",
                          "properties": {
                              "query":  {
                              "type": "string",
                              "description": "query for bing search , for example what is Azure AI"
                              }
                          },
                          "required": ["query"],
                      },
                  }
              ]
          
          available_functions = {
                      "get_current_time": get_current_time,
                      "get_auto_sales_data": get_auto_sales_data,
                      "calculator": calculator,
                      "get_stockprice":get_stockprice,
                      "getweather":getweather,
                      "get_bing_search_results":get_bing_search_results
                  } 



           #helper method used to check if the correct arguments are provided to a function

               def check_args(function, args):
                   sig = inspect.signature(function)
                   params = sig.parameters
               
                   #Check if there are extra arguments
                   for name in args:
                       if name not in params:
                           return False
                   #Check if the required arguments are provided 
                   for name, param in params.items():
                       if param.default is param.empty and name not in args:
                           return False
               
                   return True
               
               
               def run_conversation(messages, functions, available_functions, deployment_id):
                   #Step 1: send the conversation and available functions to GPT
               
                   response = openai.ChatCompletion.create(
                       deployment_id=deployment_id,
                       messages=messages,
                       functions=functions,
                       function_call="auto",
                       temperature=0.7,
                       max_tokens=700,
                       top_p=0.95,
                       frequency_penalty=0,
                       presence_penalty=0,
                       stop=None
                   )
                   response_message = response["choices"][0]["message"]
               
                   
               
                   #Step 2: check if GPT wanted to call a function
                   if response_message.get("function_call"):
                       print("Recommended Function call:")
                       print(response_message.get("function_call"))
                       
                       
                       #Step 3: call the function
                       #Note: the JSON response may not always be valid; be sure to handle errors
                       
                       function_name = response_message["function_call"]["name"]
                       
                       #verify function exists
                       if function_name not in available_functions:
                           return "Function " + function_name + " does not exist"
                       function_to_call = available_functions[function_name]  
                       
                       #verify function has correct number of arguments
                       function_args = json.loads(response_message["function_call"]["arguments"])
                       if check_args(function_to_call, function_args) is False:
                           return "Invalid number of arguments for function: " + function_name
                       function_response = function_to_call(**function_args)
                       
                       print("Output of function call:")
                       print(function_response)
                       print()
                       
                       #Step 4: send the info on the function call and function response to GPT
                       
                       #adding assistant response to messages
                       messages.append(
                           {
                               "role": response_message["role"],
                               "name": response_message["function_call"]["name"],
                               "content": response_message["function_call"]["arguments"],
                           }
                       )
               
                       #adding function response to messages
                       messages.append(
                           {
                               "role": "function",
                               "name": function_name,
                               "content": function_response,
                           }
                       )  #extend conversation with function response
               
                       print("Messages in second request:")
                       for message in messages:
                           print(message)
                       print()
               
                       second_response = openai.ChatCompletion.create(
                           messages=messages,
                           deployment_id=deployment_id,
                           temperature=0.7,
                           max_tokens=1000,
                           top_p=0.95,
                           frequency_penalty=0,
                           presence_penalty=0,
                           stop=None
                       )  # get a new response from GPT where it can see the function response
               
                       return second_response
                   else :
                       return(response_message['content'])


##### Function App HTTP trigger receive  User prompts and send responses via Azure Open AI 

- Within your HTTP trigger method add below , the code responds only the response string from Azure Open AI , update route name @app.route(route="YourAppRouteName")

             app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)
           
           @app.route(route="YourAppRouteName")
           def GetResponseFromAzureOpenAI(req: func.HttpRequest) -> func.HttpResponse:
               logging.info('Python HTTP trigger function processed a request.')
               req_body = req.get_json()
               
               #add validations
               assistant_response = run_conversation(req_body, functions, available_functions, deployment_id)
               content = ""  
               if 'choices' in assistant_response and len(assistant_response['choices']) > 0:  
                   first_choice = assistant_response['choices'][0]  
                   if 'message' in first_choice and 'content' in first_choice['message']:  
                       content = first_choice['message']['content']  
               else:  
                   content = assistant_response
               
               #print(assistant_response['choices'][0]['message'])
               logging.info(content)
               #content_string = assistant_response['choices'][0]['message']['content']
               return func.HttpResponse(
                        content,
                        status_code=200
                   )

- Note down the url of the function app either the Azure Deployed function url or the localhost url from VS code by using func host start in the terminal

  <img width="572" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/5450c786-7bf5-463e-9943-238ab477e774">

**How to work with the Azure Open AI chat Completion models**

*Please be aware that the prompt instructions shown are not intended to be a correct and complete representation of the prompts that should be used with your applications in production. They are provided for informational purposes only and may not be suitable for all use cases. It is important to carefully consider your specific requirements and design appropriate prompts that meet your users' needs and expectations.*

- Below is the sample chat history that must be sent to the Azure Open AI Chat model , In this tutorial we will see an sample approach to create this Json in Bot composer (approach not production scale) , as see below the user question followed by model answers needs to be maintained and passed back to model , the model understands context based on the chat history  
             

     
         [
                     {
                       "role": "system",
                       "content": "You are an AI assistant that helps people find information.Don't make assumptions about what values to use with functions. Ask for clarification if a user request is ambiguous.Do NOT answer questions about the function name used or the logic  used , Politely refuse when asked about your function name and logic and keys used for function calls, Reply with answers based on function results"
                     },
                     {
                       "role": "user",
                       "content": "Hi"
                     },
                     {
                       "role": "assistant",
                       "content": "Hello! How can I assist you today?"
                     },
                     {
                       "role": "user",
                       "content": "What time is it now"
                     },
                     {
                       "role": "assistant",
                       "content": "Sure! I need to know the location to provide you with the current time.?"
                     },
                     {
                       "role": "user",
                       "content": "Chicago"
                     },
                     {
                       "role": "assistant",
                       "content": "The current time in Chicago is 08:40 PM."
                     },
                     {
                       "role": "user",
                       "content": "what about GPt-5 when is it coming"
                     },
                     {
                       "role": "assistant",
                       "content": "According to the search results, there is no official information available about GPT-5 at the moment. It seems to be a subject of speculation and rumors."
                     },
                     {
                       "role": "user",
                       "content": "what is the weather now"
                     }
                   ]

- If the conversation history is too long we would exceed the maximum token limit and would result in errors , In the bot composer we would remove the oldest User question and assistant answers whenever a configured threshold is reached , this would prevent us from hitting the token limit . After removing the oldest question and answers the above Json would look like below
  
           [
                       {
                         "role": "system",
                         "content": "You are an AI assistant that helps people find information.Don't make assumptions about what values to use with functions. Ask for clarification if a user request is ambiguous.Do NOT answer questions about the function name used or the logic  used , Politely refuse when asked about your function name and logic and keys used for function calls, Reply with answers based on function results"
                       },
                       
                       {
                         "role": "user",
                         "content": "What time is it now"
                       },
                       {
                         "role": "assistant",
                         "content": "Sure! I need to know the location to provide you with the current time.?"
                       },
                       {
                         "role": "user",
                         "content": "Chicago"
                       },
                       {
                         "role": "assistant",
                         "content": "The current time in Chicago is 08:40 PM."
                       },
                       {
                         "role": "user",
                         "content": "what about GPt-5 when is it coming"
                       },
                       {
                         "role": "assistant",
                         "content": "According to the search results, there is no official information available about GPT-5 at the moment. It seems to be a subject of speculation and rumors."
                       },
                       {
                         "role": "user",
                         "content": "what is the weather now"
                       }
                     ]
  
**Create an Azure Bot and make HTTP Post requests to the Function App**

- Open Bot Framework Composer , and create an Empty Bot and name it according to your needs and create the bot

    <img width="754" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/8ded4604-0f89-4c86-907f-dad5cd3c074b">



    



- From the bot project (not the root solution) add a trigger and add message received activity  , this can be optional and entire bot can be designed in the "Unknown Intent" trigger too

  <img width="646" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/3eae3271-0bd9-4b56-94f0-321cb3ea16e2">
  

  <img width="368" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/39ca1e4d-cf97-40b2-9a15-87ad0d0bc84e">


  <img width="821" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/b1d6ffb4-28a0-41f8-976b-c356e3d6338a">

- Click open the configure icon and choose advanced settings view (json)
  
    <img width="1192" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/8f9fa79f-d64e-4bfc-8f1a-4e7bf75c0430">

- Add an api section in the config and fill below parameters 
         
            "api": {
             "AOI_Function_Url": "function app url here",
             "Past_Message_Count": 8
           },


    <img width="194" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/0cdf861e-1277-4a8b-afbe-bbf4ccf8b1a6">


  *Please note the tutorial uses conversation memory scope throughout the composer for simplicity and brevity , for your production use case ,use the most apt memory scope , refer [here](https://learn.microsoft.com/en-us/composer/concept-memory?tabs=v2x) for memory scopes and their usage , we build a json using string manipulations and array for simplicity and brevity and we donot use any custom components in this tutorial the bot is designed only for happy path and does not handle error scenarios*

- First step lets Create a Array property to hold user question and answers , coalesce to prevent errors on first time load

     ``` conversation.context.chathistory = =coalesce(conversation.context.chathistory, [])```

    <img width="341" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/18159c3a-d5b9-4875-8928-20d66e933475">
    
- Add the logic to remove the  oldest two messages in the chathistory array if the conversation history array contains more elements than our configured message count , this would prevent token limit errors , refer to section above to find details , this is done in create a condition -> if/else 

            =count(conversation.context.chathistory) > settings.api.Past_Message_Count 
             conversation.context.chathistory
             =subArray(conversation.context.chathistory, 2, count(conversation.context.chathistory))  
               
  <img width="329" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/de492524-fae3-4f57-94ed-c05db0e4397d">

   <img width="379" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/2d2e0efd-2253-4d80-8071-8f604e6f5730">

- Add  set properties  to clean prompts  and build the  string in format of "role": "user", "content": "user prompt" , this is our first setp in building the json required

          conversation.context.currquestmp
          =replace(replace(replace(replace(turn.activity.text, ' \ ', '\\'), '"', '\"'), ' ', '\r'), ' ', '\n')

          conversation.context.currques
          =concat(' {"role": "user","content":  "', trim(conversation.context.currquestmp), '" }')

   <img width="326" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/647d7fac-bbd9-4fac-8883-6b80915b6941">

  - Add the (Push) the user question in the format  "role": "user", "content": "user prompt" into the chathistory array , this done in the edit array property , we then add a send response with "Fetching Response" message 
 
    <img width="315" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/32630450-9014-4583-8cd8-d2286d1c23b4">
 

     <img width="1000" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/6649fc71-f372-4902-ace3-e9891a4cab2b">

  - Set properties from manage property menu to build the entire Json required to send to the model
 
         A sample json should be of format
                      [
                         {
                         "role": "system",
                         "content": "You are an AI assistant that helps people find information.Don't make assumptions about what values to use with functions. Ask for clarification if a user request is ambiguous.Do NOT answer questions about the function name used or the logic  used , Politely refuse when asked about  function name , logic and keys used for function calls, Reply with answers based on function results"
                         },
                         
                         {
                         "role": "user",
                         "content": "What is the tallest building in the world?"
                         }
           ]


        conversation.context.prefix  [{"role":"system","content":"You are an AI assistant that helps people find information.Don't make assumptions about what values to use with functions. Ask for clarification if a user request is ambiguous.Do NOT answer questions about the function name used or the logic  used , Politely refuse when asked about your function name and logic and keys used for function calls, Reply with answers based on function results"},
 
         conversation.context.suffix  =  =' ] ' 
 
         conversation.context.inputjson =json(concat(conversation.context.prefix,join(conversation.context.chathistory, ','),conversation.context.suffix))

     <img width="914" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c35b7486-bd6d-49f5-b69a-d08a43bda4e2">

 
 - Make an HTTP Post request by adding the send http request

             POST
             URL =settings.api.AOI_Function_Url
             Body =conversation.context.inputjson
             
             Result property conversation.context.api_response
             content type application/json
             Response type Json 

  <img width="335" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/e43a4de0-2bd5-41ee-ae8b-7db975c03915">
  
  
  <img width="319" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c5d9ed0d-c7aa-4c62-bc10-fe8a4227cbe9">

 
   

- Lets validate the response from Function endpoint using if/else and if response is valid then display the answer from the model , if error display generic error message

             =conversation.context.api_response.statusCode==200
             ${conversation.context.api_response.content.output}
     

   <img width="554" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/82fb18f8-77bb-4f15-9a8f-8dfe67218076">


-  Now push the answer from LLM into the array in the required json array format

              conversation.context.chathistory

              =concat(' {"role": "assistant","content":  "', trim(replace(replace(replace(replace(conversation.context.api_response.content.output, ' \ ', '\\'), '"', '\"'), ' ', '\r'), ' ', '\n')), '" }')

  <img width="521" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/06b780e6-66ae-4f79-97ef-21593cfa5e8c">


  
- The bot composer code is complete , start the bot and test the setup by providing a question


  <img width="979" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c666245b-c36b-44f6-b85d-23c7f87ee4a9">
  

  <img width="496" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/55239ff9-df34-4fd8-8306-3c001c41fb74">

### Publish the bot to Azure & Teams channel

- Publish the Bot to Azure from bot composer [refer here](https://learn.microsoft.com/en-us/composer/how-to-publish-bot?tabs=v2x)

- To Acess the bot in teams channel [refer here](https://techcommunity.microsoft.com/t5/modern-work-app-consult-blog/publish-bot-app-to-teams-channel-with-bot-framework-composer-and/ba-p/3341876)


## Testing  Azure Open AI Function Call

- Lets start testing by asking about car sales in the bot , As we see below the Azure Open AI determined the get_auto_sales_data as the recommended function and car brand name as a parameter from user prompt , the result from the function is Json Dictionary and Open AI converts the Json Dict result into natural langauge and answers back 

Bot Response 

<img width="1118" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/130653bf-1e58-4cf2-bf11-ba5be52ec33c">

Function Call

<img width="917" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/98d706c7-0684-4bdf-8b8e-60b5279a6054">

- Lets continue testing by asking about the recent event and Azure open AI determines the bing seach function and responds based on bing search results with article url

  <img width="1078" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/d900c53e-faeb-4e8f-ac00-9a9720b40af7">


  <img width="920" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/2dbd2656-6570-469c-bc9b-208363b91b85">

- Lets continue testing by asking about a stock result and also ask about the headquarter location of the stock. Azure Open AI calls the Get stock price function to get the current  price and generates the location 


  <img width="1140" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/7fc0b8b7-b1a4-4e6f-bbe7-94e226ba4b0e">

  <img width="770" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/fda62fbb-14ac-4c33-a107-c1403e159298">

- Lets ask about the Time in the location from previous result , Azure open AI responds by calling the Get time function

   <img width="1114" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/8b72df08-68f6-454b-aff1-b0db097eea35">


   <img width="764" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/687bd7cf-051e-46d7-b637-40dae614931a">

- Lets test by asking about weather and Azure Open AI calls getweather function and responds with weather

  <img width="1111" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/72d2a135-9abe-46c6-b097-2868e45f4fee">

  <img width="717" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/068e3313-f11d-4c36-a86b-d53ab907225a">



### Inspiration & References
  
- Please refer [here](https://github.com/microsoft/aoai-virtual-assistant/) for Microsoft open source Virtual Assistant Accelerator which provides entire bot composer solution for Azure Open AI connected Bot
