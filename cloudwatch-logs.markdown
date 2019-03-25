# Cloudwatch Logs

Cloudwatch Logs are like text log files in /var/log/ like we all know and love,
but it also has some neat features and you don't need to worry about losing
your log files if you bounce your EC2 instance or have a hard-drive failure.

## Creating a Log Group

aka Log directory

```python
import boto3

cwlogs = boto3.client('logs')

cwlogs.create_log_group(logGroupName='/myapp/errors/')
```

## Creating a Log Stream

aka "Log file", but better.

Better because each log in a log stream can have line breaks in it. 
It also understands JSON.

```python
from datetime import date
import boto3

cwlogs = boto3.client('logs')

cwlogs.create_log_stream(
    logGroupName='/myapp/errors/',
    logStreamName=f'{date.today():%Y-%m-%d}'
)
```

## Writing a log

Here's your hello world (assuming you already created the group and stream).

```python
from datetime import date
import time
import boto3

cwlogs = boto3.client('logs')
cwlogs.put_log_events(
    logGroupName='/myapp/errors/',
    logStreamName=f'{date.today():%Y-%m-%d}',
    logEvents=[
        {   
            'message': "I've got a bad feeling about this.",
            'timestamp': int(time.time() * 1000), # required
        },
    ],
)
```

## Reading logs

```python
import boto3

cwlogs = boto3.client('logs')

cwlogs.get_log_events(
    logGroupName='/myapp/errors/',
    logStreamName=f'{date.today():%Y-%m-%d}',
    startFromHead=True
)['events']
```


## Time to put on big kid pants

This ain't gonna be pretty.

This is how you should actually create a log group and log stream.

```python
import boto3

cwlogs = boto3.client('logs')

def setup_log(group, stream):
    try:
        cwlogs.create_log_stream(
            logGroupName=group,
            logStreamName=stream
        )
    except cwlogs.exceptions.ResourceNotFoundException:
        cwlogs.create_log_group(logGroupName=group)
        cwlogs.create_log_stream(
            logGroupName=group,
            logStreamName=stream
        )
    except cwlogs.exceptions.ResourceAlreadyExistsException:
        # Already exists, we didn't need to do anything
        pass
```

Usually the group will already be there. Better to avoid making extra API calls on every call, trying to create a group that
is already there. We just hit the exception once on the first try.

### Sequence Tokens

When you write a log, CloudWatch give back a token which is a handle
to the point in the log you're written up to. 

Like the seek position of a file. Except you can't just say "put it at the end please." 

Game Plan:

 1. Try to write the log
 2. If it complains, look up the latest Sequence Token and try again. 

```python
import time
from datetime import date
import boto3

cwlogs = boto3.client('logs')

# ... this uses the setup_log() from above...

def make_log_event(message):
    return {
        'message': message,
        'timestamp': int(time.time() * 1000), # required
    }

def write_log(group, stream, message, sequence_token=None):
    try:
        res = cwlogs.put_log_events(
            logGroupName=group,
            logStreamName=stream,
            logEvents=[
                make_log_event(message)
            ],
            
            # include sequenceToken kwarg *if* we got one
            **({'sequenceToken': sequence_token} if sequence_token else {})
        )
        return res['nextSequenceToken']
        
    # Log Group or Stream didn't exist yet:
    except cwlogs.exceptions.ResourceNotFoundException:
        setup_log(group, stream)
        return write_log(group, stream, message)
    
    # Sequence token was missing/bad/already used:
    except (cwlogs.exceptions.InvalidSequenceTokenException, cwlogs.exceptions.DataAlreadyAcceptedException):
        for log_stream in cwlogs.describe_log_streams(logGroupName=group, logStreamNamePrefix=stream)['logStreams']:
            if log_stream['logStreamName'] == stream:
                sequence_token = log_stream['uploadSequenceToken']
                break
        
        # retry with sequence token
        return write_log(group, stream, message, sequence_token)
    

# Example use
group = '/myapp/errors/'
stream = f'{date.today():%Y-%m-%d}'

next_sequence_token = write_log(group, stream, "I love you")
next_sequence_token = write_log(group, stream, "I know", next_sequence_token)
```

Reading logs also uses tokens as a sort-of file seek position. We'll
use a generator in case some boor wrote 500 GB of logs.

## Reading logs, big kid version

