<!--ts-->
   * [Java and Kubernetes](#java-and-kubernetes)
      * [Part one - base app:](#part-one---base-app)
         * [Requirements:](#requirements)
         * [Build and run application:](#build-and-run-application)
      * [Part two - app on Docker:](#part-two---app-on-docker)
      * [Part three - app on Kubernetes:](#part-three---app-on-kubernetes)
         * [Prepare](#prepare)
         * [Dashboard](#dashboard)
         * [Database](#database)
         * [Prepare Application](#prepare-application)
            * [(First option) Cache your already created Docker image to kubernetes](#first-option-cache-your-already-created-docker-image-to-kubernetes)
            * [(Second option) Build image inside Kubernetes](#second-option-build-image-inside-kubernetes)
         * [Deploy Application](#deploy-application)
         * [Inspections](#inspections)
         * [Manage infrastructure](#manage-infrastructure)
         * [Debug app within Kubernetes:](#debug-app-within-kubernetes)
      * [Good practices](#good-practices)
      * [Tools](#tools)
         * [Switch namespaces with <a href="https://github.com/ahmetb/kubectx"><strong>kubens</strong></a>](#switch-namespaces-with-kubens)
         * [Collect logs from all pods with <a href="https://github.com/wercker/stern"><strong>stern</strong></a>](#collect-logs-from-all-pods-with-stern)
      * [References](#references)
      * [Useful commands](#useful-commands)
         * [Display](#display)
         * [Action](#action)

<!-- Added by: santunioni, at: Tue 23 Mar 2021 08:23:24 PM -03 -->

<!--te-->

--------------------

# Java and Kubernetes

Show how you can move your spring boot application to docker and kubernetes.
This project is a demo for the series of posts on [dev.to](https://dev.to/)
https://dev.to/sandrogiacom/kubernetes-for-java-developers-setup-41nk

## Part one - base app:

### Requirements:

- **Docker and Make (Optional)**

- **Java 11**

Help to install tools:

https://github.com/sandrogiacom/k8s

### Build and run application:

Spring boot and mysql database running on docker

1. **Clone from github repository**

    1. `git clone https://github.com/sandrogiacom/java-kubernetes.git`
    2. `cd java-kubernetes`

2. **Build application**

   `mvn clean install`

3. **Start the database**

    1. `docker run --name mysql57 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=java -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=k8s_java -d mysql/mysql-server:5.7` command to run containerized MySQL   

    **OR**    

    1. use the alias `make run-db`

4. **Run application locally**

   `java --enable-preview -jar target/java-kubernetes.jar`

5. **Check**

   http://localhost:8080/app/users

   http://localhost:8080/app/hello

## Part two - app on Docker:

1. **Create a Dockerfile:**

    ```yaml
    FROM openjdk:11
    RUN mkdir /usr/myapp
    COPY target/java-kubernetes.jar /usr/myapp/app.jar
    WORKDIR /usr/myapp
    EXPOSE 8080
    ENTRYPOINT [ "sh", "-c", "java --enable-preview $JAVA_OPTS -jar app.jar" ]
    ```

2. **Build application and docker image**    

    1. `./mvnw clean install` build the app locally
    2. `docker build -f Dockerfile --force-rm -t java-k8s .` build a docker image with the app content 

    **OR**

    1. use the alias `make build`

3. **Create and run the database**    

    1. `docker run --name mysql57 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=java -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=k8s_java -d mysql/mysql-server:5.7` command to run containerized MySQL

    **OR**

    1. use the alias `make run-db`


4. **Create and run the application**    

    1. `docker run --name myapp -p 8080:8080 -d -e DATABASE_SERVER_NAME=mysql57 --link mysql57:mysql57 java-k8s` command to run containerized application from the image built

    **OR**

    1. use the alias `make run-app`

5. **Check**

    http://localhost:8080/app/users    
    http://localhost:8080/app/hello    

6. **Stop all** (if you want)    
    `docker stop mysql57 myapp`

## Part three - app on Kubernetes:

We have an application in a docker container. Now, we deploy application in a kubernetes cluster running in our machine.

### Prepare

1. **Start minikube**    

    1. start minikube with profile `dev.to` (create a cluster `dev.to` with a control plane node also called `dev.to`)    
    `minikube -p dev.to start --cpus 2 --memory=4096`
    2. enable ingress (proxy reverso que roda em cima do nginx)    
    `minikube -p dev.to addons enable ingress`
    3. enable metrics-server for inspections    
    `minikube -p dev.to addons enable metrics-server`    
    4. create a namespace dev-to for organizing    
    `minikube -p dev.to kubectl create namespace dev-to`    

    **OR**

    1. use the alias    
    `make k-setup`

2. **Check the profile IP**    
    `minikube -p dev.to ip`

### Dashboard

3. **Inspect dashboard**    
    `minikube -p dev.to dashboard`

### Database

4. **Deploy database**    

    1. optional: pull the MySQL image    
    `docker pull mysql/mysql-server:5.7`  
    1. optional: cache the image to the cluster    
    `minikube -p dev.to cache add mysql/mysql-server:5.7`    
    1. apply the [kubernetes scripts](k8s/mysql) to create MySQL deployment and service. (The images will be downloaded from inside the cluster if not already pulled.)  
    `kubectl apply -f k8s/mysql/`

    **OR**

    1. use the alias    
    `make k-deploy-db`    
    
    **check the <pod_name>**    
    `kubectl get pods -n dev-to`
   
5. **Check the logs**    
    `kubectl logs -n dev-to -f <pod_name>`

6. **Forward the database port to localhost**    
    `kubectl port-forward -n dev-to <pod_name> 3306:3306`

### Prepare Application

7. **Build app locally**    
    `./mvnw clean install`

**Choose one of the two options:**

#### (First option) Cache your already created Docker image to kubernetes

8. **Build app Docker image**   

    1. build a docker image with the app content    
    `docker build -f Dockerfile  --force-rm -t java-k8s:latest .`   

    **OR**   

    1. use the alias    
    `make k-build-app`  

7. **Push the already created image to Kubernetes**   

    1. `minikube -p dev.to cache add java-k8s:latest`   

    **OR**   

    1. use the alias    
    `make k-cache-image`   

#### (Second option) Build image inside Kubernetes
 

8. **Create docker image inside minikube machine**:   

    1. `eval $(minikube -p dev.to docker-env) && docker build -f Dockerfile --force-rm -t java-k8s:latest .`   

    **OR**   

    1. use the alias    
    `k-build-image`   

### Deploy Application

10. **Create app deployment and service**   

    1. apply the [kubernetes scripts](k8s/app)    
    `kubectl apply -f k8s/app/`   

    **OR**   

    1. use the alias    
    `make k-deploy-app`   

### Inspections

11. **SSH the cluster**    
    `minikube -p dev.to ssh`    
    **OR** (as root)    
    `docker exec -it dev.to bash`

11. **Check service**    
    `kubectl get services -n dev-to`   

11. **Check IP**    
    - profile IP    
    `minikube -p dev.to ip`   
    - service IP inside the `dev.to` profile    
    `minikube -p dev.to service myapp -n dev-to --url`    
    - service IP inside the `minikube` profile    
    `minikube service myapp -n dev-to --url`  

10. **Edit `hosts`**: map the correct IP: "<ip>	dev.local"
    1. open the file to edit/insert the mapping   
    `sudo vim /etc/hosts`  

    **OR**    

    1. insert the mapping directly    
    `sudo echo "<ip>	dev.local" >> /etc/hosts`

10. **Test (change your IP and PORT as you need)**

    1. **Get users**    
    `curl -X GET http://dev.local/app/users`    

    2. **Post new User**    
    `curl --location --request POST 'http://dev.local/app/users' --header 'Content-Type: application/json' --data-raw ' {"name": "new user", "birthDate": "2010-10-01"}'`    
  

10. **Check pods**    
    `kubectl get pods -n dev-to`    
    `kubectl -n dev-to logs myapp-<xxxxxxxxx-xxxx>`
### Manage infrastructure

10. **Check replicas**    
    `kubectl get rs -n dev-to`    

10. **Delete pod**    
    `kubectl get pods -n dev-to`    
    `kubectl delete pod -n dev-to myapp-<xxxxxxxx-xxxx>`    

10. **Scale**    
    `kubectl -n dev-to scale deployment/myapp --replicas=2`

10. **Test replicas**
    ```bash
    while true
    do curl "http://dev.local/app/hello"
    echo
    sleep 0.5
    done
    ```

10. **Test replicas with wait**
    ```bash
    while true
    do curl "http://dev.local/app/wait"
    echo
    done
    ```

### Debug app within Kubernetes:

1. Add the following environment variable to app-configmap.yaml as data.JAVA_OPTS    
    `data.JAVA_OPTS: "-agentlib:jdwp=transport=dt_socket,address=*:5005,server=y,suspend=n"`

2. Change CMD to ENTRYPOINT on Dockerfile    
    1. find one pod name for the service you want    
    `kubectl get pods -n=dev-to`    
    2. forward the port open for debugging to local-machine    
    `kubectl port-forward -n dev-to <pod_name> 5005:5005`
3. Configure a remote debug in IntelliJ

## Good practices

- Use JRE images, not JDK
- Automate everything possible
- Use Environment variables
- Health Check
- Application info
- Monitoring / Logs

## Tools
### Switch namespaces with [**kubens**](https://github.com/ahmetb/kubectx)    
- access a namespace with    
`kubens dev-to`

### Collect logs from all pods with [**stern**](https://github.com/wercker/stern)    
- Collect logs    
`stern myapp`    
- Microservices hint: put a track id (some hash) to the data for tracking the data flux     
## References
https://kubernetes.io/docs/home/    
https://minikube.sigs.k8s.io/docs/

## Useful commands

### Display
- List profiles (clusters)    
  `minikube profile list`    
- Show node processes    
  `kubectl top node`    
- Show services    
  `kubectl get services -n dev-to`  
- Show replicas    
  `kubectl get rs -n dev-to`
- Show pods    
  `kubectl get pods -n dev-to`  
- Show pod processes    
  `kubectl top pod <nome_do_pod>`  

### Action
- Delete pod    
  `kubectl delete pod -n dev-to <pod_name>`    
- Start all from beginning (make script)    
  `make k:all`      
- Start cluster    
  `minikube -p dev.to start`    
- Stop cluster    
  `minikube -p dev.to stop`    
- Delete cluster    
  `minikube -p dev.to delete`    
