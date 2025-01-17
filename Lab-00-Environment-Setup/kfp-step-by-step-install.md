## Step by step instructions for installing Kubeflow Pipelines
If you need to understand in more detail the process of installing a lightweight deployment of Kubeflow Pipelines the following instructions step through the same process as automated by the `install.sh` script.

*Before procedding make sure to enable the required Cloud Services as described in the previous section*.

The following instructions have been tested with **Cloud Shell**.

This installation requires **Terraform** and **Kustomize**. **Terraform** is pre-installed in **Cloud Shell**. To install **Kustomize** in **Cloud Shell**:
```
cd /usr/local/bin 
sudo wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv3.3.0/kustomize_v3.3.0_linux_amd64.tar.gz 
sudo tar xvf kustomize_v3.3.0_linux_amd64.tar.gz
sudo rm kustomize_v3.3.0_linux_amd64.tar.gz
```


### Provisioning infrastructure
The `terraform` folder contains Terraform configuration language scripts that provision an MVP infrastructure required to run a lightweigth deployment of Kubeflow Pipelines.

The `main.tf` file utilizes re-usable Terraform modules from https://github.com/jarokaz/terraform-gcp-kfp, specifically:
- The `modules/service_account` module that creates a GCP service account and grants to the account a set of IAM roles.
- The `modules/vpc` module that creates a VPC network
- The `modules/gke` module that creates a GKE cluster with a default node pool

The `main.tf` script creates:
- A service account to be used by GKE nodes
- A service account to be used by KFP pipelines
- A regional VPC to host a GKE cluster
- A simple GKE cluster with a single (default) node pool
- An instance of Cloud SQL hosted MySQL. For the security reasons, the created instance does not have any use accounts
- A GCS storage bucket

To apply the Terraform configurations:
1. Start **Cloud Shell**.
2. Verify that your GCP project is configured properly
```
gcloud config set project [YOUR_PROJECT_ID]
```
3. If you did not do it in the previous steps replicate the folder structure of this lab in your home directory or clone the whole repository
```
git clone https://github.com/jarokaz/mlops-labs.git
```
4. Navigate to the `terraform folder` in *Lab-00*.
```
cd mlops-labs/Lab-00-Environment-Setup/kfp/terraform
```
5. Initialize Terraform. This downloads Terraform modules used by the script, including the *Google Terraform Provider* and the modules from `jarokaz/terraform-gcp`, and initializes local Terraform state.
```
terraform init
```
6. Apply the configuration. Refer to the previous section for more detail about the variables passed to the apply command. 
```
terraform apply \
-var "project_id=[YOUR_PROJECT_ID]" \
-var "region=[YOUR_REGION]" \
-var "zone=[YOUR_ZONE]" \
-var "name_prefix=[YOUR_NAME_PREFIX]"
```
7. Review the resource configurations that will be provisioned and type `yes` to start provisioning.
8. After the process completes you can review the status of the infrastructure by
```
terraform show
```
### Deploying Kubeflow Pipelines
In this section you deploy Kubeflow Pipelines to your GKE cluster. The KFP services are configured to use the Cloud SQL and the GCS bucket provisioned in the previous step to host metadata databases and an artifacts store.

#### Creating a Kubernetes namespace
The KFP services are installed to a dedicated Kubernetes namespace. You can use any name for the namespace, e.g. *kubeflow*.

1. Get credentials to your cluster:
```
CLUSTER_NAME=$(terraform output cluster_name)
gcloud container clusters get-credentials $CLUSTER_NAME --zone [YOUR_ZONE] --project [YOUR_PROJECT_ID]
```
2. Create a namespace
```
NAMESPACE=[YOUR_NAMESPACE]
kubectl create namespace $NAMESPACE
```

#### Creating `user-gcp-sa` secret
It is recommended to run KFP pipelines using a dedicated service account. A lot of samples, including samples in this repo, assume that the credentials for the service account are stored in a Kubernetes secret named `user-gcp-sa`, under the `application_default_credentials.json` key.

