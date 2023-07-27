# A Step-by-Step Guide to Build a Llama 2 on Azure powered Teams Chat Bot with built-in Azure AI Content Safety #

The Llama-2-7b-chat model on Azure powered chat bot, utilizing advanced language models, brings a range of capabilities to enhance communication and collaboration within Microsoft Teams. With its natural language processing abilities, Llama 2 chat models can understand and respond to user queries and requests, providing relevant information and assistance.

By integrating Llama 2 chat models with Microsoft Teams, users can easily access the chat bot directly from their Teams interface. This means they can engage with LLM without leaving the Teams platform, saving time and effort. They can ask questions, seek guidance, or request information, and Llama will provide prompt and accurate responses, helping to streamline workflows and improve productivity.

Additionally, the integration with Azure AI Content Safety ensures that conversations  remain secure and free from inappropriate or harmful content. This powerful combination ensures that users can communicate and collaborate with peace of mind, knowing that the Azure AI content safety is actively monitoring and filtering any potentially harmful or offensive content.

Overall, the integration of Llama-2-7b-chat model on Azure with Teams, along with the inclusion of Azure AI Content Safety, creates a comprehensive solution that not only enhances productivity and collaboration but also prioritizes user safety and security. This makes it an ideal choice for businesses looking for efficient and secure communication tools to support their teams' workflows.

## High-level Architecture

 <img width="917" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/9da4c54a-d58d-4637-bb31-a0889f5709bd">

## Prerequisites

