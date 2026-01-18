# S3 Key concepts

## S3 Glossary & Key Concepts  
*(Networking, HTTP, API, Security focused)*

<br>

### Core concepts
| S3 Concept | Description | BIG-IP Mapping |
|-----------|-------------|---------------|
| Bucket | Top-level namespace for objects | URI parsing in iRules, path-based routing |
| Object | Data + metadata stored in a bucket| Object-aware persistence (CARP) |
| Key | Full object path (acts like filename)| Hash input for deterministic routing |
| Region | Logical location of bucket | GSLB topology & latency rules |
| Namespace | Globally unique bucket naming scope | GSLB DNS, split-horizon DNS |

<br><br>

### HTTP & REST API
|HTTP Concept	| Description	| BIG-IP Feature |
|--|--|--|
|REST API	| HTTP-based object operations	| L7 proxying|
| Endpoint	| S3 service URL	| Virtual Server| 
| Virtual-hosted style	| bucket.s3.example.com	| Host-based routing |
| Path-style	| /bucket/object	| URI parsing |
| HTTP Method	| GET/PUT/POST/DELETE	| Method-aware iRules |
| Status Code	| API result	| Retry / failover logic |
| Headers	| Metadata & auth	| Header inspection/injection |
| Query Params	| API modifiers	| Signature verification | 

<br><br>

### Authentication & Authorization

| S3 Concept	| Description	| BIG-IP Mapping | 
|-----------|-------------|---------------|
| Access Key	| Public identifier	| Header parsing, rate-limit key | 
| Secret Key	| Private signing key	| Never forwarded | 
| Signature V4	| HMAC request signature	| iRule verification / regeneration | 
| Authorization | Header	| Signature container	WAF & iRules | 
| Session Token	| Temporary creds	| JWT / OIDC validation |
| IAM Policy	| Permission rules	| L7 policy enforcement |
| Bucket Policy	| Resource ACL	| Path-based authorization |
| ACL	| Legacy permissions	| WAF validation |

<br><br>

### Security
|	Security Concept |	Description |	BIG-IP Feature |
|-----------|-------------|---------------|
|	TLS	Transport | encryption |	SSL termination |
|	mTLS |	Mutual authentication |	Client cert validation |
|	SSE |	Encryption at rest |	Backend enforcement |
|	SSE-KMS |	External keys |	Header / policy checks |
|	Object Lock |	WORM protection |	API control enforcement |
|	Versioning |	Object history |	API validation |
|	Audit Logs |	Security events |	iRules + HSL logging |

<br><br>

### Networking
|	Networking Concept |	Description |	BIG-IP Feature |
|-----------|-------------|---------------|
|	Load Balancer |	Traffic distribution |	LTM pools |
|	L7 Routing |	HTTP-aware routing |	iRules |
|	Persistence |	Same object → same node |	CARP hashing |
|	Anycast VIP |	Same IP globally |	Multi-DC VIPs |
|	GSLB |	Geo routing |	BIG-IP DNS |
|	Latency Routing |	Nearest endpoint |	RTT-based selection |
|	Health Check |	Backend health |	Active HTTP probes |

<br><br>

### Performance
|	Performance Concept	| Description	| BIG-IP Feature |
|-----------|-------------|---------------|
|	Multipart Upload	| Parallel uploads	| Connection reuse |
|	Range Request	| Partial reads	L| 7 header awareness |
|	ETag	| Object checksum	| Cache validation |
|	Parallelism	| Concurrent flows	| TCP optimization |
|	Caching	| Reuse content	| Web Acceleration |

<br><br>


### API Behavior & Edge Cases
|	API Concept |	Description |	BIG-IP Mapping |
|-----------|-------------|---------------|
|	Idempotency |	Safe retries |	Retry logic |
|	Eventual Consistency |	Delayed state |	Read routing logic |
|	Strong Consistency |	Immediate state |	Write-aware routing |
|	Pagination |	Chunked listing |	Response handling |
|	Pre-Signed URL |	Time-limited access |	Signature enforcement |

<br><br>

### Threats & Risks
|	Threat |	Description |	BIG-IP Mitigation |
|-----------|-------------|---------------|
|	Credential Leakage |	Exposed keys	Header masking |
|	SSRF |	Backend abuse |	WAF + egress control |
|	Replay Attack |	Reused request |	Timestamp validation |
|	MITM |	Traffic interception |	TLS / mTLS |
|	Excessive Permissions |	Over-privilege |	Policy enforcement |

<br><br>

### Key Takeaway
S3 is pure HTTP.

