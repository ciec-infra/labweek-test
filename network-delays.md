# XVP network delays

## Test method

`curl-format.txt`
```
time_namelookup: %{time_namelookup}\n
time_connect: %{time_connect}\n
time_appconnect: %{time_appconnect}\n
time_pretransfer: %{time_pretransfer}\n
time_redirect: %{time_redirect}\n
time_starttransfer: %{time_starttransfer}\n
———\n
time_total: %{time_total}
\n
```

`curl.sh`
```sh
#!/bin/sh

BASE_URL=https://184.120.4.166:443
CLIENT_ID=LLx
curl -k -w '@curl-format.txt' -v --location --request GET \
  "${BASE_URL}/v1/partners/sky-uk/accounts/2567546836364081639/recordings?clientId=${CLIENT_ID}&maxResults=0&limit=0&offset=0&include=programVariant" \
  --header 'Authorization: Bearer ...'
```

Splunk query
```
index=xvp-eu service.name=xvp-saved-api span.kind="server" "service.environment"=prod client.id=LLx
| eval duration_ms='event.duration'/1000000
| table  transaction.id duration_ms
```

The XVP-Saved behavior regarding its response time is well understood:
A request is either processed within ~`30`ms *OR* within ~`300`ms.
The following measurements are about the **delta** of transmission times and ignore
absolute fluctuations between ~`30`ms and ~`300`ms.

## Test results

### On-pod request (AZ eu-central-1c):

**Note:** This communication is *without* TLS and purely HTTP!

Running `curl` on the pod itself: **~0.51ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.033435 | 32.873285 |
| 0.032214 | 31.686535 |
| 0.031602 | 31.048757 |
| 0.033118 | 32.655010 |
| 0.025639 | 25.169836 |

</details>

### pod-to-pod request within the same AZ:

**Note:** This communication is *without* TLS and purely HTTP!

Running `curl` on the pod itself: **~1.20ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.361723 | 360.508780 |
| 0.377498 | 376.331919 |
| 0.357782 | 356.661085 |
| 0.366927 | 365.635200 |
| 0.390145 | 389.005016 |

</details>

### Request across AZ (same VPC)

**Note:** This communication is *without* TLS and purely HTTP!

Request across AZ (`ec1b` -> `ec1c`) *without* the actual LB: **~1.61ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.347295 | 345.903665 |
| 0.360262 | 358.657963 |
| 0.352981 | 351.122920 |
| 0.396727 | 395.148599 |

</details>

### Request within the same AZ 

Request within the same AZ but via the actual LB: **~16.81ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.046991 | 32.010490 |
| 0.043727 | 28.919159 |
| 0.049900 | 35.209236 |
| 0.055720 | 31.266365 |
| 0.048492 | 33.350223 |

</details>


### Request across regions (same AWS account)

Request across regions (`eu-west-1` -> `eu-central-1`) via LB: **~117.16ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.477819 | 358.901452 |
| 0.480778 | 359.858209 |
| 0.474857 | 359.418723 |
| 0.439248 | 319.718558 |
| 0.473090 | 362.082321 |

</details> 

### Request across the ocean from CTC

Request across the ocean from a developer machine from CTC: **~465.78ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.498804 | 31.673
| 0.499612 | 30.506
| 0.502709 | 34.003
| 0.492686 | 34.502
| 0.491124 | 36.498

</details>

### Request across the ocean 

Request across the ocean from a developer machine (LAN, Gbit Cable): **~531.63ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.585529 | 34.845203 |
| 0.565862 | 35.436274 |
| 0.552710 | 28.437258 |
| 0.550774 | 31.778616 |
| 0.563407 | 29.630569 |

</details>

### Request across the ocean w/VPN

Request across the ocean from a developer machine (LAN, Gbit Cable) while connected to Comcast VPN: **~600.36ms**

<details>
<summary>Click to expand</summary>

| `curl` `time_total` seconds | Splunk `duration_ms` |
|---|---|
| 0.968955 | 358.844676 |
| 1.013556 | 341.305411 |
| 0.911875 | 345.529085 |
| 0.937877 | 357.057260 |
| 0.912454 | 340.191317 |

</details>
