# my-sixth-project
![0](https://user-images.githubusercontent.com/64473684/85940715-b63ecf00-b93b-11ea-9aa3-81fb14ad4d9b.jpg)
**Groovy** is the most widely used language for writing a Jenkinsfile as it helps working at higher levels of abstraction and provides a higher-level abstraction to all functionalities.

The following project showcases writing such a Groovy script to achieve complete automation that the DevOps team need.

Pre-process:
I] Before starting the project make sure that the Jenkins package is updated. Use yum update jenkins command to update it.

II] Download the following Jenkins plugins:

**PostBuildScript
**Job DSL


**Project:
1] Dockerfiles for HTML and PHP containers**

**HTML:**
```javascript
FROM centos

RUN yum install httpd -y

COPY *.html /var/www/html/

EXPOSE 80

CMD /usr/sbin/httpd -DFOREGROUND && tail -f /dev/null
```

**PHP:**

```javascript
FROM centos

RUN yum install httpd -y
RUN yum install php -y

COPY *.php /var/www/html/

EXPOSE 80

CMD /usr/sbin/httpd -DFOREGROUND && tail -f /dev/null
```
The webpage code files will be stored in the same path of the Dockerfiles after the Jenkins job downloads them from the GitHub



**2] Persistent Volumes and Persistent Volume Claims:**

As the containers are running with Apache Web Server, they generate log files. These log files are very sensitive as they are used to monitor any malicious activities on the Servers. To store the log files permanently, we need to create Persistent volumes.

**HTML Server:**

**PersistentVolume:**

```javascript
apiVersion: v1

kind: PersistentVolume

metadata:
  name: http-pv
  
  labels:
    app: webapp

spec:
  storageClassName: manual
  
  capacity:
    storage: 10Gi
  
  accessModes:
    - ReadWriteOnce
  
  hostPath:
    path: "/mnt/sda1/data/http/"
```
**PersistentVolumeClaim:**

```javascript
apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: http-pv-claim

spec:
  storageClassName: manual
  
  accessModes:
    - ReadWriteOnce
  
  resources:
    requests:
      storage: 3Gi
```
**PHP Server:

**PersistentVolume:
```javascript
apiVersion: v1

kind: PersistentVolume

metadata:
  name: php-pv
  
  labels:
    app: webapp

spec:
  storageClassName: manual
  
  capacity:
    storage: 10Gi
  
  accessModes:
    - ReadWriteOnce
  
  hostPath:
    path: "/mnt/sda1/data/php/"
```
**PersistentVolumeClaim:**
```javascript
apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: php-pv-claim

spec:
  storageClassName: manual
  
  accessModes:
    - ReadWriteOnce
  
  resources:
    requests:
      storage: 3Gi
```
**3] Deployments:

We need to create deployments on which the containers will be deployed. We Mount the /var/log/httpd/ directory in each container so as to store the log data permanently

**HTML:
```javascript
apiVersion: apps/v1
kind: Deployment

metadata:
  name: http-dep


spec:
  replicas: 3
  
  selector:
    matchLabels:
      app: webapp


  template:
    metadata: 
      name: http-dep 
      labels:
        app: webapp


    spec: 
      volumes:
        - name: http-pv
          persistentVolumeClaim:
            claimName: http-pv-claim
      containers: 
        - name: http-dep
          image: dockerninad07/apache-server
          volumeMounts:
            - mountPath: "/var/log/httpd/"
              name: http-pv
```
**PHP:
```javascript
apiVersion: apps/v1
kind: Deployment

metadata:
  name: php-dep


spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp


  template:
    metadata: 
      name: php-dep 
      labels:
        app: webapp


    spec: 
      volumes:
        - name: php-pv
          persistentVolumeClaim:
            claimName: php-pv-claim

      containers: 
        - name: php-dep
          image: dockerninad07/apache-php-server
          volumeMounts:
            - mountPath: "/var/log/httpd/"
              name: php-pv
```
**4] Service:

We need to create a service for our deployment to assign a customized Port address to our Deployment.

**Service:
```javascript
apiVersion: v1
kind: Service

metadata:
  name: my-service

spec:
  type: NodePort
  
  selector: 
    app: webapp
  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30100
```
This will assign Port 30100 to our deployment. You can assign any port number in the range 30000-32767 if using the NodePort type.

As we have our configurations ready, we need to write the Groovy script to create Jobs inside Jenkins.


**Groovy Script:**
We need to write a groovy script for creating a chain of Jobs inside Jenkins

**Job 1: Code_interpreter

This Job will interpret the language of the committed code. On interpreting the language of the code, it will build the corresponding Docker image from the Dockerfile provided with the code. After the build is done, it will push the built image to the corresponding DockerHub repository.
```javascript
job("Code_Interpreter") {


  description("Code Interpreter Job")
  
  steps {
  
   scm {
     git {
       extensions {
         wipeWorkspace()
       }
     }
   }
    
   scm {
     github("Ninad07/Groovy", "master")
   }


   triggers {
     scm("* * * * *")
   }


   shell("if ls | grep php; then sudo cp -rvf * /groovy/image/php/; else sudo cp * /groovy/image/html; fi")


   if(shell("ls /groovy/code/ | grep php | wc -l")) {


     dockerBuilderPublisher {
       dockerFileDirectory("/groovy/image/html")
       fromRegistry {
         url("dockerninad07")
         credentialsId("4de5b343-12cc-4a68-9e1c-8c4d1ce40917")
       }
       cloud("Local")
    
       tagsString("dockerninad07/apache-server")
       pushOnSuccess(true)
       pushCredentialsId("8fb4a5df-3dab-4214-a8ec-7f541f675dcb")
       cleanImages(false)
       cleanupWithJenkinsJobDelete(false)
       noCache(false)
       pull(true)
     }    
      
   }
   
   else {
     
     dockerBuilderPublisher {
       dockerFileDirectory("/groovy/image/php")
       fromRegistry {
         url("dockerninad07")
         credentialsId("4de5b343-12cc-4a68-9e1c-8c4d1ce40917")
       }
       cloud("Local")


       tagsString("dockerninad07/apache-php-server")
       pushOnSuccess(true)
       pushCredentialsId("8fb4a5df-3dab-4214-a8ec-7f541f675dcb")
       cleanImages(false)
       cleanupWithJenkinsJobDelete(false)
       noCache(false)
       pull(true)
     } 
   }
  }
}
```
