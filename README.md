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

For example
```console
password=$(openssl rand -hex 12)
docker-compose exec client conjur variable values add jenkins-app/db_password $password
```
    
## 4.0 Configure Plugin
Plugin Configuration Use the Jenkins UI to configure the following:
- Conjur/Jenkins connection information.
- Credentials stored in Conjur that Jenkins needs to access. Each secret has a Conjur variable path name and a reference ID to use in the Jenkins script or project.

### 4.1 Load Jenkins Dashboard
You can load the Jenkins' dashboard via the following URL http://(YourIPAddress):8181

The username is **admin** with the password the default **344827fbdbfb40d5aac067c7a07b9230** as we have done initially in 1.0.

### 4.2 Download & install the plugin
Download jenkins-cli
```console
docker exec -it jenkins curl http://localhost:8080/jnlpJars/jenkins-cli.jar --output jenkins-cli.jar
```    
Download Conjur secrets plugin 
```console
docker exec -it jenkins curl -L https://github.com/cyberark/conjur-credentials-plugin/releases/download/v0.8.0/Conjur.hpi -o /var/jenkins_home/plugins/conjur.hpi
```
Update Credential plugin to v2.1.18 or above 
```console
docker exec -it jenkins java -jar jenkins-cli.jar -s http://admin:344827fbdbfb40d5aac067c7a07b9230@localhost:8080/ install-plugin credentials -deploy
```

### 4.3 Define connection information
There are 2 ways you can do this. Via the CLI or WebUI.
    
**Using CLI** 
    
You can create the credential by executing the following commands.
```console
docker exec -it jenkins sh -c "echo \"<com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl> 
<scope>GLOBAL</scope>
  <id>conjur-login</id>
  <description>Login Credential to Conjur</description>
  <username>host/jenkins-frontend/frontend-01</username>
  <password>${frontend_api_key}</password>
</com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl>\" \
 | java -jar jenkins-cli.jar -s http://admin:344827fbdbfb40d5aac067c7a07b9230@localhost:8080/ \
   create-credentials-by-xml system::system::jenkins _ "    
```
    
**Using Jenkins Web UI**

The following steps define the connection to the Conjur appliance. This is typically a one-time configuration.
- In a browser, go to the Jenkins UI.
- Navigate to Jenkins > Credentials > System > Global credentials > host-name.
- The host-name is a Jenkins host. It must match a host that you declared in Conjur policy.
- On the form that appears, configure the login credentials. These are credentials for the Jenkins host to log into Conjur.
    - Scope: Select Global.
    - Username: Enter host/jenkins-frontend/ , where is the network name for the Jenkins host that you declared in Conjur.
    - Password: Copy and paste the API key that was returned by Conjur when you loaded the policy declaring this host.
    - ID: The Jenkins ID, natively provided by Jenkins.
    - Description: Optional. Provide a description to identify this global credential entry.

Access to the Jenkins host and to the credentials is protected by Conjur.

When a host attempts to authenticate with Conjur, Conjur can detect if the request is originating from the host presenting the request. Conjur denies connection if the requestor is not the actual host.
- Click Save.

### 4.4 Decide whether to set up global or folder-level access to Conjur, or a combination of both.
- A global configuration allows any job to use the configuration (unless a folder-level configuration overrides the global configuration).
- A folder-level configuration is specific to jobs in the folder. Folder-level configurations override the global configuration. In a hierarchy of folders, each folder may inherit configuration information from its parent. The top level in such a hierarchy is the global configuration.
- You may set up a global configuration and override it with folder-level configurations.
    
**Using CLI**

