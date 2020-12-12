# Node application on K8s

Deploying a Docker web-app with terraform in a kubernetes cluster on GCP

[![GitHub tag](https://img.shields.io/github/tag/tmknom/terraform-aws-alb.svg)](https://registry.terraform.io/modules/tmknom/alb/aws)
[![License](https://img.shields.io/github/license/tmknom/terraform-aws-alb.svg)](https://opensource.org/licenses/MIT)

# Topics

- Terraform deployment
- create a docker node app
- Front-end nginx ingres
- Node app deployment
- Monitoring
- Chaos engineering
- CI/CD cirlceCI

## Requisites
 
- [GCP Account](https://aws.amazon.com)

- [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- An existing VPC
- Some existing subnets
- GCP secret and public keys
- Helm

 ## Requirements

| Name | Version |
|--|--|
|  terraform| >= 0.12  |
|  Helm     |    V3    |
|  K8s      | 1.16.15-gke.4300  |

## Providers

|Name| Version
|--|--|
| GCS | ~> 2.18.0 |
  
### Terraform Version

Terraform 0.12. Pin module version to ⇾ v2.0. Submit pull-requests to master branch.

#  Creating the cluster

**Description**

Terraform module which sets up an kubernetes cluster .

The following resources are created:

 - A VPC Network
 - Public subnet
 - Regional Routing
 - Kubernetes cluster
 - Service account
   
## Requisites
 
 - A bucket to store the terraform state file 
 
 It can be created it graphically on the google cloud console or use the following code to create one:
 
```sh
gsutil mb -p <project_name> -c regional -l <location> gs://<bucket_name>/

```
 
 Remember to grant read/write permissions on this bucket to our service account

## Usage

Remember to change the values to the values of your project

  ```sh
https://github.com/diiegg/gcp-terraform-k8s

cd dev

terraform init

terrafom plan

terraform apply
```


## Module
  ```sh
terraform {
  required_version = ">= 0.12.10"
}

provider "google" {
  version = "~> 2.18.0"
  project = var.project_id
  region  = var.region
  credentials   = "your json file here"
}

provider "google-beta" {
  version = "~> 2.18.0"
  project = var.project_id
  region  = var.region
  credentials  = "your json file here"
}

provider "template" {
  version = "~> 2.1"
}

// --- Computed Values ---

locals {
  cluster_name = "${var.env}-cluster"
}


// --- GKE Cluster ---

resource "google_container_cluster" "primary_cluster" {
  provider = google-beta

  // Base
  name = local.cluster_name
  project = var.project_id
  location = var.region

  // Network
  network = var.network
  subnetwork = var.subnetwork
  ip_allocation_policy {
    cluster_secondary_range_name = var.secondary_ip_range
    services_secondary_range_name = var.secondary_ip_range
  }

  // Addons and auxilliary services
  addons_config {
    http_load_balancing {
      disabled = false
    }

    horizontal_pod_autoscaling {
      disabled = false
    }
  }

  node_pool {
    name = "default-pool"
    initial_node_count = var.initial_count

    autoscaling {
      max_node_count = var.max_count
      min_node_count = var.min_count
    }

    management {
      auto_repair  = true
      auto_upgrade = true
    }

    node_config {
      image_type   = "COS"
      machine_type = var.node_machine_type
      disk_size_gb = 10
      disk_type    = "pd-standard"
      preemptible  = false

      oauth_scopes = [
        "https://www.googleapis.com/auth/cloud-platform",
      ]
    }
  }

  timeouts {
    create = "30m"
    update = "30m"
    delete = "30m"
  }
}
``` 

Now confirm if the cluster set is correct by running

  ```sh
kubectl config current-context
```

# Create a basic Docker node app

## Note: For this demo we will use this app 

**Description**

- Buidl and test the app
- Create a docker file and dockerize the app
- Push the app into google container rigestry

## Building the app

create a index.html file with the following code

  ```sh
<html>
 <head>
  <title></title>
<link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.0/css/bootstrap.min.css" rel="stylesheet">
 </head>
 <body>

  <div class="col-md-10 col-md-offset-1" style="margin-top:20px">
   <div class="panel panel-primary">
     <div class="panel-heading">
       <h3 class="panel-title">Im a node app</h3>
     </div>
      <div class="panel-body">
       <div class="alert alert-success">
          <p>Basic app for demostration using k8s and docker</p>
       </div>
         <div class="col-md-9">
           <p>Author:</p>
           <div class="col-md-3">
             </div>
         </div>
      </div>
  </div>
  </div>

<script src="https://code.jquery.com/jquery-3.1.1.slim.min.js"></script>
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/js/bootstrap.min.js"></script>
 </body>
</html>
```

Create a server file server.js with the folowing code 

  ```sh
const express = require('express');
const path = require('path');
const morgan = require('morgan');
const bodyParser = require('body-parser');

/* eslint-disable no-console */

const port = process.env.PORT || 3000;
const app = express();

app.use(morgan('dev'));
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: 'true' }));
app.use(bodyParser.json({ type: 'application/vnd.api+json' }));

app.use(express.static(path.join(__dirname, './')));

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, './index.html'));
});

app.listen(port, (err) => {
  if (err) {
    console.log(err);
  } else {
    console.log(`App at: http://localhost:${port}`);
  }
});
module.exports = app;
```

Create a file to hold the packages required for this project package.json

  ```sh
npm install
```
Test 

  ```sh
npm start
```

In a browser, navigate to http://localhost:3000 and the app show runing localy 

## Create a Docker app

**Description**

- Create Docker file
- create Docker Image
- Test image
- Tag image and pushh to gcr

## Docker file

Create a file with the name Dockerfile to dockerize the app and put the following code


  ```sh
FROM node:10-alpine
ENV NPM_CONFIG_LOGLEVEL warn
RUN mkdir -p /usr/src/app
EXPOSE 3000
WORKDIR /usr/src/app
RUN chown -R node.node /run && \
  chown -R node.node /usr/src/app
ADD package.json /usr/src/app/
RUN npm install --production
ADD . /usr/src/app/
COPY --chown=node:node . .
USER node
ENTRYPOINT ["npm", "start"]

```

**Note from sec consideration**

alpine image allow us to keep our image size down
we are including a non-root user that can use to avoid running your application container as root


Now, let’s build our image. Run:

  ```sh
docker build -t node-app .
```

To test the image run 

  ```sh
docker run -p 3000:3000 node-app
```

Navigating to http://localhost:3000 to show the app

## Tag the image and push it to GCR

To tag the image we will use the followin syntax

  ```sh
docker tag <HOSTNAME>/<YOUR-PROJECT-ID>/<IMAGE-NAME>
```

Example 
  ```sh
docker tag node-app eu.gcr.io/<YOUR-PROJECT-ID>/nodeapp:v1

```

Time to push the image to container regestry
  ```sh
gcloud docker -- push <HOSTNAME>/<YOUR-PROJECT-ID/<IMAGE-NAME>

```
Example

  ```sh
gcloud docker -- push eu.gcr.io/<YOUR-PROJECT-ID/nodeapp:v1

```

## Font-end nginx ingres

Now we have our app on the container repo, lets log in into the k8s cluster to intall nginx ingress using helm chart

Seting up Nginx ingres via Helm

  ```sh
helm install nginx-ingress stable/nginx-ingress --set controller.publishService.enabled=true


```

Lets check the ingress controler

  ```sh

kubectl get service nginx-ingress-controller

```

Configure the ingress resource file, create a file ingress.yaml


  ```sh
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nodeapp-ingress
  namespace: ingress-basic
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /?(.*)
        backend:
          serviceName: nodeapp
          servicePort: 8080
      - path: /api/?(.*)
        backend:
          serviceName: api
          servicePort: 80

```

Deploy the resource

  ```sh

kubectl apply --filename deploy.yaml

```

Testing ingress

  ```sh

kubectl get service nginx-ingress-controller

```

## Node app deployment

To deploy the docker app we need to create a file node-app.yaml

  ```sh

apiVersion: v1
kind: Service
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: nodeapp
    tier: frontend
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodeapp
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nodeapp
        tier: frontend
    spec:
      containers:
      - image: eu.gcr.io/labs-298016/nodeapp:0.1.0
        name: nodeapp
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
        ports:
        - containerPort: 8080
          name: nodeapp

```

**Note**  security consideratios

we are runing the conainer as ano root user, we blcok root containers and preventing Linux binaries run as a priviledge user

Lets deploy the app

  ```sh

kubectl apply --filename nodeapp.yaml

```

test the conection

  ```sh

kubectl get service nginx-ingress-controller

```
 navigate in the web browser uing the external ip


# Demo

[Try me](http://35.246.90.134)



# Monitoring

**Topics**

- Prometheus and Grafana
- Weave Scope
- Golden rules 


## Prometheus and grafana

Prometheus and Grafana make it extremely easy to monitor just about any metric in your Kubernetes cluster it Shows overall cluster CPU / Memory / Filesystem usage as well as individual pod, containers, systemd services.

## Weave Scope

Is a great visualization and monitoring tool for your cluster, showing
you a real-time map of your nodes, containers, and processes. 


License.
----
MIT

# References.

  

- [Terraform AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener)

- [AWS  Load balancing](https://aws.amazon.com/elasticloadbalancing/getting-started/)