1. Create a private key for the KFP service account created by Terraform in the previous step
```
KFP_SA_EMAIL=$(terraform output kfp_sa_email)
gcloud iam service-accounts keys create application_default_credentials.json --iam-account=$KFP_SA_EMAIL
```
2. Create a Kubernetes secret in your namespace
```
kubectl create secret -n $NAMESPACE generic user-gcp-sa --from-file=application_default_credentials.json --from-file=user-gcp-sa.json=application_default_credentials.json
```
3. Remove the private key
```
rm application_default_credentials.json
```
4. You can verify that the secret stores the private key by executing
```
kubectl get secret user-gcp-sa -n kubeflow -o yaml
```

#### Creating Cloud SQL database user and a Kubernetes secret to hold the user's credentials
The instance of MySQL created by Terraform in the previous step does not have any database users configured. In this step, you create a database user that will be used by KFP and ML Metadata services to access the instance. The services are configured to retrieve the database user credentials from the Kubernetes secret named `mysql-credential`. You can use any name for the database user. Since some older TFX samples assume the user named `root`, it is recommended to use this name if you intend to use the samples from the TFX site. The labs in this repo do not hard code any user names. 

1. Create a database user
```
SQL_INSTANCE_NAME=$(terraform output sql_name)
gcloud sql users create [YOUR_USER_NAME] --instance=$SQL_INSTANCE_NAME --password=[YOUR_PASSWORD] --project [YOUR_PROJECT_ID]
```
2. Create the `mysql-credential` secret to store user name and password
```
kubectl create secret -n $NAMESPACE generic mysql-credential --from-literal=username=[YOUR_USERNAME] --from-literal=password=[YOUR_PASSWORD]
```
3. You can verify that the secret was created in you namespace by executing:
```
kubectl get secret mysql-credential -n $NAMESPACE -o yaml
```
#### Deploying Kubeflow Pipelines using Kustomize
In this step you deploy Kubeflow Pipelines using **Kustomize**.

The Kustomize overlays and patches, which can be found in the `kfp/kustomize` folder, are applied on top of the Kustomize configuration from the Kubeflow Pipelines github repo. 
https://github.com/kubeflow/pipelines/tree/master/manifests/kustomize/env/gcp

The `kustomization.yaml` file in the `kfp/kustomize` folder refers to the 0.1.36 release of KFP as a base. This is the release against which the labs in this repo were tested. As the labs and KFP evolve, this will be updated to align with the required version of KFP.

The `gcp-configurations-patch.yaml` file contains patches that configure the KFP services to retrieve credentials from the secrets created in the previous steps and connection information to the Cloud SQL and the GCS bucket from the Kubernetes **ConfigMap** named `gcp-configs`.

The `gcp-configs` config map is created by **Kustomize** using *configMapGenerator* defined in the `kustomization.yaml` file. The generator is configured to retrieve connections settings from the `gcp-configs.env` environment file.


1. Retrieve connections settings from the Terraform state
```
SQL_CONNECTION_NAME=$(terraform output sql_connection_name)
BUCKET_NAME=$(terraform output artifact_store_bucket)
```
2. Assuming that you are still in the `terraform` folder, navigate to the `kustomize` folder
```
cd ../kustomize
```
3. Update the namespace in the `kustomization.yaml` file
```
kustomize edit set namespace $NAMESPACE
```
4. Create an environment file with connection settings
```
cat > gcp-configs.env << EOF
sql_connection_name=$SQL_CONNECTION_NAME
bucket_name=$BUCKET_NAME
EOF
```
5. Deploy KFP to the cluster
```
kustomize build . | kubectl apply -f -
```
6. To list the workloads comprising Kubeflow Pipelines:
```
kubectl get all -n $NAMESPACE
```


### Accessing KFP UI

After the installation completes, you can access the KFP UI from the following URL. You may need to wait a few minutes before the URL is operational.

```
echo "https://"$(kubectl describe configmap inverse-proxy-config -n kubeflow | grep "googleusercontent.com")
```

