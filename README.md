# DevOps_task6
# Problem statement....
1. Create container image that’s has Jenkins installed  using dockerfile  Or You can use the Jenkins Server on RHEL 8/7
2.  When we launch this image, it should automatically starts Jenkins service in the container.
3.  Create a job chain of job1, job2, job3 and  job4 using build pipeline plugin in Jenkins 
4.  Job2 ( Seed Job ) : Pull  the Github repo automatically when some developers push repo to Github.
5. Further on jobs should be pipeline using written code  using Groovy language by the developer
6. Job1 :  
    1. By looking at the code or program file, Jenkins should automatically start the respective language interpreter installed image container to deploy code on top of Kubernetes ( eg. If code is of  PHP, then Jenkins should start the container that has PHP already installed )
    2.  Expose your pod so that testing team could perform the testing on the pod
    3. Make the data to remain persistent using PVC ( If server collects some data like logs, other user information )
7.  Job3 : Test your app if it  is working or not.
8.  Job4 : if app is not working , then send email to developer with error messages and redeploy the application after code is being edited by the developer

# Here we begins...
1. Make a Dockerfile.

From centos:latest
run yum install wget -y
run yum install net-tools -y


run wget -O /etc/yum.repos.d/jenkins.repo  https://pkg.jenkins.io/redhat-stable/jenkins.repo
run rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
run yum upgrade -y
run yum install java -y
run yum install jenkins -y
run yum install git -y
run echo "jenkins ALL=(ALL) NOPASSED:ALL">>/etc/sudoers
run yum install python3 -y
copy sendmail.py /

RUN touch /etc/yum.repos.d/kubernetes.repo
RUN echo $'[kubernetes]\n\
name=Kubernetes\n\
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64\n\
enabled=1\n\
gpgcheck=1\n\
repo_gpgcheck=1\n\
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg' >> /etc/yum.repos.d/kubernetes.repo


RUN yum install -v -y kubectl

COPY ca.crt /root/.kube/ca.crt
COPY client.crt /root/.kube/client.crt
COPY client.key /root/.kube/client.key


COPY  config    /root/.kube/config

EXPOSE 8080
CMD java -jar /usr/lib/jenkins/jenkins.war

2. Build this Dockervile
--- docker build -t task6:v1

3. Running the container image which is created after building and exposing it at the same time.
  -- docker run -it -p 3000:8080 --name dev_task task6:v1
  
4. Creating persistent volume fot the deployment.
apiVersion: v1
 kind: PersistentVolume
 metadata:
   name: html-pv
   labels:
     type: local
   spec:
     storageClassName: manual
     capacity:
       storage: 3Gi
     accessModes:
        - ReadWriteOnce
     hostPath:
        path: "/mnt/sdr/data/website"
 
5. Creating PVC for storing permanent data.
apiVersion: v1
 kind: PersistentVolumeClaim
 metadata:
   name: html-pvc
 spec:
   storageClassName: manual
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 3Gi
       
Exposing the deployment
  apiVersion: v1
  kind: Service
  metadata:
    name: expose
  spec:
    type: NodePort
    selector: 
    app: webserver
    ports:
      - port: 80
        targetPort: 80
        nodePort: 3000
        
Creating deploy.yml file.
 apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: web-deployment
         labels:
           app: web
     spec:
       replicas: 3
         selector:
           matchLabels:
             app: server
         template:
           metadata:
             name: web-deployment
             labels:
               app: server
           spec:
             containers:
             - name: deployment-server
               image: abhi8585/web:v1
               imagePullPolicy: IfNotPresent
        
             volumeMounts:
             - mountPath: "/var/log/httpd"
               name: pvc-storage
           volumes:
           - name: pvc-storage
             persistentVolumeClaim:
               claimname: html-pvc
               
Create this file with the command.
 kubectl create -f filename
 
 Now we have to create jenkins job for downloading the github code. This job will eheck the extension of the code and the same image will be pushed to dockerhub repository.

We don't have to do the things directly but have to use the groovy code for this.
job("job1-github") {
    steps {
    scm {
          github(" abhi-85 /
DevOps_task6 ", "master")
        }
    triggers {
          scm("* * * * *")
        }
    shell("sudo cp -rvf * /groovy")
    if(shell("ls /groovy/ | grep html")) {
          dockerBuilderPublisher {
                dockerFileDirectory("/groovy/")
                cloud("docker")
    tagsString("server:v1")
                pushOnSuccess(true)

                fromRegistry {
                      url(" abhi-85")
                      credentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                }
                pushCredentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                cleanImages(false)
                cleanupWithJenkinsJobDelete(false)
                noCache(false)
                pull(true)
          }
    }
    else {
          dockerBuilderPublisher {
                dockerFileDirectory("/groovy/")
                cloud("docker")
    tagsString("web-html:v1")
                pushOnSuccess(true)

                fromRegistry {
                      url(" abhi-85")
                      credentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                }
                pushCredentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                cleanImages(false)
                cleanupWithJenkinsJobDelete(false)
                noCache(false)
                pull(true)
          }
    }
    }
    }
    
For the deployment of pods from the yml code.
 job("pod-deploy") {


  triggers {
    upstream {
      upstreamProjects("job1-github")
      threshold("SUCCESS")
    }  
  }


  steps {
    if(shell("ls /groovy | grep html")) {


      shell("if sudo kubectl get pv html-pv; then if sudo kubectl get pvc  html-pvc; then echo " alreaday there"; else kubectl create -f pvc-storage.yml; fi; else sudo kubectl create -f htmlpv.yml; sudo kubectl create -f pvc-storage.yml; fi; if sudo kubectl get deployments web-deployment ; then sudo kubectl rollout restart deployment/web-deployment ; sudo kubectl rollout status deployment/web-deployment; else sudo kubectl create -f deploy.yml; sudo kubectl create -f expose-html.yml; sudo kubectl get all; fi")       

  }
    
  }
}
Checking for the pods if they are running or not.
  job("Testing") {

steps {

    shell('export status=$(curl -siw "%{http_code}" -o /dev/null                     192.168.99.100:3000); if [ $status -eq 200 ]; then exit 0; else python3           mail.py; exit 1; fi')
  }
}

# Here my task finished.
