# Resident Services Developer Setup

## Overview
Resident services are the self-services which are used by the residents themselves via a portal. Resident Portal is a web based UI application that provides residents of a country the services related to their Unique Identification Number (UIN). To know more, read [here](https://docs.mosip.io/1.2.0/modules/resident-services).

The documentation here will guide you through the pre-requisties required for the developer' setup.

## Software setup

Below are a list of tools required in Resident Services:
1. JDK 11
2. Any IDE (like Eclipse, IntelliJ IDEA)
3. Apache Maven (zip folder)
4. pgAdmin
5. Postman
6. Git
7. Notepad++ (optional)
8. lombok.jar (file)
9. settings.xml (document)


Follow the steps below to setup Resident services on your local system

1. Download `lombok.jar` and `settings.xml` from [here](https://github.com/mosip/documentation/tree/1.2.0/docs/_files/resident-services-config-files).

2. Unzip Apache Maven and move the unzipped folder in ```C:\Program Files``` and `settings.xml` to "conf" folder `C:\Program Files\apache-maven-3.8.4\conf`.

3. Install Eclipse, open the `lombok.jar` file and wait for some time until it completes the scan for Eclipse IDE and then click `Install/Update`.

<img src="_images/lombok-configuration.png" width="750" height="450">

4. Check the Eclipse installation folder `C:\Users\userName\eclipse\jee-2021-12\eclipse` to see if the `lombok.jar` is added. Now you don't have to add the dependency of `lombok` in your `pom.xml` file separately because it will be auto-configured by Eclipse.

5. Configure the JDK (Standard VM) with your Eclipse by traversing through `Preferences → Java → Installed JREs`.

<img src="_images/installed-jre.png" width="750" height="450">
 
## Code setup

For the code setup, clone the repository and follow the steps mentioned in the [Contribution guidelines](https://docs.mosip.io/1.2.0/community/code-contributions).

### Importing and building of a project

1. Open the project folder where `pom.xml` is present.
2. Open command prompt from the same folder.
3. Now run this command to build the project ```mvn clean install -Dgpg.skip=true -DskipTests=true``` and wait for build to complete successfully.
4. After building of a project, open Eclipse and select `Import Projects → Maven → Existing Maven Projects → Next → Browse to project directory → Finish`.

<img src="_images/import-project.png" width="750" height="450">

5. After successful importing of project, update the project by right-click on `Project → Maven → Update Project`.

## Environment setup

1. For the environment setup, you need an external JAR that is available [here](https://oss.sonatype.org/#nexus-search;gav~~kernel-auth-adapter~1.2.0-SNAPSHOT~~) with different versions. (eg., you can download `kernel-auth-adapter.jar` and add to project Libraries → Classpath → Add External JARs → Select Downloaded JAR → Add → Apply and close).

<img src="_images/add-external-library.png" width="750" height="450">

2. Now, clone 'mosip-config' repository from [here](https://github.com/mosip/mosip-config). Follow all the steps of cloning a repository and adding a remote repository and constraints.

3. Create an empty folder inside the 'mosip-config' with 'sandbox-local' name and then copy and paste all config files inside 'sandbox-local' folder except .gitignore, README and LICENSE.

4. As resident-services is using two properties files, `resident-default` and `application-default`, you will have to configure them according to your environment.    The same files are available [here](https://github.com/mosip/documentation/tree/1.2.0/docs/_files/resident-services-config-files) for your reference.

5. To run the server, two files are required- [kernel-config-server.jar](https://oss.sonatype.org/#nexus-search;gav~~kernel-config-server~1.2.0-SNAPSHOT~~) and [config-server-start.bat](https://github.com/mosip/documentation/blob/1.2.0/docs/_files/resident-services-config-files/config-server-start.bat).

6. Put both the files in the same folder and change the location attribute to `sandbox-local` folder in `config-server-start.bat` file and also check the version of `kernel-config-server.jar` in the end of command. 

 For example,
 
 ```java -jar -Dspring.profiles.active=native  -Dspring.cloud.config.server.native.search-locations=file:C:\Users\myDell\mosipProject\mosip-config\sandbox-local -Dspring.cloud.config.server.accept-empty=true  -Dspring.cloud.config.server.git.force-pull=false -Dspring.cloud.config.server.git.cloneOnStart=false -Dspring.cloud.config.server.git.refreshRate=0 kernel-config-server-1.2.0-20201016.134941-57.jar```.

As mentioned in step 4, you may have to make some changes in the two properties files.

For example,

    * `mosip.mosip.resident.client.secret=xyz789` you need to add this property to be able to use a decrypted passcode and run it in your local machine. 
    * If you are running it on a server, then you have to use an encrypted passcode like this `mosip.mosip.resident.client.secret={cipher}1bdd7e59ca3a9dbe66b47db3ecb7025e66a6746911de2bd841c804f`.
    * You need to comment this out `auth.server.admin.issuer.internal.uri` in `application-default` file because you already have this `auth.server.admin.issuer.uri` so there is no need of `auth.server.admin.issuer.internal.uri`.
    * If you check the URLs present in these files, they are set to default with port no. 80 (or any other port number) but you will need to use external URL to access it.
    * In the beginning of `resident-default` file, you need to add `mosipbox.public.url=https://${domain.url}` and change all other URLs with `${mosipbox.public.url}`. 
    * It is because you will pass this domain URL in Eclipse VM arguments like this `-Ddomain.url=dev.mosip.net` which results in `mosipbox.public.url=https://dev.mosip.net` and it will connect with the Development environment.

7. Now run the server by opening the `config-server-start.bat` file.

<img src="_images/run-server.png" width="750" height="450">

Now the server would be up and running. 

Below are the configurations to be done in Eclipse:

1. Open Eclipse and run the project for one time as 'java application', so that it will create a java application which you can see in debug configurations and then change it's name. (e.g.: project name with environment - "Resident-dev").

<img src="_images/create-env-in-eclipse.png" width="750" height="450">

2. Now, open the arguments and pass this `-Ddomain.url=dev.mosip.net -Dapplication.base.url=http://localhost:8090 -Dspring.profiles.active=default -Dspring.cloud.config.uri=http://localhost:51000/config -Dspring.cloud.config.label=master` in VM arguments. 

3. Here domain URL represents the environment on which you are working (eg., it can be ```dev2.mosip.net``` or ```qa3.mosip.net```).

<img src="_images/vm-arguments.png" width="750" height="450">

4. Now, click on Apply and then debug it (it starts running). In the console, you can see a message like `"Started ResidentBootApplication in 34.078 seconds (JVM running for 38.361)"`.

## Resident services API documentation

* You can find all the APIs used in resident-services 1.2.0 [here](https://mosip.github.io/documentation/1.2.0/resident-services.html).

* You can test the APIs with the help of Swagger-UI and Postman. 

* Swagger is an interface description language for describing restful APIs expressed using JSON. You can access Swagger-UI of resident-services for dev-environment from `https://dev.mosip.net/resident/v1/swagger-ui/index.html?configUrl=/resident/v1/v3/api-docs/swagger-config` and localhost from `http://localhost:8099/resident/v1/swagger-ui/index.html?configUrl=/resident/v1/v3/api-docs/swagger-config`.

* Postman is an API platform for building and using APIs. Postman simplifies each step of the API lifecycle and streamlines collaboration so you can create better APIs—faster. It is widely used tool for API testing. Below you will find the APIs postman collection of resident-services.

* You need to download the JSON collection from [here](https://github.com/mosip/documentation/blob/1.2.0/docs/_files/resident-services-config-files/Resident-Service-APIs.postman_collection.json) and then import it in your 'postman'.

<img src="_images/import-apis-in-postman.png" width="750" height="450">

* Now, you need to create an environment as shown in the image below. 
 
This environment is created for dev and give the variable name as `url` and set both the values as `https://dev.mosip.net`.

<img src="_images/dev-env-postman.png" width="750" height="400">

* In the similar way, create another environment as seen below .
 
This environment is created for localhost and give the variable name as `url` and set both the values as `http://localhost:8099`.

<img src="_images/localhost-env-postman.png" width="750" height="400">
