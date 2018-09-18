# Streamsets (ChatBox API)

To provide a pipeline services to receive APIs, perform lookup and send back results to rabbit mq.

  - Get API request
  - Do Checking on Required Field
  - Perform Look up
  - Send back expected results to Rabbit MQ

# Pre-requisites
  - Streamsets (Current Version : 2.7.1.1, Standalone)
  - RabbitMq  (Current Version : 3.6.0 , Standalone)

### Installation

1. Install Streamsets in Docker
    - Install [streamsets](https://streamsets.com/opensource/) services following the link below.

        ```sh
        $ docker run --restart on-failure -p [port_no]:18630 -d -name streamsets-dc streamsets/datacollector
        ```
    
    - Browse through the web http:[ip-address]:[port_no] to check whether its accessible.
    Default username and password will be admin. 
    **Please kindly remove or change the default username and password to avoid unauthorized access.**
    
    - Things to take note : RabbitMq library and JDBC library are not installed in streamsets by default. To install, you may follow the guide provided here.
        - [RabbitMq](https://streamsets.com/documentation/datacollector/latest/help/#Installation/AddtionalStageLibs.html#concept_fb2_qmn_bz)
        - [JDBC](https://streamsets.com/documentation/datacollector/2.6.0.0/help/#Configuration/ExternalLibs.html#concept_pdv_qlw_ft)


    

2. Install [Rabbit Mq](https://www.rabbitmq.com/download.html)


### StreamSet Components
There are several components that been used for this ChatBot Streamsets. Click on the components belows for more information.
 - [HTTP Server](https://streamsets.com/documentation/datacollector/latest/help/index.html#Origins/HTTPServer.html)
 - [Jython Processing](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/Jython.html)
 - [Stream Selector](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/StreamSelector.html)
 - [JDBC Look Up](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/JDBCLookup.html)
 - [Field Remover](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/FieldRemover.html)
 - [RabbitMQ Producer](https://streamsets.com/documentation/datacollector/latest/help/index.html#Destinations/RabbitMQ.html)

### Process Flow
Below are the detail explanation of this pipeline.
In streamsets pipeline, its compulsory to only have **ONE** origin but multiple destination. In this instance, HTTP Server component will be the origin as the start point of API request.
The ChatBot Pipeline can be downloaded [here](http://manufacturing-giant.oss-ap-southeast-1.aliyuncs.com/esb_pipeline/chatbot/esb_Chat%20Bot.json).

**Overview of Pipeline**
[![N|Solid](http://manufacturing-giant.oss-ap-southeast-1.aliyuncs.com/esb_pipeline/chatbot/esb_ChatBot.PNG)](https://nodesource.com/products/nsolid)


1.  API Request
    Component : [HTTP Server](https://streamsets.com/documentation/datacollector/latest/help/index.html#Origins/HTTPServer.html)
    HTTP Server Origin function as an HTTP endpoint and received the api request 
from HTTP POST request. There are serveral information that you need to configure in HTTP Server Origin.
    **Http Url** : http://47.88.175.8:8088
    This url are pointing to streamsets development server.

    **Header** :
        X-SDC-APPLICATION-ID : {sdc_applicationId}
        Content-Type : application/json

    **Body**:
    ``` json 
    {
     	"Session": "{sessionId}",
     	"ResponseId": "{responseId}",
     	"QueryResult": {
     		"Parameters": {},
     		"Intent": {
     			"Name": "{name}",
     			"DisplayName": "willy-intent"
     		},
     		"IntentDetectionConfidence": 1.0,
     		"DiagnosticInfo": {}
     	},
     	"requestUTCDatetime" : "2017-12-18T15:04:00.000Z"
    }
    ```
    
    | Name | Description | Example Values | Required |
    | ------ | ------ | ------ | ------ |
    | Http Url | Streamsets api that will be call. Currently pointing to Development Streamsets Server | http://47.88.175.8:8088 | Yes |
    | X-SDC-APPLICATION-ID | (Header) A key to call to streamsets apis. Its been pre-defined in pipeline. API requests Will get error without specifiy this key in header. This is pre-defined in streamsets pipeline and its hardcoded.  | 8ca483c5-ca2c-4c5a-8246-4cb2ebae0a5b | Yes
    | Content-Type | (Header)  | application/json | Yes|
    | Session | (Body) This values passed in by Chat Bot Engine. There are no specific format on this.  | projects/dialogflow-rnd/agent/sessions/4da256d0-971c-4631-a499-7192d8053b79 | Yes
    | ResponseId | (Body) This values passed in by Chat Bot Engine.  There are no specific format on this.  | 1739aef9-e5a2-4e51-b849-d15ec2f2fa99 | Yes |
    | QueryResults/Parameters | (Body) This values passed in by Chat Bot Engine. This parameters will used for condition processing. | {"packageCode" : "PKG_01"} | No |
    | requestUTCDatetime | (Body) To get current API request datetime in UTC Format. To pass in the current datetime when this api been triggered. To store this as logging purpose. | 2017-12-18T15:04:00.000Z | Yes|
    
    **Example Curl Command:**
    ```sh
    curl -i \
	-H "Content-Type: application/json" \
	-H "X-SDC-APPLICATION-ID:8ca483c5-ca2c-4c5a-8246-4cb2ebae0a5b" \
	-X POST -d '{"Session": "session1","ResponseId": "response1","QueryResult": {"Parameters": {},"Intent": {"Name": "intent-Name","DisplayName": "intent"},"IntentDetectionConfidence": 1.0,"DiagnosticInfo": {}},"requestUTCDatetime" : "2017-12-18T15:04:00.000Z"}' \
	http://47.88.175.8:8008
    ```
    
2. Required Field Checking
    Components : [Jython Processing](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/Jython.html)
    This steps is using Jython Processor in Streamsets to custom write some checking function to do some required field checking.

3. Required Field Condition Splitting
    Components : [Stream Selector](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/StreamSelector.html)
    This Steps are to check the status Code that been generated by Step 2 - Required Field Checking. This processor will split the records to 2 different streamline based on "StatusCode". If the statusCode is equal to 200, it will flow into streamline 1, where it will continue to further processing or data. Else, the data will flow to streamline 2 to perform the next steps.

    **Streamline 1 : Process that will be run through by streamsets if all required field are fullfilled**
    **Streamline 2 : Process that will be run through by streamsets if eihter one required field are not fullfilled**

4. Streamline 1 : Process Query
    Components : [Jython Processing](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/Jython.html)
    Using Jython Processor , this components are to further to process the sql query based on the APIs content that been sent. For this pipeline, there are some required field to process the query selection. Currently, only do some simple query inside, where if the required object is not exist, system will generate a simple query and generate a variable ${lookup_query} to be used on next steps. This shall further enhance the functionality when there is more requirements.

5. Streamline 1 : LookUp Query 
    Components : [JDBC Look Up](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/JDBCLookup.html)
   This process will use the variable ${lookup_query} that been generated out from Streamline 1 : Process Query to process the JDBC Query in order to get the results from the desired database. 

6. Streamline 2 : Reformat Results
    Components : [Field Remover](https://streamsets.com/documentation/datacollector/latest/help/index.html#Processors/FieldRemover.html)
    To filter or To retain the desired object or data that needed to be further processed.

7. Rabbit MQ Producer
    Components : [RabbitMQ Producer](https://streamsets.com/documentation/datacollector/latest/help/index.html#Destinations/RabbitMQ.html)
    RabbitMQ Producer writes AMQP messages to a single RabbitMQ queue. All the desired object will be written into rabbit MQ for another process. Currently Streamsets will provide 2 type of results to user.

    **If all required field is fullfil**, these are the expected results:
    ``` json
    {
    	"Session": "projects/dialogflow-rnd/agent/sessions/4da256d0-971c-4631-a499-7192d8053b79",
    	"ResponseId": "1739aef9-e5a2-4e51-b849-d15ec2f2fa99",
    	"result": {
    		"statusCode": 200,
    		"templateId": "1",
    		"imageId": "2"
    	}
    }
    ```
    
    **If required field are not fullfil**, these are the expected results:
    ``` json
    {
    	"Session": "projects/dialogflow-rnd/agent/sessions/4da256d0-971c-4631-a499-7192d8053b79",
    	"result": {
    		"description": "ResponseId not found",
    		"statusCode": 400
    	}
    }
    ```
    
    User may consume the message using the below configuration.
    | Properties Name | Value |
    | ------ | ------ |
    | Http (Public) | 47.74.148.132 |
    | Http (Private) | 172.18.101.116 |
    | Port | 15672 (Web) , 5672 (Api) |
    | Username | chatbot |
    | password | chatbot |
    | queue | fnb |
    | format | json |


### Questions & Answers
1. X-SDC-APPLICATION-ID (Header) A key to call to streamsets apis. Its been pre-defined in pipeline. API requests Will get error without specifiy this key in header.
    Question : How do i get this?
    Answer :
    ```
    X-SDC-APPLICATION-ID is pre-defined in streamsets pipeline configuration. Its currently been hardcoded. The value can be changed in pipeline level, provided every services that been called to this api shall change the value of this key.
    ```
2. RequestUTCDatetime (Body) To get current API request datetime in UTC Format 2017-12-18T15:04:00.000Z
    Question : Why is this required? If invoked at 12:00 and send Over 13:00 , how will that affect the flow?
    Answer:
    ```
    It wont affect the flow, just thinking to use a request Datetime to do logging
    ```
3.  Session Id "projects/dialogflow-rnd/agent/sessions/4da256d0-971c-4631-a499-7192d8053b79"
    Question : Is this constuct is a must or just a key?
    Answer:
    ```
    Streamset treat it as a normal required object. Does not have any format restriction. Streamsets shall pass back the same value back to rabbitMq.
    ```
    
4. "DisplayName": "willy-intent"
    Question : Can change it to "my-intent"?
    Answer:
    ```
    Based on current user case, there is no issues in changing the display name
    ```

5.  Streamline 1 : Process Query
    Streamline 1 : LookUp Query
    Question : what  does streamline 1 means?
    Answer:
    ```
    Streamline 1 : Process that will be run through by streamsets if all required field are fullfilled
    Streamline is a temporary name been given
    ```

6.  How do i get my parameter out?
    What is templateId and ImageId?
    Answer:
    ```
    Currently in streamsets pipeline, there is one components can be used to retain the object needed.
    In this pipeline, only retain the result object, Session, and responseId.
    If parameters object is needed,  the component can be removed to retain only needed object.
    ```

7.  is the rabbit hard coded into the flow?
    if its production and non production how does it change?
    Answer:
    ```
    RabbitMQ info is configured inside pipeline.
    We can have one option to segregate the production and non production which is to take out the configuration as parameters and start the pipeline using parameters (this need further test, i tested one simple one before)
    ```

8. Can the original request been passed back as it is with the newly responses?
    Answer:
    ```
    Ya, can.
    ```

9. Is there a docker compose file.
    Answer:
    ```
    There is one docker file that been used for Aliyun Deployment. You may check the files from http://manufacturing-giant.oss-ap-southeast-1.aliyuncs.com/esb_general/streamsets_2.7.1.1.yml
    ```
    
10. if i want to invoke another pipeline how do i do it?
    i have 3 different pipeline
    how do i invoke it?
    is the splitting done in stream set?
    is all the logic done inside there?
    if its handle there,will it be maintainable if i have 100 logics branches?
   Answer:
    ```
    Streamsets contain a components to split the condition based on your configuration.
    to handle 100 logics, technically its workable , but will very hard to maintain.
    Its possible to invoke from pipeline to another pipeline.
    But one pipeline will consume one port. as its act like APIs request.
    ```


[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)


   [dill]: <https://github.com/joemccann/dillinger>
