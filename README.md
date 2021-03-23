[TOC]
# Java and Kubernetes

Show how you can move your spring boot application to docker and kubernetes.
This project is a demo for the series of posts on dev.to
https://dev.to/sandrogiacom/kubernetes-for-java-developers-setup-41nk

## Part one - base app:

### Requirements:

- **Docker and Make (Optional)**

- **Java 11**

Help to install tools:

https://github.com/sandrogiacom/k8s

### Build and run application:

Spring boot and mysql database running on docker

1. **Clone from repository**

    1. `git clone https://github.com/sandrogiacom/java-kubernetes.git`
    2. `cd java-kubernetes`

2. **Build application**

   `mvn clean install`

3. **Start the database**

    1. `docker run --name mysql57 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=java -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=k8s_java -d mysql/mysql-server:5.7` command  to run containerized MySQL   

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

We have an application and image running in docker. Now, we deploy application in a kubernetes cluster running in our machine.

### Prepare

1. **Start minikube**    

    1. `minikube -p dev.to start --cpus 2 --memory=4096` start minikube
    2. `minikube -p dev.to addons enable ingress` enable ingress
    3. `minikube -p dev.to addons enable metrics-server` enable metrics-server for inspections
    4. `kubectl create namespace dev-to` create namespace dev-to for organizing

    **OR**

    1. use the alias `make k-setup`

2. **Check IP**    
    `minikube -p dev.to ip`

### Dashboard

3. **Inspect dashboard** 
    `minikube -p dev.to dashboard`

### Database

4. **Deploy database**    

    1. `kubectl apply -f k8s/mysql/`

    **OR**

    1. use the alias `make k-deploy-db`
    
    **check the <pod_name> with** `watch kubectl get pods -n dev-to`
    

create MySQL deployment and service

5. **Check the logs**    
    `kubectl logs -n dev-to -f <pod_name>`

6. **Foward the database port to localhost**    
    `kubectl port-forward -n dev-to <pod_name> 3306:3306`



### Application

**Choose one of the two options:**

#### Build app inside Kubernetes

7. **Build app inside Kubernetes**   

    1. `mvn clean install` build app in local machine   
    2. `docker build --force-rm -t java-k8s .` build the docker image copying the app content to container`   

    **OR**   

    1. use the alias `make k-build-app`   

8. **Create docker image inside minikube machine**:   

    1. `eval $$(minikube -p dev.to docker-env) && docker build --force-rm -t java-k8s .`   

    **OR**   

    1. use the alias `k-build-image`   

#### Cache your already created Docker image to kubernetes

7. **Push the already created image to Kubernetes**   

    1. `minikube cache add java-k8s`   

    **OR**   

    1. use the alias `make k-cache-image`   


8. **Create app deployment and service**   

    1. `kubectl apply -f k8s/app/`   

    **OR**   

    1. use the alias `make k-deploy-app`   

### Inpections

9. **Check service**

    1. `kubectl get services -n dev-to` check the services    
    2. `minikube -p dev.to service -n dev-to myapp --url` access app    
    Ip examples:    
       - http://172.17.0.3:32594/app/users    
       - http://172.17.0.3:32594/app/hello    

10. **Check pods**    

    `kubectl get pods -n dev-to`
    `kubectl -n dev-to logs myapp-6ccb69fcbc-rqkpx`

### Map to dev.local

11. **Map the minikube to dev.local**    
    `minikube -p dev.to ip` 

12. **Edit `hosts`** 
    ```bash
    sudo vim /etc/hosts
    ```

12. **Replicas**
    ```bash
    kubectl get rs -n dev-to
    ```

12. **Get and Delete pod**
    ```bash
    kubectl get pods -n dev-to
    kubectl delete pod -n dev-to myapp-f6774f497-82w4r
    ```

12. **Scale**
    ```bash
    kubectl -n dev-to scale deployment/myapp --replicas=2
    ```

12. **Test replicas**
    ```bash
    while true
    do curl "http://dev.local/app/hello"
    echo
    sleep 2
    done
    ```

12. **Test replicas with wait**
    ```bash
    while true
    do curl "http://dev.local/app/wait"
    echo
    done
    ```

### Check app url

12. **Check URL**    
    `minikube -p dev.to service -n dev-to myapp --url`

12. **Change your IP and PORT as you need it**    
    `curl -X GET http://dev.local/app/users`

12. **Add new User**
    ```bash
    curl --location --request POST 'http://dev.local/app/users' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "name": "new user",
        "birthDate": "2010-10-01"
    }'
    ```

## Part four - debug app:

add   JAVA_OPTS: "-agentlib:jdwp=transport=dt_socket,address=*:5005,server=y,suspend=n"

change CMD to ENTRYPOINT on Dockerfile

`
kubectl get pods -n=dev-to
`

`
kubectl port-forward -n=dev-to <pod_name> 5005:5005
`

## KubeNs and Stern

`
kubens dev-to
`

`
stern myapp
` 

## Start all

`make k:all`


## References

https://kubernetes.io/docs/home/

https://minikube.sigs.k8s.io/docs/

## Useful commands

```
##List profiles
minikube profile list

kubectl top node

kubectl top pod <nome_do_pod>
```
