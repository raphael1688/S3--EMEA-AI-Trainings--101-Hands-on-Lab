# Rate-Limiting S3 requests

## Why would we need to rate-limit requests?

Rate-limiting is the controlled throttling of requests to an S3 service to prevent abuse, ensure stability, and optimize resource usage. Specifically, we might need to rate-limit S3 requests for several reasons:
<br>

### Protect backend infrastructure

S3 servers, whether AWS S3 or MinIO nodes, have finite CPU, memory, network bandwidth, and I/O throughput.
Uncontrolled bursts (e.g., hundreds of clients requesting large files simultaneously) can overwhelm nodes, causing slow responses or failures.
<br>

### Prevent service degradation

Rate-limiting ensures predictable latency for all clients, preventing a few heavy users from impacting others.
This is especially important in multi-tenant environments or when multiple applications share the same MinIO cluster.
<br>

### Avoid operational costs

High request rates can generate unexpected egress costs in cloud S3, or excessive load in on-prem MinIO, potentially triggering throttling or downtime.
<br>

### Mitigate abusive or malicious traffic

Rate-limiting can protect against DDoS attacks, misconfigured clients, or accidental loops, where a client floods the S3 cluster with requests.

<br><br>


## On which criteria do we want to rate-limit?

When implementing S3 request rate-limiting, you can base it on different criteria depending on your goals. Some common examples include:

| Criteria | Explanation	| Example / Rationale|
|----|---|---|
| Client identity / access key	| Rate-limit per IAM user or access key	| Prevents a single client from overwhelming the cluster while allowing others to operate normally |
| IP address / subnet	| Limit requests based on client IP	| Useful when clients don’t authenticate or you want to limit traffic per network segment but limited when multiple clients are hosted behind a proxy  |
| Bucket / object	Rate-limit | Requests to specific buckets or hot objects	| Protects popular buckets from floods without throttling other buckets |
| HTTP method / action type	| Separate limits for GET, PUT, DELETE, etc.	| PUT/POST/DELETE are usually heavier on storage or processing than GET; throttling writes protects storage I/O |
| Request size / bandwidth	| Limit based on bytes transferred per second	| Prevents large uploads or downloads from saturating network links |
| Global cluster limit	| Total requests per second across all clients	| Ensures the S3 cluster remains stable under load spikes |

<br><br>

When we debugged S3 requests in a previous lab, you probably identified the 2 following arguments in the response from the minio nodes:

```yaml
X-Ratelimit-Limit: 6974
X-Ratelimit-Remaining: 6974
```

We could also override these values when passing them to the client but all the S3 vendors don't use them and you rely on the client to honour them which is not always the case.


## iRule to rate-limit S3 traffic

```tcl
# Rate Limiting iRule
when RULE_INIT {
    # Throughput gating
    set static::S3_MAX_BPS          800000000   ;# 800 Mbps
    set static::THRESHOLD_PERCENT  85

    # Rate limit
    set static::MAX_REQUESTS_PER_WINDOW 1000
    set static::WINDOW_SECONDS          60
    set static::RETRY_AFTER             2
}

when HTTP_REQUEST {

    # ---- STEP 0: Throughput gate ----
    set current_bps [expr { [STATS::bytes in] * 8 }]
    set threshold_bps [expr {
        ($static::S3_MAX_BPS * $static::THRESHOLD_PERCENT) / 100
    }]

    if { $current_bps < $threshold_bps } {
        return
    }

    # ---- STEP 1: Extract access_key ----
    set access_key "unknown"

    if { [HTTP::header exists "Authorization"] &&
         [regexp {Credential=([^/]+)} [HTTP::header "Authorization"] -> ak] } {
        set access_key $ak
    }

    if { $access_key eq "unknown" } {
        return
    }

    # ---- STEP 2: Token bucket ----
    set key "s3_rl:$access_key"
    set now [clock seconds]

    set state [table lookup $key]

    if { $state eq "" } {
        set tokens $static::MAX_REQUESTS_PER_WINDOW
        set last_ts $now
    } else {
        lassign $state tokens last_ts
    }

    # Refill
    set elapsed [expr {$now - $last_ts}]
    if { $elapsed > 0 } {
        set refill [expr {
            ($elapsed * $static::MAX_REQUESTS_PER_WINDOW) /
            $static::WINDOW_SECONDS
        }]
        set tokens [expr {
            min($static::MAX_REQUESTS_PER_WINDOW, $tokens + $refill)
        }]
    }

    if { $tokens <= 0 } {
        HTTP::respond 429 \
            content "Too Many Requests" \
            "Retry-After" $static::RETRY_AFTER
        return
    }

    incr tokens -1

    table set $key [list $tokens $now] \
        timeout $static::WINDOW_SECONDS \
        lifetime $static::WINDOW_SECONDS
}

```

<br><br>

---

<br><br>
[Back to Agenda](/README.md)
