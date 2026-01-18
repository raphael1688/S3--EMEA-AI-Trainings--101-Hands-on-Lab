# 7. S3 Multipart

## 7.0 Introduction




## 7.1 What are S3 Multipart uploads and downloads
In S3, Multipart is a mechanism that allows a single object to be uploaded and downloaded in multiple independent parts, which are then assembled into one final object by the S3 service or the S3 client.
It was designed to make large-object transfers reliable, fast, and resumable.

Instead of uploading a large file in a single PUT request, the S3 client will:
- Split the object into multiple parts
- Upload each part independently
- Retry on failed parts

On the server, the S3 service will assemble them into one single object in the target bucket.

From the client’s perspective, the transaction is transparent and it looks like a normal S3 object.

The core reasons for multipart:
- Reliability
  - Large uploads are fragile
  - Network failures would restart the whole upload
  - Multipart allows resuming from the last successful part

- Performance
  - Parts can be uploaded in parallel
  - Uses multiple TCP streams
  - Maximizes bandwidth

- Scalability
  - Each part is handled independently

- Large object support
  - Required for objects > 5 GB (AWS S3 rule)
  - Maximum object size: 5 TB
  - Minimum part size: 5 MB (except last part)

<br><br>
## 7.2 Anatomy of multipart objects

The request flow when you do multi-part uploads is the following:

    1. POST /<bucket>/<object>?uploads                        # returns an UploadId
    2. PUT /<bucket>/<object>?partNumber=1&uploadId=XYZ       # upload the first part number
    3. PUT /<bucket>/<object>?partNumber=2&uploadId=XYZ       # upload the second part number
    4. POST /<bucket>/<object>?uploadId=XYZ                   # Comple the upload

The S3 node returns an automatic generated uploadId that is used for every part of the upload. All the parts have to land on the same node, otherwise the upload will fail


BUT, why can't we set a basic Source IP persistence. Because workloads may reside on Kubernetes clusters, you may never reach a fair load balancing across all S3 nodes.


## 7.3 Managing multipart through the BIG-IP
To handle the S3 multipart uploads and downloads we need to make sure our CARP persistence iRule takes into account the trailing information of the **uploadID** so any part request of an upload or download always land on the same pool member.

```tcl
when HTTP_REQUEST {
    set path [HTTP::path]
    set persist_key ""

    if { [regexp {^/([^/]+)/(.+)} $path -> bucket object] } {
        set persist_key "${bucket}/${object}"
    } else {
        set persist_key $path
    }

    persist carp $persist_key
    log local0. "PERSIST_KEY: $persist_key"

}
```

This iRule builds CARP persistence to ensure object locality in a distributed S3 deployment.

First, the iRule extracts, when it can, the bucket and the object names from the query.
If the 



<br><br>
[Back to Agenda](/README.md)


