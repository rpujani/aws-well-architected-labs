---
title: "FAQ"
#menutitle: "FAQs"
date: 2022-08-31T11:16:08-04:00
chapter: false
weight: 8
hidden: false
---
#### Last Updated
August 2022

If you wish to provide feedback on this lab, there is an error, or you want to make a suggestion, please email: cloud-intelligence-dashboards@amazon.com

## How do I setup the dashboards in an account other than my payer account or on top of multiple payer accounts?

This scenario allows customers with multiple payer (management) accounts to deploy all the CUR dashboards on top of the aggregated data from multiple payers. To fulfill prerequisites customers should set up or have setup a new Governance Account. The payer account CUR S3 buckets will have S3 replication enabled, and will replicate to a new S3 bucket in your separate Governance account.
{{%expand "Click here to expand step by step instructions" %}}

![Images/CUDOS_multi_payer.png](/Cost/200_Cloud_Intelligence/Images/CUDOS_multi_payer.png?classes=lab_picture_small)

**NOTE: These steps assume you've already setup the CUR to be delivered in each payer (management) account.**

#### Setup S3 CUR Bucket Replication

1. Create or go to the console of your Governance account. This is where the Cloud Intelligence Dashboards will be deployed. Your payer account CURs will be replicated to this account. Note the region, and make sure everything you create is in the same region. To see available regions for QuickSight, visit [this website](https://docs.aws.amazon.com/quicksight/latest/user/regions.html). 
2. Create an S3 bucket with enabled versioning.
3. Open S3 bucket and apply following S3 bucket policy with replacing respective placeholders {PayerAccountA}, {PayerAccountB} (one for each payer account) and {GovernanceAccountBucketName}. You can add more payer accounts to the policy later if needed.

```json
{
"Version": "2008-10-17",
"Id": "PolicyForCombinedBucket",
"Statement": [
	{
		"Sid": "Set permissions for objects",
		"Effect": "Allow",
		"Principal": {
    		"AWS": ["{PayerAccountA}","{PayerAccountB}"]
		},
		"Action": [
    		"s3:ReplicateObject",
    		"s3:ReplicateDelete"
		],
		"Resource": "arn:aws:s3:::{GovernanceAccountBucketName}/*"
	},
	{
		"Sid": "Set permissions on bucket",
		"Effect": "Allow",
		"Principal": {
		    "AWS": ["{PayerAccountA}","{PayerAccountB}"]
		},
		"Action": [
 		   "s3:List*",
 		   "s3:GetBucketVersioning",
 		   "s3:PutBucketVersioning"
		],
		"Resource": "arn:aws:s3:::{GovernanceAccountBucketName}"
	},
	{
		"Sid": "Set permissions to pass object ownership",
		"Effect": "Allow",
		"Principal": {
		    "AWS": ["{PayerAccountA}","{PayerAccountB}"]
		},
		"Action": [
    		"s3:ReplicateObject",
    		"s3:ReplicateDelete",
    		"s3:ObjectOwnerOverrideToBucketOwner",
    		"s3:ReplicateTags",
    		"s3:GetObjectVersionTagging",
    		"s3:PutObject"
		],
		"Resource": "arn:aws:s3:::{GovernanceAccountBucketName}/*"
	}
]
}
```

This policy supports objects encrypted with either SSE-S3 or not encrypted objects. For SSE-KMS encrypted objects additional policy statements and replication configuration will be needed: see https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html

#### Set up S3 bucket replication from each Payer (Management) account to S3 bucket in Governance account

This step should be done in each payer (management) account.

1. Open S3 bucket in Payer account with CUR.
2. On Properties tab under Bucket Versioning section click Edit and set bucket versioning to Enabled.
3. On Management tab under Replication rules click on Create replication rule.
4. Specify rule name.

![Images/s3_bucket_replication_1.png](/Cost/200_Cloud_Intelligence/Images/s3_bucket_replication_1.png?classes=lab_picture_small)

5. Select Specify a bucket in another account and provide Governance account id and bucket name in Governance account.
6. Select Change object ownership to destination bucket owner checkbox.
7. Select Create new role under IAM Role section.

![Images/s3_bucket_replication_2.png.png](/Cost/200_Cloud_Intelligence/Images/s3_bucket_replication_2.png?classes=lab_picture_small)

8. Leave rest of the settings by default and click Save.

#### Copy existing objects from CUR S3 bucket to S3 bucket in Governance account

This step should be done in each payer (management) account.

Sync existing objects from CUR S3 bucket to S3 bucket in Governance account.

	aws s3 sync s3://{curBucketName} s3://{GovernanceAccountBucketName} --acl bucket-owner-full-control

After performing this step in each payer (management) account S3 bucket in Governance account will contain CUR data from all payer accounts under respective prefixes.

#### Prepare Glue Crawler 

These actions should be done in Governance account

1. Open AWS Glue Service in AWS Console in the same region where S3 bucket with aggregated CUR data is located and go to Crawlers section
2. Click Add Crawler
3. Specify Crawler name and click Next
4. In Specify crawler source type leave settings by default. Click Next

![Images/glue_1.png](/Cost/200_Cloud_Intelligence/Images/glue_1.png?classes=lab_picture_small)

5. In Add a data store select S3 bucket name with aggregated CUR data and add following exclusions **.zip, **.json, **.gz, **.yml, **.sql, **.csv, **/cost_and_usage_data_status/*, aws-programmatic-access-test-object. Click Next

![Images/glue_2.png](/Cost/200_Cloud_Intelligence/Images/glue_2.png?classes=lab_picture_small)

6. In Add another data store leave No by default. Click Next

7. In Choose an IAM role select Create an IAM role and provide role name. Click Next

![Images/glue_1.png](/Cost/200_Cloud_Intelligence/Images/glue_1.png?classes=lab_picture_small)

8. In Create a schedule for this crawler select Daily and specify Hour and Minute for crawler to run

9. In Configure the crawler’s output choose Glue Database in which you’d like crawler to create a table or add new one. Select Create a single schema for each S3 path checkbox. Select Add new columns only and Ignore the change and don’t update the table in the data catalog in Configuration options. Click Next 

*Please make sure Database name doesn’t include ‘-’ character*

![Images/glue_4.png](/Cost/200_Cloud_Intelligence/Images/glue_4.png?classes=lab_picture_small)

10. Crawler configuration should look as on the screenshot below. Click Finish

11. Resume deployment methodoly of choice from previous page. 

{{% /expand%}}

## I'm getting an error in QuickSight that is saying Athena timed out?

For very large CUR files, Athena may time out trying to query the data for summary_view. In Athena, find the summary_view view, click the three dots next to it and select show/edit query. Modify the following:

- [Request Athena](https://docs.amazonaws.cn/en_us/athena/latest/ug/service-limits.html?) DML query timeout increase via support case. 
- Or, adjust the granularity to monthly, by changing 'day' to 'month' in row 6. 
- Or, adjust the look back from '7' months to desired time-frame in row 75.
- In QuickSight, refresh your dataset. 

## I am getting the error StartQueryExecution operation: No output location provided. How do I fix this?

An output location is required either through the Workgroup result configuration setting or as an API input. 
{{%expand "Click here to expand step by step instructions" %}}

1. Visit Athena and click on the three line hamburger icon on the top left. 

![images/athenahelp1.png](/Cost/200_Cloud_Intelligence/Images/athenahelp1.png?classes=lab_picture_verysmall)

1. Select Workgroups. Click on primary (or the workgroup you wish to use), and from the actions menu select edit. 

![images/athenahelp15.png](/Cost/200_Cloud_Intelligence/Images/athenahelp15.png?classes=lab_picture_verysmall)

1. Find the Query results configuration section, designate a bucket for your query results, and click save. 

![images/athenahelp2.png](/Cost/200_Cloud_Intelligence/Images/athenahelp2.png?classes=lab_picture_verysmall)

{{% /expand%}}

## How do I limit access to the data in the Dashboards using row level security? 

Do you want to give access to the dashboards to someone within your organization, but you only want them to see data from accounts or business units associated with their role or position? You can use row level seucirty in QuickSight to accomplish limiting access to data by user. In these steps below, we will define specific Linked Account IDs against individual users. Once the Row-Level Security is enabled, users will continue to load the same Dashboards and Analyses, but will have custom views that restrict the data to only the Linked Account IDs defined.

**Video Tutorial**

{{< rawhtml >}}
<iframe width="560" height="315" src="https://www.youtube.com/embed/EFyWEyeXQlE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
{{< /rawhtml >}}

{{%expand "Click here to expand step by step instructions" %}}

**Considerations:**

* The permissions dataset can't contain duplicate values. Duplicates are ignored when evaluating how to apply the rules.

* Each user or group specified can see only the rows that match the field values in the dataset rules.

* If you add a rule for a user or group and leave all other columns with no value (NULL), you grant them access to all the data.

* If you don't add a rule for a user or group, that user or group can't see any of the data.
	* If the userbase of QuickSight will be changing frequently, consider storing your csv in S3 rather than a local file.

* The full set of rule records that are applied per user must not exceed 999. This applies to the total number of rules that are directly assigned to a user name plus any rules that are assigned to the user through group names.

* If a field includes a comma (,) Amazon QuickSight treats each word separated from another by a comma as an individual value in the filter. For example, in ('AWS', 'INC'), AWS,INC is considered as two strings: AWS and INC. To filter with AWS,INC, wrap the string with double quotation marks in the permissions dataset.

#### Create a CSV file defining users and Linked Account IDs

Create a CSV file that looks something like this:

	username,account_id
	user1@amazon.co.uk,"123456123456"
	user1@amazon.co.uk,"987654987654"
	user2@amazon.fr,"123456123456"
	user3@amazon.com,"789123456123"

Any Account IDs that you wish the given user to see should be defined in the account_id field of the CSV. Create a separate row for a single username having access to multiple account IDs. Ensure there are no spaces after your final quote character. Name this file something similar to CUDOS_Dataset_rules.csv

If you want to use QuickSight groups the CSV input file is slightly different.  Instead of UserName as the initial field you have to use GroupName.  Also, you can only use Users or Groups in the input file, not both. [This page](https://docs.aws.amazon.com/quicksight/latest/user/restrict-access-to-a-data-set-using-row-level-security.html#create-data-set-rules-for-row-level-security) provides more details. QuickSight groups can only be created and managed via the QuickSight CLI. There is no UI for this in the QuickSight console.

You now have 2 options on how to proceed:

1. Keep your CSV file local and create a new Dataset by Uploading a File as the Data Source
2. Upload your CSV to S3 and create a new Dataset with an S3 Manifest file as the Data Source

#### Using a local CSV file as the data source

Create a new Dataset using the CSV file above as the Data Source: Click New Dataset and select Upload a file. Locate your CUDOS_Dataset_rules.csv and a Preview will appear.

![Images/rowlevelsec1.PNG](/Cost/200_Cloud_Intelligence/Images/rowlevelsec1.PNG?classes=lab_picture_small)

Click Edit settings and prepare data. Verify that the account_id field is a String data type. If it appears as an Integer, change the data type for account_id to String.

![Images/ChangeDataType.png](/Cost/200_Cloud_Intelligence/Images/ChangeDataType.png?classes=lab_picture_small)

The reason we need to do this is that we will lose any leading or trailing zeroes on an Account ID if it remains as an Integer value. This is also the same Data Type used for the account_id field in all the CUDOS Datasets.

Save the CUDOS_Dataset_rules.csv dataset.

1. Select the dataset name
2. Click Row-level security
![Images/RLS.png](/Cost/200_Cloud_Intelligence/Images/RLS.png?classes=lab_picture_small)
3. Select CUDOS_Dataset_rules.csv
![Images/ChooseDataset.png](/Cost/200_Cloud_Intelligence/Images/ChooseDataset.png?classes=lab_picture_small)
4. Click Apply dataset
5. Click Apply dataset again
6. Wait until you see Selected dataset rules populate with CUDOS_Dataset_rules
![Images/AppliedDataset.png](/Cost/200_Cloud_Intelligence/Images/AppliedDataset.png?classes=lab_picture_small)
7. Click x to exit
8. Click x to exit
9. Repeat steps for all CUDOS-related datasets (See below for a list of current Dataset Names)

Once you have applied the CUDOS_Dataset_rules S3 Dataset to all your CUDOS datasets, visit the CUDOS Dashboard as a user who is defined in the csv file, and confirm the Account IDs shown are only the ones specified in that file.

#### Use S3 as the data source

Upload your csv file to the Athena query location bucket. eg. aws-athena-query-results-123456123456-us-east-1

Create an S3 manifest file that looks something like this:

	{
  	"entries": [
    	{"url":"s3://aws-athena-query-results-123456123456-us-east-1/CUDOS_Dataset_rules.csv", 	"mandatory":true},
 	 ]
	}

This manifest file can be saved locally, or uploaded to the same S3 bucket where the csv file is stored. Save this file as something similar to CUDOS_manifest.json.

Back in the QuickSight Admin Console, click New Dataset and select S3. Name the Dataset CUDOS_Dataset_rules and locate your CUDOS_manifest.json file either by entering the S3 URL where it is stored, or choosing to upload from your local machine.

![Images/S3-URL.png](/Cost/200_Cloud_Intelligence/Images/S3-URL.png?classes=lab_picture_small)

Click Edit/preview data.

![Images/S3-EditPreview.png](/Cost/200_Cloud_Intelligence/Images/S3-EditPreview.png?classes=lab_picture_small)

Verify that the account_id field is a String data type. If it appears as an Integer, change the data type for account_id to String.

![Images/S3-ChangeDataType.png](/Cost/200_Cloud_Intelligence/Images/S3-ChangeDataType.png?classes=lab_picture_small)

Save the CUDOS_Dataset_rules dataset.

On each of the datasets that the CUDOS dashboard is using, define Row Level Security by following these steps:

1. Select the dataset name
2. Click Row-level security

![Images/RLS.png](/Cost/200_Cloud_Intelligence/Images/RLS.png?classes=lab_picture_small)

3. Select CUDOS_Dataset_rules

![Images/S3-ChooseDataset.png](/Cost/200_Cloud_Intelligence/Images/S3-ChooseDataset.png?classes=lab_picture_small)

4. Click Apply dataset
5. Click Apply dataset again
6. Wait until you see Selected dataset rules populate with CUDOS_Dataset_rules

![Images/S3-AppliedDataset.png](/Cost/200_Cloud_Intelligence/Images/S3-AppliedDataset.png?classes=lab_picture_small)

7. Click x to exit
8. Click x to exit
9. Repeat steps for all CUDOS-related datasets (See below for a list of current Dataset Names)

Once you have applied the CUDOS_Dataset_rules S3 Dataset to all your CUDOS datasets, visit the CUDOS Dashboard as a user who is defined in the csv file, and confirm the Account IDs shown are only the ones specified in that file.

{{% /expand%}}

## How do I fix the COLUMN_GEOGRAPHIC_ROLE_MISMATCH error? 

When attempting to deploy the dashboard manually, some users get an error that states COLUMN_GEOGRAPHIC_ROLE_MISMATCH. 
{{%expand "Click here to expand answer" %}}

This error is caused by there being too many [data source connectors](https://docs.aws.amazon.com/quicksight/latest/user/working-with-data-sources.html) in QuickSight with the same name. To check how many data source connectors you have, visit QuickSight datasets and click on **new datasets**. Scroll to the bottom and note how many Athena data connectors there are with the same name. 

![Images/Sduplicatedataset.png](/Cost/200_Cloud_Intelligence/Images/duplicatedataset.png?classes=lab_picture_small)

Unless you know which datasets are tied to which data sources, it is faster to simply delete all the Cloud Intelligence Dashboards data sources and data sets from QuickSight, and start adding them again, this time only using a single data source. This is described in detail in [this lab under the manual deployment option](https://wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/cost-usage-report-dashboards/dashboards/2a_cost_intelligence_dashboard/) as step 22. You should only have one data source for all your Cloud Intelligence Dashboard datasets, including customer_all. If you wish to use separate data sources, they must not have the same name. 

{{% /expand%}}

## How do I fix the ‘product_cache_engine’ cannot be resolved error? 

When attempting to deploy the dashboard, some users get an error that states `product_cache_engine` cannot be resolved. 
{{%expand "Click here to expand answer" %}}

This view is dependent on having or historically having an RDS database instance and an ElastiCache cache instance run in your organization. If you get the error that the column `product_database_engine` or `product_deployment_option` does not exist, then you do not have any RDS database instances running. There are two options to resolve this.

### Option 1
 To make this column show up in the CUR spin up a database in the RDS service, let it run for a couple of minutes and in the next integration of the crawler the column will appear. If you get the error that the column `product_cache_engine` does not exist, then you do not have any ElastiCache cache instances running. To make this column show up in the CUR spin up an ElastiCache cache instance in the ElastiCache service, let it run for a couple of minutes and in the next integration of the crawler the column will appear. You can verify this by running the Athena query: SHOW COLUMNS FROM tablename - and replace the tablename accordingly after selecting the correct CUR database in the dropdown on the left side in the Athena view.

### Option 2
Follow the below steps to remove the colum. Be aware this will mean if you do add ElastiCache instances to your accounts you should put this back.

1. In Amazon Athena click 'Show Edit Query' for ``kpi_instance_all``
2. Remove *`product_cache_engine`* and remove the last *Group by* number
3. Run query
4. If you are running from CloudShell re-run the deploy command
5. Go to Amazon Quicksight 
6. Find the ``kpi_instance_all`` dataset
7. Click on *'Edit Dataset'* 
8. On the top right of the screen click *'save and publish'*


{{% /expand%}}

{{% notice tip %}}
This page will be updated regularly with new answers. If you have a FAQ you'd like answered here, please reach out to us here cloud-intelligence-dashboards@amazon.com. 
{{% /notice %}}
