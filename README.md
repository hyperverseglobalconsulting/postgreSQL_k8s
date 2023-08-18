# Kubernetes Project: PostgreSQL Deployment

## Overview
This project automates the deployment of PostgreSQL on an AWS EKS cluster. By utilizing Terraform for infrastructure provisioning and Ansible for configuration management, we've streamlined the process to ensure a smooth and efficient deployment. Once set up, users can access the PostgreSQL database either through the psql command-line interface from a bastion instance or programmatically via the psycopg2 Python library.
## Prerequisites
- AWS Account
- Terraform installed
- Ansible installed
- Access to AWS S3 bucket for Terraform state management

## Setup Instructions
### 1. Infrastructure Setup
- #### Modify Bucket Names
Edit the bucket names specified in main.tf and variables.tf to fit your desired AWS environment.

- #### Initialize Terraform:
Navigate to the project's root directory and run:
`terraform init`

- #### Apply Terraform Configuration
Deploy the AWS resources:
`./apply_infrastructure.sh`

- #### Retrieve Bastion Host IP
Once Terraform has finished provisioning the resources, get the public IP of the bastion instance:
`terraform output bastion_public_ip`

- #### Accessing Key Pair
Terraform script will create a key pair and save the private key as `postgresql-k8s.pem` in the current directory. Ensure you keep this key secure.

### Accessing PostgreSQL

#### Via `psql`

1.  SSH into the bastion host:
    
    `ssh -i postgresql-in-eks.pem ec2-user@$(terraform output bastion_public_ip)` 
    
2.  Extract the PostgreSQL root password:
    
    `password=$(kubectl get secret postgresql -o jsonpath='{.data.postgresql-password}' | base64 --decode)` 
    
3.  Access the PostgreSQL instance using the extracted password:
    
    `psql --host=localhost --port=5432 --username=postgres --password=$password` 
    

#### Via `psycopg2`

Make sure your Python script is equipped with the `get_postgresql_password` function to pull the root password from the Kubernetes secret:
  

    from psycopg2 import connect
    from kubernetes import client, config
    import base64
    
    def get_postgresql_password():
        config.load_kube_config()
        v1 = client.CoreV1Api()
        secret = v1.read_namespaced_secret(name="postgresql", namespace="default")
        encoded_password = secret.data["postgresql-password"]
        decoded_password = base64.b64decode(encoded_password).decode('utf-8')
        client.ApiClient().close()
        return decoded_password
    
    password = get_postgresql_password()
    conn = connect(host='localhost', port=5432, dbname='mydatabase', user='postgres', password=password)


#### Via DB Client such as pgAdmin or DBeaver

The PostgreSQL password is additionally saved in AWS Secrets Manager and can be retrieved using the following AWS CLI command:

`aws secretsmanager get-secret-value --secret-id PostgreSQLPassword --region <region> | jq -r .SecretString` 

The script for applying the Terraform script also does port-forwarding to allow connection to `localhost`. You can use the following URL to connect to the PostgreSQL server:

`postgresql://<credentials>@localhost:5432/?sslmode=disable` 

----------

You can adapt the placeholders (`<dbname>`, `<credentials>`, `<region>`, etc.) to your specific setup.
### 3. Destroy infrastructure
Destroy the AWS resources:
`./destroy_infrastructure.sh`

  ## Conclusion
Successfully deploying PostgreSQL on Kubernetes offers a scalable and resilient database platform that's optimized for cloud-native applications. By leveraging AWS resources through Terraform and orchestrating deployments with Ansible, you can achieve a streamlined database setup. The Bitnami Helm chart ensures that best practices are followed for the deployment, storage, and management of PostgreSQL, simplifying maintenance and updates. It's essential to prioritize security by managing database credentials securely and setting up proper access controls. When the setup is no longer needed, or when migrating to different configurations, the Helm chart facilitates easy upgrades and teardowns. This guide serves as a comprehensive resource for initializing, accessing, and managing the PostgreSQL deployment on Kubernetes within AWS.
