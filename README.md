# cloud-dw-tools
Tools to build data pipelines in the cloud for data analysts.

## Purpose:
The goal for this project is to give data analysts a low cost abilty to build and produce reliable low-medium volume data pipelines themselves and not need to turn to data engineers for the task, thus speeding up their overall productivity and lowering costs

This is done by reducing the task of setting up data pipelines into: 

1. Write ETL logic as a SQL query
2. Copy the SQL query into a JSON object
3. Configure a LAMBDA function in AWS, uploading JSON file into a S3 bucket

Moving the ETL logic into SQL removes the need for analysts to have to learn scripting languages like Python or R in addition to linux server management and command line tools like GIT thus reducing the required training time.  This enables you to draw from a larger pool of potential candidates for your data analyts.  Conversely using standard SQL removes the need to acquire and setup costly proprietary ETL tools like Informatica.

Moving the ETL logic into the database itself also removes the need to maintain ETL servers as all the work is done in the database.  Thus maintainence is kept with the DBAs.  

Similarly moving the scheduling function to AWS Lambda funtion also removes the need setup and maintain linux servers in the cloud just to setup daemons - this improves the reliability of the ETL jobs as linux boxes won't go down, suffer from memory leaks or provide yet another attack surface for hackers.

As a result, this provides an overall reduction in costs - open source software, AWS per-use metering - and time - no need to wait on data engeineer resources to build pipelines.



### Initial Setup of cloud-dw-tools:
1. setup an S3 bucket on your AWS with appropriate security permissions.  
 Since you will be storing connection information including DB passwords and private keys, DO NOT make this bucket publically accessible.

2. upload the redshift_exec_script.zip file to your S3 bucket.  
 This is a fully contained execution environment including the lamba function script and required supporting libraires for the AWS Lambda function to run.  

 Copy the URL of the redshift_exec_script.zip file - you will need this when configuring your LAMBDA functions

3. configure a JSON file with your database connection information and upload it to your S3 bucket.  The db connnection JSON should have the following fields:
	..***host:** ip address or ip-routable name of your database server
	..***dbname:** the name of the target database
	..***port:** the port number for the Lambda function to connect to
	..***login:** database login name
	..***password:** password for the database login

 You may refer to sample_rs_conn_strings.json file included in the repository for an example of the JSON file

4. Configure an AWS execution role.  You may need to have your AWS admin do this for you.
 In general, your AWS execution role should have permissions to execute Lambda scripts and access the server where your database resides.  If you plan on using the read_gheet() function, your AWS role will also need to be able to call out to the larger internet (or at least access the Google servers where your Google sheet is locaeted).


5. if you plan on using the read_gsheet() function to load data on a google sheet into your database, you will need to retrieve a Google API private key and upload that to the s3 bucket. 

To do that, you will need to enable server to server integration on your google apps account:
	a. Go to https://console.developers.google.com/apis/credentials
	b. Create Credentials -> Service Account Key
		1. select the custom service account
		2. select JSON
	c. *download the credential json file and then upload it to your S3 bucket*  
	d. Enable the API - https://console.developers.google.com/start/api?id=sheets.googleapis.com
	e. In the JSON file, look at the client_email - keep that email address for later use

You may refer to "sample-gsheet-service-acct-private-key.json" to see this Google API JSON key file should look like.

Note: If your google account is centrally managed, you may need for your Google Apps admin to do this for you.



**Documentation on Functions:**

## exec_query()

This Python 2.7 Lambda function that retrieves a set of SQL instructions in the form of a JSON object in stored in an S3 bucket
and executes them on your redshift database.  Pair with cloudwatch event triggers to generate scheduled ETL function.

Note: The function will execute all your SQL commands in a transaction and rollback the entire transaction any command fail.

To use the function: 
1. Write your ETL routine as a SQL query that runs against the database

2. create a JSON file to encapsulate your ETL routine.  Refer to the sample_etl_query.json file as an example.  
Your JSON should have the following fields:
	**query_name:**  the name of your etl query - like update table
	**query_array:** an list of JSON array that specify your ETL query
		**element 0:** the actual SQL query code.  JSON doesn't support line breaks so you will need minify your code using a tool like https://www.cleancss.com/json-minify/
		**element 1:** the message you want to display/log before the command 
		**element 2:** the message to display/log if that command should fail

3. Upload the ETL json file to your S3 bucket.

