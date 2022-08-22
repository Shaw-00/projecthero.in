# Project Hero- Cloud Infra and Architecture Documentation: -


Cloud: Google Cloud Provider

## 1. Setup on Google Kubernetes Engine(GKE): -

### 1.a. Created 3 separate projects for each environment: -
| Environment | Project ID  |
| :---         |     :---:      |
| Dev   | hero-app-dev-346507     |
| Staging | hero-app-staging |
| Production     | hero-app-prod       |


Within each of these projects we have created a GKE cluster where we have deployed our web-app, server, admin app.

### 1.b. The deployments are created using CI/CD through GitHub Actions wherein the following parameters are passed as environment variables: -

PROJECT_ID: The respective project id for that environment( described above). </br>
SERVICE_ACCOUNT_KEY: The service account created for access on the GKE cluster. </br>
IMAGE: The image pushed through Docker build in CI/CD on the Google Artifact Registry(GAR) . </br>
REPOSITORY: Repository which we create on GCP under GAR where our images will reside. </br>
GKE_ZONE: The zone where our project is created i.e. asia-south1 . </br>
GKE_CLUSTER: Cluster created under GKE for our deployments and services. </br>
GKE_LOCATION: Region where we define our GKE cluster i.e. asia-south1. </br>
DEPLOYMENT_NAME: The name of the deployment for our application/server. </br>
CONTAINER_NAME: The container name within GAR storing our pushed images. </br>


## 2. GKE Deployment CI/CD yaml example: -
```
name: build and deploy
on:
  push:
    branches:
      - dev
env:
  PROJECT_ID: ${{ secrets.GKE_DEV_PROJECT }}
  SERVICE_ACCOUNT_KEY: ${{ secrets.GKE_DEV_SA_KEY }}
  IMAGE: app
  REPOSITORY: ph-dev
  GKE_ZONE: asia-south1
  GKE_CLUSTER: ph-dev
  GAR_LOCATION: asia-south1
  DEPLOYMENT_NAME: ph-dev
  CONTAINER_NAME: ph-dev
jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Set branch name
        run: echo "BRANCH=${GITHUB_REF##*/}" >> $GITHUB_ENV
      - name: Set branch environments
        run: |-
          echo "NEXT_PUBLIC_SERVER_URL=${{secrets.NEXT_PUBLIC_STAGE_SERVER_URL_VAR}}" >> "$GITHUB_ENV"
      - name: Checkout
        uses: actions/checkout@v2
 
      # Setup gcloud CLI
      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@94337306dda8180d967a56932ceb4ddcf01edae7
        with:
          service_account_key: ${{ env.SERVICE_ACCOUNT_KEY }}
          project_id: ${{ env.PROJECT_ID }}
 
      # Configure docker to use the gcloud command-line tool as a credential helper
      - name: Configure docker with gcloud
        run: |-
          gcloud --quiet auth configure-docker "$GAR_LOCATION-docker.pkg.dev"
      # Get the GKE credentials so we can deploy to the cluster
      - name: Configure GKE Credentials
        uses: google-github-actions/get-gke-credentials@fb08709ba27618c31c09e014e1d8364b02e5042e
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}
          credentials: ${{ env.SERVICE_ACCOUNT_KEY }}
 
      # Build the Docker image
      - name: Build docker image
        run: |-
          docker build \
            --tag "$GAR_LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$GITHUB_SHA" \
            --build-arg NEXT_PUBLIC_SERVER_URL_VAR="$NEXT_PUBLIC_SERVER_URL" \
            .
      # Push the Docker image to Google Container Registry
      - name: Publish to GAR
        run: |-
          docker push "$GAR_LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$GITHUB_SHA"
      # Deploy the Docker image to the GKE cluster
      - name: Deploy
        run: |-
          kubectl --namespace default set image deployment/$DEPLOYMENT_NAME $CONTAINER_NAME=$GAR_LOCATION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$GITHUB_SHA
          kubectl --namespace default rollout status deployment/$DEPLOYMENT_NAME
```          


### 2.a.Following are the steps to be implemented to create deployment( manually) the first time, post which the deployment will be carried out through CI/CD: -

1. Create a repository within Google Artifact Registry.
2. Push your app image to the GAR repository, using GCloud CLI or by running the CI/CD once.
3. Get the entire image address with a tag from the GAR repository.
4. Create a deployment manifest as below: -
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: ph-prod-admin-app
  name: ph-prod-admin-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ph-prod-admin-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ph-prod-admin-app
    spec:
      containers:
      - image: asia-south1-docker.pkg.dev/hero-app-prod/ph-prod-admin-app/app@sha256:acfe47089388bdc8d938cfe552f80ae27c1cdf1d64554b59e765ec243640fa00
        name: app
        imagePullPolicy: Always
        ports:
          - containerPort: 8080
        livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 1
        readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 1
            successThreshold: 1
            timeoutSeconds: 2
 ```


Within the GCloud CLI, use the following command to point your cli to the cluster
gcloud container clusters get-credentials CLUSTER_NAME --region asia-south1 --project PROJECT_ID

Copy the entire image address to the spec.containers.image key in the deployment yaml.
Run the following command in the GCloud CLI to now create the deployment within the cluster:
kubectl create -f PATH_OF THE_DEPLOYMENT_YAML***

NOTE***- The above steps need to be carried out from local for first time deployment, post which deployments shall be carried out through the CI/CD workflows.


### 2.b. Creating a service for our deployment: -

We need to create a service on our pods to expose them to the network. This can be provisioned on the GKE portal or through the service yaml file on your local.

Following is a sample service yaml file:

```
apiVersion: v1
kind: Service
metadata:
  name: ph-stage-admin-app-service
  labels:
    run: ph-stage-admin-app-service
