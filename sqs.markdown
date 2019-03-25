# SQS

Basic operations for an SQS queue using boto3

## SQS Queue Creation

```python
import boto3
sqs = boto3.resource("sqs")

# Safe to call even if the queue already exists
q = sqs.create_queue(QueueName="my-queue")
```

## SQS Queue Lookup

```python
# Safely get a queue by name that already exists
try:
    q = sqs.get_queue_by_name(QueueName='my-queue')
except sqs.meta.client.exceptions.QueueDoesNotExist:
    # handle missing queue
    pass
```

## SQS Queue Producer

```python
import boto3
sqs = boto3.resource("sqs")
q = sqs.get_queue_by_name(QueueName='my-queue')
q.send_message(MessageBody='world')
```

## SQS Queue Bulk Producer

```python
import boto3
sqs = boto3.resource("sqs")
q = sqs.create_queue(QueueName="my-queue")

# Accepts up to 10 messages at a time:
q.send_messages(Entries=[
    {'MessageBody': 'hello'},
    {'MessageBody': 'world'},
])
```

## SQS Queue Consumer

```python
import boto3
sqs = boto3.resource("sqs")
q = sqs.create_queue(QueueName="my-queue")
for message in q.receive_messages():
    message.body == 'world'
    
    # if you don't delete the message it will be retried
    message.delete()
```   

## Lambda SQS Queue Consumer

```python
import boto3

def lambda_handler(event, context):
    """
    Lambda may recieve multiple queue messages, depending on SQS config. 
    
    Messages are automatically deleted if the lambda exits normally.
    
    If you can't process a message, raise an exception. 
    
    If you need to raise an exception but processing other messages passed
    to this lambda a second time will be a problem, configure SQS to send 
    one message at a time to the lambda.
    """"
    for message in event['Records']:
        message['body'] == 'world'
```

## SQS Queue Teardown

```python
import boto3
sqs = boto3.resource("sqs")
q = sqs.get_queue_by_name(QueueName='my-queue')
q.delete()
```
