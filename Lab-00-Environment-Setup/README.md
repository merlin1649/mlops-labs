# Setting up an MLOps environment on GCP.

The labs in this repo are designed to run in a reference MLOps environment. The environment is configured to support effective development and operationalization of production grade ML workflows.

![Reference topolgy](/images/lab_300.png)

The core services in the environment are:
- AI Platform Notebooks - ML experimentation and development
- AI Platform Training - scalable, serverless model training
- AI Platform Prediction - scalable, serverless model serving
- Dataflow - distributed data processing
- BigQuery - analytics data warehouse
- Cloud Storage - unified object storage
- TensorFlow Extended/Kubeflow Pipelines (TFX/KFP) - machine learning pipelines
- Cloud SQL - machine learning metadata  management
- Cloud Build - CI/CD
    
In the reference environment, all services are provisioned in the same [Google Cloud Project](https://cloud.google.com/storage/docs/projects). Before proceeding make sure that your account has access to the project and is assigned to the **Owner** or **Editor** role.

## Copy the installation files to Cloud Shell
Although you can run the installation from any workstation configured with *Google Cloud SDK* and *Terraform*, the following instructions have been based on and tested with [Cloud Shell](https://cloud.google.com/shell/).

In the home directory of your **Cloud Shell**, replicate the folder structure of this lab. If you prefer, you can clone the whole repo using `git clone` command:
```
git clone https://github.com/jarokaz/mlops-labs.git
```


## Enabling the required cloud services

In addition to the [services enabled by default](https://cloud.google.com/service-usage/docs/enabled-service), the following additional services must be enabled in the project hosting an MLOps environment:

1. Compute Engine
1. Container Registry
1. AI Platform Training and Prediction
1. IAM
1. Dataflow
1. Kubernetes Engine
1. Cloud SQL
1. Cloud SQL Admin
1. Cloud Build
1. Cloud Resource Manager

Use [GCP Console](https://console.cloud.google.com/) or `gcloud` command line interface in [Cloud Shell](https://cloud.google.com/shell/docs/) to [enable the required services](https://cloud.google.com/service-usage/docs/enable-disable) . 

You can use the `enable_apis.sh` script to enable the required services from **Cloud Shell**.
```
./enable.sh
```

*Make sure that the Cloud Build service account (that was created when you enabled the Cloud Build service) is granted the Kubernetes Engine Developer role.*



## Deploying Kubeflow Pipelines 

The below diagrame shows an MVP infrastructure for a lightweight deployment of Kubeflow Pipelines on GCP:

![KFP Deployment](/images/kfp.png)

The environment includes:
- A VPC to host GKE cluster
- A GKE cluster to host KFP services
- A Cloud SQL managed MySQL instance to host KFP and ML Metadata databases
- A Cloud Storage bucket to host artifact repository

The KFP services are deployed to the GKE cluster and configured to use the Cloud SQL managed MySQL instance. The KFP services access the Cloud SQL through [Cloud SQL Proxy](https://cloud.google.com/sql/docs/mysql/sql-proxy). External clients use [Inverting Proxy](https://github.com/google/inverting-proxy) to interact with the KFP services.


The provisioning of the infrastructure components and installation of Kubeflow Pipelines has been automated with Terraform and Kustomize. The Terraform HCL configurations can be found in the `kfp/terraform` folder. The Kustomize overlays are in the `kfp/kustomize` folder.

To deploy Kubeflow Pipelines:

1. Open **Cloud Shell**
2. Install **Kustomize** 
```
cd /usr/local/bin 
sudo wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv3.3.0/kustomize_v3.3.0_linux_amd64.tar.gz 
sudo tar xvf kustomize_v3.3.0_linux_amd64.tar.gz
sudo rm kustomize_v3.3.0_linux_amd64.tar.gz
```
3. Start installation by executing the `install.sh` script from the `/Lab-00-EnvironmentSetup/kfp` folder.
```
./install.sh [PROJECT_ID] [REGION] [ZONE] [PREFIX] [NAMESPACE] [SQL_USERNAME] [SQL_PASSWORD]
```
Where:
- `[PROJECT_ID]` - your project ID
- `[REGION]` - the region for a Cloud SQL instance
- `[ZONE]` - the zone for a GKE cluster
- `[PREFIX]` - the name prefix that will be added to the names of provisioned resources
- `[SQL_USERNAME]` - the Cloud SQL user that will be used by KFP services to acces the Cloud SQL instance
- `[SQL_PASSWORD]` - the password of the Cloud SQL user



## Accessing KFP UI

After the installation completes, you can access the KFP UI from the following URL. You may need to wait a few minutes before the URL is operational.

```
echo "https://"$(kubectl describe configmap inverse-proxy-config -n kubeflow | grep "googleusercontent.com")
```
