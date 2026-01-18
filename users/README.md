# 6. Users and Identity

<br><br>
## 6.1 Introduction

IAM roles in AWS
Authentication, credentials with S3v4 and JWT

<br><br>
---
## 6.2 Creating a user
So far, we have been using the **admin** user but we surely want to identifiy users individually.

```shell
ALICE_ACCESS_KEY="alice"
ALICE_SECRET_KEY="myPass12345"

mc admin user add minio-101 $ALICE_ACCESS_KEY $ALICE_SECRET_KEY

mc alias set alice-minio-101 http://10.1.10.101:9000 "$ALICE_ACCESS_KEY" "$ALICE_SECRET_KEY"

```

if you would try accessing object1 in bucket1 it should fail:
```shell
mc ls alice-minio-101/bucket1/
```

You should get a **mc: <ERROR> Unable to list folder. Access Denied.**

why? because, except **admin** users do not have any permissions by default and credentials alone do nothing; so after creating a user, we should create an IAM Policy and assign it to the user.

<br><br>
---
## 6.3 Create the IAM Policy

Create a json file called readonly.json:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::bucket1/*"]
    }
  ]
}
```

then create the policy on the minio node and assign it to **alice**:
```shell
mc admin policy create minio-101 readonly readonly.json
mc admin policy attach minio-101 readonly --user alice
```

Now let's try accessing the object for alice:

```shell
mc head alice-minio-101/bucket1/exercice1/object1.txt
This is an object for Bucket1
```

Now if you try to upload a file to bucket1:

```shell
echo "This is another file Alice cannot PUT but she can GET" > /tmp/object2.txt
mc cp /tmp/object2.txt alice-minio-101/bucket1/exercice1/object2.txt
mc: <ERROR> Failed to copy `/tmp/object2.txt`. Insufficient permissions to access this path `http://10.1.10.101:9000/bucket1/exercice1/object2.txt`
```
The request fails because you did not add PUT and POST permission to the bucket in the IAM Policy associated with alice.


<br><br>
---
## 6.4 Wildcard Policies

To make it easier to manage, we could simplify drasticaly the IAM Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:"],
      "Resource": ["*"]
    }
  ]
}
```
This is *admin* access!. It is often used during testing and somehow forgotten leading to full access on all objects and all buckets.

<br><br>
---
## 6.5 Network filtering

we want all the clients - wheteher they are humans or apps - to access their allowed buckets through the BIG-IP only.
All S3 vendors includes ACLs on their solutions

add the following configuration to allow direct requests working around the BIG-IP if routing and firewalling are not configured properly.

By the way, let's secure a bit further our monitors so the bucket **monitor** is only accessible by our BIG-IP Self IP addresses:

```shell
vi monitor-bucket-policy.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBigIPHealthcheckByIP",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::monitor/monitor"],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.1.10.20/32"
        }
      }
    }
  ]
}
```


Use exact object path, not *

Principal: "*" is mandatory for anonymous access

aws:SourceIp must match the IP MinIO actually sees i.e. the IP address of the BIG-IP Self-IP or the SNAT IP address (you can use /24 if you want to grant to the whole subnet.

```shell
mc anonymous set-json monitor-bucket-policy.json minio-105/monitor/monitor
mc anonymous set-json monitor-bucket-policy.json minio-104/monitor/monitor
```

Now if you try to curl from the client, you should be reject:

```shell
curl http://10.1.10.105:9000/monitor/monitor -vvv
*   Trying 10.1.10.105:9000...
* Connected to 10.1.10.105 (10.1.10.105) port 9000 (#0)
> GET /monitor/monitor HTTP/1.1
> Host: 10.1.10.105:9000
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Accept-Ranges: bytes
< Content-Length: 319
< Content-Type: application/xml
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 188BA5DE528F7055
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 6972
< X-Ratelimit-Remaining: 6972
< X-Xss-Protection: 1; mode=block
< Date: Sat, 17 Jan 2026 22:40:52 GMT
<
<?xml version="1.0" encoding="UTF-8"?>
* Connection #0 to host 10.1.10.105 left intact
<Error><Code>AccessDenied</Code><Message>Access Denied.</Message><Key>monitor</Key><BucketName>monitor</BucketName><Resource>/monitor/monitor</Resource><RequestId>188BA5DE528F7055</RequestId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>
```


whereas when you run it from the big-ip shell it works:

```shell
curl http://10.1.10.105:9000/monitor/monitor -vvv
*   Trying 10.1.10.105...
* Connected to 10.1.10.105 (10.1.10.105) port 9000 (#0)
> GET /monitor/monitor HTTP/1.1
> Host: 10.1.10.105:9000
> User-Agent: curl/7.47.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Content-Length: 3
< Content-Type: text/plain
< ETag: "d36f8f9425c4a8000ad9c4a97185aca5"
< Last-Modified: Sat, 17 Jan 2026 22:13:38 GMT
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 188BA5C07D7A3155
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 6972
< X-Ratelimit-Remaining: 6972
< X-Xss-Protection: 1; mode=block
< Date: Sat, 17 Jan 2026 22:38:44 GMT
<
OK
* Connection #0 to host 10.1.10.105 left intact
```



<br><br>
---
## 6.6 BIG-IP Recommandations and configurations
### 6.7.1 Goals

We have two major goals: 
- to control that only allowed users can access S3 objects (prevent non allowed users)
- to control that an authenticated user can only access what they are supposed to access (separation of concerns and privilege escalation)


<br><br>

### 6.7.2 Reject Anonymous S3 requests

A request is considered **anonymous** if **no authentication information** is present. 

We have seen in the **basics** session the anatomy of an authenticated and anonymous requests. It contains the following headers:
- Authorization: AWS4-HMAC-SHA256 Credential=admin/20260113/us-east-1/s3/aws4_request,SignedHeaders=host;x-amz-content-sha256;x-amz-date;x-amz-decoded-content-length,Signature=**REDACTED**
- X-Amz-Content-Sha256: STREAMING-AWS4-HMAC-SHA256-PAYLOAD

If **none** of these are present the request is anonymous.

Here what we want to process on the BIG-IP protecting our S3 services:
- Detect authentication inspecting the HTTP headers
- Block anonymous access early
- Return a valid S3 XML error so it does not break any S3 client or SDK.
- Avoid TCP resets or plain HTTP errors
- Should be compatible with:
  - MinIO
  - AWS S3
  - Dell ECS
  - NetApp StorageGRID

```tcl
when HTTP_REQUEST {

    set is_authenticated 0

    if { [HTTP::header exists "Authorization"] } {
        set auth_hdr [HTTP::header value "Authorization"]

        if { $auth_hdr starts_with "AWS4-HMAC-SHA256" } {
            set is_authenticated 1
        }
    }

    if { !$is_authenticated } {

        HTTP::respond 403 content \
"<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<Error><Code>AccessDenied</Code><Message>Access Denied.</Message><Resource>[HTTP::path]</Resource><RequestId>[clock clicks]</RequestId></Error>" \
"Content-Type" "application/xml"

        return
    }
}
```


### 6.7.3 separate Virtual Server roles
Each S3 nodes within a cluster is reachable by the same IP address regardless it is for a GET, POST, PUT, DELETE or LIST action. But the processes and people in charge of PUT'ing objects in buckets are different than the apps or users requesting these objects. 
Same, if you have objects that can be consumed by a user, is that user granted to DELETE these objects?

This is why IAM Policies are meant but it can easily be messed up a policy can be associated to multiple users, on multiple buckets and with different grants. The first one that matches is the one that is applied.

<!-- TO VERIFY THIS STATEMENT -->


What we can do to mitigate privilege abuses is to separate Virtual Services based on the desired role:

- Virtual Server 1: read.s3.f5demo.com (10.1.10.103)
  LTM Policy: readPolicy
    rule1:
    conditions (all match)
        hostname read.s3.f5demo.com
        client IP address 10.1.10.0/24
        method: GET
    actions: 
        forward to pool
        otherwise drop

- Virtual Server 1: write.s3.f5demo.com (10.1.10.104)
  LTM Policy: writePolicy
    rule1:
    conditions (all match)
        hostname read.s3.f5demo.com
        client IP address 10.1.10.0/24
        method: GET, PUT
    actions: 
        forward to pool
        otherwise drop

- Virtual Server 1: delete.s3.f5demo.com (10.1.10.105)
  LTM Policy: readPolicy
    rule1:
    conditions (all match)
        hostname delete.s3.f5demo.com
        client IP address 10.1.10.0/24
        method: DELETE
    actions: 
        forward to pool
        otherwise drop

 <!--Verify if that scales or if required to separate to different Virtual Servers-->       



<br><br>
[Back to Agenda](/README.md)
