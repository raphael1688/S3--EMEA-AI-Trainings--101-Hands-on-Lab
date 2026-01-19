# S3 Protocol Security Class – S3 Lab Guide

![F5](https://upload.wikimedia.org/wikipedia/fr/thumb/a/a3/F5_Networks_%28logo%29.svg/120px-F5_Networks_%28logo%29.svg.png?20200624213146)



## Agenda

:spiral_notepad:   [1. Introduction](introduction/README.md)

Here we are introducing the goals and duration of this lab.
<br>

:spiral_notepad:   [2. Lab Architecture](architecture/README.md)

This part explains the components of the UDF lab.
<br>

:spiral_notepad:   [3. Key concepts](concepts/README.md)

S3 glossary, mostly networking, HTTP and Security oriented.
Minio `mc` client cheat-sheet that can be used for self testings and PoCs.
<br>

:computer:   [4. S3 Basics](basics/README.md)

This part of the lab is hands-on on basic `mc`commands to understand how to create an S3 bucket, how to upload and download objects, understand the anatomy of an S3 request and how to deal with public buckets.
<br>

:computer:   [5. BIG-IP Configuration](bigip/README.md)

In this part of the lab, we will demonstrate how to leverage pre-built TCP and Client SSL profiles, introduced in TMOS v21.0 and how to configure persistence, oneConnect and monitors.
The lab is also pre-provisioned with F5 AppStudy Tool (AST) enriched with minio metrics scraped directly from the Minio nodes and event logs streamed from the BIG-IP.
<br>

:computer:   [6. Users and Identity](users/README.md)

Managing users authentication and privileges is probably the most complex part when configuring S3 services. You can be quickly overloaded when dealing with multiple users, buckets, and IAM policies.
The risk is introducing anonymous access, over-permissive policies and exposing confidential data to non allowed users. As F5, We cannot prevent from creating over-permissive policies but we can segregate traffic and enforce only required S3 actions and reject anonymous access to S3 services.
<br>

:computer:   [7. Multi-part GET or PUT](multipart/README.md)

Multipart uploads and downloads are very common in S3 requests because they are first mandatory when dealing with big objects and improve performances by parralelizing requests.
Therefore, you need to land every request parts on the same backend node so uploads and downloads are handled by the same S3 node. This is another example where CARP persistence is a strong requirement.
<br>

:computer:   [8. Pre-Signed URLs](presigned/README.md)

when the storage owner wants to share an object for downloading or an object placeholder for upload to different users or different applications without having to create them credentials and IAM privileges you can create pre-signed URLs.
Pre-signed URLs are generated curl commands with a pre-built bearer token valid for a limited period of time (by default 7 days). There are very convenient, but as all convenient things, they can be a concern with abused and not shared securely.
<br>

:computer:   [9. Metadata Poisoning and Metadata stored XSS](poisoning/README.md)

So far, we demonstrated where BIG-IP LTM can bring great values in delivering and secure the delivery of S3 objects. In this lab part, we will look into two use-cases where F5 A.WAF can protect your S3 services from attacks through S3 Metadata.
S3 Metadata headers are often underlooked and not inspected because their role is to offer descriptive labels on the objects they are transported with. By default they are not signed, not processed and their role is for classification only.
In fact they are more used and useful than what everyone thinks and could be a huge attack vector for several attacks. We are going through 2 of them in the two labs.
<br>

:computer:   [11. Rate Limiting](ratelimit/README.md)

when dealing with S3 infrastructure, you are oftenly dealing with large objects and huge throughputs. You can add as many disks and storage nodes to the S3 clusters, it won't automatically increase the throughput and the number of requests. The bottleneck will first become the network infrastructure.
Rate-limiting protects infrastructure, improves reliability, prevents abuse, and controls costs.

:computer:   [12. Advanced S3 Traffic Steering](steering/README.md)

Work In Progress.
<br>

:computer:   [13. Performance Testing guidance](performances/README.md)

This lab shares feedback and guidance some rules we get by experience when running performance testings. 
<br>

:spiral_notepad:   [14. Final Takeaways](conclusion/README.md)

