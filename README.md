# A Step-by-Step Guide to Build a Versatile Multi-Topic Digital Assistant Chatbot powered by Azure Open AI Function Calling #

A Multi-Topic Digital Assistant Chatbot is an intelligent conversational AI solution built to cater to diverse subject matters and assist users. By harnessing the capabilities of natural language processing and machine learning, it comprehends user queries and offers pertinent information or executes tasks accordingly. This advanced chatbot is designed to seamlessly understand and address a broad spectrum of topics, providing valuable assistance to users with accuracy and efficiency.

Users  interact effortlessly through natural language input via Bot channels like Teams. The Function App acts as a bridge, connecting the Bot to Azure Open AI, sending functions and prompts. Azure Open AI processes user input , understands the user's query in depth, extracting the intended meaning and relevant details. This information is then used to provide specific function and parameters. The Function App executes the function, and the response is sent back to Azure Open AI. The Bot then delivers valuable insights and information to the user from the final response from Azure Open AI.

## High-level Architecture

 <img width="734" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/d790b4ad-dc05-4b31-85c4-cfd3dc9af48b">

## High-level Flow

The High level flow  involves the following steps:

- Users input queries using natural language through Bot channels such as Teams.

- The Bot communicates with a Function App, which then communicates with Azure Open AI with the available functions and user prompts.

- Azure Open AI responds by providing the specific function to be executed, along with the parameters extracted from the user's input.

- The Function App executes the function, and the response is sent back to Azure Open AI.

- The Bot then responds to the user with the final natural language response received from Azure Open AI.
  


## Teams Screen Grab


 

# TBD #


## Prerequisites

