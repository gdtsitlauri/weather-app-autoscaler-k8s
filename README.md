# Weather App with a Custom Kubernetes Autoscaler

**Can a service scale its own pods from the request rate it measures, instead of from CPU load?**

A 7-day weather forecast service (Node.js, WeatherAPI.com) deployed on Kubernetes, with an autoscaler
built into the service itself: every 5 seconds it measures the requests per second it received and
sets the number of replicas of its own deployment through the Kubernetes API. A load generator that
follows a sine wave and a live chart of the replica count show the autoscaler at work. A team project
for a course at the University of Thessaly; `report.pdf` (in Greek) describes the design.

| part | what it does |
| --- | --- |
| `server.js` | the service: `/weather` (forecast from WeatherAPI.com), `/metrics` (replica history), `/demand` (load test); counts requests and runs the autoscaling loop |
| autoscaler | every 5 s: requests per second = requests / 5, desired replicas = ceil(rps / 1) between 1 and 5, set through the `deployments/scale` subresource |
| load test | `/demand` runs `wrk` for 60 s with a number of connections that follows a sine wave (20 ± 20) |
| `index.html` | forecast form and a Chart.js graph of the replica count, refreshed every 5 s |
| `k8s/` | Deployment, Service, RBAC that lets the pod change only the scale of its own deployment, and an optional Horizontal Pod Autoscaler for comparison |

## Limitations (reported as such)

- Each replica runs the autoscaling loop and counts only the requests it receives itself, so with several
  replicas behind the Service the measured rate is that pod's share of the traffic.
- One pod per request per second and a maximum of 5 replicas are the values used in the project; they
  are constants in `server.js`.

## Folder map

```
weather-app-autoscaler-k8s/
  server.js                 the service and its autoscaling loop
  index.html                forecast form and replica chart
  k8s/                      deployment.yaml, service.yaml, rbac-scalers.yaml, hpa.yaml
  Dockerfile, docker-compose.yml, package.json, package-lock.json
  report.pdf                the project report (in Greek)
```

## Running

Requirements: Docker, a Kubernetes cluster (1.25 or later) with `kubectl`, and a WeatherAPI.com key
(free plan). The manifests use the namespace `gtsitlaouri-priv` of the course cluster; change it to yours.

```sh
docker build -t <registry>/weather-app:latest .
docker push <registry>/weather-app:latest           # and set this image in k8s/deployment.yaml

kubectl create secret generic weather-api --from-literal=api-key=<your WeatherAPI.com key>
kubectl apply -f k8s/rbac-scalers.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml                       # optional, needs metrics-server
```

The key reaches the service only through the `WEATHER_API_KEY` environment variable, which the
Deployment fills from the Secret; it is not stored in the code or the image. Locally:
`WEATHER_API_KEY=<key> docker compose up`.

Then open `http://<node-ip>:<node-port>/index.html`, ask for a forecast, and press **Demand** to start
the sine-wave load; the chart shows the replicas following it. `kubectl get pods` and
`kubectl logs deployment/weather-app-deployment` show the same from the cluster side.

## Authors and license

George David Tsitlauri, Dimitris Christou and Nikiforos Planakis, University of Thessaly. MIT license ([LICENSE](LICENSE)).
