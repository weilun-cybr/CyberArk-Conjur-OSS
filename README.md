# CyberArk-Conjur-OSS

## Overview
The conjur-credentials-plugin makes secrets stored in an existing Conjur database available to Jenkins jobs. Jenkins jobs can authenticate to Conjur and access specific secret values for which they have authorization. You store and manage the secrets in Conjur.

We provide the plugin binary, which you install and configure on your Jenkins host.

On the Conjur side, policy defines the Jenkins host and gives it privileges to access Conjur. Conjur policy also defines the variables that will hold the secret values and authorizes Jenkins to access them. The secret values are loaded and managed in Conjur. Policy is also used to set up automatic rotation for supported variables.

When all configurations are in place, Jenkins scripts and projects simply reference the variable using the configured Jenkins ID.

Please note that this tutorial refers to offical Conjur tutorial at https://docs.conjur.org/Latest/en/Content/Integrations/jenkins.htm.

In this tutorial, you will learn how to secure Jenkins pipelines using Conjur & credentials plugins.

The Conjur Jenkins plugin retrieves secrets from Conjur for use in Jenkins pipeline scripts or Freestyle projects.

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

Visit the Jenkins Console. This should be https://<IPAddress>:8181.

While prompt for initial password, input the response by this command
  
```console
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

In the next screen, select **Install suggested plugins**.

If any of the plugin failed to be installed, don't worry, we will make sure the necessary plugins work properly in the next few steps.

Create an user called **admin** & password **344827fbdbfb40d5aac067c7a07b9230** to complete the setup


### 1.1 Note on SELinux and Container Volumes
Put in something

## 2.0 Setup host prerequisites
Start something else

### 2.1 Note on SELinux and Container Volumes
Talk about that something else. 
```console
My word.
Oh my god.
```
