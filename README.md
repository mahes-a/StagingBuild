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

- The bot maintains conversation history to be passed to the model and ensures conversation history is mainatained only for configurable turns to prevent hitting token limits, the state management in the bot is a sample and may not be suited for Production workloads 

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

**Create an Azure Machine learning real-time inference endopint that hosts Llama-2-7b-chat model with built in Azure AI Content Safety**
