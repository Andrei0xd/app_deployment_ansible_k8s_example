# 1. Single-node Kubernetes cluster deployment of a simple node app and a database

Running the playbook deploy_app_playbook.yml with the inventory file in this repo will fully deploy the app inside /app - Docker's to-do example app, via minikube, on the host VM given to the inventory

For quickly deploying a single-node cluster of a simple app such as this one, I chose to simply use minikube and ansible's k8s module, to apply the changes to the cluster.


The playbook calls the role app_deployment_role, which is easily extendable to other platforms, but has been created and tested for ubuntu. This role will:

- Ensure that docker exists on the VM and will install it if it does not. Also installs pip and the docker python package, since ansible's docker module needs that.

- Pull the image described in defaults/main.yml or described in any other way as ansible vars, either by setting them in the playbook, giving them per host in the inventory file, or giving them as extra vars when calling the playbook. The image is pulled locally only when PULL_IMAGE_LOCAL is set to true

- Ensure kubectl and minikube are installed, or will install them otherwise. Also installs pip and the docker kubernetes package, since ansible k8s module needs that.

- Start minikube

- Create the following resources:
    - The namespace, being given as a var in invetory or in any other way.
    - A docker registry secret, so it can pull the image. The auth user/password are being given as vars
    - A PersistentVolumeClaim for the mysql service
    - A kubernetes secret to store the mysql password
    - The MySQL Deployment, which will be set in the namespace given above and have labels that are also given as vars in the inventory. Same for its replica count. This deployment's container runs the mysql:5.7.37 official docker image, and also through vars set in the inventory, a user and its password, and also a new database, will be created when the container is created.
    - The service for this mysql deployment, routing traffic to port 3306.
    - Same thing for the to-do app, but the traffic is being rotued to port 3000, on which the node server is listening.
    - An ingress resource, routing traffic to our todo-service on port 3000
    - An ingress controller, being created with minikube's nginx ingress addon
    
- At the end, it is just using ngrok to easily expose all this publicly.
___

# 2. Updating the app/redeploying/CD:

- I have edited the Github Workflow given in the original to-do app repo, such that on a push, pull, or on workflow_dispatch (manually running a github action), the app's image is being rebuilt and pushed onto my docker registry. That is also being done by storing the docker registry password in the repo's secrets.

- I chose not to run the whole playbook again in the workflow, but that is easily done with a single task to fully redeploy the app
___

# 3. Variables needed to be set



## Most of the required vars have default values already set in the roles /defaults/main.yml file

- DOCKER_REPO: this is the repo in which the image is present
- APP_VERSION: the app's version, being treated as the images tag, e.g. :latest
- PULL_IMAGE_LOCAL: bool that pulls the image via the docker cli after installing docker, was useful for testing
- mysql_user: the user that will be created on the mysql server
- mysql_database_name: the database that will be created on the mysql server
- EXPOSE_WITH_NGROK: run ngrok at the end to publicly expose the app.



## Preferrably per host:

- app_deployment_name: label for the app name
- mysql_deployment_name: label for the mysql resource
- deployment_env: label for different environments, e.g. could be set to dev/prod/staging
- namespace: deployment's namespace in which all resources will be created
- replica_count



## These vars are treated as secrets and I am taking them from an Ansible Vault:

- DOCKERHUB_USERNAME
- DOCKERHUB_EMAIL
- DOCKERHUB_TOKEN
- MYSQL_PASSWORD: This is the password that will be set for the newly added user, when the database is created. It is also used by the node app to connect to the mysql server.

# 4. Accessing the app:

The app is currently running at https://a420-4-231-238-161.ngrok-free.app/ (and will hopefully still be by the time you access it)
