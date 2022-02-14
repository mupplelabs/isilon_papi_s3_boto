<div class="cell markdown">

# Python with BOTO3 and OneFS Platform API for S3 Failover

</div>

<div class="cell markdown">

**Assumptions:**

1.  S3 is configured and enabled on both clusters
2.  buckets are configured on both clusters
3.  users are equally created on both clusters
4.  S3 Users are configured with the following RBAC Permission
    ISI\_PRIV\_LOGIN\_PAPI and (ptional) ISI\_PRIV\_NS\_IFS\_ACCESS
5.  each user has already S3 Keys generated on both clusters If not the
    Function newS3Key() below can be used to generate them.
6.  S3 Buckets are equally set up on both clusters

</div>

<div class="cell markdown">

## Setup and Requirements

</div>

<div class="cell code" data-execution_count="3">

``` python
# Requirements
import boto3
import urllib3
from pprint import pprint

# isi_sdk 
from __future__ import print_function
import time
import isi_sdk_9_1_0
from isi_sdk_9_1_0.rest import ApiException

# Disable SSL selfsigned certificate warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
```

</div>

<div class="cell code" data-execution_count="15">

``` python
# Config Variables OneFS
myHost = '192.168.188.192' # source cluster
#myHost = '192.168.188.201' # target cluster
myEndpointURL = 'https://' + myHost + ':9021'
OneFSUser = 'root'
OneFSPw = 'a'
```

</div>

<div class="cell markdown">

## Get the S3 Keys using the OneFS SDK

</div>

<div class="cell code" data-execution_count="7">

``` python
def myS3keys(user, password, papiHost):
    # Configure HTTP
    configuration = isi_sdk_9_1_0.Configuration()
    configuration.username = user
    configuration.password = password
    configuration.verify_ssl = False
    configuration.host = "https://" + papiHost + ":8080"

    # create an instance of the API class
    api_instance = isi_sdk_9_1_0.ProtocolsApi(isi_sdk_9_1_0.ApiClient(configuration))

    try:
        api_response = api_instance.list_s3_mykeys()
    except ApiException as e:
        print("Exception when calling ProtocolsApi->list_s3_mykeys: %s\n" % e)
    else :
        return api_response
        
my = myS3keys(OneFSUser, OneFSPw, myHost)
pprint(my)
myAccessID = my.keys.access_id
mySecretKey = my.keys.secret_key
```
</div>

<div class="cell markdown">

## Generate a new S3 Key using the OneFS SDK

</div>

<div class="cell code" data-execution_count="6">

``` python
def newS3key(user, password, papiHost):
    # Configure HTTP
    configuration = isi_sdk_9_1_0.Configuration()
    configuration.username = user
    configuration.password = password
    configuration.verify_ssl = False
    configuration.host = "https://" + papiHost + ":8080"

    # create an instance of the API class
    api_instance = isi_sdk_9_1_0.ProtocolsApi(isi_sdk_9_1_0.ApiClient(configuration))
    s3_mykey = isi_sdk_9_1_0.S3Key() # S3Key | 
    force = True # bool | Forces to create new key. (optional)

    try:
        api_response = api_instance.create_s3_mykey(s3_mykey, force=force)
    except ApiException as e:
        print("Exception when calling ProtocolsApi->create_s3_mykey: %s\n" % e)
    else :
        return api_response
        
my = newS3key(OneFSUser, OneFSPw, myHost)
pprint(my)
myAccessID = my.keys.access_id
mySecretKey = my.keys.secret_key
```

</div>

<div class="cell markdown">

## List objects with BOTO - lookup a new Key Pair in case of a authentication error...

**Note: There are other reasons why an S3 call could return a 403 Error.
=\> This is for demonstraion purpose only\!** <br> It can be potentially
dangerous to automatially get the keys and just retry with out further
sanity checks.

</div>

<div class="cell code" data-execution_count="17">

``` python
def listObjects(myAccID, mySecret, myURL, bucket):
    global myAccessID
    global mySecretKey
    # S3 Config
    # disable unsigned SSL warning
    urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

    from botocore import config
    from botocore.exceptions import ClientError

    my_config = config.Config(
        region_name = "",
        signature_version = 'v4',
        retries = {
            'max_attempts' : 10,
            'mode' : 'standard'
        }
    )

    # create a S3 Session Object
    session = boto3.session.Session()

    s3_client = session.client(
        service_name='s3',
        aws_access_key_id=myAccID,
        aws_secret_access_key=mySecret,
        endpoint_url=myURL,
        use_ssl=True,
        verify=False
    )

    try:
        pprint(s3_client.list_objects(Bucket=bucket))
    except ClientError as e:
        eStatus = e.response['ResponseMetadata']['HTTPStatusCode']
        print(e.response['Error']['Message'])
        if eStatus == 403 :
            print("Time to add the call to get a new key")
            my = myS3keys(OneFSUser, OneFSPw, myHost)
            pprint(my)
            myAccessID = my.keys.access_id
            mySecretKey = my.keys.secret_key
            listObjects(myAccessID, mySecretKey, myURL, bucket)

listObjects(myAccessID, mySecretKey, myEndpointURL, 'omnibot')
```

</div>
