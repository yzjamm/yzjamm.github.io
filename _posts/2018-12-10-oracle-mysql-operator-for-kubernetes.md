---
title: Introducing the Oracle MySQL Operator for Kubernetes
date: 2018-12-10
comments: true
categories: [MySQL, Kubernetes, StudyNotes]
tags: [MySQL, Kubernetes, Docker, Study Notes]
typora-copy-images-to: ../assets/img
---
Oracle recently open sourced a Kubernetes operator for MySQL that makes running and managing MySQL on Kubernetes easier.



## References

- https://learnk8s.io/blog/installing-docker-and-kubernetes-on-windows
- https://blogs.oracle.com/developers/introducing-the-oracle-mysql-operator-for-kubernetes
- https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md




## Study Notes

**Start cmd.exe as administrator, execute this long command:**

```powershell
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```

**Install docker for windows and cmder**

```powershell
choco search docker
choco install docker-for-windows -y
choco install cmder -y
```

**Enable Kubernetes for docker**

![1544477338793](/assets/img/1544477338793.png)



**Run cmder.exe as administrator and run following commands**

```powershell
git clone https://github.com/oracle/mysql-operator.git
cd mysql-operator
choco install kubernetes-helm
kubectl create ns mysql-operator
helm init
```



**Install mysql-operator**

```powershell
λ helm install --name mysql-operator mysql-operator
NAME:   mysql-operator
LAST DEPLOYED: Mon Dec 10 16:07:15 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/ClusterRole
NAME            AGE
mysql-agent     1s
mysql-operator  1s

==> v1beta1/ClusterRoleBinding
mysql-operator  1s
mysql-agent     1s

==> v1beta1/Deployment
mysql-operator  1s

==> v1/Pod(related)

NAME                             READY  STATUS             RESTARTS  AGE
mysql-operator-56d78bfc88-w9kgz  0/1    ContainerCreating  0         1s

==> v1/ServiceAccount

NAME            AGE
mysql-operator  9s
mysql-agent     9s

==> v1beta1/CustomResourceDefinition
mysqlclusters.mysql.oracle.com         5s
mysqlbackupschedules.mysql.oracle.com  1s
mysqlbackups.mysql.oracle.com          1s
mysqlrestores.mysql.oracle.com         1s


NOTES:
Thanks for installing the MySQL Operator.

Check if the operator is running with

kubectl -n mysql-operator get po
```

**Check pods status**

```powershell
λ kubectl -n mysql-operator get po
NAME                              READY     STATUS    RESTARTS   AGE
mysql-operator-56d78bfc88-w9kgz   1/1       Running   0          9s
```

