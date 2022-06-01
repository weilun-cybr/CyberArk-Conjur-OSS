# CyberArk-Conjur-OSS

## Overview
The conjur-credentials-plugin makes secrets stored in an existing Conjur database available to Jenkins jobs. Jenkins jobs can authenticate to Conjur and access specific secret values for which they have authorization. You store and manage the secrets in Conjur.

We provide the plugin binary, which you install and configure on your Jenkins host.

On the Conjur side, policy defines the Jenkins host and gives it privileges to access Conjur. Conjur policy also defines the variables that will hold the secret values and authorizes Jenkins to access them. The secret values are loaded and managed in Conjur. Policy is also used to set up automatic rotation for supported variables.

When all configurations are in place, Jenkins scripts and projects simply reference the variable using the configured Jenkins ID.

Please note that this tutorial refers to offical Conjur tutorial at https://docs.conjur.org/Latest/en/Content/Integrations/jenkins.htm.

In this tutorial, you will learn how to secure Jenkins pipelines using Conjur & credentials plugins.

The Conjur Jenkins plugin retrieves secrets from Conjur for use in Jenkins pipeline scripts or Freestyle projects.

## Requirements
This tutorial is tested in AWS Linux. We will assume that you have docker and docker-compose installed. Follow the guide here. 
https://www.cyberciti.biz/faq/how-to-install-docker-on-amazon-linux-2/

You will also need jq installed.
**sudo yum install jq**

## Benefits
The Conjur Jenkins integration provides the following advantages to Jenkins DevOps administrators:
- Security. Secret values are stored and obtained securely. Secrets are not exposed in Jenkins jobs or referenced files.
- Central management. Secrets are managed in a central location, either in Conjur or in the CyberArk Vault if you are using the Vault Conjur Synchronizer.
- Automatic rotation. Secret value rotations are recommended for security. Conjur handles rotation so that no changes are required on the Jenkins side.
- Segregation of duties. Jenkins DevOps administrators are isolated from secrets management.
- Segregation of duties.The plugin supports Jenkins scripts or projects. It supports global or folder-specific configurations.
- Simplification. The plugin simplifies Jenkins job and project creation by requiring only a reference ID to a secret.
- Familiarity. The plugin is configured using the Jenkins UI, a familiar interface for Jenkins users.

## 1.0 Setup Jenkins
Launch Jenkins as a Docker Container with the following command:
```console
mkdir jenkins_home
docker run -d -u root --name jenkins \
    -p 8181:8080 -p 50000:50000 \
    -v ${pwd}/jenkins_home:/var/jenkins_home \
    jenkins/jenkins:lts
```

It will take a few minutes to start Jenkins. Meanwhile, let's move on to next step and start deploying Conjur

Visit the Jenkins Console. This should be **http://(yourIPAddress):8181**.

While prompt for initial password, input the response by this command
  
```console
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

In the next screen, select **Install suggested plugins**.

If any of the plugin failed to be installed, don't worry, we will make sure the necessary plugins work properly in the next few steps.

Create an user called **admin** & password **344827fbdbfb40d5aac067c7a07b9230** to complete the setup

## 2.0 Setup Conjur
If you'd need more details, check out https://www.conjur.org/get-started/install-conjur.html.

### 2.1 Download & Pull
In your terminal, download the Conjur quick-start configuration. It will take a few minutes to pull the images.

```console
curl -o docker-compose.yml https://quincycheng.github.io/docker-compose.quickstart.yml
docker-compose pull
```

### 2.2 Generate Master Key
Generate your master data key and load it into the environment:

```console
docker-compose run --no-deps --rm conjur data-key generate > data_key
```
**Prevent data loss: The conjurctl conjur data-key generate command gives you a master data key. Back it up in a safe location.**

### 2.3 To run the Conjur server, database and client:

```console
export CONJUR_DATA_KEY="$(< data_key)"
docker-compose up -d
```

To verify the containers are up & running, run **docker ps**

The result should be similar to the following:
![Alt text](docker-ps.PNG?raw=true "docker ps")

To create a default account (eg. quick-start):

```console
docker-compose exec conjur conjurctl account create quick-start | tee admin_key
```

If there are errors returned, it is likely that the container is still spinning up. Please repeat this step by running docker-compose command to create the account again.

**Prevent data loss: The conjurctl account create command gives you the public key and admin API key for the account you created. Back them up in a safe location.**

**Please backup the API key for admin for logging in to the system**

## 3.0 Conjur Policies
This section describes all configuration requirements for the Jenkins plugin. It includes Conjur policy requirements, SSL certificate preparation, and Jenkins plugin configuration. We will focus on policy requirements in this step.

### 3.1 Login to Conjur
Initialize Conjur Client

```console
docker-compose exec client conjur init -u conjur -a quick-start
```
 
Logon to Conjur

```console
export admin_api_key="$(cat admin_key|awk '/API key for admin/ {print $NF}'|tr '  \n\r' ' '|awk '{$1=$1};1')"
docker-compose exec client conjur authn login -u admin -p $admin_api_key
```

It should display **Logged in** once you are successfully logged in.

### 3.2 Required Conjur Configurations
The following configurations are required on Conjur:
- Declare the Jenkins host in Conjur policy.
- Declare variables (secrets) in Conjur policy.
- Load secret values into Conjur.

### 3.3 Declare Jenkins host in Conjur policy
The following steps create Conjur policy that defines the Jenkins host and adds that host to a layer.
- Declare a policy branch for Jenkins & save it as a .yml file

```console
docker-compose exec client bash
cat > conjur.yml << EOF
- !policy
  id: jenkins-frontend
