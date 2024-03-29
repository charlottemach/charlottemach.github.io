---
layout: post
title: "Using AWS Lambda to write from S3 to DynamoDB"
date: 2021-12-16
tags: python lambda s3 dynamodb cdk
---

Creating a lambda function triggered by an S3 event can be done in different ways. Using a blueprint, the console or AWS CDK.

We'll take a look at creating a Python Lambda function created from Typescript CDK code.

### CDK Resources

We create a separate CDK Stack for an S3 bucket
```javascript
    const bucket = new s3.Bucket(this, 'S3Bucket', {
      bucketName: 'somebucketnamethatsnottaken',
      autoDeleteObjects: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      encryption: s3.BucketEncryption.S3_MANAGED,
    });
    bucket.grantRead(new AccountRootPrincipal());
```
a DynamoDB table
```javascript
    const dynamoTable = new Table(this, 'DynamoDb', {
      partitionKey: {
        name: 'id',
        type: AttributeType.STRING
      },
      tableName: 'sometable',
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });
```
and finally a Lambda function.
```javascript
    const lambdaFunction = new lambda.Function(this, "Lambda", {
      runtime: lambda.Runtime.PYTHON_3_9,
      code: new lambda.AssetCode("./lambda"),
      handler: "lambda_function.lambda_handler",
      vpc: vpc,
    });

    // this grants the Lambda function rights to read from the bucket
    bucket.grantRead(lambdaFunction);
    // this grants the Lambda function right to write to DynamoDB
    dynamoTable.grantReadWriteData(lambdaFunction);

    // this adds a trigger for any time an object is put into the bucket
    const s3PutEventSource = new lambdaEventSources.S3EventSource(bucket, {
      events: [
        s3.EventType.OBJECT_CREATED_PUT
      ]
    });
    lambdaFunction.addEventSource(s3PutEventSource);
```
The typescript files will usually be in the `lib` folder (easy to get starting using the [AWS docs](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html#getting_started_bootstrap)) and are added to the app in `bin`. The above Lambda function code is expected to be found in a file `lambda/lambda_function.py` in the same directory as `bin` and `lib`.

### Python code

This function consists of two simple pieces, reading the CSV file from S3 and writing its contents to the DynamoDB table.

```python
"""Imports CSV from S3 to DynamoDB"""
import uuid
import boto3

TABLE = 'sometable'

s3 = boto3.resource('s3')
db_table = boto3.resource('dynamodb').Table(TABLE)

def write_to_dynamodb(header, arr):
    """Assigns random id to csv line and writes to table"""
    items = dict(zip(header, arr))
    items['id'] = str(uuid.uuid4())
    return db_table.put_item(Item=items)

def lambda_handler(event, context):
    """Gets CSV and splits into lines"""
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    tmp_file = '/tmp/' + key

    s3.meta.client.download_file(bucket, key, tmp_file)
    with open(tmp_file, 'r') as f:
        header = next(f).rstrip().split(';')
        for line in f:
            if line != '\n':
                arr = line.rstrip().split(';')
                write_to_dynamodb(header, arr)
    return {'statusCode': 200}
```

### Deployment

Now all you have to do is create your stack using `cdk deploy` and testing the function by adding a CCSV file to your S3 bucket.
```csv
User; Id; First Name; Age
spidey;123;Peter;22
mi6;007;James;60
McTestface;7;testy;4
```