```python
import boto3

cwlogs = boto3.client('logs')

def read_logs(group, stream):
    try:
        log_events = cwlogs.get_log_events(
            logGroupName=group,
            logStreamName=stream,
            startFromHead=True
        )
    except cwlogs.exceptions.ResourceNotFoundException:
        return

    while True:
        if not log_events['events']:
            # alternatively we could sleep or continue here and read more
            # logs as they are written
            break

        for event in log_events['events']:
            yield event

        log_events = cwlogs.get_log_events(
            logGroupName=group,
            logStreamName=stream,
            nextToken=log_events['nextForwardToken']
        )
        

# example use
for log_entry in read_logs("/myapp/errors/", f'{date.today():%Y-%m-%d}'):
    print(log_entry)
```

Finally, you could make a nice abstraction

```python
class Logger(object):
    def __init__(self, group, stream):
        self.group = group
        self.stream = stream
        self.token = None
    
    def __call__(self, message):
        self.token = write_log(self.group, self.stream, message, self.token)
        
    def __iter__(self):
        return read_logs(self.group, self.stream)


# Example usage:

log = Logger("/myapp/errors/", f'{date.today():%Y-%m-%d}')

log("Never tell me the odds")

for message in log:
    print(f'{message["timestamp"]}: {message["message"]}')
```

## Leveling up with JSON

Everything so far can be accomplished with Python’s built-in `logging` module and [Watchtower](https://github.com/kislyuk/watchtower).

Let’s get into the black magic.

First I'll fill up the log with random garbage

```python
import json
import random
import time
import uuid
import boto3

cwlogs = boto3.client('logs')

for i in range(100):
    # if you want to know for sure you are working with an
    # empty stream, give it a unique name :)
    group = '/myapp/test/'
    stream = uuid.uuid4().hex

    # function from above
    setup_log(group, stream)
    
    # Limits on bulk-creating log events:
    #   - max of 1 MB of log data per call
    #   - max of 10k log events per call
    #   - time stamps cannot span more than 24 hours
    cwlogs.put_log_events(
        logGroupName=group,
        logStreamName=stream,
        logEvents=[
            {
                # generate fake data:
                'message': json.dumps({
                    'i': i,
                    'iEven': not (i % 2),
                    'nest': {'big': 'bird'},
                    'values': [random.random(), random.random(), random.random()],
                    'uuidname': uuid.uuid4().hex[:2],
                }),
                'timestamp': int(time.time() * 1000),
            }
            for _ in range(1000)
        ]
    )
```

### Now let's search!

Cloudwatch Logs know JSON.

Oh… wait.

Uh oh. 

We only know the Log Group name (`/myapp/test/`). nbd.

```python
import boto3

cwlogs = boto3.client('logs')

cwlogs.filter_log_events(
    logGroupName='/myapp/test/',
    filterPattern='{$.uuidname = "aa"}',
    
    # This is for pagination like we did in `read_logs()`
    # nextToken='...',
    
    # Slower but combines results from multiple streams in
    # a single response. Usually you want False
    interleaved=True
)

# The Log data: 
response['events']

# The streams that were searched:
response['searchedLogStreams'] 
```

Let’s play! 

Wrap it in curly braces to indicate it’s a JSON filter. The `$` represents the JSON object in the log.

```python
import boto3

cwlogs = boto3.client('logs')

GROUP = '/myapp/test/'

def get_logs(json_filter):
    return cwlogs.filter_log_events(
        logGroupName=GROUP,
        filterPattern=json_filter,
    )['events']
    
# Some examples:
get_logs('{ $.i > 90 }')
get_logs('{ $.i = 90 }')
get_logs('{ $.i <= 90 }')
get_logs('{ $.i != 90 }')
get_logs('{ $.uuidname = "aa" }')

# checking for true/false/null
get_logs('{ $.iEven IS TRUE }') 
get_logs('{ $.iEven IS FALSE }') 
get_logs('{ $.iEven IS NULL }')  # no matches in our test data

# nested objects
get_logs('{ $.nest.big = "bird" }')

# array indexing
get_logs('{ $.values[0] < 0.1 }')

# anything with uuidname starting with "a" (* is wildcard)
get_logs('{ $.uuidname = "a*" }')

# paren groupings and boolean AND/OR
get_logs('{ $.i != 95 && ($.uuidname = "aa" || $.uuidname = "ab") }')

# Check if a key is present
get_logs('{ $.missingValue NOT EXISTS }')

# null is separate from missing. Doesn't match any logs:
get_logs('{ $.missingValue IS NULL }')


```