BIG-IP adds intelligence: security, routing, observability, and control.
<br><br>



---

<br><br>

## MinIO Client (`mc`) – One-Page Cheat Sheet

**Purpose:** Manage S3-compatible object storage (MinIO, AWS S3, etc.)  
**Scope:** Usage, networking, security, admin, and troubleshooting

## Basics

### Install
```bash
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```
<br>

Check version
```bash
mc version
```
<br><br>

### Aliases (Endpoints)

:warning: Alias = endpoint + credentials

<br>
Add an S3 / MinIO endpoint

```bash
mc alias set myminio https://s3.example.com ACCESS_KEY SECRET_KEY
```

<br>
List aliases

```bash
mc alias list
```
<br>

Remove alias
```bash
mc alias rm myminio
```

<br>
<br>


### Buckets

<br>

List buckets
```bash
mc ls myminio
```

<br>

Create bucket
```bash
mc mb myminio/photos
```

<br>

Remove bucket
```bash
mc rb myminio/photos
```

<br>

Bucket info
```bash
mc stat myminio/photos
```

<br><br>

### Objects

List objects
```bash
mc ls myminio/photos
```
<br>

Upload file
```bash
mc cp image.jpg myminio/photos/
```

<br>

Download file
```bash
mc cp myminio/photos/image.jpg .
```

<br>

Recursive upload
```bash
mc cp --recursive ./data myminio/photos/
```

<br>

Remove object
```bash
mc rm myminio/photos/image.jpg
```

<br>

Object metadata
```bash
mc stat myminio/photos/image.jpg
```

<br><br>

### Sync & Mirroring


One-way sync
```bash
mc mirror ./localdir myminio/photos
```

<br>

Reverse sync
```bash
mc mirror myminio/photos ./localdir
```

<br>

Continuous replication
```bash
mc mirror --watch ./data myminio/photos
```

<br><br>

### Access Control & Security

Anonymous access
```bash
mc anonymous set download myminio/photos
```

<br>

Remove anonymous access
```bash
mc anonymous set none myminio/photos
```

<br>

IAM-style policies
```bash
mc admin policy list myminio
mc admin policy info myminio readwrite
```

<br>

Users & Admin (MinIO only)
Create user
```bash
mc admin user add myminio user1 password123
```

<br>

Enable / disable user
```bash
mc admin user enable myminio user1
mc admin user disable myminio user1
```

<br>

Attach policy
```bash
mc admin policy attach myminio readwrite --user user1
```

<br><br>

### Encryption

Server-Side Encryption (SSE-S3)
```bash
mc encrypt set sse-s3 myminio/photos
```

<br>

SSE-KMS
```bash
mc encrypt set sse-kms mykey myminio/photos
```

<br>

Check encryption
```bash
mc encrypt info myminio/photos
```

<br><br>

### Versioning & Object Lock

Enable versioning
```bash
mc version enable myminio/photos
```

<br>

Suspend versioning
```bash
mc version suspend myminio/photos
```

<br>

Enable object lock (on creation only)
```bash
mc mb --with-lock myminio/securebucket
```

<br><br>

### Events & Notifications
List events
```bash
mc event list myminio/photos
```

<br>

Add event
```bash
mc event add myminio/photos arn:minio:sqs::1:webhook --event put
```

<br><br>

### Networking & Debugging

Enable debug (VERY useful)
```bash
mc --debug ls myminio/photos
```

<br>

Trace requests
```bash
mc admin trace myminio
```

<br><br>

### Performance

Multipart uploads (automatic)
```bash
mc cp largefile.iso myminio/photos/
```

<br>

Parallel copy
```bash
mc cp --recursive --json ./data myminio/photos
```

<br><br>

### Pre-Signed URLs
Generate download URL
```bash
mc share download myminio/photos/image.jpg
```

<br>

Generate upload URL
```bash
mc share upload myminio/photos/
```

<br> <br>

## Common Errors & Fixes
| Error	| Cause	| Fix |
|---|---|---|
| SignatureDoesNotMatch	|Clock skew	| Sync time (NTP)| 
| AccessDenied	| Policy issue	| Check user/bucket policy| 
| Connection refused	| Network/TLS	| Verify endpoint & cert |
| Slow uploads	| MTU / TCP	| Enable BIG-IP TCP profiles |


## Pro Tips
```bash
mc --debug ...
```
displays the raw HTTP request of the S3 action (very useful to understand the traffic patterns and build your iRules).




<br><br>

---

<br><br>

[Back to Agenda](/README.md)