- Azure subscription with access enabled for the Azure OpenAI service. You can request access [here](https://aka.ms/oaiapply). 
- Visual Studio Code
- Install Bot Composer [here](https://learn.microsoft.com/en-us/composer/install-composer?tabs=windows).
- Teams for work or school , You can download from [here](https://go.microsoft.com/fwlink/?linkid=2187327&Lmsrc=groupChatMarketingPageWeb&Cmpid=directDownloadWin64&clcid=0x409&culture=en-us&country=us)
- Bing search resource
- Weather API from [here](https://rapidapi.com/apishub/api/yahoo-weather5)
- Stock API from [here](https://rapidapi.com/alphavantage/api/alpha-vantage)
-  Car Sales dataset from [here](https://www.carpro.com/blog/full-year-2022-national-auto-sales-by-brand) in a CSV format

## Things to Note 

- When using function calling to integrate language models with other tools and systems, like any AI system, it is essential to acknowledge and address the potential risks involved. It is crucial to have a clear understanding of these risks and take appropriate measures to ensure responsible use of the capabilities. Refer [here](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/function-calling#using-function-calling-responsibly) for responsible function calling
  
- Function calling is in preview and supported only from OPENAI_API_VERSION of "2023-07-01-preview"
  
-  The bot maintains conversation history to be passed to the model and ensures conversation history is maintained only for pre-configured number of conversation turns to prevent hitting token limits in its memory, the state management in the bot is a sample and may not be suited for Production workloads 

- Please note that this tutorial is intended for explorative and illustrative purposes only. It is meant to inspire ideas and should not be taken as prescriptive advice. Any implementation of the techniques described in this tutorial as part of your application should be thoroughly validated and evaluated to ensure accuracy, validity, compatibility with your specific use case and technical environment.

- Please be aware that the prompt instructions shown are not intended to be a correct and complete representation of the prompts that should be used with your applications in production. They are provided for informational purposes only and may not be suitable for all use cases. It is important to carefully consider your specific requirements and design appropriate prompts that meet your users' needs and expectations.

- While LLMs have tremendous potential across many industries and use cases, it is essential to ensure that they are built in a safe and responsible manner. This includes taking steps to mitigate potential risks and ensure that the model will not cause harm to users or result in reputational damage to organizations. It is important to carefully consider the ethical implications of LLMs and to develop appropriate safeguards to protect against potential harms.
  
- The tutorial uses developer freemium sample APIs as they are a convenient way to experiment and test integration with various services but  it is crucial to thoroughly validate and test the free sample API before integrating it into your application. This includes checking for error handling, response validation, and understanding any potential limitations or restrictions.
  
- There are no warranties of any kind, express or implied, about the completeness, accuracy, reliability, suitability or availability with respect to the information provided. 


## Steps


**Function App to serve insights**

##### Creating Function App 

- The Sample uses Python Azure Functions developed in Visual Studio Code , Please refer [here](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=python) on how to create , add triggers and deploy azure functions via VS code

##### Function App Configurations 
- Configure the keys and endpoints in local.settings.json

  
         {
           "IsEncrypted": false,
           "Values": {
             "AzureWebJobsStorage": "",
             "FUNCTIONS_WORKER_RUNTIME": "python",
             "OPEN_AI_DEPLOYMENT": "",
             "OPENAI_API_BASE": "https://youropeniresourcename.openai.azure.com",
             "OPENAI_API_VERSION": "2023-07-01-preview",
             "OPENAI_API_KEY": "",
             "bing_subscription_key": "",
             "bing_search_endpoint": "https://api.bing.microsoft.com/v7.0/search",
             "vantage_key": "",
             "vantage_url": "",
             "yahoo_weather_url": "",
             "yahoo_weather_key": ""
           }
         }


- To fill in configurations follow below

-   In Azure Open AI Create a GPT 3.5 deployment 

   <img width="864" alt="image" src="https://github.com/mahes-a/Wingman/assets/120069348/dd9b5f1e-7ee2-47aa-9ade-bd2c750ffa62">

 - Note down the OPENAI_API_KEY and endpoint, they can be retrieved from Azure Portal Azure OpenAI resource under Keys and Endpoint section
    
      <img width="620" alt="image" src="https://github.com/mahes-a/Wingman/assets/120069348/646b0ebf-bf99-45cd-bc90-ee0fcf384b1c">

  - From your Bing Search resource in Azure Keys and Endpoints section , copy the keys

      <img width="947" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/be07fc54-3099-42db-8792-74d5da16a8c1">

  - Function calling is in preview and supported only from "OPENAI_API_VERSION" of "2023-07-01-preview"
    
  - For Stock and Weather go to the developer api links listed in Prerequisites section
    
  - Copy the car sales dataset in csv format in to the Function App solution
    
  - Add packages to the requirements.txt file-
              azure-functions
              openai
              pytz
              pandas
    
    <img width="497" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/2f5fa787-8560-4e3d-b010-10323e7ecac7">

##### Creating the API and arithmetic functions

- In your function App add all the functions with API calls
  
- We have below  functions
  -  Get current time for a location using pytz
  -  Basic Arithmetic calculator
  -  Return top 1 search result from bing search (modify as needed to return more)
  -  Use Pandas dataframe to filter data from Car sales dataset
  -  Get Current stock price using the freemium developer api
  -  Get Current weather  using the freemium developer api
  
*The functions provided in this sample are modified to return simple strings from Json results without robust exception handling , Please esnure to thoroughly validate and test the freemium sample API before integrating it into your application. This includes checking for error handling, response validation, and understanding any potential limitations or restrictions and more* 

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

**How to work with the Llama-2-7b-chat deployment models**

*Please be aware that the prompt instructions shown are not intended to be a correct and complete representation of the prompts that should be used with your applications in production. They are provided for informational purposes only and may not be suitable for all use cases. It is important to carefully consider your specific requirements and design appropriate prompts that meet your users' needs and expectations.*

- Below is the sample chat history that must be sent to the Llama-2-7b-chat model , In this tutorial we will see an sample approach to create this Json in Bot composer (approach not production scale) , as see below the user question followed by model answers needs to be maintained and passed back to model , the model understands context based on the chat history  
             
- The system role/message is optional, but it's recommended to at least include a basic one to get the best results.     
     
       {
        "input_data": {
          "input_string": [
            {
              "role": "system",
              "content": "You are an AI assistant that helps people find information. You are friendly and concise and provide only short factual answers"
            },
            {
              "role": "user",
              "content": "I am going to Paris, what should I see?"
            },
            {
              "role": "assistant",
              "content": "Paris, the capital of France, is known for its stunning architecture, art museums, historical landmarks, and romantic atmosphere. Here are some of the top attractions to see in Paris:\n\n1. The Eiffel Tower: The iconic Eiffel Tower is one of the most recognizable landmarks in the world and offers breathtaking views of the city.\n2. The Louvre Museum: The Louvre is one of the world's largest and most famous museums, housing an impressive collection of art and artifacts, including the Mona Lisa.\n3. Notre-Dame Cathedral: This beautiful cathedral is one of the most famous landmarks in Paris and is known for its Gothic architecture and stunning stained glass windows.\n\nThese are just a few of the many attractions that Paris has to offer. With so much to see and do, it's no wonder that Paris is one of the most popular tourist destinations in the world."
            },
            {
              "role": "user",
              "content": "What is so great about #1?"
            },
            {
              "role": "assistant",
              "content": "The Eiffel Tower is considered one of the greatest engineering feats of the 19th century and is an iconic symbol of Paris and France. Here are some reasons why it's so great:\n\n1. Unique Design: The Eiffel Tower's unique design, with its lattice-like structure and iron beams, makes it a striking and recognizable landmark.\n2. Breathtaking Views: The Eiffel Tower offers panoramic views of the city of Paris and its surroundings, making it a popular spot for tourists and locals alike.\n3. Historical Significance: The Eiffel Tower was built for the 1889 World's Fair and was initially intended to be a temporary structure. However, it has become a permanent fixture in Paris and a symbol of French culture and engineering.\n4. "
            },
            {
              "role": "user",
              "content": "What is the tallest building in the world?"
            },
            {
              "role": "assistant",
              "content": "As of 2021, the Burj Khalifa in Dubai, United Arab Emirates is the tallest building in the world, standing at a height of 828 meters (2,722 feet). It was completed in 2010 and has 163 floors. The Burj Khalifa is not only the tallest building in the world but also holds several other records, such as the highest occupied floor, highest outdoor observation deck, elevator with the longest travel distance, and the tallest freestanding structure in the world."
            },
            {
              "role": "user",
              "content": "and in Africa?"
            },
            {
              "role": "assistant",
              "content": "In Africa, the tallest building is the Carlton Centre, located in Johannesburg, South Africa. It stands at a height of 50 floors and 223 meters (730 feet). The CarltonDefault Centre was completed in 1973 and was the tallest building in Africa for many years until the construction of the Leonardo, a 55-story skyscraper in Sandton, Johannesburg, which was completed in 2019 and stands at a height of 230 meters (755 feet). Other notable tall buildings in Africa include the Ponte City Apartments in Johannesburg, the John Hancock Center in Lagos, Nigeria, and the Alpha II Building in Abidjan, Ivory Coast"
            },
            {
              "role": "user",
              "content": "and in Europe?"
            }
          ],
          "parameters": {
            "max_length": 2048,
            "temperature": 0.6,
            "top_p": 0.9,
            "do_sample": true,
            "max_new_tokens": 256
              }
             }
           }

- If the conversation history is too long we would exceed the maximum token limit and would result in errors , In the bot composer we would remove the oldest User question and assitant answers whenever a configured threshold is reached , this would prevent us from hitting the token limit . After removing the oldest question and answers the above Json would look like below

         {
        "input_data": {
          "input_string": [
            {
              "role": "system",
              "content": "You are an AI assistant that helps people find information. You are friendly and concise and provide only short factual answers"
            },
            
            {
              "role": "user",
              "content": "What is the tallest building in the world?"
            },
            {
              "role": "assistant",
              "content": "As of 2021, the Burj Khalifa in Dubai, United Arab Emirates is the tallest building in the world, standing at a height of 828 meters (2,722 feet). It was completed in 2010 and has 163 floors. The Burj Khalifa is not only the tallest building in the world but also holds several other records, such as the highest occupied floor, highest outdoor observation deck, elevator with the longest travel distance, and the tallest freestanding structure in the world."
            },
            {
              "role": "user",
              "content": "and in Africa?"
            },
            {
              "role": "assistant",
              "content": "In Africa, the tallest building is the Carlton Centre, located in Johannesburg, South Africa. It stands at a height of 50 floors and 223 meters (730 feet). The CarltonDefault Centre was completed in 1973 and was the tallest building in Africa for many years until the construction of the Leonardo, a 55-story skyscraper in Sandton, Johannesburg, which was completed in 2019 and stands at a height of 230 meters (755 feet). Other notable tall buildings in Africa include the Ponte City Apartments in Johannesburg, the John Hancock Center in Lagos, Nigeria, and the Alpha II Building in Abidjan, Ivory Coast"
            },
            {
              "role": "user",
              "content": "and in Europe?"
            }
          ],
          "parameters": {
            "max_length": 2048,
            "temperature": 0.6,
            "top_p": 0.9,
            "do_sample": true,
            "max_new_tokens": 256
          }
        }
      }
  
**Create an Azure Bot and make HTTP Post requests to real-time inference endpoint that hosts Llama-2-7b-chat model with built in Azure AI Content Safety**

- Open Bot Framework Composer , and create an Empty Bot and name it according to your needs and create the bot

    <img width="985" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/f37bd7be-9831-4f84-a574-2d1d7bc9f969">


    <img width="754" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c97f207c-f052-49a9-b2c7-b78134032d04">


- From the bot project (not the root solution) add a trigger and add message received activity  , this can be optional and entire bot can be designed in the "Unknown Intent" trigger too

  <img width="646" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/60e67404-9e5d-4a1a-b4bb-86534cc9320c">
  

  <img width="368" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/39ca1e4d-cf97-40b2-9a15-87ad0d0bc84e">


  <img width="821" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/b1d6ffb4-28a0-41f8-976b-c356e3d6338a">

- Click open the configure icon and choose advanced settings view (json)
  
    <img width="1192" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/8f9fa79f-d64e-4bfc-8f1a-4e7bf75c0430">

- Add an api section in the config and fill below parameters , the URL , key and deployment name can be retrieved from the consume section of AML deployment 

       "api": {
        "AML_Llama_Inference_Url": "",
        "AML_Llama_Inference_Key": "",
        "Past_Message_Count": 8,
        "AML_Llama_Deployment_Name": ""
      },

    <img width="497" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/eb6d0e82-e1d8-4b99-9d60-05d14187e25b">

  *Please note the tutorial uses conversation memory scope throught the composer for simplicity and brevity , for your production use case ,use the most apt memory scope , refer [here](https://learn.microsoft.com/en-us/composer/concept-memory?tabs=v2x) for memory scopes and their usage , we build a json using string manipulations and array for simplicity and brevity and we donot use any custom components in this tutorial the bot is designed only for happy path and doesnot handle error sceanrios*

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
 
         The json should be of format
                     {
           "input_data": {
           "input_string": [
           {
           "role": "system",
           "content": "You are an AI assistant that helps people find information. You are friendly and concise and provide only short factual answers"
           },
           
           {
           "role": "user",
           "content": "What is the tallest building in the world?"
           }
           ],
           "parameters": {
           "max_length": 2048,
           "temperature": 0.6,
           "top_p": 0.9,
           "do_sample": true,
           "max_new_tokens": 256
              }
            }
           }


        conversation.context.prefix  ' { "input_data": { "input_string": [ {"role": "system","content": "You are an AI assistant that helps people find information.You are friendly and concise and provide only short factual answers"}, '
 
         conversation.context.suffix  = '  ], "parameters": { "max_length": 3000, "temperature": 0.9, "top_p": 0.6, "do_sample": true, "max_new_tokens": 350 } } } '
 
         conversation.context.inputjson =json(concat(conversation.context.prefix,join(conversation.context.chathistory, ','),conversation.context.suffix))

    <img width="1004" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/db830189-d17f-4f12-b0f9-84c93c002edb">
 
 - Make an HTTP Post request by adding the send http request

             POST
             URL =settings.api.AML_Llama_Inference_Url
             Body =conversation.context.inputjson
             Headers
             Authorization =concat('Bearer ',settings.api.AML_Llama_Inference_Key)
             azureml-model-deployment =settings.api.AML_Llama_Deployment_Name
             Result property conversation.context.api_response
             content type application/json
             Response type Json 

  <img width="335" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/e43a4de0-2bd5-41ee-ae8b-7db975c03915">
  

   <img width="1009" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/3793a8a5-a56d-4c89-909a-fa68cfcff8ad">

- Lets validate the response from AML endpoint using if/else and if response is valid then display the answer from the model , if error display generic error message

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

### AI Content Safety monitoring from Content Safety studio 

- To validate our AI content Safety , login into Content Safety Studio [here](https://contentsafety.cognitive.azure.com/monitor)
  
- Monitor the model traffic against content safety categories

   <img width="1178" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/7f2556df-a423-4f95-8d18-e2bd829514d6">

 
### Inspiration & References

- Refer [here](https://techcommunity.microsoft.com/t5/ai-machine-learning-blog/introducing-llama-2-on-azure/ba-p/3881233) to know more about Llama 2 on Azure
  
- Please refer [here](https://github.com/microsoft/aoai-virtual-assistant/) for Microsoft open source Virtual Assistant Accelerator which provides entire bot composer solution for Azure Open AI connected Bot
