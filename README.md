# Build a Chatbot for your documents using: Amazon Bedrock, Amazon Lex and Amazon Kendra

## Overview

In this project, we present an architectural design illustrating the creation of a Generative AI chatbot fueled by data extracted from documents. The solution is orchestrated through an AWS Lambda function, seamlessly integrating Amazon Bedrock, Amazon Kendra, and Amazon Lex. Amazon Bedrock, coupled with the adaptable Amazon Titan model (which can easily be substituted with any other Amazon Bedrock model), empowers the chatbot with Generative AI capabilities. Amazon Kendra plays a pivotal role in document search, enabling the solution to extract relevant context for user queries. Finally, Amazon Lex is employed to deliver a conversational interface to users, complete with the retention of previous user questions. This retention feature becomes particularly crucial when users pose follow-up questions to the bot.

### Architecture 

![architecture (1)](https://github.com/cupumelody/build-a-chatbot/assets/145847069/37960f1c-469f-416c-a129-2746435ffec6)


## User request flow

Let’s go through the request flow to understand what happens at each step, as shown in architecture;

1. If Lex has an intent to address the question then Lex will handle the response.
2. If Lex can't handle the question then it will be sent to the Lambda function.
3. The Lambda function leverages the ConversationalRetrievalChain to call the Bedrock endpoint to formulate a prompt based on the question and conversation history.
4. The Lambda function sends that prompt to the Kendra retrieve API, which returns a set of context passages.
5. The Lambda function sends the prompt along with the context to the Bedrock endpoint to generate a response.
6. The Lambda sends the response along with message history back to the user via Lex. Lex stores the message history in a session.

**_NOTE:_**  In steps 3 and 5 we will use two different LLMs via the Bedrock service. This is an impactful feature of both Bedrock and LangChain that allow customers to make cost/ performance decisions on which model to use for various use cases.

## Getting Started

### Prerequisites

For this solution, you need the following prerequisites:

* The [AWS Command Line Interface (CLI)](https://aws.amazon.com/cli/) installed and [configured for use](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).
* The [Docker CLI](https://docs.docker.com/get-docker) installed for building the lambda container image.
* Python 3.9 or later, to package Python code for Lambda
   `Note`: I recommend that you use a [virtual environment](https://docs.python.org/3.9/library/venv.html) or [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) to isolate the solution from the rest of your Python environment.
* An IAM role or user with sufficient permissions to create an Amazon Kendra Index, invoke Amazon Bedrock, create an IAM Role and Policy, and create a Lambda function.

### Deploy the sample solution

From within the directory where you've downloaded this sample code from GitHub, first run the following command to initialize the environment and create an ECR repository for our Lambda Function's image. Then run the next command to generate the cloudformation stack and deploy the resources.

`Note`: Ensure docker is running prior to running the commands below.

```bash
bash ./helper.sh init-env
 …
{
   "repository": {
      …
   }
}
```

```bash
bash ./helper.sh cf-create-stack
 …
Successfully created CloudFormation stack.
```

### Setup Amazon Kendra

Navigate to the Amazon S3 service in the AWS console. 
<br>
Once there, find the bucket that was created via the CloudFormation template. 
<br>
The bucket name should begin with `llm-lex-chatbot`. 
<br>
Then, upload the text documents that you would like to test with this RAG chatbot.

With the documents uploaded, we can now sync the data with Amazon Kendra. 
<br>
Navigate to the Amazon Kendra service in the AWS console. 
<br>
Select `Indexes` from the menu on the left. Then click on the index that was created from the CloudFormation template. 
<br>
The index name should be `KendraChatbotIndex`. 
<br>
After the index is selected, navigate to `Data source` on the left menu bar. Select the data source named `KendraChatbotIndexDataSource` by clicking the radio button to the left of the item. 
<br>
Once it is selected then click the `Sync now` button and wait for the sync to complete (depending on the number of documents you've uploaded to S3 this process can take several minutes).

![kendraimage](https://github.com/cupumelody/build-a-chatbot/assets/145847069/1986d7ba-f8e1-41a1-b3ac-544a12082de4)


### Setup Model Access in Amazon Bedrock

If this is your first time using Amazon Bedrock in the AWS account you're deploying this bot in, you will first need to request access to Bedrock hosted models. 

Start by making sure you're in us-east-1 (or the appropriate region if you changed these config parameters for the deployment).
<br>
And then navigate to the Bedrock console. 
<br>
A welcome prompt will pop up to request model access. 
<br>
If the prompt has already been dismissed you can navigate to the `Base models` page via the left navigation bar, and then click the `Request model access` button in the blue banner at the top of the page.

<img width="1256" alt="requestmodel" src="https://github.com/cupumelody/build-a-chatbot/assets/145847069/06dc8928-72a6-4453-9f1d-6c31880c2bad">


Within the Model access page click the `Edit` button at the top right.
<br>
Select the model(s) that you would like access to, or use the select all option at the top left to enable access to all Bedrock models (note that this will not enable models that are still in Preview).  

<img width="1256" alt="modelaccessselect" src="https://github.com/cupumelody/build-a-chatbot/assets/145847069/7c29839d-a049-4d78-9f96-ae84ae040467">

Once your selections have been made, click the orange 'Save changes' button at the bottom right of the page and proceed to the next step.

<img width="1256" alt="savemodels" src="https://github.com/cupumelody/build-a-chatbot/assets/145847069/07055ef9-4841-4115-b972-392f4b52c414">


### Test via the Amazon Lex interface

At this point we have a fully functioning chatbot. 
<br>
Navigate to the Amazon Lex service in the AWS console to test the functionality. 
<br>
Once there, select the `Chatbot` bot that the CloudFormation template created. 
<br>
In the menu on the left, select `English` under `All languages`. 
<br>
At the top of the screen you will see a `Version` dropdown. 
<br>
Select `Version 1` from the dropdown, as the Draft version isn't associated with our Alias and Lambda function. 
<br>
Confirm the Alias of `ChatbotTestAlias` which is associated with our lambda function. Use the chat interface to test the bot. 
<br>
Remember that this bot leverages retrieval augmented generation and your documents to answer questions. 
<br>
So if you have not uploaded data that is pertinent to the questions you ask (and thus the context in your document retrieval does not answer the question), then the bot will return a response of "I don't know" or similar.

Also, the example includes a Lex intent to reset a password. 
<br>
So try asking a question like, "I forgot my password, can you help?". 
<br>
This intent will be served through Lex without the need to interact with the LLM via the fallback intent. 
<br>
This is useful when you want to serve a fixed set of user intents, independent of using Generative AI.

![leximage](https://github.com/cupumelody/build-a-chatbot/assets/145847069/caf58c66-7e9b-42fd-91c0-14a472f591ed)


## Cleanup

First delete all files in the WebDataS3Bucket that was created for this sample. 
<br>
Then you can delete all associated AWS resources via this command:

```bash
bash ./helper.sh cf-delete-stack
 …
```

👍
