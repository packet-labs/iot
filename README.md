# Packet Labs IoT workshop

This workshop deploys compute, storage, networking, and an IoT application to Packet.com.

## Conceptual architecture

Diagram:

![Conceptual architecture](/docs/images/conceptual.png)

Private components:

* Kubernetes - provisioned with [Terraform](https://www.terraform.io)
* TLS termination - via [cert-manager](https://cert-manager.io)
* MQTT Connector - [openfaas-incubator/mqtt-connector](https://github.com/openfaas-incubator/mqtt-connector)
* Database/storage - [Postgresql](https://www.postgresql.org)
* Docker registry - deployed externally, i.e. the Docker Hub.

Components exposed with TLS / Ingress or NodePort:

* Ingress Controller - [Traefik v1](https://github.com/containous/traefik) (HostPort 80/443)
* Serverless compute platform - [OpenFaaS](https://github.com/openfaas/faas)
* MQTT Broker - [emitter.io](https://emitter.io) (NodePort) - 30080/30443
* Business intelligence - [Metabase](https://www.metabase.com) (Ingress/TLS)
* Metrics visualization - [Grafana](https://grafana.com) (Ingress/TLS)

## Getting started

Everything you need to deploy this workshop is available in this repository.

> Note: This repository is designed to be used with your own domain name and a number of DNS records. This enables TLS termination (HTTPS) to be used for exposed services. If you are working in development, you can skip the domain and TLS steps. 

### 1) Clone the repo

```sh
git clone https://github.com/packet-labs/iot
```

### 2) Create a bare-metal Kubernetes cluster

You will need to [install Terraform](https://www.terraform.io) for this step.

* Set your Packet API and project ID in the .tf files in `/k8s`.

* Enter [the `k8s` folder](/k8s/) and apply the terraform plan.

* Find the IP of one of the nodes in the cluster from your Packet dashboard or the state file in /k8s/

Create four DNS A records (replace `example.com` with your domain):

* A `gateway.example.com` - IP
* A `grafana.example.com` - IP
* A `metabase.example.com` - IP
* A `emitter.example.com` - IP

> You can register for a domain at [Google Domains](https://domains.google) or [Namecheap.com](https://namecheap.com) for a few dollars. You can also configure your domain there, after purchase.

Some commands will be run from your laptop, so make sure you install Kubectl

* [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### 3) Install Postgres via KubeDB and helm

You will need to install helm for this step.

* Install [postgresql](/postgresql/)

### 4) Install OpenFaaS

* Install [openfaas](/openfaas/) to provide compute and events

* Deploy the [OpenFaaS services](/openfaas/services/) for the application

You will also deploy the [schema.sql](/openfaas/services/schema.sql) at this time for `drone_position` and `drone_event`.

### 65 TLS for OpenFaaS

* Install cert-manager

    ```sh
    k3sup app install cert-manager
    ```

* Install an Ingress record for your OpenFaaS gateway

    ```sh
    k3sup app install openfaas-ingress \
    --domain gateway.example.com \
    --email openfaas@example.com \
    --ingress-class traefik
    ```

* Add TLS for Grafana

    Edit `./openfaas/grafana-ingress.yaml` and edit `grafana.example.com` replacing `example.com` with your domain.

    Now run:

    ```sh
    kubectl apply -f ./openfaas/grafana-ingress.yaml
    ```

### 6) Add the MQTT Broker (Emitter.io)

* Install [emitter](/emitter/)

### 7) Add the OpenFaaS MQTT-Connector

The MQTT-Connector is used to trigger functions and services in response to messages generated by the event source. It runs inside the Kubernetes cluster and is private with no ingress.

* Install [OpenFaaS MQTT-Connector](/openfaas/mqtt-connector/)

### 8) Add Grafana for function/service visualization

Grafana packages a pre-compiled dashboard for OpenFaaS to show metrics like throughput and latency.

* [Deploy and Grafana](/grafana/)

### 9) Send Drone Data

You can now send data to emitter from your drone clients.  Use the [drone simulator](/test/) to generate realistic data for use with visualization tools.

## Todo/backlog

Documentation and due dilligence:

- [x] Document: Add k3s Terraform instructions - Cody
- [x] Document: OpenFaaS deployment - Alex
- [x] Document: Postgresql lightweight option - Alex
- [x] Document: K8s: how to get a local KUBECONFIG file - Alex
- [x] Document: Conceptual architecture diagram - Alex
- [x] Document: Grafana deployment to visualize functions - Alex
- [x] Document: Add TLS termination documentation - Alex
- [ ] Select deploy business insights software (Metabase / Redash?) - Alan & team
- [ ] Write deployment instructions for Metabase or Redash - Alan & team

Data-feed generation for end-to-end testing:
- [ ] Create Python or Node app to publish generated MQTT data - Alan & team

Functions / application services:
- [x] Create Postgesql schema asset for drone geo-location data - Alex
- [x] Create Postgesql schema asset for drone event data that can support the following events (jsonb or string of JSON) - Alex

* Collision Avoidance
* Airspace Violation
* Device Health Report
* Low Battery
* Excessive Battery Usage
* Device Fault

- [x] Create OpenFaaS Function: insert geo row (using `node12` template) - Alex
- [ ] Create OpenFaaS Function: insert event row (using `node12` template) - Alan & team 

- [ ] Create OpenFaaS Function: process geo data for collision or airspace (using `node12` template) - Alan & team

This should detect, log the event, and emit a message back thru mqtt to the violating drones

- [x] Create OpenFaaS Function: Query drone positions for mapbox (using `node12` template)
- [x] Create OpenFaaS Function: Query drone events for mapbox (using `node12` template)
- [x] Create OpenFaaS service for viewing data (mapbox for geo) (using `node12` template) - Alex
- [ ] Enhance OpenFaaS service for live streaming / positional viewing data (mapbox for geo) (using `node12` template) - Alan & team

* Static example with OpenStreetMap https://github.com/alexellis/sf-assets
* Pure express - https://github.com/openfaas-incubator/node10-express-service
* Mix of function/express - https://github.com/openfaas/openfaas-cloud/blob/master/dashboard/of-cloud-dashboard/handler.js
* Static example - https://www.openfaas.com/blog/serverless-static-sites/

Suggested schema ('drone_status'):

* ID 
* Name
* Altitude 
* GPS Latitude
* GPS Longitude
* Velocity (sent as a vector)
* Battery
* Temperature

Suggested schema ('drone_event'):

* ID 
* Event Type
* Data 
