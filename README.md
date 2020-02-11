# OCI List Instances API


This instruction is to show, how to run the developed List Instances API in either inside a Docker Container or inside a Kubernetes Cluster and then use Oracle Cloud Infrastructure API Gateway to expose it.

This is split in four buckets. Firstly we will see how to build the Docker container to generate the List of Instances, export it to a CSV file & upload it to Object Storage Bucket. Secondly, we will build a Docker Container for the actual Python Flask API and push it to a Docker repository. Third, we will deploy it your OKE Cluster and at the end we will expose it through OCI API Gateway.

We will use OCI Cloud Shell to build everything off, so there is no need to have any intervention on your local machine. Below is a diagram that depicts our Architecture which will be implemented by end of this instruction.



![API-GW-Arch](https://github.com/stretchcloud/OCI-APIGW-Demo-API/blob/master/API-GW-Arch.png)





## 1. Clone the repository, build & run the Docker Container



Login to your Cloud Shell Console and clone this repository. If you don't know how to login to the Cloud Shell, then visit this [Lab Guide](https://oke-rancher.gitbook.io/oke-rancher/) to see how to do it yourself. OCI Cloud Shell is a free feature for the developers who would like to simplify things at the time of development and run the commands within the shell without any extra step to perform the OCI CLI config.

```bash
$ git clone https://github.com/stretchcloud/OCI-APIGW-Demo-API
$ cd OCI-APIGW-Demo-API/ListInstances
```

You need to edit the config file according to your OCI credentials and also upload the Private Key in PEM format. These files are already in the directory. Just edit and paste your content. If you are unsure, then follow this [link](https://docs.cloud.oracle.com/en-us/iaas/Content/API/Concepts/sdkconfig.htm) to get your Keys in there. While editing the config file leave the ***key_file*** parameter as it is in the file.

```bash
Run this to build the Docker container, push it to a repository and run the container to generate & upload the CSV file for the Instances.

$ docker build -t listinstances:latest .
$ docker tag listinstances:latest <docker-hub-handle>/listinstances:latest
$ docker login
$ docker push <docker-hub-handle>/listinstances:latest
$ docker run --name listinstancesapp <docker-hub-handle>/listinstances:latest
```





## 2. Build the Docker Container for the Flask API

Dockerfile to create the Python Flask API has been already pushed to the repo. You need to build the Docker Container of the API and then push it to a Docker Registry. You need to use the same upstream Docker Image to deploy this on top of your OKE Cluster.

To build this image, you need to copy your OCI Config file content along with the Private Key content in PEM format here. This is the same step that you have performed earlier. So, you can just copy paste the same file in this directory.

```bash
$ cp OCI-APIGW-Demo-API/ListInstances/config OCI-APIGW-Demo-API/Flask-API/
$ cp OCI-APIGW-Demo-API/ListInstances/oci_api_key.pem OCI-APIGW-Demo-API/Flask-API/
$ cd OCI-APIGW-Demo-API/Flask-API
$ docker build -t flaskapp:latest .
$ docker tag flaskapp:latest <docker-hub-handle>/flaskapp:latest
$ docker push <docker-hub-handle>/flaskapp:latest
```



This will build the Docker Container for the Flask API and push it your Docker Repository. We will use this in our next step to deploy it on top of OKE Cluster.



## 3. Deploy it on OKE Cluster

We will assume that you already have a OKE Cluster deployed and ready to be consumed. If not then you can follow this [link](https://oke-rancher.gitbook.io/oke-rancher/ ) to create one using Rancher Management Console.

We have provided the Kubernetes manifests inside the K8S-Deployment folder. Let's use it to deploy the API.



```bash
$ cd OCI-APIGW-Demo-API/K8S-Deployment

Here edit the apiapp-deployment.yaml file and change the image: jit2600/flaskapp:latest to your Docker Registry and image that you have pushed to in step 2.

$ kubectl apply -f .
$ kubectl get svc flaskapp

Make a note of the Public IP (coming from OCI Loadbalancer) as we need it to expose via API Gateway in the next section.
```



This will deploy the Flask API on top of your OKE Cluster and have a OCI Loadbalancer backed public IP to access it. Make sure that you have port 5000 open in the OCI Security Lists.





## 4. Deploy the API Gateway & API Deployment Resources

This is the last step of our demo. You need to create a API Gateway along with API Gateway Deployments using the provided JSON file.



```bash
$ oci iam compartment list

Make a note of the Compartment OCID where you need to deploy the API Gateway

$ oci network vcn list --compartment-id <compartment-ocid>

Make a note of the OCID of the VCN where you want to deploy the API Gateway

$ oci network subnet list --compartment-id <compartment-ocid> --vcn-id <vcn-ocid>

Make a note of the Subnet OCID where the API Gateway is finally going to be connected.

Open up the createapigw.sh file and change the <compartment-ocid> with your recorded Compartment OCID and change the <subnet-ocid> with your recorded Subnet OCID. Make the script executable and run the script.

$ cd OCI-APIGW-Demo-API/API-GW
$ chmod +x createapigw.sh
$ ./createapigw.sh

This will deploy your API Gateway and wait for the state to become ACTIVE. From the outout make a note of the OCID of the API Gateway as well. You need it for the next step where you will use it to create the Deployment.
```



You just create the API Gateway, let's create the API Deployment that will expose the Flask API using API Gateway.

To create the Deployment, you need to use the OKE Loadbalancer Public IP, edit the provided JSON file and change the <OKE-LB-SVC-PUB-IP> in both the JSON file with your OKE LB Public IP. Let's create the Deployment now.



```bash
$ oci api-gateway deployment create --gateway-id <API-GW-OCID> --compartment-id <compartment-ocid> --path-prefix "/v1" --specification file://simple-deployment-json

$ oci api-gateway deployment create --gateway-id <API-GW-OCID> --compartment-id <compartment-ocid> --path-prefix "/v2" --specification file://wildcard-deployment.json

This will create two deployments, the first one being the one to list all of the instances and the second one where we will provide the shape of the instance and it will query the database to match and retrieve the result. Make a note of the Deployment OCID for both the deployment, this is required to retrieve the endpoint.

We have used context variables to policies and HTTP back end definitions for the path specification of our API.

To access the API Endpoint using API Gateway, you need to retrieve the endpoint of the Deployment. Run this to retrieve the endpoint

$ oci api-gateway deployment get --deployment-id <deployment-ocid> | grep -i endpoint
$ curl -X GET -H "Content-type: application/json" <endpoint-url>/instances | python -m json.tool
$ curl -X GET -H "Content-type: application/json" <endpoint-url>/instances/{instance-shape} | python -m json.tool

This will fetch the result from the SQLite Database and the json.tool will format the output in a human readable format.
```



# API Spec

The API has been implemented as per the below HTTP endpoint.

If you want to, you can expand the capabilities of the API, such as, search the Instances based on family or running status etc.

| HTTP Verb | Path                 | Request Content-Type | Request body | Response Content-Type | Example response body                                        |
| --------- | -------------------- | -------------------- | ------------ | --------------------- | ------------------------------------------------------------ |
| GET       | `/instances`         | `application/json`   | -            | `application/json`    | `{"data": [{"AD": "EU-FRANKFURT-1-AD-3", "Compartment": "Dyn", "Instances-env.Operation": "n/a", "Licensed": "BYOL", "MEMORY": "15", "Name": "fra-01-wp", "OCPU": "1", "OS": "Canonical-Ubuntu-16.04-2018.10.16-0", "PrivateIP": "10.0.2.3 ", "PublicIP": "130.1.1.1 ", "SSD TB": "0", "Service": "Compute", "Shape": "VM.Standard2.1", "State": "RUNNING", "Version": "n/a"}]}` |
| GET       | `/instances/{shape}` | `application/json`   | -            | `application/json`    | `{"data": [{"AD": "EU-FRANKFURT-1-AD-3", "Compartment": "Dyn", "Instances-env.Operation": "n/a", "Licensed": "BYOL", "MEMORY": "15", "Name": "fra-01-wp", "OCPU": "1", "OS": "Canonical-Ubuntu-16.04-2018.10.16-0", "PrivateIP": "10.0.2.3 ", "PublicIP": "130.1.1.2", "SSD TB": "0", "Service": "Compute", "Shape": "VM.Standard2.1", "State": "RUNNING", "Version": "n/a"}]}` |