You can configure Jenkins manually or by executing the following commands. Please remember to change the IP Address of the **applianceURL**. 
```console
docker exec -it jenkins bash
cat >>/var/jenkins_home/org.conjur.jenkins.configuration.GlobalConjurConfiguration.xml<<EOF
<?xml version='1.1' encoding='UTF-8'?>
<org.conjur.jenkins.configuration.GlobalConjurConfiguration plugin="Conjur@0.2">
  <conjurConfiguration>
    <applianceURL>http://**(YourIPAddress)**:8080</applianceURL>
    <account>quick-start</account>
    <credentialID>conjur-login</credentialID>
    <certificateCredentialID></certificateCredentialID>
  </conjurConfiguration>
EOF
exit
```
If the above command returns an error, it is likely that Jenkins is still being restarted. Please wait for a while and try again
    
**Enable Conjur secrets plugin**
```console
docker exec -it jenkins java -jar jenkins-cli.jar -s http://admin:344827fbdbfb40d5aac067c7a07b9230@localhost:8080/ install-plugin https://github.com/cyberark/conjur-credentials-plugin/releases/download/v0.8.0/Conjur.hpi -restart
```

Jenkins will be restarted, you may need to wait for 1-2 min and login to Jenkins dashboard again to proceed.

### 4.5 Verify Credential created
Access http://(YourIPAddress):8181/credentials/store/system/domain/_/credential/conjur-login/update

You may need to login to the Jenkins dashboard again as the previous step involves restarting of Jenkins. The username is admin with the password the default 344827fbdbfb40d5aac067c7a07b9230.
    
![Alt text](4.5-Verify-Credential-created.PNG?raw=true "Verify Creds")
    
### 4.6 Verify Global Configuration
Access http://(YourIPAddress):8181/configure

You should be able to a section named Conjur Appliance with details configured.
![Alt text](4.6-Verify-Global-Configuration.PNG?raw=true "Verify Config")

## 5.0 Configure Folder
You can override the global configuration by setting the Conjur Appliance information at Folder level, if the checkbox **Inherit from parent**? is checked, it means it will ignore the values set here, and go up the level navigating to the parent folder, or taking the global configuration if all folder up the hierarchy are inheriting from parent. In our example code, we are not inheriting it. Make changes if you want to. Also remember to change the **applianceURL** to yours. 

### 5.1 Create a folder
```console
docker exec -it jenkins bash
cat >> conjur_folder << _EOF_
<?xml version='1.1' encoding='UTF-8'?>
<com.cloudbees.hudson.plugins.folder.Folder plugin="cloudbees-folder@6.4">
  <actions/>
  <description></description>
  <properties>
    <org.conjur.jenkins.configuration.FolderConjurConfiguration plugin="Conjur@0.2">
      <inheritFromParent>false</inheritFromParent>
      <conjurConfiguration>
        <applianceURL>http://192.168.2.206:8080</applianceURL>
        <account>quick-start</account>
        <credentialID>conjur-login</credentialID>
        <certificateCredentialID></certificateCredentialID>
      </conjurConfiguration>
    </org.conjur.jenkins.configuration.FolderConjurConfiguration>
    <org.jenkinsci.plugins.pipeline.modeldefinition.config.FolderConfig plugin="pipeline-model-definition@1.2.7">
      <dockerLabel></dockerLabel>
      <registry plugin="docker-commons@1.11"/>
    </org.jenkinsci.plugins.pipeline.modeldefinition.config.FolderConfig>
    <com.cloudbees.hudson.plugins.folder.properties.FolderCredentialsProvider_-FolderCredentialsProperty>
      <domainCredentialsMap class="hudson.util.CopyOnWriteMap$Hash">
        <entry>
          <com.cloudbees.plugins.credentials.domains.Domain plugin="credentials@2.1.18">
            <specifications/>
          </com.cloudbees.plugins.credentials.domains.Domain>
          <java.util.concurrent.CopyOnWriteArrayList>
            <org.conjur.jenkins.ConjurSecrets.ConjurSecretCredentialsImpl plugin="Conjur@0.2">
              <id>DB_PASSWORD</id>
              <description>Conjur Demo Folder DB Password</description>
              <variablePath>jenkins-app/db_password</variablePath>
            </org.conjur.jenkins.ConjurSecrets.ConjurSecretCredentialsImpl>
          </java.util.concurrent.CopyOnWriteArrayList>
        </entry>
      </domainCredentialsMap>
    </com.cloudbees.hudson.plugins.folder.properties.FolderCredentialsProvider_-FolderCredentialsProperty>
  </properties>
  <folderViews class="com.cloudbees.hudson.plugins.folder.views.DefaultFolderViewHolder">
    <views>
      <hudson.model.AllView>
        <owner class="com.cloudbees.hudson.plugins.folder.Folder" reference="../../../.."/>
        <name>All</name>
        <filterExecutors>false</filterExecutors>
        <filterQueue>false</filterQueue>
        <properties class="hudson.model.View$PropertyList"/>
      </hudson.model.AllView>
    </views>
    <tabBar class="hudson.views.DefaultViewsTabBar"/>
  </folderViews>
  <healthMetrics>
    <com.cloudbees.hudson.plugins.folder.health.WorstChildHealthMetric>
      <nonRecursive>false</nonRecursive>
    </com.cloudbees.hudson.plugins.folder.health.WorstChildHealthMetric>
  </healthMetrics>
  <icon class="com.cloudbees.hudson.plugins.folder.icons.StockFolderIcon"/>
</com.cloudbees.hudson.plugins.folder.Folder>
_EOF_
cat conjur_folder | java -jar jenkins-cli.jar -s http://admin:344827fbdbfb40d5aac067c7a07b9230@localhost:8080/ create-job "Conjur Demo"
exit
```
### 5.2 Verify Folder settings
Access http://(YourIPAddress):8181/job/Conjur%20Demo/configure
![Alt text](5.2-Verify-Folder-settings.PNG?raw=true "Verify Folder")    
    