- Azure subscription with Azure Machine Learning resource
- Deploying Llama 2 models requires GPU compute of V100 / A100 SKUs. You can view and request AzureML compute quota [here](https://ml.azure.com/quota).
- Install Bot Composer [here](https://learn.microsoft.com/en-us/composer/install-composer?tabs=windows).
- Teams for work or school , You can downlaod from [here](https://go.microsoft.com/fwlink/?linkid=2187327&Lmsrc=groupChatMarketingPageWeb&Cmpid=directDownloadWin64&clcid=0x409&culture=en-us&country=us)

## Things to Note 

- The bot maintains conversation history to be passed to the model and ensures conversation history is mainatained only for configurable turns to prevent hitting token limits in its memory, the state management in the bot is a sample and may not be suited for Production workloads 

- Please note that this tutorial is intended for explorative and illustrative purposes only. It is meant to inspire ideas and should not be taken as prescriptive advice. Any implementation of the techniques described in this tutorial as part of your application should be thoroughly validated and tested to ensure accuracy, validity, compatibility with your specific use case and technical environment.

- Please be aware that the  prompt instructions shown are not intended to be a correct and complete representation of the prompts that should be used with your applications in production. They are provided for informational purposes only and may not be suitable for all use cases. It is important to carefully consider your specific requirements and design appropriate prompts that meet your users' needs and expectations.

- While LLMs have tremendous potential across many industries and use cases, it is essential to ensure that they are built in a safe and responsible manner. This includes taking steps to mitigate potential risks and ensure that the model will not cause harm to users or result in reputational damage to organizations. It is important to carefully consider the ethical implications of LLMs and to develop appropriate safeguards to protect against potential harms.

## Technical Flow

- Azure Bot's Teams channel is enabled and deployed in Teams app. 
  
- The Teams channel Bot makes HTTP Post requests with user prompt history to the Azure Machine learning real-time inference endopints  
  
- Azure Machine learning real-time inference endopints hosts the Llama-2-7b-chat model with built in Azure AI Content Safety monitoring
  
- The User prompts are monitored , validated and filtered by  Azure AI Content Safety resource
  
- Safe Prompts are sent to Llama-2-7b-chat model and response from Llama-2-7b-chat model is also monitored and validated by the Azure AI Content Safety resource
  
- Safe Responses are sent back to Azure Bot via the Azure Machine learning real-time inference endopints

## Steps

*As noted in the Prerequisites sections Deploying Llama 2 models requires GPU compute of V100 / A100 SKUs , Please ensure you have quota available for v100 or A100* 

**Create an Azure Machine learning real-time inference endopint that hosts Llama-2-7b-chat model with built in Azure AI Content Safety**

##### Deploying Llama-2-7b-chat model with Azure AI Content Safety

- Login into your ML Studio [here](https://ml.azure.com/)

   <img width="998" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/b905bda5-79df-41cf-b444-28beb0543953">

- Select your workspace , Select "Model Catalog" menu from left and click on View models in Introducing Llama 2 section

   <img width="1128" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/e443154b-f7aa-4852-adeb-59b9d92c5c83">

- Choose Llama-2-7b-chat model 

     <img width="1111" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/96b62efa-e9b5-42a4-a2f9-40997fcba1df">


     <img width="563" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/67ed9385-7b78-4f12-bfb5-cb5b92768f63">

-  Click on Deploy -> Real-time endpoint option and ensure you chose Enable Azure AI Content Safety (Recommended) option and click proceed. Multiple layers of safety mechanism are recommended for LLM's for Responsible AI 

    <img width="512" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c901f8ba-86cb-4ae9-9909-7e7eb3985e7b">

    <img width="465" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/96d5e3a7-c7b5-41aa-941c-3cc4e2a70a21">

    
-  This will open a notebook

     <img width="998" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/5d13e2f4-c2a0-4d80-85f8-da9a114b7b83">

- Clone the notebook and open llama-safe-online-deployment.ipynb

     <img width="1019" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/6f7c1b10-1c54-46a4-bade-8b8a79d17e9d">

 -  Choose your compute , either your pre-created conmpute instance or serverless if you dont have compute instance created

-  Choose the model_name as "Llama-2-7b-chat" , update endpoint_name , deployment_name  if needed and  The sku_name would be the A100 or V100 GPU SKU , at present  it defaults to Standard_NC24s_v3 SKU and execute the notebook

    <img width="554" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/355f73d1-3f0d-47b2-a301-910f558096a5">


-  Uncomment and install dependencies

    <img width="534" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/dc5bf161-2344-4dd2-88aa-4432d6b7562e">

-  Run the Get Credential section

    <img width="766" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/3c8fe189-5599-452f-a3b3-525e93f089e8">

-  Enter your AML subscription_id , resource group and aml workspace name and execute

   <img width="625" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/88be6260-acf2-4748-bec8-f5e33853da2f">

   
- Kindly go through the Note section to undesrtand the  region to deploy the Azure AI Content Safety and Choose the region for deploying the content safety resource according to your needs and compliance laws , The default choses "east us" , Please change according to your needs

     <img width="994" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/e5a24d4f-3ec8-44e3-a182-274b8c24224e">

- Create Azure content safety resource
  
    <img width="749" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/8f16e7bf-28f2-42f4-ae0f-7220e0841b68">

    
- Optional , You can search for the content safety resource in the Azure Portal

     <img width="1169" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/33dd7224-cf94-4ec0-9287-b34c19f2e00a">

- Run 3.1 and 3.2 to create a endpoint for the Llama-2-7b-chat , this would take few minutes

  <img width="736" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/b0894558-35d6-4e29-b375-896248762550">

- Deploy the Llama 2 model with the content safety resource to the endpoint created in previous steps , this would take several minutes , wait for completion 
  
    <img width="698" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/7230fa15-a23c-4577-81d0-5c7384ecf10e">

- Once Successful , switch to the endpoints section and click on the Llama2 endpoint in the Real-time endpoint section

    <img width="877" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/cfd87fef-ec4b-4c86-ac3c-a50f4c5b6564">

    <img width="937" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/5ea8d98c-7891-48bc-837c-95317fee6867">


- Lets select the test tab and provide a input json to test the Llama-2-7b-chat deployment

   <img width="881" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/c62aa452-5c1f-4667-aa12-a0618c82c49d">

- Optional Sample inputs and ouputs are available in the Llama-2-7b-chat model page , to navigate to the model page select your workspace , Select "Model Catalog" menu from left and click on View models in Introducing Llama 2 section Choose Llama-2-7b-chat model 

    <img width="563" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/52fefbac-d2e7-4e58-973a-124a95b1b2b1">

  
- In the endpoints section and click on the Llama2 endpoint in the Real-time endpoint section and select the Consume tab , Note down the Rest endpoint and Authentication key these will be used to make the request from the Bot
 
    <img width="856" alt="image" src="https://github.com/mahes-a/StagingBuild/assets/120069348/6132b03f-ea92-4145-963a-74160a358ff8">

**How to work with the Llama-2-7b-chat deployment models**

*Please be aware that the  prompt instructions shown are not intended to be a correct and complete representation of the prompts that should be used with your applications in production. They are provided for informational purposes only and may not be suitable for all use cases. It is important to carefully consider your specific requirements and design appropriate prompts that meet your users' needs and expectations.*

- Below is the sample chat history that must be sent to the Llama-2-7b-chat model , In this tutorial we will see an sample approach to create this Json in Bot composer (approach not production scale) , as see below the user question followed by model answers needs to be maitained and passed back to model , the model understabds context based on the chat history  
             
      
     
       {
        "input_data": {
          "input_string": [
            {
              "role": "system",
              "content": "You are an AI assistant that helps people find information.You are friendly and concise and provide only short factual answers"
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
              "content": "You are an AI assistant that helps people find information.You are friendly and concise and provide only short factual answers"
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
  
**Create an Azure Bot and make HTTP Post requests to real-time inference endopint that hosts Llama-2-7b-chat model with built in Azure AI Content Safety**
