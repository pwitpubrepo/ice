# Ice
[![Build Status](https://travis-ci.org/Teevity/ice.svg?branch=master)](https://travis-ci.org/Teevity/ice)

## Intro
Ice provides a birds-eye view of our large and complex cloud landscape from a usage and cost perspective.  Cloud resources are dynamically provisioned by dozens of service teams within the organization and any static snapshot of resource allocation has limited value.  The ability to trend usage patterns on a global scale, yet decompose them down to a region, availability zone, or service team provides incredible flexibility. Ice allows us to quantify our AWS footprint and to make educated decisions regarding reservation purchases and reallocation of resources.

Ice is a Grails project. It consists of three parts: processor, reader and UI. Processor processes the Amazon detailed billing file into data readable by reader. Reader reads data generated by processor and renders them to UI. UI queries reader and renders interactive graphs and tables in the browser.

## What it does
Ice communicates with AWS Programmatic Billing Access and maintains knowledge of the following key AWS entity categories:
- Accounts
- Regions
- Services (e.g. EC2, S3, EBS)
- Usage types (e.g. EC2 - m1.xlarge)
- Cost and Usage Categories (On-Demand, Reserved, etc.)
The UI allows you to filter directly on the above categories to custom tailor your view.

In addition, Ice supports the definition of Application Groups. These groups are explicitly defined collections of resources in your organization. Such groups allow usage and cost information to be aggregated by individual service teams within your organization, each consisting of multiple services and resources. Ice also provides the ability to email weekly cost reports for each Application Group showing current usage and past trends.

When representing the cost profile for individual resources, Ice will factor the depreciation schedule into your cost contour, if so desired.  The ability to amortize one-time purchases, such as reservations, over time allows teams to better evaluate their month-to-month cost footprint.

## Screenshots
1. Summary page grouped by accounts
![Summary page grouped by accounts](https://github.com/Netflix/ice/blob/master/screenshots/ss_summary.png?raw=true)

2. Detail page with throughput metrics and grouped by products
![Detail page with throughput metrics and grouped by products](https://github.com/Netflix/ice/blob/master/screenshots/ss_detail.png?raw=true)

3. Reservation page grouped by on-demand, un-used, reserved, upfront costs
![Reservation page](https://github.com/Netflix/ice/blob/master/screenshots/ss_reservation_byreservation.png?raw=true)

4. Reservation page with on-demand cost and grouped by instance types
![Reservation page with on-demand cost and grouped by instance types](https://github.com/Netflix/ice/blob/master/screenshots/ss_reservation_bytype.png?raw=true)

5. Breakdown page of Application Groups
![Breakdown page of Application Groups](https://github.com/Netflix/ice/blob/master/screenshots/ss_breakdown_appgroup.png?raw=true)

## Prerequisite:

1. First sign up for Amazon's programmatic billing access [here](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/detailed-billing-reports.html) to receive detailed billing(hourly) reports. Verify you receive monthly billing file in the following format: `<accountid>-aws-billing-detailed-line-items-<year>-<month>.csv.zip`.
2. Install Grails 2.4.4 and set GRAILS_HOME and JAVA_HOME
3. Ice uses [highstock](http://www.highcharts.com/) to generate interactive graphs. Please make sure you acquire the proper license before using it.


## Basic setup:
Using basic setup, you don't need any extra code change and you will use the provided bootstrap.groovy. You will need to construct your own ice.properties file and have it in your classpath (I put it at src/java). You can use sample.properties (located at src/java) file as the template.

1. Processor configuration

  1.1 Enable processor in ice.properties:

          ice.processor=true

  1.2 In ice.properties, set up the local directory where the processor can copy the billing file to and store the output files. The directory must pre-exist. For example:

          ice.processor.localDir=/mnt/ice_processor

  1.3 In ice.properties, set up the working s3 bucket and working s3 bucket file prefix to upload the processed output files which will be read by reader. Ice must have read and write access to the s3 bucket. For example:

          ice.work_s3bucketname=work_s3bucketname
          ice.work_s3bucketprefix=work_s3bucketprefix/

  1.4 If running locally, set the following system properties at runtime. ice.s3AccessToken is optional.

          ice.s3AccessKeyId=<accessKeyId>
          ice.s3SecretKey=<secretKey>
          ice.s3AccessToken=<accessToken>

  If running on a ec2 instance and you want to use the credentials in the instance metadata, you can leave the above properties unset.

  1.5 In ice.properties, specify the start time in millis for the processor to start processing billing files. For example, if you want to start processing billing files from April 1, 2013. If this property is not set, Ice will set startmillis to be the beginning of current month.

          ice.startmillis=1364774400000

  1.6 Set up s3 billing bucket in ice.properties. If you have multiple payer accounts, you will need to specify multiple values for each property.

          # s3 bucket name where the billing files are. multiple bucket names are delimited by ",". Ice must have read access to billing s3 bucket.
          ice.billing_s3bucketname=billing_s3bucketname1,billing_s3bucketname2
          # prefix of the billing files. multiple prefixes are delimited by ","
          ice.billing_s3bucketprefix=,
          # location for the billing bucket.  It should be specified for buckets using v4 validation
          # update 20170907 by HouYu Li <hyli@pixelworks.com> : Now support S3 bucket in China Region. If a China Region S3 bucket is configured, all other non China Region S3 buckets will be ignored.
          ice.billing_s3bucketregion=eu-west-1,eu-central-1
          # specify your payer account id here if across-accounts IAM role access is used. multiple account ids are delimited by ",". "ice.billing_payerAccountId=,222222222222" means assumed role access is only used for the second bucket.
          ice.billing_payerAccountId=,123456789012
          # specify the assumed role name here if you use IAM role access to read from billing s3 bucket. multiple role names are delimited by ",". "ice.billing_accessRoleName=,ice" means assumed role access is only used for the second bucket.
          ice.billing_accessRoleName=,ice
          # specify external id here if it is used. multiple external ids are delimited by ",". if you don't use external id, you can leave this property unset.
          ice.billing_accessExternalId=

  Tip: If you have multiple payer accounts, or Ice is running from a different account of the s3 billing bucket, for example Ice is running in account "test", while the billing files are written to bucket in account "prod", account "test" does not have read access to those billing files because Amazon created them. In this case, the recommended way is to use cross-accounts IAM roles. E.g. you can create an assumed role "ice". In "prod" account, grant assumed role "ice" with read access to billing bucket, then specify ice.billing_accessRoleName=ice. You can also create a secondary s3 bucket in account "prod" and grant read access to account "test", and then create a billing file poller running in account "prod" to copy billing files to the secondary bucket.

  1.7 Specify account id and account name mappings in ice.properties. This is for readability purposes. For example:

          ice.account.account1=123456789011
          ice.account.account2=123456789012
          ice.account.account3=123456789013

  1.8 If you have reserved ec2 instances, please also make sure Ice has the permission to make ec2 call describeReservedInstancesOfferings, which is used to get ri prices.

2. Reader configuration

  2.1 Enable reader in ice.properties:

          ice.reader=true

  2.2 In ice.properties, set up the local directory where the reader will copy files to. The local directory must pre-exist. For example:

          ice.reader.localDir=/mnt/ice_reader

    Make sure the local directory is different if you run processor and reader on the same instance.

  2.3 Same as 1.3

  2.4 Same as 1.4

  2.5 Same as 1.5

  2.6 Specify your organization name in ice.properties. This will show up in the UI header.

          ice.companyName=Your Company Name

  2.7 You can choose to show cost in currency other than "$". To enable other currency, specify the following properties in ice.properties:

          # Specify your currency sign here. The default value is $. For other currency symbols, you can use UTF-8 code, e.g. for ¥, you can use ice.currencySign=\u00A5
          ice.currencySign=£
          # Specify your currency conversion rate here. The default value is 1. If 1 pound = 1.5 dollar, then the rate is 0.6666667.
          ice.currencyRate=0.6666667

  2.8 By default, Ice pulls in [Highstock](https://www.highcharts.com/) from its CDN.

          ice.highstockUrl=https://example.com/js/highstock.js

3. Running Ice

  After the processor and reader setup, you can choose to run the processor and reader on the same or different instances. Running on different instances is recommended. For the first time, you should FIRST RUN PROCESSOR. Make sure you see non-empty output files in your working s3 bucket. Then run reader and browse to http://localhost:8080/ice/dashboard/summary.

  Here are the steps of getting ice running locally:

  3.1 Pull the project

  3.2 Run `grails wrapper` from the project root directory. This step will pull all necessary jar from maven central.

  3.3 Construct ice.properties for processor and make sure ice.properties is added to directory src/java

  3.4 Run Ice processor. From project root directory, run `./grailsw run-app`. Note you may need to add system properties like `./grailsw -Dice.s3AccessKeyId=<s3AccessKeyId> -Dice.s3SecretKey=<s3SecretKey> run-app`. To verify Ice processor runs successfully, make sure you see un-empty output files in your working S3 bucket, e.g. tagdb_all, cost_weekly_all, cost_daily_all_2013, etc.

  3.5 Repeat steps 3.3 and 3.4 to run Ice reader.

  Tip: Sometimes you want to re-start from a clean slate. To do that:

  a) Get the latest code

  b) Delete all files from your working s3 bucket under the working prefix

  c) Delete all files from your local ice directory (for processor and reader)

  d) Start Ice in processor mode. Make sure it runs correctly.

  e) Then start Ice in reader mode.

## Ice Cookbook:

1. A community cookbook is available for deploying Ice via Chef here https://github.com/mdsol/ice_cookbook.

## Ice Docker Image:

1. A community image is available for deploying Ice via Docker here https://github.com/jonbrouse/docker-ice


## Advanced options:
Options with * require writing your own code.

1. Basic reservation service

  If you have reserved instances in your accounts, you may want to make use of the reservation view in the UI, where you can browse/analyze your on-demand, unused reserved instance usage & cost of different instance types in different regions, zones and accounts. in Bootstrap.groovy, BasicReservationService is used. You can specify reservation period and reservation utilization type in ice.properties:

          # reservation period, possible values are oneyear, threeyear
          ice.reservationPeriod=threeyear
          # reservation utilization, possible values are LIGHT, HEAVY
          ice.reservationUtilization=HEAVY

2. Reservation capacity poller

  To use BasicReservationService, you should also run reservation capacity poller, which will call ec2 API (describeReservedInstances) to poll reservation capacities for each reservation owner account defined in ice.properties. The reservation capacities history is stored in a file in s3 bucket. To run reservation capacity poller, following steps below:

    2.1 Set ice.reservationCapacityPoller=true in ice.properties

    2.2 Make sure you set up reservation owner accounts in ice.properties. For example:

          ice.owneraccount.account1=
          ice.owneraccount.account2=account3,account4
          ice.owneraccount.account5=account6

    2.3 If you need to poll reservation capacity of different accounts, set up IAM roles. Then specify the assumed roles and external ids in ice.properties. For example, if assumed role "ice" is used:

          ice.owneraccount.account1.role=ice
          ice.owneraccount.account2.role=ice
          ice.owneraccount.account5.role=ice

        If you use external id too, specify it like following:

          ice.owneraccount.account1.externalId=


3. On-demand instance cost alert

  You can set set an on-demand instance cost threshold so that alert emails will be sent through Amazon SES if the threshold is reached within last 24 hours. The alert emails will be sent no more than once a day. The following properties are needed in ice.properties:

          # url prefix, e.g. http://ice.netflix.com/. This is used to create the link in alert email
          ice.urlPrefix=
          # from email address, this email must be registered in ses.
          ice.fromEmail=
          # ec2 ondemand hourly cost threshold to send alert email. The alert email will be sent at most once per day.
          ice.ondemandCostAlertThreshold=250
          # ec2 ondemand hourly cost alert emails, separated by ","
          ice.ondemandCostAlertEmails=

4. Sharing reserved instances among accounts (*)

  All linked accounts under the same payer account can share each other's reservations (see http://docs.aws.amazon.com/awsaccountbilling/latest/about/AboutConsolidatedBilling.html).

  If reserved instances are shared among accounts, please specify them in ice.properties. For example:

          # set reservation owner accounts. In the example below, account1, account2, account3 and account4 are linked under the same payer account. account5, account6 are linked under the same payer account.
          # if reservation capacity poller is enabled, the poller will try to poll reservation capacity through ec2 API (describeReservedInstances) for each reservation owner account.
          ice.owneraccount.account1_name=account2_name,account3_name,account4_name
          ice.owneraccount.account2_name=account1_name,account3_name,account4_name
          ice.owneraccount.account5_name=account6_name

  If different accounts have different AZ mappings, you will also need to subclass BasicAccountService and override method getAccountMappedZone to provide correct AZ mapping.

5. Customized reservation service (*)

  Reserved instance prices in BasicReservationService are copied from Amazon's ec2 price page as of Jun 1, 2013. Your accounts may have different reservation prices (e.g. Amazon may change prices in the future). In this case, you need to write a subclass of BasicReservationService to provide the correct pricing.

6. Resource service (*)

  To use the breakdown feature and application group feature, first make sure you signed up the beta version of detailed billing file with resources and tag. Verify you receive monthly billing file in the this format: `<accountid>-aws-billing-detailed-line-items-with-resources-and-tags-<year>-<month>.csv.zip`. Then you will need to subclass abstract class ResourceService and have your own bootstrap.groovy create ProcessorConfig and ReaderConfig. See SampleMapDbResourceService for a sample of subclass.

  If your custom tags have limited number of value combinations (e.g. < 100), you can choose to set the following parameter in ice.properties, and Ice will generate resource group values for each line item in the billing file. Please be VERY careful about using this feature. Resource group values are generated by concatenating values of of all custom tags. If it results in a long list of resource group values, Ice performance will be greatly affected. Please make sure the custom tags exist in the header of the billing file.

          # specify your custom tags here. Multiple tags are delimited by ",". If specified, BasicResourceService will be used to generate resource groups for you.
          # PLEASE MAKE SURE you have limited number (e.g. < 100) of unique value combinations from your custom tags, otherwise Ice performance will be greatly affected.
          # update 20170906 by HouYu Li <hyli@pixelworks.com> : Now tags are not connected with "_" automatically. It supports custom combinations of tags. See Readme.md.
          ice.customTags=tag1,tag2

  You will need to ensure that any tag you wish to use in ICE is ticked in the "Manage Cost allocation report" page here: https://portal.aws.amazon.com/gp/aws/developer/account?ie=UTF8&action=cost-allocation-report

  Any tag that you have created yourself (as opposed to being automatically generated by AWS) will require you to use the ice.customTags= parameter in the following way. See this example:

  ice.customTags=user:CostCenter,User:Environment

  Update 20170906, by hyli@pixelworks.com .
    Tags are not connected with "_" automatically. Each tag, as separated by "," , will form a standalone resource group. You can setup custom combinations of tags in the "ice.customTags" options. See following example.

    Example: You have following custom tags for an EC2 instance.
        Department : IT
        Project : awsusage
        Owner : hyli

    Then you can use following settings to form 2 resource groups.
        ice.customTags=user:Department&user:Project,user:Owner

    The resulting 2 resource groups will be:
        IT + awsuage
        hyli

7. Weekly cost email per application group (*)

  If you have resource service enabled, you can use BasicWeeklyCostEmailService to send weekly cost emails. You can use the default BasicS3ApplicationGroupService, or you can have your own ApplicationGroupService implementation.

8. Throughput metric service (*)

  You may also want to show your organization's throughput metric alongside usage and cost. You can choose to implement interface ThroughputMetricService, or you can simply use the existing BasicThroughputMetricService. Using BasicThroughputMetricService requires the throughput metric data to be stores monthly in files with names like <filePrefix>_2013_04, <filePrefix>_2013_05. Data in files should be delimited by new lines. <filePrefix> is specified when you create BasicThroughputMetricService instance.

9. Blended Costs
  By default, unblended costs are shown. You can show Blended costs with the following configuration:

        ice.use_blended=true

10. Extra Grails configuration file

  If you need to setup custom Grails settings, you can specify an additional configuration file to be loaded by Grails by setting the ``ice.config.location`` system property to the location of that file.

  See http://docs.grails.org/2.4.4/guide/single.html#configExternalized for more information.

## Example IAM Permissions

Grant the following permissions to either an instance role, or the user running the reports:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1421551747000",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeReservedInstances",
        "ec2:DescribeReservedInstancesOfferings"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "Stmt1418665415000",
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListAllMyBuckets",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::work-bucket-name/*"
      ]
    },
    {
      "Sid": "Stmt1418665415001",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::work-bucket-name"
      ]
    },
    {
      "Sid": "Stmt1418665415000a",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListAllMyBuckets",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::billing-reports-bucket/*"
      ]
    },
    {
      "Sid": "Stmt1418665415001a",
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::billing-reports-bucket"
      ]
    }
  ]
}
```


## Support

Please use the [Ice Google Group](https://groups.google.com/d/forum/iceusers) for general questions and discussion.

## Download Snapshot Builds
Download snapshot builds here: [https://netflixoss.ci.cloudbees.com/job/ice-master/](https://netflixoss.ci.cloudbees.com/job/ice-master/)

## License

Copyright 2014 Netflix, Inc.

Licensed under the Apache License, Version 2.0 (the “License”); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
