# Results

Observations from the runs on the course cluster: 3 nodes with 2 vCPU and 4 GB each, Kubernetes 1.25,
Node.js 18, wrk 4.3.0. Latency and throughput were read from the wrk output, the number of pods from
`kubectl get pods`, CPU from `kubectl top pods`, and the replica history from the Chart.js graph. Each
scenario ran once; the values are approximate, as read during the runs.

## Scenarios

| scenario | load | pods | latency | CPU per pod |
| --- | --- | --- | --- | --- |
| baseline | about 5 requests/s, no `/demand` | 1 throughout | mean 80 ms | under 20 mCPU |
| sine wave (`/demand`) | 20 + 20 sin(2 pi t / 60) connections, peak about 40 requests/s | 1 → 3 → 5 → 2 → 1 in each 60 s cycle; at the peak the autoscaler asked for 40 and was capped at 5 | mean under 120 ms, p99 under 200 ms | up to about 400 mCPU |
| constant high load | `wrk -t4 -c100 -d60s`, about 80 requests/s | 5 throughout (80 asked, capped at 5) | 150 to 180 ms, stable | about 450 mCPU |

Under the constant high load each pod used close to half a CPU core.

## With and without the custom autoscaler

The same load scenarios, once with a fixed deployment of 2 pods and once with the custom autoscaler:

| metric | 2 fixed pods | custom autoscaler |
| --- | ---: | ---: |
| baseline latency | 80 ms | 80 ms |
| peak latency | 300 ms | 150 ms |
| pods at the peak | 2 | 5 |
| throughput | 50 requests/s | 80 requests/s |

The load generator of `/demand` runs inside the pod and targets that pod's own port, so the extra replicas
started by the autoscaler do not receive its requests. The table records what was observed; it does not
isolate how much of the difference the extra replicas caused.

## Feedback loop

- The autoscaler samples the request rate every 5 s, which sets the delay before it reacts.
- A call to the `deployments/scale` subresource of the Kubernetes API took 150 to 200 ms on average,
  added to every scaling decision.
