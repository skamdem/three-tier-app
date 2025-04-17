# K8s for three tier app

the 3-tier app uses 
- nginx 
- flask 
- mysql

## folder structure
```
/flask_docker
	Dockerfile -> docker file for creating the application image
	requirements.txt -> requirements file for flask and mysql
	/templates
		index.html -> render the test application
	test-app1.yaml -> pod manifest linked to the application image
	view.py -> flask application
/flask_read_docker
	Dockerfile -> docker file for creating the application image
	read-app1.yaml
	read.py
	requirements.txt
	/templates
		index.html
/ingress
	app1-ingress.yaml -> ingress resource exposing the application through **test-app1-service**
	complete-app1-ingress.yaml -> updated ingress resource (to later replace the above ingress) exposing two applications through **test-app1-service** and **read-app1-service**
/mysql 
	mysql-deploy.yaml -> mysql database **deployment** and **service** manifests
	mysql-storage.yaml -> mysql **pv** and **pvc**
/security
	network_policy_allow_app1.yaml -> network policies that limit all traffic into data1 to only pods in the namespace app1
	network_policy_deny.yaml -> default deny network policy
	secure-read-app1.yaml -> secure application **read-app1** pod manifest 
/setup
	ingress_create.yaml
	storage.yaml
```
	
## 1/10 prerequisites
\# setup nginx ingress controller in a namespace **ingress-nginx**
$ kubectl -n ingress-nginx get all 
there should be 
- a deployment **ingress-nginx-controller **
- a service **ingress-nginx-controller **

\# docker running on port **5000** for the local image **registry**
$ docker ps

\# there is a host entry in **/etc/hosts** for our ingress controller to resolve to
$ grep app /etc/hosts

$ cat /etc/hosts
\# could contain at bottom
> 127.0.0.1 controlplane
> 172.30.2.2 node01
> 172.30.1.2 application.lab.mine

## 2/10 set up the namespaces
$ kubectl create namespace app1
$ kubectl create namespace data1
\# confirm namespaces were created
$ kubectl get ns

## 3/10 set up the secrets 
\# create a secret to use with the mysql database
$ kubectl -n data1 create secret generic mysql --from-literal=mysql-root-password='Very$ecure1#' 
\# confirm the secret that was created
$ kubectl get secrets -n data1

## 4/10 set up the storage from folder **mysql**
$ kubectl create -f mysql/mysql-storage.yaml -f mysql/mysql-deploy.yaml
\# confirm the resources were created
$ kubectl -n data1 get pv,pvc 

## 5/10 setup the application from folder **flask_docker**
\# create the flask docker container image for the application
$ cd flask_docker
\# create the docker image
$ docker image build -t flask_docker .

\# tag and push the image to the local repository
$ docker tag flask_docker localhost:5000/flask_docker
$ docker push localhost:5000/flask_docker

6/10 create a pod with the new image to run the application from folder **flask_docker**
$ kubectl create -f flask_docker/test-app1.yaml
\# confirm pod test-app1 is running
$ kubectl -n app1 get pods -o wide
\# create service **test-app1-service** for pod **test-app1**
$ kubectl -n app1 expose pod test-app1 --port=6000  --name=test-app1-service 
\# confirm pod test-app1 is exposed on port 6000
$ kubectl -n app1 describe service test-app1 

## 7/10 expose the application via an ingress controller from folder **ingress**
\# create the ingress resource pointing to the application
$ kubectl create -f ingress/app1-ingress.yaml
\# test the ingress routing to the application
$ curl application.lab.mine:30080/test

## 8/10 data portion of the three tier app from folder **mysql**
\# confirm database resources previously created
$ kubectl get svc -n data1
$ kubectl describe svc mysql-service -n data1
$ kubectl get deployments -n data1
$ kubectl get pods -o wide -n data1 --show-labels

\# load the database with sample data to read out from our application
\# deploy a *temporary pod* **mysql-client** to connect to the mysql database
$ k run mysql-client -n data1 --image=mysql:5.7 -it --rm --restart=Never -- /bin/bash
\# above command takes you to a shell inside the temporary pod

\# connect to database host (mysql-service) and put information into the database
$ mysql -h mysql-service -uroot -p'Very$ecure1#'
CREATE DATABASE visitors;
use visitors;
CREATE TABLE persons (personID int, FirstName varchar(255), LastName varchar(255));
INSERT INTO persons VALUES ('1', 'phillip', 'devnull');
INSERT INTO persons VALUES ('2', 'het', 'tanis');
\# leave the mysql client
$ exit
\# test reading the table created
$ mysql -h mysql-service -uroot -p'Very$ecure1#' -e 'use visitors; show tables; select * from persons'
\# leave the mysql-client pod
$ exit

## 9/10 build the read application in flask to read the data from that database from folder **flask_read_docker**
\# test the basic application functionality to the database
$ cd flask_read_docker
\# create the docker image
$ docker image build -t flask_read_docker .
\# tag and push the image to the local repository
$ docker tag flask_read_docker localhost:5000/flask_read_docker
$ docker push localhost:5000/flask_read_docker
\# create a pod **read-app1** to run the flask application from the new image
$ kubectl create -f flask_read_docker/read-app1.yaml
\# confirm pod read-app1 is running
$ kubectl -n app1 get pods -o wide
\# create service read-app1-service for exposing read-app1 
$ kubectl -n app1 expose pod read-app1 --port=6000  --name=read-app1-service 
\# confirm pod read-app1 is exposed on port 6000 through read-app1-service 
$ kubectl -n app1 describe service read-app1-service
\# delete old ingress and redeploy the new ingress resource pointing to BOTH applications
$ kubectl -n app1 delete ingress ingress 
$ kubectl create -f ingress/complete-app1-ingress.yaml

# test the ingress connection to BOTH applications
$ curl application.lab.mine:30080/test
$ curl application.lab.mine:30080/read

## 10/10 securing the applications from folder **security**
\# create network policy to deny anything into data1
$ kubectl create -f network_policy_deny.yaml

\# create network policy that limit all traffic into namespace **data1** on port 3306 to only pods in the namespace **app1**
$ kubectl create -f network_policy_allow_app1.yaml

\# confirm the ingress connection to the application is still working
$ curl application.lab.mine:30080/test
$ curl application.lab.mine:30080/read

\# create a mysql image-based pod in the default namespace and verify that it no longer connects to mysql-service in namespace **data1**
$ kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- /bin/bash
$ mysql -h mysql-service.data1.svc.cluster.local -uroot -p'Very$ecure1#' -e 'use visitors; show tables; select * from persons'
\# above command should hang without connnecting

\# create a mysql image-based pod in the **app1** namespace and verify it still connects to mysql-service in namespace **data1**
$ kubectl -n app1 run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- /bin/bash
$ mysql -h mysql-service.data1.svc.cluster.local -uroot -p'Very$ecure1#' -e 'use visitors; show tables; select * from persons'

\# redeploy the read application in a secure fashion
$ kubectl -n app1 delete pod read-app1 
\# create a replacement pod for added security of 
- *non-root* user
- disallowed privilege escalation
- read only filesystems 

$ kubectl create -f secure-read-app1.yaml
\# verify connectivity
$ curl application.lab.mine:30080/read

next step: add a flask application that can **write** data