4. Configure the lambda function:

	a. create a new lambda function in AWS
	b. Function Code:  select "Python 2.7" Runtime and  choose "Upload a file from S3", S3 Link URL: copy the URL of the execution library in #2 of the initial setup, e.g. "https://s3-us-west-1.amazonaws.com/s3_bucket_name/redshift_exec_script.zip"
	c. Handler: "exec_redshift_prod_s3_v3.exec_query"
	d. Environment Variables:  you should have the following environment variables
		**s3_bucket_name** : <the name of your AWS s3 bucket>
		**connStringJSON** : <the name of the connection string JSON object uploaded to the s3 bucket in the Step #3 of the initial setup>
		**querySpecJSON** : <the name of the ETL commands JSON objecti you uploaded to s3 in Step #3 above>
	e. Basic Settings:
			set Memory to at least 512 MB.  1GB or higher is recommended
			set Timeout to at least 2 mins. 3-4mins is recommended to larger ETL commands
	f. Execution Role: whatever execution role your AWS admin is configured.
	g. Network: please refer to your AWS admin for approriate VPC,Subnets and Security Groups (if needed) config settings to access your dataabase server

5. To Schedule recurring execution of the script, select appropriate Cloudwatch Event Triggers on Designer card.  
 (You will need to configure such triggers if you have not already done so).
 
6. Save the function and test it 
 (you will need to define a test event if testing for the first time.  The default test event is fine)


## read_gsheet()
This function load the contents of a google sheet range into a database table on AWS Redshift
This function destructive overwrites and reloads the entire contents of the specficied database table

To enable this function, you will need to set up appropriate permissions on the google sheet and load the google sheet credentials into a specified file in an AWS S3 bucket

#### Setup instructions:
1. on the Google sheet, give view permissions to the email address specified in the JSON API key file on Step 5e of the initial setup;
	go to the sheet -> share ->enter that email address

2. configure a load specs JSON file that specifies which cell range on the Google sheet to upload and to which table.  You JSON should have the following fields:

	**src_gsheet_ID** : this is a reference to the Google sheet ID.  You can find it in the URL of your Google sheet.  

		e.g. For instance if your google sheet URL https://docs.google.com/spreadsheets/d/1UXo3WiTTBQmXqkHUcgWpjk88Z6TJNFC69r1J83HI6af/  the ID is "1UXo3WiTTBQmXqkHUcgWpjk88Z6TJNFC69r1J83HI6af"

	**src_gsheet_range** : specifies the worksheet and cell range to load.  the format is "<sheetname>!<cellrange>"  e.g. "PartyTime!A:D" would tell it to load columns A to D on worksheet called "PartyTime".  The google sheet range must be a continguous block of data

	**tgt_db_schema** : specifies the database schema of the table you want to load the data do.

	**tgt_table** : the name of the table you want ot load the table too

		e.g. if you want to load the data to table highgarden.soliders then "tgt_db_schema" : "highgarden" and "tgt_table" : "soldiers"

 You may refer to sample_gsheet_load_specs.json for an example


3. Configure the Lambda function
	a. create a new lambda function in AWS
	b. Function Code:  select "Python 2.7" Runtime and  choose "Upload a file from S3", S3 Link URL: copy the URL of the execution library in #2 of the initial setup, e.g. "https://s3-us-west-1.amazonaws.com/s3_bucket_name/redshift_exec_script.zip"
	c. Handler: "exec_redshift_prod_s3_v3.read_gsheet"
	d. Environment Variables:  you should have the following environment variables
		s3_bucket_name : the name of your AWS s3 bucket
		connStringJSON : the name of the connection string JSON object uploaded to the s3 bucket in the Step #3 of the initial setup
		gsheetCredsJSON : the name of the GoogleAPI Key JSON file you uploaded on Step #5c of the initial setup
		gsheetUpdateSpecsJSON : the filename of the gsheet load specs JSON object configured in Step 2
	e. Basic Settings:
			set Memory to at least 512 MB.  1GB or higher is recommended
			set Timeout to at least 2 mins. 3-4mins is recommended to larger ETL commands
	f. Execution Role: whatever execution role your AWS admin is configured.
	g. Network: please refer to your AWS admin for approriate VPC,Subnets and Security Groups (if needed) config settings to access your dataabase server


##### NOTES:
-the first row in the google sheet range will be interpreted as the column names
-all data will be imported as text fields so you will may to do additional type conversions on your SQL queries


