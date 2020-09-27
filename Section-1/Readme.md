# Push MySQL Enterprise Edition Image 
## 1. MySQL Enterprise Edition
Download MySQL Enterprise docker image from https://support.oracle.com.
sign in to your Oracle account, and perform these steps once you are on the landing page:
    
    Select the Patches and Updates tab.

    Go to the Patch Search region and, on the Search tab, switch to the Product or Family (Advanced) subtab.

    Enter “MySQL Server” for the Product field, and the desired version number in the Release field.

    Use the dropdowns for additional filters to select Description—contains, and enter “Docker” in the text field.

    Click the Search button and, from the result list, select the version you want, and click the Download button.

    In the File Download dialogue box that appears, click and download the .zip file for the Docker image. 

Unzip the downloaded .zip archive to obtain the tarball inside (mysql-enterprise-server-version.tar) 

Login into Openshift as system/admin through command line and create new project called "db-mysql-dev"
```
oc login -u system -p admin
oc new-project db-mysql-dev --description="This is example of MySQL deployment on Openshift" --display-name="db-mysql-dev"
oc project db-mysql-dev
```

Copy the tar ball inside minishift. 
On minishift, push docker image into docker registry and get \<dockerID>
```
docker load -i mysql-enterprise-server-version.tar
docker images 
```

Login into internal docker registry and tag the MySQL docker image
```
docker login -u `oc whoami` -p `oc whoami --show-token` 172.30.1.1:5000
docker tag <dockerID> 172.30.1.1:5000/db-mysql-dev/mysql-enterprise-server-8021:latest
```

Push the image into Openshift internal image registry
```
docker push 172.30.1.1:5000/db-mysql-dev/mysql-enterprise-server-8021:latest
```
## 2. MySQL Router
Pull MySQL Router docker image from public repository and get router's \<dockerID>
```
docker pull mysql/mysql-router
docker images
```
Tag the MySQL Router docker image and push the image into Openshift internal image registry
```
docker tag <RouterDockerID> registry.<openshift URL and domain>:443/db-mysql-dev/mysql-router
docker push registry.<openshift URL and domain>:443/db-mysql-dev/mysql-router
```

