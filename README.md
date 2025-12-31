
# Java Application Deployment on Kubernetes

<!-- <br> -->

## Project Overview

This project demonstrates deploying a **Java Spring Boot (PetClinic) application** on Kubernetes with a **MySQL database**, following production-grade DevOps practices. The setup uses Kubernetes primitives such as **Namespaces, Deployments, StatefulSets, Services, ConfigMaps, and Secrets** to achieve a clean, scalable, and secure architecture.

---

## Objectives

- Deploy a Java application on Kubernetes
    
- Deploy a MySQL database as a stateful workload
    
- Use ConfigMaps and Secrets for configuration management
    
- Enable rolling updates with zero downtime
    
- Observe application metrics using Kubernetes tools
    

---

## Architecture Overview

![architecture](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/architecture.png)


**Namespaces**

- `pet-clinic-db` → Database components
    
- `pet-clinic-app` → Application components
    

**High-Level Flow**

```
User
 ↓
Kubernetes Service
 ↓
Java Application Pod
 ↓
MySQL Service
 ↓
MySQL Pod (StatefulSet)
```

---
You will practically use the following key Kubernetes objects. It will help you understand how these objects can be used in real-world project implementations:

1. Deployment
2. HPA
3. ConfigMap
4. Secrets
5. StatefulSet
6. Service
7. Namespace

It also covers key concepts such as:

1. Creating an application properties file from ConfigMap to be consumed by the app
2. Startup, readiness, and liveness probes
3. Consuming secret and configmap data using Environment Variables
4. MySQL initialization script from ConfigMap to create tables

# Step-by-Step Implementation


## Build and Create Java Application Docker Image

Given below is the step-by-step guide to building and deploying Java Apps with MySQL on Kubernetes.

Clone the Git repository below, which contains the source code to build the application. We will using the publicly available pet clinic spring boot app for this setup.

```
https://github.com/spring-projects/spring-petclinic.git
```

### Step 1: Build the Java Application