EOF
exit
```
- You may change the id in the above example if desired
- Load the policy into Conjur under root:

```console
docker-compose exec client conjur policy load --replace root /conjur.yml
```

- Declare the layer and Jenkins host in another file. Copy the following policy as a template & save it.

```console
docker-compose exec client bash
cat > jenkins-frontend.yml << EOF
- !layer
- !host frontend-01
- !grant
  role: !layer
  member: !host frontend-01
EOF
exit
```

This policy does the following:
- Declares a layer that inherits the name of the policy under which it is loaded. In our example, the layer name will become jenkins-frontend.
- Declares a host named frontend-01
- Adds the host into the layer. A layer may have more than one host. Change the following items:
- Change the host name to match the DNS host name of your Jenkins host. Change it in both the !host statement and the !grant statement.
- Optionally declare additional Jenkins hosts. Add each new host as a member in the !grant statement.

Load the policy into Conjur under the Jenkins policy branch you declared previously:

```console
docker-compose exec client conjur policy load jenkins-frontend /jenkins-frontend.yml | tee frontend.out
```

As it creates each new host, Conjur returns an API key.

We will use the host entity later within this tutorial, so let's put it in memory:

```console
export frontend_api_key=$(tail -n +2 frontend.out | jq -r '.created_roles."quick-start:host:jenkins-frontend/frontend-01".api_key')
echo $frontend_api_key
```

Save the API keys returned in the previous step. You need them later when configuring Jenkins credentials for logging into Conjur.

### 3.4 Declare variables in Conjur policy
The following steps create Conjur policy that defines each variable and provides appropriate privileges to the Jenkins layer to access those variables.

If variables are already defined, you need only add the Jenkins layer to an existing permit statement associated with the variable. The following steps assume that the required variables are not yet declared in Conjur.

Declare a policy branch for the application & save it

```console
docker-compose exec client bash
cat > conjur2.yml << EOF
- !policy
  id: jenkins-app
EOF
exit
```

You may change the id in the above example.

Load the policy into Conjur:

```console
docker-compose exec client conjur policy load root /conjur2.yml
```

Declare the variables, privileges, and entitlements. Copy the following policy as a template:

```console
docker-compose exec client bash
cat > jenkins-app.yml << EOF
#Declare the secrets required by the application

- &variables
  - !variable db_password
  - !variable db_password2

# Define a group and assign privileges for fetching the secrets

- !group secrets-users

- !permit
  resource: *variables
  privileges: [ read, execute ]
  roles: !group secrets-users

# Entitlements that add the Jenkins layer of hosts to the group  

- !grant
  role: !group secrets-users
  member: !layer /jenkins-frontend
EOF
exit
```

This policy does the following:
- Declares the variables to be retrieved by Jenkins.
- Declares the groups that have read & execute privileges on the variables.
- Adds the Jenkins layer to the group. The path name of the layer is relative to root.

Change the variable names, the group name, and the layer name as appropriate.

Load the policy into Conjur under the Jenkins policy branch you declared previously:

```console
docker-compose exec client conjur policy load jenkins-app /jenkins-app.yml
```

### 3.5 Set variable values in Conjur
Use the Conjur CLI or the UI to set variable values.

The CLI command to set a value is:

conjur variable values add <policy-path-of-variable-name> <secret-value>

For example:
```console
password=$(openssl rand -hex 12)
docker-compose exec client conjur variable values add jenkins-app/db_password $password
```
