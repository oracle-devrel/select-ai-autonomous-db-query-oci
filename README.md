# select-ai-autonomous-db-query-oci

[![License: UPL](https://img.shields.io/badge/license-UPL-green)](https://img.shields.io/badge/license-UPL-green) [![Quality gate](https://sonarcloud.io/api/project_badges/quality_gate?project=oracle-devrel_select-ai-autonomous-db-query-oci)](https://sonarcloud.io/dashboard?id=oracle-devrel_select-ai-autonomous-db-query-oci)

# Integrating OCI Generative AI with Select AI and APEX to query data using natural language


## Additional Resources

- [Use Select AI to Generate SQL from Natural Language Prompts](https://blogs.oracle.com/machinelearning/post/introducing-natural-language-to-sql-generation-on-autonomous-database)
- [DBMS_CLOUD_AI Views](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/dbms-cloud-ai-views.html)


## Introduction

One remarkable capability of Large Language Models (LLMs) is their ability to generate code, including Structured Query Language (SQL) for databases. LLMs can understand the natural language question and generate a corresponding SQL query as an output. 

A large amount of enterprise data resides in databases, and the ability to query databases in natural language makes databases more accessible to users. Business users and data analysts can ask questions about data in natural language. For technical users, it reduces the time required to generate queries, simplifies query building, and helps to minimize or eliminate specialized SQL knowledge.

The Select AI feature allows Autonomous Database to use Large Language Models (LLMs) to convert user's input text into Oracle SQL.

In this blog, we will see how to integrate an OCI Generative AI model with Select AI to query data stored in an Autonomous database using natural language prompts.

## Architecture

The following diagram depicts the reference architecture.

![Architecture][7]


[7]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONT9EC5056C729042FBBE19E18E48D924D1/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

## Components

Following are the components used in the reference architecture.

[Autonomous Database][8] is a fully automated database service that makes it easy to develop and deploy application workloads, including Data Warehousing, Transaction Processing, JSON-centric Applications, and Oracle APEX Low Code Development.


[8]: https://www.oracle.com/in/autonomous-database/

[Select AI][9] This feature allows Autonomous Database to use Large Language Models (LLMs) to convert the user's input text into Oracle SQL. The database augments the user-specified prompt with database metadata to mitigate hallucinations from the LLM. The augmented prompt is then sent to the user-specified LLM to produce the query. This metadata may include schema definitions, table and column comments, and content available from the data dictionary and catalog. 


[9]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/sql-generation-ai-autonomous.html

[OCI Generative AI][10] is a fully managed service that provides a set of customizable large language models (LLMs) that cover a wide range of use cases, including writing assistance, summarization, analysis, and chat.


[10]: https://www.oracle.com/in/artificial-intelligence/generative-ai/large-language-models/

[APEX][11] is a fully managed, low-code application development platform for building and deploying applications in Oracle Cloud. 


[11]: https://apex.oracle.com/en/

## Prerequisites

- Access to an Oracle Cloud Infrastructure cloud account, an Autonomous Database instance, and OCI Generative AI service.
- You have database Admin access and access to a schema where you want to use Select AI.

## Steps

Following are the steps to integrate Select AI with OCI Generative AI service.

1. The [DBMS_CLOUD_AI][12] package in Autonomous Database enables integration with a user-specified LLM. This package works with AI providers like OpenAI, Cohere, Azure OpenAI Service, and Oracle Cloud Infrastructure Generative AI. 


[12]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html#GUID-000CBBD4-202B-4E9B-9FC2-B9F2FF20F246

As the first step, grant the EXECUTE privilege on the DBMS_CLOUD_AI package to the schema where business data resides.

Run the following command as ADMIN.    
    
    GRANT EXECUTE ON DBMS_CLOUD_AI TO <<schema name>>;

1. To access OCI Generative AI service from Select AI, set the required IAM [policies][14]. An example is given below.

    allow group `<your-group-name>` to manage generative-ai-family in compartment `<your-compartment-name>`

2. Connect as an ADMIN user to the database and enable OCI [resource principal][16] authentication using [DBMS_CLOUD_ADMIN.ENABLE_PRINCIPAL_AUTH][17] procedure.

	    BEGIN
    
            DBMS_CLOUD_ADMIN.ENABLE_PRINCIPAL_AUTH(provider => 'OCI',username =>'<<your schema name>>');
    
        END;

3. Create a new AI profile in your business schema using [DBMS_CLOUD_AI.CREATE_PROFILE][19] procedure.    

		BEGIN
           DBMS_CLOUD_AI.CREATE_PROFILE(
          'OCI_GENAI',                               
          '{
            "provider": "oci",
            "credential_name": "OCI$RESOURCE_PRINCIPAL",
            "object_list": [
                             {
                             "owner": "<<your schema name>>"
                             }
                            ],
            "model": "cohere.command-light",
            "oci_runtimetype": "COHERE",
            "temperature":"0.4"
          }');
		END;

	Attributes of an AI profile help to manage and configure the behavior of the AI profile. For the full list of attributes refer to [this][21].

	- **provider**: oci  
	- **credential_name**: OCI$RESOURCE_PRINCIPAL
	- **object_list**: It is an array of JSON objects specifying the owner and object names that are eligible for natural language translation to SQL. To include all objects of a given user, omit the "name" and only specify the "owner" key in the JSON object. 
	- **model**: The name of the AI model being used. Pretrained models for OCI Generative AI are all supported by Select AI. The default is cohere.command. Custom models can also be supplied with their full OCIDs.
	- **oci_runtimetype**: This attribute indicates the [runtime type ][22]of the provided model. This attribute is required when the model attribute is specified. 
	- **temperature**: Temperature is used to tune the degree of randomness in the text generated. Lower temperatures mean fewer random generations.

4. Set an AI profile before running Select AI commands. [DBMS_CLOUD_AI.SET_PROFILE][23] procedure sets the AI profile for the current session.
    
    EXEC DBMS_CLOUD_AI.set_profile('OCI_GENAI');

5. After setting an AI profile for the database session, any SQL statement with the prefix [SELECT AI ][25]is considered a natural language prompt. The `AI` keyword in a `SELECT` statement instructs the SQL execution engine to use the LLM identified in the active AI profile to process natural language and generate SQL. You cannot run PL/SQL statements, DDL statements, or DML statements using the `AI` keyword.

	The syntax for running AI prompt is:
    
	    SELECT AI action natural_language_prompt;

	Actions can be runsql, showsql, narrate, and chat.

	**runsql**, runs the provided SQL command using a natural language prompt. This is the default action and it is optional to specify this parameter. **showsql**, displays the SQL statement for a natural language prompt. **narrate**, explains the output of the prompt in natural language. **chat** is for general conversation with LLM. An example of using Select AI is shown below.
    
	    SELECT AI show all employees in Accounting department;

	Select AI is not supported in Database Actions or APEX Service. You can use only [DBMS_CLOUD_AI.GENERATE][28] function.
    
	    SELECT DBMS_CLOUD_AI.GENERATE(prompt       => 'show all employees in Accounting department',
                                        profile_name => 'OCI_GENAI'
                                      )
        FROM dual;

6. If you need a UI to run your prompts, one option is to use Oracle APEX.

	In the APEX workspace, create an Application with a page.

	Create a Page Item called PROMPT of type Textarea and a button to submit the prompt.

	![APEX][30]


	[30]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONT647046228E1E44BD83D6E1B8E5E28539/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

	Create a **Classic Report Region** and Type as **Function Body returning SQL Query**. Enable **Use Generic Column Names** property and enter number of columns in **Generic Column Count.**

	Enter PL/SQL Function Body as follows. `DBMS_CLOUD_AI.GENERATE` returns the SQL query using SELECT AI.

		BEGIN
			IF :PROMPT IS NOT NULL THEN
				RETURN DBMS_CLOUD_AI.GENERATE(:PROMPT,
					profile_name => 'OCI_GENAI');
			END IF;
		END;

	![APEX][32]


	[32]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONTFCFF38F7F0964C6F86CB31AE1238E129/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

	In the Attributes tab of the Report region, set **Heading** Type as Column Names

	![apex][33]


	[33]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONT38F0386BDD4D40E29117B761C5C608A6/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

	When the page is run, you can ask a question and get a response from the database. The schema used in this example is the HR sample schema.

	Entering a prompt, _show all employees in Accounting Department_, returns the results as below.

	![APEX][34]


	[34]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONTCE354B261F0944C6A8CF5C354D2488B1/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

	Entering a prompt, _show all locations in the US_ ,returns the results as below.

	![APEX][35]


	[35]: https://www.ateam-oracle.com/content/published/api/v1.1/assets/CONT2104F086B4104816AB40CB7B154A00F2/Medium?cb=_cache_538e&channelToken=12f676b76bf44b4e9b22e6b36ebfe358&format=jpg

## Conclusion

The ability of LLMs to generate queries adds more value to enterprise data. It substantially enhances the user experience, making data access more efficient. However, it's important to remember that LLMs are susceptible to hallucinations, and their generated results may not always be accurate. There's a possibility that a specific natural language prompt may not produce the intended SQL query, or the generated SQL may not be executable. Despite these limitations, the integration of LLMs remains a valuable asset for data management, provided users remain aware of its potential inaccuracies and exercise caution in interpreting and implementing generated queries.

The content in this demo is created by following [the steps in this blog post.](https://www.ateam-oracle.com/post/using-oci-genai-with-select-ai-and-apex-to-query-data-using-natural-language)

[2]: https://www.ateam-oracle.com/authors/rekha-mathew

[22]: https://docs.oracle.com/en-us/iaas/api/#/en/generative-ai-inference/20231130/datatypes/LlmInferenceRequest
[23]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html#ADBSB-GUID-E353F87C-C787-441D-B34D-7117B474A610
[19]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html#ADBSB-GUID-D51B04DE-233B-48A2-BBFA-3AAB18D8C35C
[14]: https://docs.oracle.com/en-us/iaas/Content/generative-ai/iam-policies.htm
[16]: https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/resource-principal.html#GUID-E283804C-F266-4DFB-A9CF-B098A21E496A
[17]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-admin.html#ADBSB-GUID-DB61623B-E4AF-4F82-9AFE-DF58601D4852
[21]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html#ADBSB-GUID-12D91681-B51C-48E0-93FD-9ABC67B0F375
[25]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/sql-generation-ai-autonomous.html#ADBSB-GUID-B3E0EE68-3B4C-4002-9B45-BBE258A2F15A
[28]: https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html#GUID-7B438E87-0E9A-4318-BA01-3BE1A5851229

## Contributing

This project is open source.  Please submit your contributions by forking this repository and submitting a pull request!  Oracle appreciates any contributions that are made by the open source community.

## License
Copyright (c) 2022 Oracle and/or its affiliates.

Licensed under the Universal Permissive License (UPL), Version 1.0.

See [LICENSE](LICENSE) for more details.

ORACLE AND ITS AFFILIATES DO NOT PROVIDE ANY WARRANTY WHATSOEVER, EXPRESS OR IMPLIED, FOR ANY SOFTWARE, MATERIAL OR CONTENT OF ANY KIND CONTAINED OR PRODUCED WITHIN THIS REPOSITORY, AND IN PARTICULAR SPECIFICALLY DISCLAIM ANY AND ALL IMPLIED WARRANTIES OF TITLE, NON-INFRINGEMENT, MERCHANTABILITY, AND FITNESS FOR A PARTICULAR PURPOSE.  FURTHERMORE, ORACLE AND ITS AFFILIATES DO NOT REPRESENT THAT ANY CUSTOMARY SECURITY REVIEW HAS BEEN PERFORMED WITH RESPECT TO ANY SOFTWARE, MATERIAL OR CONTENT CONTAINED OR PRODUCED WITHIN THIS REPOSITORY. IN ADDITION, AND WITHOUT LIMITING THE FOREGOING, THIRD PARTIES MAY HAVE POSTED SOFTWARE, MATERIAL OR CONTENT TO THIS REPOSITORY WITHOUT ANY REVIEW. USE AT YOUR OWN RISK. 