### 5.3 Create a Conjur Secret in the folder
```console
docker exec -it jenkins bash
echo '<org.conjur.jenkins.conjursecrets.ConjurSecretCredentialsImpl plugin="Conjur@0.5">
        <id>DB_PASSWORD</id>
        <description>Conjur Demo Folder Credentials</description>
        <variablePath>jenkins-app/db_password</variablePath>
      </org.conjur.jenkins.conjursecrets.ConjurSecretCredentialsImpl>' | java -jar jenkins-cli.jar -s http://admin:344827fbdbfb40d5aac067c7a07b9230@localhost:8080/ create-credentials-by-xml "folder::item::Conjur Demo" "(global)"
exit
```
    
### 5.4 Verify Conjur Secret settings
Access http://(YourIPAddress):8181/job/Conjur%20Demo/credentials/store/folder/domain/_/credential/DB_PASSWORD/update
![Alt text](5.4-Verify-Conjur-Secret-settings.PNG?raw=true "Verify SecretSettings")  

## 6.0 From Pipeline Scripts
Conjur supports both Jenkins pipeline script & Freestyle project. If you'd like to use Freestyle project, please skip this step and continue to the next step.

### 6.1 Create a pipeline script
You can create a pipeline script item at http://(YourIPAddress):8181/job/Conjur%20Demo/newJob

Enter **Demo Script** as the item name Choose Pipeline & click OK
![Alt text](6.0-FromPipelineScripts.PNG?raw=true "Create Pipeline")
    
Scroll down to Pipeline. You may need to uncheck Use Groovy Sandbox to make the Script field appear

Copy & Paste the pipeline script to Script field & click Save
```console
node {
      stage('Work') {
         withCredentials([conjurSecretCredential(credentialsId: 'DB_PASSWORD', variable: 'SECRET')]) {
            echo "Hello World $SECRET"
         }
      }
      stage('Results') {
         echo "Finished!"
      }
}
```
![Alt text](6.0-FromPipelineScripts-TheScript.PNG?raw=true "The Script")
    
###6.1 Build It
Build it (optional)
Click Build Now to build it

You should be able to find Hello World **** in the log. Note that you may need to look at the Console Output for this.
![Alt text](6.0-FromPipelineScripts-Console.PNG?raw=true "The Console Output")


