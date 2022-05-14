# rainbow-fintech
An offline & mobile first solution for protection and ease of investing for underserved population.

## Tech
- Backend: NodeJS
- IAAS:
    1. Db: `AWS Dynamo`
    2. Compute: `AWS Lambda`
    3. Gateway: `AWS API Gateway`
    4. Cache: `AWS Elasticache`
    5. Broker: `AWS SQS`
- CPAAS:
  - `Expensive`, `difficult`, `scalable` (evaluated, but not used in this project):
    - Receiving:
      - [textlocal](https://textlocal.in) +919220592205 `NLLG7` (prefix)
      - [ifttt (sms → webhook)](https://ifttt.com/android_messages)
    - Sending:
      - [twilio](https://twilio.com)
      - [gupshup](https://enterprise.smsgupshup.com)
  - `Inexpensive`, `easy`, `very small scale` (used in this project):
    - Android phone connected to internet, is installed with 2 specific apps, [IFTTT](https://ifttt.com/android_device) & [Pushbullet](https://www.pushbullet.com/)
    - Users send sms to this phone, an ifttt recipe triggers on incoming sms containing a keyword
    - Trigger reads sms content and POST's to Webhook which is hosted in AWS
    - Webhook processes the sms content & calls pushbullet api to send sms via same android phone
    - Inexpensive, since only sms sending charge as per carrier tariff
    - No need to dealing with CPAAS service providers
    - Caveats:
      - Android phone must be online all the time with IFTTT and Pushbullet running in the background

## Architecture
![Architecture](https://github.com/abhisekpadhi/rain-bank/blob/main/docs/infra.png "aws design")

## Db schema
- table: `userAccountIdMapping`
```
phone (pk)
id
```

- table: `userAccount`
```
id (pk)
name
phone
loc
pan
verification
balance
currentActive
createdAt
 ```

- table: `userTxn`
```
txnId (pk)
firstParty
secondParty
requestType
money
status
createdAt
currentActive
```

- table: `floatingCashRequest`
indicates latest ask for floating cash request id
```
phone(pk)
id
```

- table: `userRequest`
```
id (pk)
phone
requestType
where
money
otherAccount
status
extraInfo
currentActive
createdAt
```

- table: `userAccountLedger`
```
id (pk)
phone
op
note
money
openingBalance
createdAt
```

- table `userBucket`
```
phoneWithBucketName (pk)
balance
```

- enums

```javascript
const requestType = {
  register: 'register',
  collect: 'collect',
  pay: 'pay',
  transfer: 'transfer',
  seeSaved: 'seeSaved',
  sip: 'sip',
  bucket: 'bucket',
  findDeposit: 'findDeposit',
  findWithdraw: 'findWithdraw'
}

const txnStatus = {
  created: 'created',
  success: 'success',
  failed: 'failed',
}

const op = {
    debit: 'debit',
    credit: 'credit'
}
```

## event schema
- Message body published to sqs
```json
{ 
  "sender": "919439831236", 
  "content": "rain balance"
}
```

## Docs
- [Dynamodb document client](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/dynamodb-example-document-client.html)
- Setup awscli for deploy script `deploy.sh`:
```
python3 -m venv .venv
.venv/bin/pip install awscli
```
- To build & deploy:

sink (receive incoming sms)
```shell
./deploy.sh sink
```

worker (process sms)
```shell
./deploy.sh worker
```

- To build lambda layer

1. create directory structure:
```shell
mkdir -p aws-sdk-layer/nodejs
cd aws-sdk-layer/nodejs
```

2. Use amazon linux 2 compatible environment to install required packages:
```shell
docker run --entrypoint "" -v "$PWD":/var/task "public.ecr.aws/lambda/nodejs:14" /bin/sh -c "npm install aws-sdk redis node-fetch@2.6.7; exit"
```

3. zip it
```shell
zip -r ../package.zip ../
```

4. create layer
```shell
Name: <anything>
Upload zip
Runtimes: Node.js 14.x
```

5. Use the layer in lambda function.
This prevents bundling `node_modules` with zip that is deployed to lambda. Reduces the lambda zip package size and prevents version conflicts with lambda preinstalled packages.

---
**Note:** If `elasticache` is in private subnet
- Lambda function needs to be inside vpc and must be associated with privates subnets same as `elasticache`
- AWS VPC endpoints needs to be setup for - `SQS` & `DynamoDB`, since requests for these services travels through internet
- Environment variables (Key: Value), needs to be set for lambda functions:
```shell
ACCESS_KEY_ID:	---
QUEUE_URL:	https://---.fifo
REDIS_ENDPOINT:	redis://---.aps1.cache.amazonaws.com:6379
REGION:	ap-south-1
SECRET_ACCESS_KEY:	---
```
- Modify config: timeout to 30seconds at least, Memory to 256mb
- Add follwing policies to the role (not ideal setup, just a shotgun approach):
```shell
AWSLambdaBasicExecutionRole-64...ef	(already exist, add the remaining 👇)
AmazonSQSFullAccess
AmazonElastiCacheFullAccess
AmazonDynamoDBFullAccess
AWSLambdaDynamoDBExecutionRole
AdministratorAccess
AWSLambdaSQSQueueExecutionRole
AWSLambdaInvocation-DynamoDB
AWSLambdaVPCAccessExecutionRole
```
---

Tests checklist (manual)
---
- [x] Register new account
- [x] ATM Deposit
- [x] ATM Withdraw
- [x] See account balance
- [x] Find ATM for deposit
- [x] Find ATM for withdrawal
- [x] Find guranteed return investment opportunity
- [x] Hassle free systematic & flexible investment
