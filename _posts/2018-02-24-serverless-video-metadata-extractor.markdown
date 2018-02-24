---
layout: default
title:  "Build a Video Metadata Extraction Service with Serverless and AWS Rekognition"
description: "In this post I'll show you how to build an automated system for extracting the identities of celebrities from video content by using Serverless framework to link together a few AWS services."
date:   2018-02-24 08:28:24 +1100
readableDate: "February 24th, 2018"
categories: serverless
---

In this post I'll demonstrate how to build an automated system for extracting metadata from video content using the [Serverless Framework](https://serverless.com). We'll use AWS Rekognition's celebrity identification functionality to process mp4 files uploaded to an S3 bucket, then store the generated metadata in JSON format alongside the original video in S3. 

If this is your first time with Serverless it's probably worth having a run through the [AWS quick start guide](https://serverless.com/framework/docs/providers/aws/guide/quick-start/) first. However if you want to just jump straight in go ahead, as I'll cover some of the basics as we go.

For reference, you can find the full example code for this walk-through on my [Github](https://github.com/twosmalltrees/serverless-video-metadata-generator).

### What We'll Be Building

Before we actually get started with the implementation, it'll help to have an understanding of what we're trying to create.

1. A video file is uploaded to our S3 Bucket.  
2. This upload triggers a Lambda function (extractMetadata), which calls out to the AWS Rekognition startCelebrityRecognition endpoint to begin an analysis job.  
3. When the analysis job completes, Rekognition publishes a success message to an SNS topic.  
4. The SNS message triggers a second Lambda function (saveMetadata), which retrieves the generated celebrity metadata from Rekognition and saves it alongside the original video in S3.  

![Metadata Extractor Architecture]({{ "/assets/images/metadata-extractor-architecture.jpeg"}})

### Step 1: Basic Set Up

First, if you haven't already you'll need to globally install Serverless in order to run CLI commands. 

```bash
$ npm install -g serverless
```

Next up we'll create a new Serverless project:

```bash
$ serverless create --template aws-nodejs --path metadata-extractor
$ cd metadata-extractor
```

Note the `--template` and `--path` flags, used to specify the serverless template type (in this case aws-nodejs) and project directory (which will also be our project name). 

At this point if you `cd` into the project directory you'll see two files have been auto generated - `serverless.yml` and `handler.js`. These are the only files we'll need to create this service.  `serverless.yml` is where we define and configure the AWS resources requried for our service, and `handler.js` where we'll implement our Lambda code. 

### Step 2: Configuring AWS Resoures - serverless.yml

Lets start off with `serverless.yml`. On opening this file you'll see quite a lot of mostly commented code. This is provided as reference to the various configuration options avaialble in Serverless - so it's worth having a read through. Once you're done, delete everything! We'll start from scratch.

#### Defining a Few Custom Properties

First off, add the below to `serverless.yml`:

```yml
# serverless.yml

service: metadata-extractor

custom:
  bucketName: your-bucket-name-goes-here
  bucketArn: arn:aws:s3:::${self:custom.bucketName}/*
  snsTopicName: your-sns-topic-name-goes-here
  snsTopicArn: arn:aws:sns:${env:AWS_REGION}:${env:AWS_ACCOUNT_ID}:${self:custom.snsTopicName}
  snsPublishRoleName: snsPublishRole
  snsPublishRoleArn: arn:aws:iam::${env:AWS_ACCOUNT_ID}:role/${self:custom.snsPublishRoleName}

```

Looking at the above, you'll see that we've named the service `metadata-extractor`, and also define a number of custom properties:

* **bucketName** - The name of the uploads bucket. You'll probably want to rename this.
* **bucketARN** - The ARN of the upload bucket, constructed with the bucketName in the standard S3 ARN format.
* **snsTopicName** -  The name of the SNS topic that Rekognition will use to notify of job completion. Again, rename this to whatever you'd like.
* **snsTopicArn** - The ARN of the above SNS topic, constructed using the AWS region, AWS account id and topic name. Note that region and account id are references to environment variables.
* **snsPublishRoleName** - The name of an IAM role (that we'll define later), which is passed to Rekognition to allow publishing notifications to our SNS topic.
* **snsPublishRoleArn** - The ARN of the above named role.

Using the syntax `${self:custom.someVariableName}` we're able to reference these properties elsewhere within our serverless.yml file.

#### Setting Up Environment Variables and Extending the Lambda IAM Role

Still working in `serverless.yml`, add the following:

```yml
# serverless.yml, continued...

provider:
  name: aws
  runtime: nodejs6.10
  environment:
    SNS_PUBLISH_ROLE_ARN: ${self:custom.snsPublishRoleArn}
    SNS_TOPIC_ARN: ${self:custom.snsTopicArn}
  iamRoleStatements:
    - Effect: Allow
      Action: 
        - rekognition:StartCelebrityRecognition
        - rekognition:GetCelebrityRecognition
      Resource: '*'
    - Effect: Allow
      Action:
        - iam:GetRole
        - iam:PassRole
      Resource: ${self:custom.snsPublishRoleArn}
    - Effect: Allow
      Action:
        - s3:GetObject
        - s3:PutObject
      Resource: ${self:custom.bucketArn}

```

Here we're adding the provider configuration. This includes specifying the cloud services provider (aws), the runtime (nodejs6.10). We also define a couple of environment variables to be made available in the Lambda runtime - the SNS publishing role ARN, and the SNS topic ARN. These are defined through references to the custom properties we defined previously.

Aditionally, we extend the default IAM role of the Lambda functions with permissions to start and get the results of the Rekognition job, to get and pass the SNS publish role to Rekognition, and to get objections from and put objects into our S3 bucket.

#### Defining the Lambdas and Event Sources

Next up, you'll see that we've defined the two function mentioned earlier - `extractMetadata` and `saveMetadata`:

```yml
# serverless.yml, continued...

functions:
  extractMetadata:
    handler: handler.extractMetadata
    events:
      - s3: 
          bucket: ${self:custom.bucketName}
          event: s3:ObjectCreated:*
          rules:
            - suffix: .mp4
  saveMetadata:
    handler: handler.saveMetadata
    events: 
      - sns: ${self:custom.snsTopicName}

```

For `extractMetadata`, we map it to the extractMetadata handler via the handler property (the implementation for which we'll define later in handler.js). We also assign an event to act as a trigger for the function. As discussed previously, for the extractMetadata function this is a an upload (ObjectCreated) to the uploads bucket. 

We also set a rule that the uploaded file must end in .mp4 to trigger the Lambda invocation - It's **very important** to set this rule, as it prevents the Lambda from triggering when we save the generated JSON file - which would result in an infinite loop, and a rapidly growing AWS bill.

In the case of `saveMetadata`, we map it to the saveMetadata handler, and add the SNS queue as the event trigger. As with the S3 bucket, Serverless will ensure the SNS topic is created for us.

#### Defining a custom IAM Role to Provide Rekognition publishing rights to SNS

One last thing before we move on to the function implementation - we need to define a custom IAM role in the resources section of `serverless.yml`. This is the IAM Role that will be passed to AWS Rekognition to provide it with the required permissions to publish notifications to the SNS topic. 

Add the following:

```yml
# serverless.yml, continued...

resources:
  Resources:
    snsPublishRole:
      Type: AWS::IAM::Role
      Properties:
        RoleName: ${self:custom.snsPublishRoleName}
        AssumeRolePolicyDocument: 
          Version: '2012-10-17'
          Statement: 
            - Effect: Allow
              Principal: 
                Service: 
                  - rekognition.amazonaws.com
              Action: 
                - sts:AssumeRole
        Policies:
          - PolicyName:  snsPublishPolicy
            PolicyDocument: 
              Version: '2012-10-17'
              Statement: 
                - Effect: Allow
                  Action: 
                    - sns:Publish
                  Resource: ${self:custom.snsTopicArn}

```

### Step 3: Lambda Implementaion - handler.js

To finish off our metadata extraction service, we need to define the two handler functions referenced in `serverless.yml` (*extractMetadata* and *saveMetadata*). 

#### Kick Off Metadata Extraction

Lets start off with *extractMetadata*. Add the following to `handler.js`:

```js
// handler.js

const AWS = require('aws-sdk');
const rekognition = new AWS.Rekognition();

module.exports.extractMetadata = (event, context, callback) => {
  const bucketName = event.Records[0].s3.bucket.name;
  const objectKey = event.Records[0].s3.object.key;

  const params = {
    Video: {
      S3Object: {
        Bucket: bucketName,
        Name: objectKey
      }
    },
    NotificationChannel: {
      RoleArn: process.env.SNS_PUBLISH_ROLE_ARN,
      SNSTopicArn: process.env.SNS_TOPIC_ARN,
    },
  };

  rekognition.startCelebrityRecognition(params).promise()
    .then((res) => {
      const response = {
        statusCode: 200,
        body: JSON.stringify(res),
      };
      callback(null, response);      
    })
    .catch((err) => {
      callback(err, null);      
    });
};

```

In the code above, you'll see we first extract the bucketName and objectKey from the event source (the S3 upload).

From here it's just a matter of calling `startCelebrityRekognition`, provided by the AWS Rekognition SDK. We also pass through a set of params which identify the location of the video to analyse in S3, the SNS topic ARN to which the success notification is to be published, and the IAM Role ARN required to publish to the specified topic.

#### Get the Results and Save to S3

Next, we define *saveMetadata*:

```js
// handler.js, continued...

const s3 = new AWS.S3();

module.exports.saveMetadata = (event, context, callback) => {
  const message = JSON.parse(event.Records[0].Sns.Message);
  const jobId = message.JobId;   
  const bucketName = message.Video.S3Bucket;  
  const objectKey = message.Video.S3ObjectName;
  const metadataObjectKey = objectKey + '.people.json';


  const rekognitionParams = {
    JobId: jobId,
  };

  rekognition.getCelebrityRecognition(rekognitionParams).promise()
    .then((res) => {
      const s3Params = {
        Bucket: bucketName,
        Key: metadataObjectKey,
        Body: JSON.stringify(res),
      };
      s3.putObject(s3Params).promise()
        .then((res) => {
          const response = {
            statusCode: 200,
            body: JSON.stringify(res),
          };
          callback(null, response);
        });
    })
    .catch((err) => {
      callback(err, null); 
    });
};

```

Above, we pull out quite a few details from the event source (the SNS success notification), then make a call to getCelebrityRekognition (passing in the Rekognition jobId), which retrieves the generated celebrity recognition metadata. Using the S3 SDK, we then push a the metadata (as a .json file) to the location of the original video file.

### Wrapping Up

At this point, the service should be ready to test. The simplest way to do so is to open up the AWS console in browser, navigate to your bucket, and manually upload an `.mp4`. 

If all goes well, you should soon see the generated .json metadata file alongside the uploaded mp4. If Rekognition has done its job, this should identify any celebrities present in the video, along with matching timecodes for when they appeared. 

If something goes wrong, open Cloudwatch in the AWS console and start debugging from the Lambda logs. Also remember you can check out the full code on the [Github repo](https://github.com/twosmalltrees/serverless-video-metadata-generator).