spec:
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
  selector:
    app: ph-stage-admin-app
  sessionAffinity: None
  type: NodePort
```  

The important key-value pairs to make note of from the above sample are as follows: -

i. metadata.name: usually kept based on the app service which you need to create. <br/>
ii. spec.ports,targetPort: The port number on which your app/ server should be running inside the container. <br/>
iii. selector.app: The app or name of the deployment which you want to expose through this service. <br/>
iv. spec.type: Choose your service type based on the requirement- NodePort/ClusterIP/ExternalLoadBalancer. <br/>

For creating the service from the yaml we use the same command as for deployment:

gcloud container clusters get-credentials CLUSTER_NAME --region asia-south1 --project PROJECT_ID
kubectl create -f PATH_OF THE_SERVICE_YAML



#### 2.c.i. Creating Ingress for our service(HTTP access): -

Ingress needs to be created to expose our service through an external load balancer allowing us to access our app/server through a public IP.

GCP allows us to create Ingress directly from the portal where we just need to specify the backend service which we had created above.

We can also create Ingress from a yaml, same as deployments and services. Following is a sample ingress yaml: -

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ph-dev-server-ingress
spec:
  defaultBackend:
    service:
      name: ph-dev
      port:
        number: 80

The important parameters to be noted from the above yaml are as follows: -

i. metadata.name: This should be the name of the ingress you’re creating. <br/>
ii. spec.defaultBackend.service.name: Name of the service over which you want to create your ingress. <br/>
iii. spec.defaultBackend.service.port,number: Port number defined in the service yaml. <br/>

The sequence of commands to create the ingress is similar to the ones we used in the creation of the service and deployment yamls:
```
gcloud container clusters get-credentials CLUSTER_NAME --region asia-south1 --project PROJECT_ID
kubectl create -f PATH_OF THE_INGRESS_YAML
```


#### 2.c.ii. Setting up HTTPS access for the server/app: -

Firstly we need to create and add a SSL certificate that holds the domain with which we have created the A record on our DNS provider.
Here we are using Google managed certificates for this purpose. Following is a sample of a     Google managed certificate:
```
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: managed-cert-ph-dev-web
spec:
  domains:
    - dev-booking.projecthero.in
```

The key parameters to be implemented here in the above sample are”
i.  metadata.name: The name of the certificate. This will need to be annotated in the ingress yaml.
We need to reserve the Ingress’ IP address to use it as a static-ip.

Once the certificate is added we need to configure Load Balancer for HTTPS access. Here within the Front-end rules, we need to add HTTPS, Port: 443, static-ip which we just reserved and select the certificate which we just added. 

We would need to add the following annotations to the Ingress yaml, in metadata.annotations: -
```
ingress.gcp.kubernetes.io/pre-shared-cert: The name of the certificate
ingress.kubernetes.io/https-forwarding-rule: The forwarding rule fetched from the Load balancer.
ingress.kubernetes.io/https-target-proxy The target proxy fetched from the Load balancer.
ingress.kubernetes.io/ssl-cert: The name of the certificate.
kubernetes.io/ingress.global-static-ip-name: name of the static ip
networking.gke.io/managed-certificates: name of the certificate.
 ```
Now we need to apply these changes on the GCP console or through CLI using the following command: - <br/>
``` kubectl apply -f PATH_OF THE_INGRESS_YAML ```
 
***Time-taking issues: -
i. The DNS can take upto 72 hours to propagate worldwide after A and AAAA records are added.
ii. When we add multiple domains in a certificate, all domains need to be resolved and be in active status. Only then the Cert will become Active.
These are referenced from Google cloud documentations.
 
 
### 3.a.Pub/Sub topics and subscription creation on Console: -
 
Go to Pub/Sub on the GCP console. Select Create Topic and fill in the required fields like topic ID and select Google-managed key.
Creating Subscriptions through console, following fields need to be added: -
i. Subscription ID
ii. The name of the topic to which the subscription shall direct the messages.
iii. Delivery Type: Pull/Push
iv. Message retention duration.
v. Expiry period of the messages.
vi. Acknowledgement Deadline
vii. Message Ordering
viii. Dead Lettering.
 
 
### 3.b. Pub/Sub Topics and Subscription creation using Terraform: -
 
Clone the infra-as-code repo on the local.
Navigate to the repo, and run terraform init from cli.***
Open the variables.tf file
Enter the name of the topics within the default array.
Enter the subscription within the default array for the subscription list.
Run terraform plan and then terraform apply on cli.
``` 
variable "google_pubsub_topic" {
  type=list
  default=[]
}
 
variable "google_pubsub_subscription" {
  type=list
  default=[]
}
 
variable "google_project_service" {
  default="hero-app-dev-346507"
}
 
variable "region" {
  default="asia-south1"
}
```
 
The default array in the topic and subscription list.