Now, CD into the **spring-petclinic** directory and run the below command to [build the application using maven.](https://devopscube.com/build-java-application-using-maven/)

```
mvn clean install
```

If you want to skip the test during the build, use the **`-DskipTests`** flag .

### Step 2: Build a Docker Image of the Application

To [build the docker image](https://devopscube.com/build-docker-image/) of the application, create a Dockerfile using the below content inside the **spring-petclinic** directory.

```
FROM techiescamp/jre-17:1.0.0
COPY /target/*.jar /app/java.jar
EXPOSE 8080
```

This dockerfile uses **`techiescamp/jre-17:1.0.0`** as the base image which has JRE installed in it and it copies the JAR file from the target folder and pastes it inside the docker image inside the **/app** folder as **java.jar**.

It also exposes port 8080, because the Java application runs on port 8080.

Run the below command from the same directory where the [Dockerfile](https://devopscube.com/create-dockerfile-using-docker-init/) is present to build the docker image

```
docker build -t kube-petclinic-app:3.0.0 .
```

**kube-petclinic-app:3.0.0** is the name and tag I have given to my docker image.

Run the `docker images` command to check if your docker image has been created successfully.

### Step 3: Push the image to DockerHub

Before pushing the image to the DockerHub repository, you have to tag the image that you want to push, use the below to tag the image

```
docker tag name:v1 docker_username/imagename:v1
```

In this command **kube-petclinic-app:3.0.0** is the name of the image I build and **techiescamp/kube-petclinic-app:3.0.0** is my DockerHub repository and image tag.

Now, push the image to DockerHub using the command. Before proceeding, ensure you have **logged in to docker hub** from the CLI.

```
docker push username/kube-petclinic-app:3.0.0
```

## Deploy Java & MySQL On Kubernetes

Clone the Git repository given below which contains all the YAML manifest we used in this guide under **03-java-app-deployment** folder.

```
git clone https://github.com/techiescamp/kubernetes-projects.git
```

In the **03-java-app-deployment** folder, you can see the YAML manifests in the below structure.

```
.
├── java-app
│   ├── configmap.yml
│   ├── hpa.yml
│   └── java.yml
├── mysql
│   ├── configmap.yml
│   └── mysql.yml
└── secret.yml
```

Given below are the steps to deploy Java and MySQL on Kubernetes
### 1. Namespace Creation

Namespaces are used to logically isolate the database and application components.

```
kubectl create ns pet-clinic-app

kubectl create ns pet-clinic-db
```

![namespace](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090341.png)

This improves security, manageability, and clarity.

---
## 2.Create Secrets

CD into the **03-java-app-deployment** directory and run the **secret.yml** in both **pet-clinic-app** and **pet-clinic-db** namespaces because the secret contains the MySQL login credentials which are needed in both MySQL and app deployment process.

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-cred
  namespace: pet-clinic-app
type: Opaque
data:
  username: Y3J1bmNob3Bz
  password: Y3J1bmNob3BzQDEyMzQ=
```

Just change the namespace and run the **secret.yml** file two times to create a secret in both **pet-clinic-app** and **pet-clinic-db** namespaces.

```
data:
  username: Y3J1bmNob3Bz
  password: Y3J1bmNob3BzQDEyMzQ=
```

You can see the data in the above block is given in base64-encoded values because the secret won't be created unless you specify your username and password in **base64-encoded** values.

You can use the **echo** command with **base64** to encode your data, the echo command will give the encoded values of your data. The command to encode your data is given below

```
echo -n '<data>' | base64
```

For example, I set my **username** as **crunchops**, then the echo command will be

```
echo -n 'crunchops' | base64
```

This command will give the encoded values of my data.

Now, create the secret in both namespaces using the following command.

```
kubectl apply -f secret
```

Run the following command to verify the secret is created in both namespace

```
kubectl get secret -n pet-clinic-app

kubectl get secret -n pet-clinic-db
```

![secret](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090401.png)


---
### Step 3: Create ConfigMap for MySQL

CD into the **03-java-app-deployment/mysql** directory and run the **configmap.yml**.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  namespace: pet-clinic-db
data:
  init.sql: |
    CREATE USER '$MYSQL_USER'@'%' IDENTIFIED BY '$MYSQL_PASSWORD';
    GRANT ALL PRIVILEGES ON petclinic.* TO '$MYSQL_USER'@'%';
    FLUSH PRIVILEGES;

    USE petclinic;

    CREATE TABLE IF NOT EXISTS vets (
      id INT(4) UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
      first_name VARCHAR(30),
      last_name VARCHAR(30),
      INDEX(last_name)
    ) engine=InnoDB;

....... View Complete File content on the mysql/configmap.yml file ......
```

This creates a configmap **mysql-configmap** on the **pet-clinic-db** namespace, that contains the SQL command to create a user, password, and tables on the database, and to insert data on the table.

It gets the username and password from env variables, which are retrieved from the secret.

Run the below command to create a configmap.

```
kubectl apply -f configmap.yml
```

Now, run the following command to verify if the ConfigMap is created

```
kubectl get cm -n pet-clinic-db
```

![configmap](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090425.png)

---
### Step 4: Deploy MySQL Database

CD into the **03-java-app-deployment/mysql** directory and run the **mysql.yml**.

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: pet-clinic-db
spec:
  serviceName: "mysql"
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:latest
        env:
        - name: MYSQL_RANDOM_ROOT_PASSWORD
          value: "yes"
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-cred
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-cred
              key: password
        - name: MYSQL_DATABASE
          value: petclinic
        volumeMounts:
        - name: mysql-config
          mountPath: /docker-entrypoint-initdb.d
        resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "200m"
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-configmap
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: pet-clinic-db
spec:
  type: NodePort
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 30244
```

This file deploys a **MySQL** statefulset database on the **pet-clinic-db** namespace and also a nodeport service **mysql-service** on the **pet-clinic-db** namespace so we can log in to MySQL from outside the cluster.

It gets the database username and password from the secret you created before.

The **`docker-entrypoint-initdb.d`** directory is part of the MySQL image. Any MySQL scripts placed in this folder will be executed during MySQL startup. It is primarily used for running database initialization scripts.

In our setup, we placed an init script (**`init.sql`**) in this folder through a ConfigMap (**mysql-configmap**). **`init.sql`** creates the username, password, and tables for our application. The script retrieves the username and password from environment variables set by the secret object using **`secretKeyRef`**

Run the below command to deploy the MySQL database.

```
kubectl apply -f mysql.yml
```

Now, run the following command to check if the MySQL StatefulSet has been created.

```
kubectl get sts -n pet-clinic-db
```

![stateful](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090444.png)


---
### Step 5: Create ConfigMap for Java Application

CD into the **03-java-app-deployment/java-app** folder and run the **configmap.yml**.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-app-config
  namespace: pet-clinic-app
data:
  application.properties: |
    # database init, supports mysql too
    database=mysql
    spring.datasource.url=jdbc:mysql://mysql-service.pet-clinic-db.svc.cluster.local:3306/petclinic
    spring.datasource.username=${DB_USERNAME}
    spring.datasource.password=${DB_PASSWORD}
    # SQL is written to be idempotent so this is safe
    spring.sql.init.mode=always

    # Web
    spring.thymeleaf.mode=HTML

    # JPA
    spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
    spring.jpa.open-in-view=true

    # Internationalization
    spring.messages.basename=messages/messages

    # Actuator
    management.endpoints.web.exposure.include=*
    # Logging
    #logging.config=classpath:logback-spring.xml
    logging.level.org.springframework=INFO
    # logging.level.org.springframework.web=DEBUG
    # logging.level.org.springframework.context.annotation=TRACE

    # Maximum time static resources should be cached
    spring.web.resources.cache.cachecontrol.max-age=12h
```

Run the below command to create a configmap.

```
kubectl apply -f configmap.yml
```

This creates a configmap **java-app-config** on the **pet-clinic-app** namespace, which contains content of the application.properties file which helps the application to connect with the MySQL database.

The username and password are given as a variable so that it can get the username and password from the env variables, which are retrieved from the secret.

Run the following command to check if the configmap is created

```
kubectl get cm -n pet-clinic-app
```

![configmap](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090508.png)

---
### Step 6: Deploy Java Application

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: pet-clinic-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
      - name: java-app-container
        image: techiescamp/kube-petclinic-app:3.0.0
        env:
            - name: DB_USERNAME
              valueFrom:
                  secretKeyRef:
                    name: mysql-cred
                    key: username
            - name: DB_PASSWORD
              valueFrom:
                  secretKeyRef:
                    name: mysql-cred
                    key: password
        resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "200m"
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: java-app-config
          mountPath: "/opt/config"
        command: ["java", "-jar", "/app/java.jar", "--spring.config.location=/opt/config/application.properties", "--spring.profiles.active=mysql"]
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          failureThreshold: 30
          periodSeconds: 20
          initialDelaySeconds: 180
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          periodSeconds: 15
          failureThreshold: 3
          timeoutSeconds: 10
      volumes:
      - name: java-app-config
        configMap:
          name: java-app-config

---
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
  namespace: pet-clinic-app
spec:
  selector:
    app: java-app
  type: NodePort
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
    nodePort: 32000
```

This file deploys the Java application **java-app** on the **pet-clinic-app** namespace and also a nodeport service **java-app-service** on the **pet-clinic-app** namespace so we can access the application on the web browser.

It gets the database username and password from the secret you created before and the **application.properties** file gets the username and password from the **env** during startup.

The command is used to run the application **JAR** file along with the **application.properties** file so that the application can access the MySQL database.

Also, I have added health checks for this application:

1. **startupProbe** - This keeps the readinessProbe and livenessProbe on hold until the application is started and configured to check the path **/actuator/health**.
2. **readinessProbe** - This checks if the application is ready to accept traffic using the readiness path **/actuator/health/readiness**. If this probe fails, it won't receive any traffic.
3. **livenessProbe** - Checks if the application is up and running using the liveness path **/actuator/health/liveness**. If the probe fails, Kubernetes will restart the pod.

Run the below command to deploy the Java application.

```
kubectl apply -f java.yml
```

After deploying the application, run the following command to check if the deployment pods are up and running. It takes minimum 2min for the pods to get ready.

```
kubectl get deploy -n pet-clinic-app
```

![deploy](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090526.png)

---
### Step 7: Deploy HorizontalPodAutoscaler (HPA)

For HPA to work, you need to have a metrics server running in the cluster.

If you don't have a metrics server, you can install the **Metrics server** using the following command.

```
kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml
```

Now, run the top command to check if the metrics server is installed properly

```
kubectl top po -n pet-clinic-app
```
![hpa](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090546.png)

---
Now, deploy the **`HorizontalPodAutoscaler`** in your cluster.

You can find the below **HPA** file inside the **03-java-app-deployment/java-app** directory.

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: java-app-hpa
  namespace: pet-clinic-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: java-app
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

This will deploy the HPA in the pet-clinic-app namespace.

It is configured to scale the target deployment **Java app** if the pod's CPU utilization is 50% and its maximum replica count is 3.

Run the following command to deploy HPA

```
kubectl apply -f hpa.yml
```

Now, run the following command to check if the HPA is deployed

```
kubectl get hpa -n pet-clinic-app
```

![hpa](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-31%20090602.png)

You can see the CPU usage of the target deployment and the number of pods available.


---
### Step 8: Access the Java Application

Once the pod is up and running, check if the application is deployed properly by trying to access it on the browser, search on the browser as **`{node-IP:nodeport}`** you will get the following

![Final Output](https://raw.githubusercontent.com/vijay-devops-lab/java_app_in_kubernetes/refs/heads/main/assets/Screenshot%202025-12-30%20230407.png)



---



## Key Learnings

- Difference between Stateful and Stateless workloads
    
- Secure configuration management using Secrets and ConfigMaps
    
- Kubernetes DNS and service discovery
    
- Rolling deployments with zero downtime
    
- Observing live resource usage
    

---

## Tools and Technologies Used

- Kubernetes (kubectl)
    
- Docker
    
- Java (Spring Boot – PetClinic)
    
- MySQL
    
- YAML manifests
    
- Metrics Server
    

---

## Future Enhancements

- Add readiness and liveness probes
    
- Configure Horizontal Pod Autoscaler (HPA)
    
- Expose application using Ingress
    
- Implement CI/CD pipeline
    
- Add monitoring with Prometheus and Grafana
    
- Package deployment using Helm
    

---

## Conclusion

This project reflects a real-world Kubernetes deployment workflow and demonstrates core DevOps principles such as scalability, reliability, security, and maintainability. It forms a strong foundation for advanced Kubernetes and cloud-native projects.
