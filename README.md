<!-- Optei por criar uma VM Oracle-Linux no virtual Box, onde preferi deixar o firewald e o selinux ativado visando um ambiente mais seguro, porém devido a problemas de instalação de dependências acabei desativando o selinux-->

<!-- Primeiro fiz a instalação das dependências -->

sudo dnf update -y  
sudo dnf install -y git vim
sudo dnf config-manager --add-repo=https://download.docker.com/linux/oracle/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
docker --version
docker ps

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

<!-- Liberei algumas portas precisaria no firewald -->

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp 
sudo firewall-cmd --permanent --add-port=10252/tcp 
sudo firewall-cmd --permanent --add-port=2379-2380/tcp 
sudo firewall-cmd --permanent --add-port=30000-32767/tcp 
sudo firewall-cmd --reload

<!-- Logo depois clonei o repositório -->

git clone https://github.com/WAProject/devops-microservices-demo.git

<!-- Criei e conigurei o cluster Kubernetes -->

sudo kubeadm init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
minikube start
cd Backend
docker build -t localhost:5001/meu-backend:latest -f Docker/api/Dockerfile.api .
cd ..
cd Frontend
docker build -t localhost:5001/meu-frontend:latest -f Docker/api/Dockerfile.api .
docker tag meu-backend:latest localhost:5000/meu-backend
docker tag meu-frontend:latest localhost:5000/meu-frontend
docker push localhost:5000/meu-backend
docker push localhost:5000/meu-frontend

<!-- Depois disso criei PostgreSQL no Kubernetes(adicionarei os arquivos ao repo) -->

vim postgresql-pvc.yaml
vim postgresql-deployment.yaml
vim postgresql-service.yaml

kubectl apply -f postgresql-pvc.yaml
kubectl apply -f postgresql-deployment.yaml
kubectl apply -f postgresql-service.yaml


<!-- Depois criei os yaml do Backend e Frontend(adicionarei os arquivos ao repo) -->

vim backend-deployment.yaml
vim frontend-deployment.yaml
vim backend-service.yaml
vim frontend-service.yaml

kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-service.yaml

<!-- Fiz as configs de HPA(adicionarei os arquivos ao repo) -->

kubectl get deployment metrics-server -n kube-system
kubectl autoscale deployment meu-backend --max=10 --min=1 --resource=memory --percent=70

vim hpa-backend.yaml

kubectl apply -f hpa-backend.yaml


<!-- Ao finalizar e tudo estiver funcionamento é importante reativar o selinux e fazer um trobleshoting em caso de problemas na aplicação -->

<!----------------------------------------------- Script completo que fiz na VM ------------------------------------------------------------>
 
dev@Titan:~$ ssh dev2@10.0.0.108
The authenticity of host '10.0.0.108 (10.0.0.108)' can't be established.
ED25519 key fingerprint is SHA256:YtLGKdPnmWmLY/wE3nTTNx+cK2t1FhbNB1s/+E6YrdU.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:30: [hashed name]
    ~/.ssh/known_hosts:33: [hashed name]
    ~/.ssh/known_hosts:34: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.0.108' (ED25519) to the list of known hosts.
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Mar 18 20:16:28 2025
[dev2@localhost ~]$ sudo su
[sudo] password for dev2: 
[root@localhost dev2]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
[root@localhost dev2]# sudo setenforce 0
[root@localhost dev2]# sudo vi /etc/selinux/config
[root@localhost dev2]# sudo reboot
[root@localhost dev2]# Connection to 10.0.0.108 closed by remote host.
Connection to 10.0.0.108 closed.
dev@Titan:~$ ssh dev2@10.0.0.108
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Mar 18 21:19:30 2025 from 10.0.0.113
[dev2@localhost ~]$ sudo su
[sudo] password for dev2: 
[root@localhost dev2]# docker ps -a
bash: docker: command not found
[root@localhost dev2]# sudo yum install -y docker
sudo systemctl enable --now docker
Oracle Linux 9 BaseOS Latest (x86_64)                                  41 kB/s | 4.2 kB     00:00    
Oracle Linux 9 BaseOS Latest (x86_64)                                 8.7 MB/s |  53 MB     00:06    
Oracle Linux 9 Application Stream Packages (x86_64)                    74 kB/s | 4.5 kB     00:00    
Oracle Linux 9 Application Stream Packages (x86_64)                   8.8 MB/s |  52 MB     00:05    
Oracle Linux 9 UEK Release 7 (x86_64)                                  68 kB/s | 3.5 kB     00:00    
Oracle Linux 9 UEK Release 7 (x86_64)                                 9.2 MB/s |  59 MB     00:06    
Last metadata expiration check: 0:00:05 ago on Tue 18 Mar 2025 09:40:10 PM -03.
Dependencies resolved.
======================================================================================================
 Package               Architecture   Version                             Repository             Size
======================================================================================================
Installing:
 podman-docker         noarch         4:5.2.2-13.0.1.el9_5                ol9_appstream         285 k
Upgrading:
 podman                x86_64         4:5.2.2-13.0.1.el9_5                ol9_appstream          16 M
Installing dependencies:
 passt                 x86_64         0^20240806.gee36266-6.el9_5         ol9_appstream         205 k
 passt-selinux         noarch         0^20240806.gee36266-6.el9_5         ol9_appstream          27 k

Transaction Summary
======================================================================================================
Install  3 Packages
Upgrade  1 Package

Total download size: 17 M
Downloading Packages:
(1/4): passt-0^20240806.gee36266-6.el9_5.x86_64.rpm                   2.1 MB/s | 205 kB     00:00    
(2/4): podman-docker-5.2.2-13.0.1.el9_5.noarch.rpm                    1.3 MB/s | 285 kB     00:00    
(3/4): passt-selinux-0^20240806.gee36266-6.el9_5.noarch.rpm            23 kB/s |  27 kB     00:01    
(4/4): podman-5.2.2-13.0.1.el9_5.x86_64.rpm                           7.2 MB/s |  16 MB     00:02    
------------------------------------------------------------------------------------------------------
Total                                                                 7.1 MB/s |  17 MB     00:02     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                              1/1 
  Installing       : passt-0^20240806.gee36266-6.el9_5.x86_64                                     1/5 
  Running scriptlet: passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             2/5 
  Installing       : passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             2/5 
  Running scriptlet: passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             2/5 
  Upgrading        : podman-4:5.2.2-13.0.1.el9_5.x86_64                                           3/5 
  Installing       : podman-docker-4:5.2.2-13.0.1.el9_5.noarch                                    4/5 
  Running scriptlet: podman-2:4.4.1-3.0.1.el9.x86_64                                              5/5 
  Cleanup          : podman-2:4.4.1-3.0.1.el9.x86_64                                              5/5 
  Running scriptlet: passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             5/5 
  Running scriptlet: podman-2:4.4.1-3.0.1.el9.x86_64                                              5/5 
  Verifying        : passt-0^20240806.gee36266-6.el9_5.x86_64                                     1/5 
  Verifying        : passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             2/5 
  Verifying        : podman-docker-4:5.2.2-13.0.1.el9_5.noarch                                    3/5 
  Verifying        : podman-4:5.2.2-13.0.1.el9_5.x86_64                                           4/5 
  Verifying        : podman-2:4.4.1-3.0.1.el9.x86_64                                              5/5 

Upgraded:
  podman-4:5.2.2-13.0.1.el9_5.x86_64                                                                  
Installed:
  passt-0^20240806.gee36266-6.el9_5.x86_64       passt-selinux-0^20240806.gee36266-6.el9_5.noarch     
  podman-docker-4:5.2.2-13.0.1.el9_5.noarch     

Complete!
Failed to enable unit: Unit file docker.service does not exist.
[root@localhost dev2]# podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
[root@localhost dev2]# sudo yum remove -y podman podman-docker
Dependencies resolved.
======================================================================================================
 Package                  Architecture Version                             Repository            Size
======================================================================================================
Removing:
 podman                   x86_64       4:5.2.2-13.0.1.el9_5                @ol9_appstream        55 M
 podman-docker            noarch       4:5.2.2-13.0.1.el9_5                @ol9_appstream        11 k
Removing dependent packages:
 cockpit-podman           noarch       63.1-1.el9                          @AppStream           620 k
Removing unused dependencies:
 conmon                   x86_64       2:2.1.7-1.el9_2                     @AppStream           170 k
 passt                    x86_64       0^20240806.gee36266-6.el9_5         @ol9_appstream       831 k
 passt-selinux            noarch       0^20240806.gee36266-6.el9_5         @ol9_appstream       196 k
 shadow-utils-subid       x86_64       2:4.9-6.el9                         @anaconda            215 k

Transaction Summary
======================================================================================================
Remove  7 Packages

Freed space: 57 M
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                              1/1 
  Erasing          : podman-docker-4:5.2.2-13.0.1.el9_5.noarch                                    1/7 
  Erasing          : cockpit-podman-63.1-1.el9.noarch                                             2/7 
  Running scriptlet: podman-4:5.2.2-13.0.1.el9_5.x86_64                                           3/7 
  Erasing          : podman-4:5.2.2-13.0.1.el9_5.x86_64                                           3/7 
  Erasing          : passt-0^20240806.gee36266-6.el9_5.x86_64                                     4/7 
  Erasing          : passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             5/7 
  Running scriptlet: passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             5/7 
  Erasing          : conmon-2:2.1.7-1.el9_2.x86_64                                                6/7 
  Erasing          : shadow-utils-subid-2:4.9-6.el9.x86_64                                        7/7 
  Running scriptlet: shadow-utils-subid-2:4.9-6.el9.x86_64                                        7/7 
  Verifying        : cockpit-podman-63.1-1.el9.noarch                                             1/7 
  Verifying        : conmon-2:2.1.7-1.el9_2.x86_64                                                2/7 
  Verifying        : passt-0^20240806.gee36266-6.el9_5.x86_64                                     3/7 
  Verifying        : passt-selinux-0^20240806.gee36266-6.el9_5.noarch                             4/7 
  Verifying        : podman-4:5.2.2-13.0.1.el9_5.x86_64                                           5/7 
  Verifying        : podman-docker-4:5.2.2-13.0.1.el9_5.noarch                                    6/7 
  Verifying        : shadow-utils-subid-2:4.9-6.el9.x86_64                                        7/7 

Removed:
  cockpit-podman-63.1-1.el9.noarch              conmon-2:2.1.7-1.el9_2.x86_64                        
  passt-0^20240806.gee36266-6.el9_5.x86_64      passt-selinux-0^20240806.gee36266-6.el9_5.noarch     
  podman-4:5.2.2-13.0.1.el9_5.x86_64            podman-docker-4:5.2.2-13.0.1.el9_5.noarch            
  shadow-utils-subid-2:4.9-6.el9.x86_64        

Complete!
[root@localhost dev2]# sudo yum install -y docker-engine
Last metadata expiration check: 0:03:09 ago on Tue 18 Mar 2025 09:40:38 PM -03.
No match for argument: docker-engine
Error: Unable to find a match: docker-engine
[root@localhost dev2]# docker ps -a
bash: docker: command not found
[root@localhost dev2]# sudo yum install docker
Last metadata expiration check: 0:03:30 ago on Tue 18 Mar 2025 09:40:38 PM -03.
Dependencies resolved.
======================================================================================================
 Package                 Arch        Version                             Repository              Size
======================================================================================================
Installing:
 podman-docker           noarch      4:5.2.2-13.0.1.el9_5                ol9_appstream          285 k
Installing dependencies:
 conmon                  x86_64      3:2.1.12-1.el9                      ol9_appstream           56 k
 passt                   x86_64      0^20240806.gee36266-6.el9_5         ol9_appstream          205 k
 passt-selinux           noarch      0^20240806.gee36266-6.el9_5         ol9_appstream           27 k
 podman                  x86_64      4:5.2.2-13.0.1.el9_5                ol9_appstream           16 M
 shadow-utils-subid      x86_64      2:4.9-10.el9_5                      ol9_baseos_latest       85 k

Transaction Summary
======================================================================================================
Install  6 Packages

Total download size: 17 M
Installed size: 56 M
Is this ok [y/N]: ^COperation aborted.
[root@localhost dev2]# sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
[root@localhost dev2]# sudo yum install -y docker-ce docker-ce-cli containerd.io
Docker CE Stable - x86_64                                              36 kB/s |  65 kB     00:01    
Last metadata expiration check: 0:00:02 ago on Tue 18 Mar 2025 09:45:16 PM -03.
Dependencies resolved.
======================================================================================================
 Package                          Architecture  Version                 Repository               Size
======================================================================================================
Installing:
 containerd.io                    x86_64        1.7.25-3.1.el9          docker-ce-stable         43 M
 docker-ce                        x86_64        3:28.0.1-1.el9          docker-ce-stable         20 M
 docker-ce-cli                    x86_64        1:28.0.1-1.el9          docker-ce-stable        8.3 M
Installing weak dependencies:
 docker-buildx-plugin             x86_64        0.21.1-1.el9            docker-ce-stable         16 M
 docker-ce-rootless-extras        x86_64        28.0.1-1.el9            docker-ce-stable        3.2 M
 docker-compose-plugin            x86_64        2.33.1-1.el9            docker-ce-stable         15 M

Transaction Summary
======================================================================================================
Install  6 Packages

Total download size: 106 M
Installed size: 422 M
Downloading Packages:
(1/6): docker-buildx-plugin-0.21.1-1.el9.x86_64.rpm                   4.1 MB/s |  16 MB     00:04    
(2/6): docker-ce-28.0.1-1.el9.x86_64.rpm                              4.6 MB/s |  20 MB     00:04    
(3/6): docker-ce-rootless-extras-28.0.1-1.el9.x86_64.rpm              2.5 MB/s | 3.2 MB     00:01    
(4/6): docker-ce-cli-28.0.1-1.el9.x86_64.rpm                          2.7 MB/s | 8.3 MB     00:03    
(5/6): docker-compose-plugin-2.33.1-1.el9.x86_64.rpm                  4.1 MB/s |  15 MB     00:03    
(6/6): containerd.io-1.7.25-3.1.el9.x86_64.rpm                        3.5 MB/s |  43 MB     00:12    
------------------------------------------------------------------------------------------------------
Total                                                                 8.5 MB/s | 106 MB     00:12     
Docker CE Stable - x86_64                                              17 kB/s | 1.6 kB     00:00    
Importing GPG key 0x621E9F35:
 Userid     : "Docker Release (CE rpm) <docker@docker.com>"
 Fingerprint: 060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35
 From       : https://download.docker.com/linux/centos/gpg
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                              1/1 
  Installing       : docker-buildx-plugin-0.21.1-1.el9.x86_64                                     1/6 
  Running scriptlet: docker-buildx-plugin-0.21.1-1.el9.x86_64                                     1/6 
  Installing       : docker-compose-plugin-2.33.1-1.el9.x86_64                                    2/6 
  Running scriptlet: docker-compose-plugin-2.33.1-1.el9.x86_64                                    2/6 
  Installing       : docker-ce-cli-1:28.0.1-1.el9.x86_64                                          3/6 
  Running scriptlet: docker-ce-cli-1:28.0.1-1.el9.x86_64                                          3/6 
  Installing       : containerd.io-1.7.25-3.1.el9.x86_64                                          4/6 
  Running scriptlet: containerd.io-1.7.25-3.1.el9.x86_64                                          4/6 
  Installing       : docker-ce-rootless-extras-28.0.1-1.el9.x86_64                                5/6 
  Running scriptlet: docker-ce-rootless-extras-28.0.1-1.el9.x86_64                                5/6 
  Installing       : docker-ce-3:28.0.1-1.el9.x86_64                                              6/6 
  Running scriptlet: docker-ce-3:28.0.1-1.el9.x86_64                                              6/6 
  Verifying        : containerd.io-1.7.25-3.1.el9.x86_64                                          1/6 
  Verifying        : docker-buildx-plugin-0.21.1-1.el9.x86_64                                     2/6 
  Verifying        : docker-ce-3:28.0.1-1.el9.x86_64                                              3/6 
  Verifying        : docker-ce-cli-1:28.0.1-1.el9.x86_64                                          4/6 
  Verifying        : docker-ce-rootless-extras-28.0.1-1.el9.x86_64                                5/6 
  Verifying        : docker-compose-plugin-2.33.1-1.el9.x86_64                                    6/6 

Installed:
  containerd.io-1.7.25-3.1.el9.x86_64                 docker-buildx-plugin-0.21.1-1.el9.x86_64       
  docker-ce-3:28.0.1-1.el9.x86_64                     docker-ce-cli-1:28.0.1-1.el9.x86_64            
  docker-ce-rootless-extras-28.0.1-1.el9.x86_64       docker-compose-plugin-2.33.1-1.el9.x86_64      

Complete!
[root@localhost dev2]# sudo systemctl enable --now docker
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.
[root@localhost dev2]# docker --version
Docker version 28.0.1, build 068a01e
[root@localhost dev2]# cd /home/dev2/
[root@localhost dev2]# ls
apache-tomcat-7.0.76.tar.gz
[root@localhost dev2]# rm -r apache-tomcat-7.0.76.tar.gz 
rm: remove regular file 'apache-tomcat-7.0.76.tar.gz'? y
[root@localhost dev2]# docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[root@localhost dev2]# sudo yum install -y curl
curl -LO "https://dl.k8s.io/release/v1.25.4/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
Last metadata expiration check: 0:09:00 ago on Tue 18 Mar 2025 09:45:16 PM -03.
Package curl-7.76.1-23.el9.x86_64 is already installed.
Dependencies resolved.
======================================================================================================
 Package             Architecture       Version                   Repository                     Size
======================================================================================================
Upgrading:
 curl                x86_64             7.76.1-31.el9             ol9_baseos_latest             303 k
 libcurl             x86_64             7.76.1-31.el9             ol9_baseos_latest             283 k

Transaction Summary
======================================================================================================
Upgrade  2 Packages

Total download size: 586 k
Downloading Packages:
(1/2): libcurl-7.76.1-31.el9.x86_64.rpm                               1.6 MB/s | 283 kB     00:00    
(2/2): curl-7.76.1-31.el9.x86_64.rpm                                  219 kB/s | 303 kB     00:01    
------------------------------------------------------------------------------------------------------
Total                                                                 420 kB/s | 586 kB     00:01     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                              1/1 
  Upgrading        : libcurl-7.76.1-31.el9.x86_64                                                 1/4 
  Upgrading        : curl-7.76.1-31.el9.x86_64                                                    2/4 
  Cleanup          : curl-7.76.1-23.el9.x86_64                                                    3/4 
  Cleanup          : libcurl-7.76.1-23.el9.x86_64                                                 4/4 
  Running scriptlet: libcurl-7.76.1-23.el9.x86_64                                                 4/4 
  Verifying        : curl-7.76.1-31.el9.x86_64                                                    1/4 
  Verifying        : curl-7.76.1-23.el9.x86_64                                                    2/4 
  Verifying        : libcurl-7.76.1-31.el9.x86_64                                                 3/4 
  Verifying        : libcurl-7.76.1-23.el9.x86_64                                                 4/4 

Upgraded:
  curl-7.76.1-31.el9.x86_64                        libcurl-7.76.1-31.el9.x86_64                       

Complete!
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   138  100   138    0     0    539      0 --:--:-- --:--:-- --:--:--   536
100 42.9M  100 42.9M    0     0  2620k      0  0:00:16  0:00:16 --:--:-- 2354k
[root@localhost dev2]# curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11913  100 11913    0     0  18819      0 --:--:-- --:--:-- --:--:-- 18819
Downloading https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
helm not found. Is /usr/local/bin on your $PATH?
Failed to install helm
	For support, go to https://github.com/helm/helm.
[root@localhost dev2]# echo $PATH
/root/.local/bin:/root/bin:/sbin:/bin:/usr/sbin:/usr/bin
[root@localhost dev2]# helm version
bash: helm: command not found
[root@localhost dev2]# export PATH=$PATH:/usr/local/bin
[root@localhost dev2]# export PATH=$PATH:/usr/local/bin
[root@localhost dev2]# source ~/.bash_profile
[root@localhost dev2]# source ~/.bashrc
[root@localhost dev2]# helm version
version.BuildInfo{Version:"v3.17.2", GitCommit:"cc0bbbd6d6276b83880042c1ecb34087e84d41eb", GitTreeState:"clean", GoVersion:"go1.23.7"}
[root@localhost dev2]# sudo firewall-cmd --list-ports
8080/tcp
[root@localhost dev2]# sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=3030/tcp                                    
success
Warning: ALREADY_ENABLED: 8080:tcp
success
success
[root@localhost dev2]# sudo firewall-cmd --reload
success
[root@localhost dev2]# sudo firewall-cmd --list-ports
3030/tcp 8000/tcp 8080/tcp
[root@localhost dev2]# git clone https://github.com/WAProject/devops-microservicesdemo.git
Cloning into 'devops-microservicesdemo'...
Username for 'https://github.com': 
Password for 'https://github.com': 
remote: Repository not found.
fatal: Authentication failed for 'https://github.com/WAProject/devops-microservicesdemo.git/'
[root@localhost dev2]# git clone https://github.com/WAProject/devops-microservicesdemo.git
Cloning into 'devops-microservicesdemo'...
Username for 'https://github.com': Renan3709
Password for 'https://Renan3709@github.com': 
remote: Support for password authentication was removed on August 13, 2021.
remote: Please see https://docs.github.com/get-started/getting-started-with-git/about-remote-repositories#cloning-with-https-urls for information on currently recommended modes of authentication.
fatal: Authentication failed for 'https://github.com/WAProject/devops-microservicesdemo.git/'
[root@localhost dev2]# git clone https://github.com/WAProject/devops-microservices-demo.git
Cloning into 'devops-microservices-demo'...
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 14 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (14/14), done.
[root@localhost dev2]# ls
devops-microservices-demo
[root@localhost dev2]# cd devops-microservices-demo/
[root@localhost devops-microservices-demo]# ls
Backend  Frontend
[root@localhost devops-microservices-demo]# helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
[root@localhost devops-microservices-demo]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@localhost devops-microservices-demo]# helm install my-postgresql bitnami/postgresql \
  --set postgresqlPassword=senha123 \
  --set postgresqlDatabase=meubanco
Error: INSTALLATION FAILED: Kubernetes cluster unreachable: the server could not find the requested resource
[root@localhost devops-microservices-demo]# kubectl cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
error: the server doesn't have a resource type "services"
[root@localhost devops-microservices-demo]# minikube status
bash: minikube: command not found
[root@localhost devops-microservices-demo]# cat ~/.kube/config
cat: /root/.kube/config: No such file or directory
[root@localhost devops-microservices-demo]# curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
sudo chmod +x /usr/local/bin/minikube
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  119M  100  119M    0     0  8747k      0  0:00:13  0:00:13 --:--:-- 9343k
[root@localhost devops-microservices-demo]# minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: ssh, none

⛔  Exiting due to RSRC_INSUFFICIENT_CORES:  has less than 2 CPUs available, but Kubernetes requires at least 2 to be available

[root@localhost devops-microservices-demo]# nproc
1
[root@localhost devops-microservices-demo]# minikube start --cpus=2
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: ssh, none

⛔  Exiting due to RSRC_INSUFFICIENT_CORES:  has less than 2 CPUs available, but Kubernetes requires at least 2 to be available

[root@localhost devops-microservices-demo]# exitxvxzvw,kdamsexit
bash: exitxvxzvw,kdamsexit: command not found
[root@localhost devops-microservices-demo]# exit
exit
[dev2@localhost ~]$ exit
logout
Connection to 10.0.0.108 closed.
dev@Titan:~$ ssh dev2@10.0.0.108
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Mar 18 21:27:09 2025 from 10.0.0.113
[dev2@localhost ~]$ sudo su
[sudo] password for dev2: 
[root@localhost dev2]# minikube start
bash: minikube: command not found
[root@localhost dev2]# minikube start
bash: minikube: command not found
[root@localhost dev2]# minikube start
bash: minikube: command not found
[root@localhost dev2]# which minikube
/usr/bin/which: no minikube in (/root/.local/bin:/root/bin:/sbin:/bin:/usr/sbin:/usr/bin)
[root@localhost dev2]# ^[[200~minikube version
bash: $'\E[200~minikube': command not found
[root@localhost dev2]# minikube version
bash: minikube: command not found
[root@localhost dev2]# curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  119M  100  119M    0     0  8961k      0  0:00:13  0:00:13 --:--:-- 9574k
[root@localhost dev2]# chmod +x minikube
[root@localhost dev2]# sudo mv minikube /usr/local/bin/
[root@localhost dev2]# minikube version
bash: minikube: command not found
[root@localhost dev2]# ls -l /usr/local/bin/minikube
-rwxr-xr-x 1 root root 125267037 Mar 18 22:22 /usr/local/bin/minikube
[root@localhost dev2]# echo $PATH
/root/.local/bin:/root/bin:/sbin:/bin:/usr/sbin:/usr/bin
[root@localhost dev2]# source ~/.bashrc
[root@localhost dev2]# minikube version
bash: minikube: command not found
[root@localhost dev2]# echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
[root@localhost dev2]# source ~/.bashrc
[root@localhost dev2]# echo $PATH
/root/.local/bin:/root/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
[root@localhost dev2]# echo $PATH
/root/.local/bin:/root/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
[root@localhost dev2]# minikube version
minikube version: v1.35.0
commit: dd5d320e41b5451cdf3c01891bc4e13d189586ed-dirty
[root@localhost dev2]# minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: none, ssh
🛑  The "docker" driver should not be used with root privileges. If you wish to continue as root, use --force.
💡  If you are running minikube within a VM, consider using --driver=none:
📘    https://minikube.sigs.k8s.io/docs/reference/drivers/none/

❌  Exiting due to DRV_AS_ROOT: The "docker" driver should not be used with root privileges.

[root@localhost dev2]# exit
exit
[dev2@localhost ~]$ minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
👎  Unable to pick a default driver. Here is what was considered, in preference order:
    ▪ docker: Not healthy: "docker version --format {{.Server.Os}}-{{.Server.Version}}:{{.Server.Platform.Name}}" exit status 1: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.48/version": dial unix /var/run/docker.sock: connect: permission denied
    ▪ docker: Suggestion: Add your user to the 'docker' group: 'sudo usermod -aG docker $USER && newgrp docker' <https://docs.docker.com/engine/install/linux-postinstall/>
💡  Alternatively you could install one of these drivers:
    ▪ kvm2: Not installed: exec: "virsh": executable file not found in $PATH
    ▪ qemu2: Not installed: exec: "qemu-system-x86_64": executable file not found in $PATH
    ▪ podman: Not installed: exec: "podman": executable file not found in $PATH
    ▪ virtualbox: Not installed: unable to find VBoxManage in $PATH

❌  Exiting due to DRV_NOT_HEALTHY: Found driver(s) but none were healthy. See above for suggestions how to fix installed drivers.

[dev2@localhost ~]$ sudo usermod -aG docker $USER
[dev2@localhost ~]$ newgrp docker
[dev2@localhost ~]$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[dev2@localhost ~]$ minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: ssh, none

⛔  Exiting due to RSRC_INSUFFICIENT_CONTAINER_MEMORY: docker only has 1499MiB available, less than the required 1800MiB for Kubernetes

[dev2@localhost ~]$ exit
exit
[dev2@localhost ~]$ exit
logout
Connection to 10.0.0.108 closed.
dev@Titan:~$ ssh dev2@10.0.0.108
dev2@10.0.0.108's password: 
Permission denied, please try again.
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Mar 18 22:20:16 2025 from 10.0.0.113
[dev2@localhost ~]$ minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.46 ...
💾  Downloading Kubernetes v1.32.0 preload ...
    > preloaded-images-k8s-v18-v1...:  333.57 MiB / 333.57 MiB  100.00% 6.40 Mi
    > gcr.io/k8s-minikube/kicbase...:  500.31 MiB / 500.31 MiB  100.00% 5.25 Mi
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🐳  Preparing Kubernetes v1.32.0 on Docker 27.4.1 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner

❗  /usr/local/bin/kubectl is version 1.25.4, which may have incompatibilities with Kubernetes 1.32.0.
    ▪ Want kubectl v1.32.0? Try 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
[dev2@localhost ~]$ minikube kubectl -- get pods -A
    > kubectl.sha256:  64 B / 64 B [-------------------------] 100.00% ? p/s 0s
    > kubectl:  54.67 MiB / 54.67 MiB [--------------] 100.00% 3.22 MiB p/s 17s
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-668d6bf9bc-gbgf2           1/1     Running   0             105s
kube-system   etcd-minikube                      1/1     Running   0             112s
kube-system   kube-apiserver-minikube            1/1     Running   0             112s
kube-system   kube-controller-manager-minikube   1/1     Running   0             112s
kube-system   kube-proxy-fbx68                   1/1     Running   0             105s
kube-system   kube-scheduler-minikube            1/1     Running   0             112s
kube-system   storage-provisioner                1/1     Running   1 (74s ago)   108s
[dev2@localhost ~]$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.32.0/bin/linux/amd64/kubectl"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   220  100   220    0     0    856      0 --:--:-- --:--:-- --:--:--   856
[dev2@localhost ~]$ chmod +x ./kubectl
[dev2@localhost ~]$ sudo mv ./kubectl /usr/local/bin/kubectl
[sudo] password for dev2: 
[dev2@localhost ~]$ kubectl version --client
/usr/local/bin/kubectl: line 1: syntax error near unexpected token `<'
/usr/local/bin/kubectl: line 1: `<?xml version='1.0' encoding='UTF-8'?><Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Details>No such object: kubernetes-release/release/v1.32.0/bin/linux/amd64/kubectl</Details></Error>'
[dev2@localhost ~]$ curl -LO "https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   138  100   138    0     0    704      0 --:--:-- --:--:-- --:--:--   700
100 54.6M  100 54.6M    0     0  8810k      0  0:00:06  0:00:06 --:--:-- 9262k
[dev2@localhost ~]$ chmod +x ./kubectl
[dev2@localhost ~]$ sudo mv ./kubectl /usr/local/bin/kubectl
[dev2@localhost ~]$ kubectl version --client
Client Version: v1.32.0
Kustomize Version: v5.5.0
[dev2@localhost ~]$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dev2@localhost ~]$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    97  100    97    0     0    391      0 --:--:-- --:--:-- --:--:--   391
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 6808k  100 6808k    0     0  2918k      0  0:00:02  0:00:02 --:--:-- 5985k
[dev2@localhost ~]$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.26.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
[dev2@localhost ~]$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:43753
CoreDNS is running at https://127.0.0.1:43753/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dev2@localhost ~]$ helm install my-postgresql bitnami/postgresql \
  --set postgresqlPassword=senha123 \
  --set postgresqlDatabase=meubanco
Error: INSTALLATION FAILED: repo bitnami not found
[dev2@localhost ~]$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
[dev2@localhost ~]$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
[dev2@localhost ~]$ helm install my-postgresql bitnami/postgresql \
  --set postgresqlPassword=senha123 \
  --set postgresqlDatabase=meubanco
NAME: my-postgresql
LAST DEPLOYED: Tue Mar 18 22:42:05 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 16.5.2
APP VERSION: 17.4.0

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    my-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run my-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.4.0-debian-12-r8 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host my-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/my-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
[dev2@localhost ~]$ kubectl get pods
NAME              READY   STATUS              RESTARTS   AGE
my-postgresql-0   0/1     ContainerCreating   0          20s
[dev2@localhost ~]$ kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
[dev2@localhost ~]$ kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          12m
[dev2@localhost ~]$ kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
[dev2@localhost ~]$ ^[[200~kubectl get secrets --namespace default
-bash: $'\E[200~kubectl': command not found
[dev2@localhost ~]$ kubectl get secrets --namespace default
NAME                                  TYPE                 DATA   AGE
my-postgresql                         Opaque               1      13m
sh.helm.release.v1.my-postgresql.v1   helm.sh/release.v1   1      13m
[dev2@localhost ~]$ ^[[200~kubectl get secret my-postgresql --namespace default -o yaml
-bash: $'\E[200~kubectl': command not found
[dev2@localhost ~]$ kubectl get secret my-postgresql --namespace default -o yaml
apiVersion: v1
data:
  postgres-password: OEltM3lRcDBtbw==
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: my-postgresql
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-03-19T01:42:06Z"
  labels:
    app.kubernetes.io/instance: my-postgresql
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: postgresql
    app.kubernetes.io/version: 17.4.0
    helm.sh/chart: postgresql-16.5.2
  name: my-postgresql
  namespace: default
  resourceVersion: "570"
  uid: 2d76a4b1-8299-493c-95d6-75342ac2987d
type: Opaque
[dev2@localhost ~]$ ^[[200~echo "OEltM3lRcDBtbw==" | base64 --decode
-bash: $'\E[200~echo': command not found
[dev2@localhost ~]$ echo "OEltM3lRcDBtbw==" | base64 --decode
8Im3yQp0mo[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend
[dev2@localhost devops-microservices-demo]$ cd Frontend/Docker/api/
[dev2@localhost api]$ docker build -t meu-frontend .
[+] Building 0.4s (1/1) FINISHED                                                       docker:default
 => [internal] load build definition from Dockerfile                                             0.2s
 => => transferring dockerfile: 2B                                                               0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ docker build -f Dockerfile.api -t meu-frontend .
[+] Building 2.8s (8/9)                                                                docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 251B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               2.0s
 => [internal] load .dockerignore                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => CANCELED [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed  0.4s
 => => resolve docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.1s
 => => sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd0 10.41kB / 10.41kB  0.0s
 => => sha256:246e088f8d9bae1efc1d3f0ab4a800616403520b3aecd810f30e27dc9d69e0b4 1.75kB / 1.75kB   0.0s
 => => sha256:bea3dea871787412b8fa17205b317e46dd3f37bdfb1952d6022290726219b9bc 5.28kB / 5.28kB   0.0s
 => => sha256:6e909acdb790c5a1989d9cfc795fda5a246ad6664bb27b5c688e2b734b2c5fad 0B / 28.20MB      0.5s
 => => sha256:da1cbe0d584f421af08256ce9d3d64b361a433126b700d76ac42c23da4259346 0B / 14.93MB      0.5s
 => [internal] load build context                                                                0.2s
 => => transferring context: 2B                                                                  0.0s
 => CACHED [2/5] WORKDIR /app                                                                    0.0s
 => ERROR [3/5] COPY frontend_service.py .                                                       0.0s
 => ERROR [4/5] COPY templates/ templates/                                                       0.0s
------
 > [3/5] COPY frontend_service.py .:
------
------
 > [4/5] COPY templates/ templates/:
------
Dockerfile.api:6
--------------------
   4 |     
   5 |     COPY frontend_service.py .
   6 | >>> COPY templates/ templates/
   7 |     
   8 |     RUN pip install fastapi uvicorn requests jinja2
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref 0e0b5173-a2f2-4dd4-9f5f-adca7ed167e1::t5w6ffu746r3rj4shp9e9trci: "/templates": not found
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ pwd
/home/dev2/devops-microservices-demo/Frontend
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ docker build -f Dockerfile.api -t meu-frontend .

[+] Building 0.3s (1/1) FINISHED                                                       docker:default
 => [internal] load build definition from Dockerfile.api                                         0.1s
 => => transferring dockerfile: 2B                                                               0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile.api: no such file or directory
[dev2@localhost Frontend]$ docker build -f Docker/Dockerfile.api -t meu-frontend .
[+] Building 0.2s (1/1) FINISHED                                                       docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 2B                                                               0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile.api: no such file or directory
[dev2@localhost Frontend]$ cd /home/dev2/devops-microservices-demo/Frontend/Docker/api
[dev2@localhost api]$ docker build -f Dockerfile.api -t meu-frontend .
[+] Building 1.9s (8/9)                                                                docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 251B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               1.2s
 => [internal] load .dockerignore                                                                0.0s
 => => transferring context: 2B                                                                  0.0s
 => CANCELED [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed  0.2s
 => => resolve docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.1s
 => => sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd0 10.41kB / 10.41kB  0.0s
 => => sha256:246e088f8d9bae1efc1d3f0ab4a800616403520b3aecd810f30e27dc9d69e0b4 1.75kB / 1.75kB   0.0s
 => => sha256:bea3dea871787412b8fa17205b317e46dd3f37bdfb1952d6022290726219b9bc 5.28kB / 5.28kB   0.0s
 => [internal] load build context                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => CACHED [2/5] WORKDIR /app                                                                    0.0s
 => ERROR [3/5] COPY frontend_service.py .                                                       0.0s
 => ERROR [4/5] COPY templates/ templates/                                                       0.0s
------
 > [3/5] COPY frontend_service.py .:
------
------
 > [4/5] COPY templates/ templates/:
------
Dockerfile.api:6
--------------------
   4 |     
   5 |     COPY frontend_service.py .
   6 | >>> COPY templates/ templates/
   7 |     
   8 |     RUN pip install fastapi uvicorn requests jinja2
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref 0e0b5173-a2f2-4dd4-9f5f-adca7ed167e1::z2l26vncznpec05846d0fxakt: "/templates": not found
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ vim Dockerfile.api 
[dev2@localhost api]$ sudo vim Dockerfile.api 
[sudo] password for dev2: 
[dev2@localhost api]$ docker build -f Dockerfile.api -t meu-frontend .
[+] Building 1.2s (8/9)                                                                docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 254B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               0.6s
 => [internal] load .dockerignore                                                                0.0s
 => => transferring context: 2B                                                                  0.0s
 => CANCELED [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed  0.3s
 => => resolve docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.1s
 => => sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd0 10.41kB / 10.41kB  0.0s
 => => sha256:246e088f8d9bae1efc1d3f0ab4a800616403520b3aecd810f30e27dc9d69e0b4 1.75kB / 1.75kB   0.0s
 => => sha256:bea3dea871787412b8fa17205b317e46dd3f37bdfb1952d6022290726219b9bc 5.28kB / 5.28kB   0.0s
 => [internal] load build context                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => CACHED [2/5] WORKDIR /app                                                                    0.0s
 => ERROR [3/5] COPY frontend_service.py .                                                       0.0s
 => ERROR [4/5] COPY ../templates/ templates/                                                    0.0s
------
 > [3/5] COPY frontend_service.py .:
------
------
 > [4/5] COPY ../templates/ templates/:
------
Dockerfile.api:6
--------------------
   4 |     
   5 |     COPY frontend_service.py .
   6 | >>> COPY ../templates/ templates/
   7 |     
   8 |     RUN pip install fastapi uvicorn requests jinja2
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref 0e0b5173-a2f2-4dd4-9f5f-adca7ed167e1::qi97kyomimn1gsz0ypve8rv84: "/templates": not found
[dev2@localhost api]$ cd /home/dev2/devops-microservices-demo/Frontend
[dev2@localhost Frontend]$ docker build -f Docker/api/Dockerfile.api -t meu-frontend .
[+] Building 19.3s (10/10) FINISHED                                                    docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 254B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               0.6s
 => [internal] load .dockerignore                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  8.3s
 => => resolve docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.1s
 => => sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd0 10.41kB / 10.41kB  0.0s
 => => sha256:246e088f8d9bae1efc1d3f0ab4a800616403520b3aecd810f30e27dc9d69e0b4 1.75kB / 1.75kB   0.0s
 => => sha256:bea3dea871787412b8fa17205b317e46dd3f37bdfb1952d6022290726219b9bc 5.28kB / 5.28kB   0.0s
 => => sha256:cec49b84de9df5777ee48546b2fd933b7ae87ec6746f4aa181d6893175331a21 3.51MB / 3.51MB   1.2s
 => => sha256:6e909acdb790c5a1989d9cfc795fda5a246ad6664bb27b5c688e2b734b2c5fa 28.20MB / 28.20MB  4.7s
 => => sha256:da1cbe0d584f421af08256ce9d3d64b361a433126b700d76ac42c23da425934 14.93MB / 14.93MB  4.3s
 => => sha256:9a95d1744747ce9c4e854b9f5822065468265f57a58dae4d8fcec30f1e60216f 249B / 249B       1.5s
 => => extracting sha256:6e909acdb790c5a1989d9cfc795fda5a246ad6664bb27b5c688e2b734b2c5fad        0.7s
 => => extracting sha256:cec49b84de9df5777ee48546b2fd933b7ae87ec6746f4aa181d6893175331a21        0.1s
 => => extracting sha256:da1cbe0d584f421af08256ce9d3d64b361a433126b700d76ac42c23da4259346        0.5s
 => => extracting sha256:9a95d1744747ce9c4e854b9f5822065468265f57a58dae4d8fcec30f1e60216f        0.0s
 => [internal] load build context                                                                0.2s
 => => transferring context: 2.13kB                                                              0.0s
 => [2/5] WORKDIR /app                                                                           0.1s
 => [3/5] COPY frontend_service.py .                                                             0.2s
 => [4/5] COPY ../templates/ templates/                                                          0.2s
 => [5/5] RUN pip install fastapi uvicorn requests jinja2                                        8.1s
 => exporting to image                                                                           1.3s 
 => => exporting layers                                                                          1.2s 
 => => writing image sha256:be7e7835279125c70804ca734c8856745e4343387be01f49c7cd6388a50c4781     0.0s 
 => => naming to docker.io/library/meu-frontend                                                  0.0s 
[dev2@localhost Frontend]$ cd ..                                                                      
[dev2@localhost devops-microservices-demo]$ ls                                                        
Backend  Frontend
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd Docker/
[dev2@localhost Docker]$ vim api/Dockerfile.api 
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ docker build -t meu-backend .
[+] Building 0.3s (1/1) FINISHED                                                       docker:default
 => [internal] load build definition from Dockerfile                                             0.1s
 => => transferring dockerfile: 2B                                                               0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost Backend]$ docker build -f Docker/api/Dockerfile.api -t meu-backend .
[+] Building 60.4s (9/9) FINISHED                                                      docker:default
 => [internal] load build definition from Dockerfile.api                                         0.0s
 => => transferring dockerfile: 232B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               6.3s
 => [internal] load .dockerignore                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.0s
 => [internal] load build context                                                                0.1s
 => => transferring context: 1.57kB                                                              0.0s
 => CACHED [2/4] WORKDIR /app                                                                    0.0s
 => [3/4] COPY backend_service.py .                                                              0.1s
 => [4/4] RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary                            52.4s
 => exporting to image                                                                           0.8s 
 => => exporting layers                                                                          0.7s 
 => => writing image sha256:78edb7c73a01e0495e4b4dd87bf5c8de448c91850d958b95008a663da52bfee2     0.0s 
 => => naming to docker.io/library/meu-backend                                                   0.0s 
[dev2@localhost Backend]$ docker push meu-frontend
Using default tag: latest
The push refers to repository [docker.io/library/meu-frontend]
1e973fabd059: Preparing 
6d5262e89875: Preparing 
6e10d14e2005: Preparing 
393195fdbe17: Preparing 
0796a33961ef: Preparing 
04f6e4cfc28e: Waiting 
140ec0aa8af0: Waiting 
1287fbecdfcc: Waiting 
denied: requested access to the resource is denied
[dev2@localhost Backend]$ docker login

USING WEB-BASED LOGIN

i Info → To sign in with credentials on the command line, use 'docker login -u <username>'
         

Your one-time device confirmation code is: LQZL-CSFC
Press ENTER to open your browser or submit your device code here: https://login.docker.com/activate

Waiting for authentication in the browser…

failed waiting for authentication: We found an existing Docker account for 'renan6rodrigues@outlook.com'.
      Sign-in using your password to complete your account verification process.
[dev2@localhost Backend]$ docker push meu-frontend
Using default tag: latest
The push refers to repository [docker.io/library/meu-frontend]
1e973fabd059: Preparing 
6d5262e89875: Preparing 
6e10d14e2005: Preparing 
393195fdbe17: Preparing 
0796a33961ef: Preparing 
04f6e4cfc28e: Waiting 
140ec0aa8af0: Waiting 
1287fbecdfcc: Waiting 
denied: requested access to the resource is denied
[dev2@localhost Backend]$  cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend
[dev2@localhost devops-microservices-demo]$ mkdir kubernetes
mkdir: cannot create directory ‘kubernetes’: Permission denied
[dev2@localhost devops-microservices-demo]$ sudo mkdir kubernetes
[sudo] password for dev2: 
[dev2@localhost devops-microservices-demo]$ vim kubernetes/deployment-frontend.yaml
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          63m
[dev2@localhost devops-microservices-demo]$ kubectl exec -it nome-do-pod -- bash
Error from server (NotFound): pods "nome-do-pod" not found
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          64m
[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ exit
exit
[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  password authentication failed for user "postgres"
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  password authentication failed for user "postgres"
I have no name!@my-postgresql-0:/$ 
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  database "meubanco" does not exist
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  database "meubanco" does not exist
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
I have no name!@my-postgresql-0:/$ ^C
I have no name!@my-postgresql-0:/$ exit
exit
command terminated with exit code 130
[dev2@localhost devops-microservices-demo]$ echo "OEltM3lRcDBtbw==" | base64 --decod
8Im3yQp0mo[dev2@localhost devops-microservices-demo]$ echo "OEltM3lRcDBtbw==" | base64 --decod
8Im3yQp0mo[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ psql -U postgres -d meubanco
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  database "meubanco" does not exist
I have no name!@my-postgresql-0:/$ psql -U postgres
Password for user postgres: 
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  password authentication failed for user "postgres"
I have no name!@my-postgresql-0:/$ psql -U postgres
Password for user postgres: 
psql (17.4)
Type "help" for help.

postgres=# \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules | 
  Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-
----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
=c/postgres          +
           |          |          |                 |             |             |        |           | 
postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
=c/postgres          +
           |          |          |                 |             |             |        |           | 
postgres=CTc/postgres
(3 rows)


postgres=# 
postgres=# exit
could not save history to file "//.psql_history": Read-only file system
I have no name!@my-postgresql-0:/$ exit
exit
[dev2@localhost devops-microservices-demo]$ kubectl get pods -l app=my-postgresql
No resources found in default namespace.
[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ kubectl get pods -l app=my-postgresql
bash: kubectl: command not found
I have no name!@my-postgresql-0:/$ kubectl get pods -l app=my-postgresqlkubectl get pods -l app=my-postgresql
^C
I have no name!@my-postgresql-0:/$ exit
exit
command terminated with exit code 130
[dev2@localhost devops-microservices-demo]$ kubectl get pods -l app=my-postgresql
No resources found in default namespace.
[dev2@localhost devops-microservices-demo]$ helm list --all-namespaces
NAME         	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART            	APP VERSION
my-postgresql	default  	1       	2025-03-18 22:42:05.826674822 -0300 -03	deployed	postgresql-16.5.2	17.4.0     
[dev2@localhost devops-microservices-demo]$ helm status my-postgresql
NAME: my-postgresql
LAST DEPLOYED: Tue Mar 18 22:42:05 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 16.5.2
APP VERSION: 17.4.0

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    my-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run my-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.4.0-debian-12-r8 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host my-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/my-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
[dev2@localhost devops-microservices-demo]$ kubectl get pods -n default
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          74m
[dev2@localhost devops-microservices-demo]$ export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
[dev2@localhost devops-microservices-demo]$ kubectl run my-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.4.0-debian-12-r8 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql --host my-postgresql -U postgres -d meubanco -p 5432
^C
[dev2@localhost devops-microservices-demo]$ kubectl run my-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.4.0-debian-12-r8 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql --host my-postgresql -U postgres -d meubanco -p 5432
psql: error: connection to server at "my-postgresql" (10.96.188.183), port 5432 failed: FATAL:  database "meubanco" does not exist
pod "my-postgresql-client" deleted
pod default/my-postgresql-client terminated (Error)
[dev2@localhost devops-microservices-demo]$ kubectl get pods -n default
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          76m
[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ kubectl get pods
bash: kubectl: command not found
I have no name!@my-postgresql-0:/$ exit
exit
command terminated with exit code 127
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          79m
[dev2@localhost devops-microservices-demo]$ kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP    81m
my-postgresql      ClusterIP   10.96.188.183   <none>        5432/TCP   79m
my-postgresql-hl   ClusterIP   None            <none>        5432/TCP   79m
[dev2@localhost devops-microservices-demo]$ kubectl get pods
/
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          80m
-bash: /: Is a directory
[dev2@localhost devops-microservices-demo]$ kubectl exec -it my-postgresql-0 -- bash
I have no name!@my-postgresql-0:/$ psql -U postgres
Password for user postgres: 
psql (17.4)
Type "help" for help.

postgres=# CREATE DATABASE meubanco;
CREATE DATABASE
postgres=# \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules | 
  Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-
----------------------
 meubanco  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
=c/postgres          +
           |          |          |                 |             |             |        |           | 
postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
=c/postgres          +
           |          |          |                 |             |             |        |           | 
postgres=CTc/postgres
(4 rows)






































postgres=# exit
could not save history to file "//.psql_history": Read-only file system
I have no name!@my-postgresql-0:/$ exit
exit
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
[dev2@localhost kubernetes]$ sudo su
[sudo] password for dev2: 
[root@localhost kubernetes]# vim deployment-frontend.yaml
[root@localhost kubernetes]# vim deployment-backend.yaml
[root@localhost kubernetes]# vim deployment-backend.yaml
[root@localhost kubernetes]# kubectl get svc
E0319 00:16:44.777420   59005 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
E0319 00:16:44.779299   59005 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
E0319 00:16:44.781643   59005 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
E0319 00:16:44.783301   59005 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
E0319 00:16:44.784990   59005 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@localhost kubernetes]# # Permitir o tráfego na porta 5432 (PostgreSQL)
sudo firewall-cmd --zone=public --add-port=5432/tcp --permanent

# Recarregar as regras do firewall para aplicar as mudanças
sudo firewall-cmd --reload
success
success
[root@localhost kubernetes]# minikube status
🤷  Profile "minikube" not found. Run "minikube profile list" to view all profiles.
👉  To start a cluster, run: "minikube start"
[root@localhost kubernetes]# minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: none, ssh
🛑  The "docker" driver should not be used with root privileges. If you wish to continue as root, use --force.
💡  If you are running minikube within a VM, consider using --driver=none:
📘    https://minikube.sigs.k8s.io/docs/reference/drivers/none/

❌  Exiting due to DRV_AS_ROOT: The "docker" driver should not be used with root privileges.

[root@localhost kubernetes]# exit
exit
[dev2@localhost kubernetes]$ minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Using the docker driver based on existing profile
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.46 ...
🏃  Updating the running docker "minikube" container ...
🐳  Preparing Kubernetes v1.32.0 on Docker 27.4.1 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
[dev2@localhost kubernetes]$ sudo systemctl status kubelet
Unit kubelet.service could not be found.
[dev2@localhost kubernetes]$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dev2@localhost kubernetes]$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   106m
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ vim hpa-backend.yaml
[dev2@localhost kubernetes]$ sudo vim hpa-backend.yaml
[sudo] password for dev2: 
[dev2@localhost kubernetes]$ kubectl apply -f kubernetes/deployment-frontend.yaml
error: the path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$  path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
 path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
^C
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ [dev2@localhost kubernetes]$  path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
 path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
^C
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ 

^C
[dev2@localhost kubernetes]$ [dev2@localhost kubernetes]$  path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
 path "kubernetes/deployment-frontend.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
error: no objects passed to apply
[dev2@localhost kubernetes]$ 
^C
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ 

^C
[dev2@localhost kubernetes]$ cat deployment-frontend.yaml
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ depmod 
deployment-backend.yaml   deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-frontend.yaml 
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ sudo vim deployment-frontend.yaml 
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
deployment.apps/frontend created
service/frontend created
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml 
deployment.apps/backend created
service/backend created
[dev2@localhost kubernetes]$ kubectl apply -f hpa-backend.yaml 
horizontalpodautoscaler.autoscaling/backend-hpa created
[dev2@localhost kubernetes]$ kubectl get pods
NAME                        READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb    0/1     ErrImagePull       0          32s
frontend-6f594cc7d6-zjhgt   0/1     ImagePullBackOff   0          50s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                        READY   STATUS         RESTARTS   AGE
backend-7f7fc54dcd-xzmcb    0/1     ErrImagePull   0          44s
frontend-6f594cc7d6-zjhgt   0/1     ErrImagePull   0          62s
[dev2@localhost kubernetes]$ kubectl get hpa
NAME          REFERENCE            TARGETS                 MINPODS   MAXPODS   REPLICAS   AGE
backend-hpa   Deployment/backend   memory: <unknown>/70%   1         10        1          32s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                        READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb    0/1     ImagePullBackOff   0          59s
frontend-6f594cc7d6-zjhgt   0/1     ImagePullBackOff   0          77s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                        READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb    0/1     ErrImagePull       0          65s
frontend-6f594cc7d6-zjhgt   0/1     ImagePullBackOff   0          83s
[dev2@localhost kubernetes]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED             SIZE
meu-backend                   latest    78edb7c73a01   About an hour ago   185MB
meu-frontend                  latest    be7e78352791   About an hour ago   153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago        1.31GB
kindest/node                  <none>    36d37c652064   23 months ago       936MB
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ sudo vim deployment-frontend.yaml 
[dev2@localhost kubernetes]$ sudo vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
deployment.apps/frontend-deployment created
service/frontend-service created
deployment.apps/backend-deployment created
service/backend-service created
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          3m47s
backend-deployment-7ffbf79d9f-s98jm    0/1     ErrImagePull       0          7s
frontend-6f594cc7d6-zjhgt              0/1     ImagePullBackOff   0          4m5s
frontend-deployment-65bdb668df-vsxkp   0/1     ErrImagePull       0          7s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          3m53s
backend-deployment-7ffbf79d9f-s98jm    0/1     ErrImagePull       0          13s
frontend-6f594cc7d6-zjhgt              0/1     ImagePullBackOff   0          4m11s
frontend-deployment-65bdb668df-vsxkp   0/1     ErrImagePull       0          13s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          4m4s
backend-deployment-7ffbf79d9f-s98jm    0/1     ImagePullBackOff   0          24s
frontend-6f594cc7d6-zjhgt              0/1     ImagePullBackOff   0          4m22s
frontend-deployment-65bdb668df-vsxkp   0/1     ImagePullBackOff   0          24s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          4m11s
backend-deployment-7ffbf79d9f-s98jm    0/1     ImagePullBackOff   0          31s
frontend-6f594cc7d6-zjhgt              0/1     ImagePullBackOff   0          4m29s
frontend-deployment-65bdb668df-vsxkp   0/1     ErrImagePull       0          31s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          4m16s
backend-deployment-7ffbf79d9f-s98jm    0/1     ErrImagePull       0          36s
frontend-6f594cc7d6-zjhgt              0/1     ImagePullBackOff   0          4m34s
frontend-deployment-65bdb668df-vsxkp   0/1     ErrImagePull       0          36s
[dev2@localhost kubernetes]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED             SIZE
meu-backend                   latest    78edb7c73a01   About an hour ago   185MB
meu-frontend                  latest    be7e78352791   About an hour ago   153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago        1.31GB
kindest/node                  <none>    36d37c652064   23 months ago       936MB
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ docker build -t meu-frontend:latest .  

[+] Building 0.4s (1/1) FINISHED                                                       docker:default
 => [internal] load build definition from Dockerfile                                             0.2s
 => => transferring dockerfile: 2B                                                               0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-xzmcb               0/1     ImagePullBackOff   0          5m53s
backend-deployment-7ffbf79d9f-s98jm    0/1     ImagePullBackOff   0          2m13s
frontend-6f594cc7d6-zjhgt              0/1     ErrImagePull       0          6m11s
frontend-deployment-65bdb668df-vsxkp   0/1     ImagePullBackOff   0          2m13s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-xzmcb" deleted
pod "backend-deployment-7ffbf79d9f-s98jm" deleted
pod "frontend-6f594cc7d6-zjhgt" deleted
pod "frontend-deployment-65bdb668df-vsxkp" deleted
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED        SIZE
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago   97MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago   89.7MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago   69.6MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago   94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago   150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago   61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago   736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago    31.5MB
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ docker build -f Docker/api/Dockerfile.api -t meu-frontend .
[+] Building 0.0s (0/0)                                                                docker:default
WARNING: current commit information was not captured by the build: failed to read current commit information with git rev-parse --is-inside-work-tree
ERROR: resolve : lstat Docker: no such file or directory
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Frontend/
[dev2@localhost Frontend]$ docker build -f Docker/api/Dockerfile.api -t meu-frontend .
[+] Building 21.0s (10/10) FINISHED                                                    docker:default
 => [internal] load build definition from Dockerfile.api                                         0.1s
 => => transferring dockerfile: 254B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               1.9s
 => [internal] load .dockerignore                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  9.2s
 => => resolve docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.1s
 => => sha256:bea3dea871787412b8fa17205b317e46dd3f37bdfb1952d6022290726219b9bc 5.28kB / 5.28kB   0.0s
 => => sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd0 10.41kB / 10.41kB  0.0s
 => => sha256:246e088f8d9bae1efc1d3f0ab4a800616403520b3aecd810f30e27dc9d69e0b4 1.75kB / 1.75kB   0.0s
 => => sha256:6e909acdb790c5a1989d9cfc795fda5a246ad6664bb27b5c688e2b734b2c5fa 28.20MB / 28.20MB  4.7s
 => => sha256:da1cbe0d584f421af08256ce9d3d64b361a433126b700d76ac42c23da425934 14.93MB / 14.93MB  2.8s
 => => sha256:cec49b84de9df5777ee48546b2fd933b7ae87ec6746f4aa181d6893175331a21 3.51MB / 3.51MB   2.6s
 => => sha256:9a95d1744747ce9c4e854b9f5822065468265f57a58dae4d8fcec30f1e60216f 249B / 249B       2.9s
 => => extracting sha256:6e909acdb790c5a1989d9cfc795fda5a246ad6664bb27b5c688e2b734b2c5fad        0.9s
 => => extracting sha256:cec49b84de9df5777ee48546b2fd933b7ae87ec6746f4aa181d6893175331a21        0.1s
 => => extracting sha256:da1cbe0d584f421af08256ce9d3d64b361a433126b700d76ac42c23da4259346        0.5s
 => => extracting sha256:9a95d1744747ce9c4e854b9f5822065468265f57a58dae4d8fcec30f1e60216f        0.0s
 => [internal] load build context                                                                0.2s
 => => transferring context: 2.13kB                                                              0.0s
 => [2/5] WORKDIR /app                                                                           0.1s
 => [3/5] COPY frontend_service.py .                                                             0.2s
 => [4/5] COPY ../templates/ templates/                                                          0.2s
 => [5/5] RUN pip install fastapi uvicorn requests jinja2                                        7.9s
 => exporting to image                                                                           0.9s 
 => => exporting layers                                                                          0.8s 
 => => writing image sha256:899327028402bf28057dc926f512a69da5b9272f105f150b5545d1fa55109065     0.0s 
 => => naming to docker.io/library/meu-frontend                                                  0.0s 
[dev2@localhost Frontend]$ cd ..                                                                      
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ docker build -f Docker/api/Dockerfile.api -t meu-backend .
[+] Building 12.1s (9/9) FINISHED                                                      docker:default
 => [internal] load build definition from Dockerfile.api                                         0.1s
 => => transferring dockerfile: 232B                                                             0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                               0.6s
 => [internal] load .dockerignore                                                                0.1s
 => => transferring context: 2B                                                                  0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a  0.0s
 => [internal] load build context                                                                0.1s
 => => transferring context: 1.57kB                                                              0.0s
 => CACHED [2/4] WORKDIR /app                                                                    0.0s
 => [3/4] COPY backend_service.py .                                                              0.1s
 => [4/4] RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary                             9.8s
 => exporting to image                                                                           0.9s 
 => => exporting layers                                                                          0.9s 
 => => writing image sha256:e30e4bd204a30c4966663f7d5d6049b17e782f1a00623ab2a17e3e2f71f9f4dd     0.0s 
 => => naming to docker.io/library/meu-backend                                                   0.0s 
[dev2@localhost Backend]$ docker images                                                               
REPOSITORY                                TAG        IMAGE ID       CREATED         SIZE              
meu-backend                               latest     e30e4bd204a3   2 minutes ago   185MB
meu-frontend                              latest     899327028402   3 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago    97MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago    69.6MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago    89.7MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago    94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago    150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago    61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago    736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago     31.5MB
[dev2@localhost Backend]$ kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
error: the path "deployment-frontend.yaml" does not exist
error: the path "deployment-backend.yaml" does not exist
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-d5k29               0/1     ImagePullBackOff   0          8m47s
backend-deployment-7ffbf79d9f-c54gm    0/1     ImagePullBackOff   0          8m47s
frontend-6f594cc7d6-sxsqd              0/1     ImagePullBackOff   0          8m47s
frontend-deployment-65bdb668df-jlgfd   0/1     ImagePullBackOff   0          8m47s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-d5k29               0/1     ImagePullBackOff   0          9m14s
backend-deployment-7ffbf79d9f-c54gm    0/1     ImagePullBackOff   0          9m14s
frontend-6f594cc7d6-sxsqd              0/1     ImagePullBackOff   0          9m14s
frontend-deployment-65bdb668df-jlgfd   0/1     ImagePullBackOff   0          9m14s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-d5k29" deleted
pod "backend-deployment-7ffbf79d9f-c54gm" deleted
pod "frontend-6f594cc7d6-sxsqd" deleted
pod "frontend-deployment-65bdb668df-jlgfd" deleted
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-7f7fc54dcd-7qmnn               0/1     ErrImagePull   0          15s
backend-deployment-7ffbf79d9f-2mbcc    0/1     ErrImagePull   0          15s
frontend-6f594cc7d6-g7wv5              0/1     ErrImagePull   0          15s
frontend-deployment-65bdb668df-x4lr6   0/1     ErrImagePull   0          15s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-7qmnn" deleted
pod "backend-deployment-7ffbf79d9f-2mbcc" deleted
pod "frontend-6f594cc7d6-g7wv5" deleted
pod "frontend-deployment-65bdb668df-x4lr6" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-7f7fc54dcd-pj4fq               0/1     ErrImagePull        0          7s
backend-deployment-7ffbf79d9f-dpncc    0/1     ContainerCreating   0          7s
frontend-6f594cc7d6-942zk              0/1     ContainerCreating   0          7s
frontend-deployment-65bdb668df-nr6h4   0/1     ContainerCreating   0          7s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-pj4fq" deleted
pod "backend-deployment-7ffbf79d9f-dpncc" deleted
pod "frontend-6f594cc7d6-942zk" deleted
pod "frontend-deployment-65bdb668df-nr6h4" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-7f7fc54dcd-r4xcs               0/1     ErrImagePull   0          9s
backend-deployment-7ffbf79d9f-l4h8p    0/1     ErrImagePull   0          9s
frontend-6f594cc7d6-47kt2              0/1     ErrImagePull   0          9s
frontend-deployment-65bdb668df-9nf6j   0/1     ErrImagePull   0          9s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-r4xcs" deleted
pod "backend-deployment-7ffbf79d9f-l4h8p" deleted
pod "frontend-6f594cc7d6-47kt2" deleted
pod "frontend-deployment-65bdb668df-9nf6j" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-7f7fc54dcd-8mrjv               0/1     ContainerCreating   0          3s
backend-deployment-7ffbf79d9f-hxxgd    0/1     ContainerCreating   0          3s
frontend-6f594cc7d6-6ccx6              0/1     ContainerCreating   0          3s
frontend-deployment-65bdb668df-m8gk7   0/1     ContainerCreating   0          3s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-8mrjv" deleted
pod "backend-deployment-7ffbf79d9f-hxxgd" deleted
pod "frontend-6f594cc7d6-6ccx6" deleted
pod "frontend-deployment-65bdb668df-m8gk7" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-gxxlp               0/1     ImagePullBackOff   0          15s
backend-deployment-7ffbf79d9f-8sg4j    0/1     ErrImagePull       0          15s
frontend-6f594cc7d6-dfsz9              0/1     ErrImagePull       0          15s
frontend-deployment-65bdb668df-xmb8g   0/1     ErrImagePull       0          15s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-7f7fc54dcd-gxxlp               0/1     ImagePullBackOff   0          18s
backend-deployment-7ffbf79d9f-8sg4j    0/1     ErrImagePull       0          18s
frontend-6f594cc7d6-dfsz9              0/1     ErrImagePull       0          18s
frontend-deployment-65bdb668df-xmb8g   0/1     ErrImagePull       0          18s
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-7f7fc54dcd-gxxlp" deleted
pod "backend-deployment-7ffbf79d9f-8sg4j" deleted
pod "frontend-6f594cc7d6-dfsz9" deleted
pod "frontend-deployment-65bdb668df-xmb8g" deleted
[dev2@localhost kubernetes]$ kubectl delete deployment frontend backend
deployment.apps "frontend" deleted
deployment.apps "backend" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-deployment-7ffbf79d9f-qhr2z    0/1     ErrImagePull        0          8s
frontend-6f594cc7d6-gvk2x              0/1     Terminating         0          8s
frontend-deployment-65bdb668df-b8dqq   0/1     ContainerCreating   0          8s
[dev2@localhost kubernetes]$ kubectl delete deployment frontend backend
Error from server (NotFound): deployments.apps "frontend" not found
Error from server (NotFound): deployments.apps "backend" not found
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-qhr2z    0/1     ImagePullBackOff   0          22s
frontend-deployment-65bdb668df-b8dqq   0/1     ErrImagePull       0          22s
[dev2@localhost kubernetes]$ kubectl delete deployment backend-deployment-7ffbf79d9f-qhr2z
Error from server (NotFound): deployments.apps "backend-deployment-7ffbf79d9f-qhr2z" not found
[dev2@localhost kubernetes]$ kubectl delete deployment frontend-deployment-65bdb668df-b8dqq
Error from server (NotFound): deployments.apps "frontend-deployment-65bdb668df-b8dqq" not found
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-deployment-7ffbf79d9f-qhr2z" deleted
pod "frontend-deployment-65bdb668df-b8dqq" deleted
[dev2@localhost kubernetes]$ kubectl delete pods --all
pod "backend-deployment-7ffbf79d9f-8phfm" deleted
pod "frontend-deployment-65bdb668df-cwcc2" deleted
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-r6hf6    0/1     ErrImagePull   0          8s
frontend-deployment-65bdb668df-v2xjx   0/1     ErrImagePull   0          8s
[dev2@localhost kubernetes]$ kubectl delete deployment backend
kubectl delete deployment frontend
Error from server (NotFound): deployments.apps "backend" not found
Error from server (NotFound): deployments.apps "frontend" not found
[dev2@localhost kubernetes]$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
backend-deployment    0/1     1            0           17m
frontend-deployment   0/1     1            0           17m
[dev2@localhost kubernetes]$ kubectl get replicasets
NAME                             DESIRED   CURRENT   READY   AGE
backend-deployment-7ffbf79d9f    1         1         0       17m
frontend-deployment-65bdb668df   1         1         0       17m
[dev2@localhost kubernetes]$ kubectl delete deployment backend-deployment
kubectl delete deployment frontend-deployment
deployment.apps "backend-deployment" deleted
deployment.apps "frontend-deployment" deleted
[dev2@localhost kubernetes]$ kubectl get pods
No resources found in default namespace.
[dev2@localhost kubernetes]$ kubectl apply -f deployment-frontend.yaml
kubectl apply -f deployment-backend.yaml
deployment.apps/frontend-deployment created
service/frontend-service unchanged
deployment.apps/backend-deployment created
service/backend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ErrImagePull   0          6s
frontend-deployment-65bdb668df-6jnfc   0/1     ErrImagePull   0          6s
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   13 minutes ago   185MB
meu-frontend                              latest     899327028402   14 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago     97MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago     89.7MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago     69.6MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago     94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago     150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago     61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago     736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago      31.5MB
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          24s
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          24s
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          2m13s
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          2m13s
[dev2@localhost kubernetes]$ kubectl describe pod backend-deployment-7ffbf79d9f-txhtf
Name:             backend-deployment-7ffbf79d9f-txhtf
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 00:51:52 -0300
Labels:           app=backend
                  pod-template-hash=7ffbf79d9f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.41
IPs:
  IP:           10.244.0.41
Controlled By:  ReplicaSet/backend-deployment-7ffbf79d9f
Containers:
  backend:
    Container ID:   
    Image:          meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      your_postgres_user
      POSTGRES_PASSWORD:  your_postgres_password
      POSTGRES_DB:        your_postgres_db
      POSTGRES_HOST:      postgres-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hdg5j (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-hdg5j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m29s                default-scheduler  Successfully assigned default/backend-deployment-7ffbf79d9f-txhtf to minikube
  Normal   Pulling    61s (x4 over 2m29s)  kubelet            Pulling image "meu-backend:latest"
  Warning  Failed     60s (x4 over 2m26s)  kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     60s (x4 over 2m26s)  kubelet            Error: ErrImagePull
  Normal   BackOff    9s (x8 over 2m25s)   kubelet            Back-off pulling image "meu-backend:latest"
  Warning  Failed     9s (x8 over 2m25s)   kubelet            Error: ImagePullBackOff
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          4m13s
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          4m13s
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
kubectl apply -f deployment-frontend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          4m20s
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          4m20s
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          4m36s
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          4m36s
[dev2@localhost kubernetes]$ ^[[200~docker images
-bash: $'\E[200~docker': command not found
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   19 minutes ago   185MB
meu-frontend                              latest     899327028402   20 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago     97MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago     69.6MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago     89.7MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago     94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago     150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago     61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago     736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago      31.5MB
[dev2@localhost kubernetes]$ minikube stop
minikube start
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
    ▪ MINIKUBE_ACTIVE_DOCKERD=minikube
✨  Using the docker driver based on existing profile
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.46 ...
📌  Noticed you have an activated docker-env on docker driver in this terminal:
❗  Please re-eval your docker-env, To ensure your environment variables have updated ports:

	'minikube -p minikube docker-env'

	
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.32.0 on Docker 27.4.1 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   20 minutes ago   185MB
meu-frontend                              latest     899327028402   21 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago     97MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago     89.7MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago     69.6MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago     94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago     150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago     61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago     736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago      31.5MB
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
kubectl apply -f deployment-frontend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ErrImagePull   0          7m44s
frontend-deployment-65bdb668df-6jnfc   0/1     ErrImagePull   0          7m44s
[dev2@localhost kubernetes]$ kubectl describe pod backend-deployment-7ffbf79d9f-txhtf
Name:             backend-deployment-7ffbf79d9f-txhtf
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 00:51:52 -0300
Labels:           app=backend
                  pod-template-hash=7ffbf79d9f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.44
IPs:
  IP:           10.244.0.44
Controlled By:  ReplicaSet/backend-deployment-7ffbf79d9f
Containers:
  backend:
    Container ID:   
    Image:          meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      your_postgres_user
      POSTGRES_PASSWORD:  your_postgres_password
      POSTGRES_DB:        your_postgres_db
      POSTGRES_HOST:      postgres-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hdg5j (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-hdg5j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason          Age                     From               Message
  ----     ------          ----                    ----               -------
  Normal   Scheduled       7m59s                   default-scheduler  Successfully assigned default/backend-deployment-7ffbf79d9f-txhtf to minikube
  Normal   Pulling         5m4s (x5 over 7m58s)    kubelet            Pulling image "meu-backend:latest"
  Warning  Failed          5m3s (x5 over 7m55s)    kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed          5m3s (x5 over 7m55s)    kubelet            Error: ErrImagePull
  Warning  Failed          2m53s (x20 over 7m54s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff         2m40s (x21 over 7m54s)  kubelet            Back-off pulling image "meu-backend:latest"
  Normal   SandboxChanged  62s                     kubelet            Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff         25s (x2 over 56s)       kubelet            Back-off pulling image "meu-backend:latest"
  Warning  Failed          25s (x2 over 56s)       kubelet            Error: ImagePullBackOff
  Normal   Pulling         11s (x3 over 61s)       kubelet            Pulling image "meu-backend:latest"
  Warning  Failed          9s (x3 over 57s)        kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed          9s (x3 over 57s)        kubelet            Error: ErrImagePull
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ docker ps -a
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS                       PORTS     NAMES
fd449b1a0743   6e38f40d628d                 "/storage-provisioner"   2 minutes ago    Up 2 minutes                           k8s_storage-provisioner_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_5
49fa4c29f675   c69fa2e9cbf5                 "/coredns -conf /etc…"   2 minutes ago    Up 2 minutes                           k8s_coredns_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_2
53a91a5a276c   6e38f40d628d                 "/storage-provisioner"   2 minutes ago    Exited (1) 2 minutes ago               k8s_storage-provisioner_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_4
7f3f22832b64   040f9f8aac8c                 "/usr/local/bin/kube…"   2 minutes ago    Up 2 minutes                           k8s_kube-proxy_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_2
58439cc29184   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_2
991de034d545   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_frontend-deployment-65bdb668df-6jnfc_default_13c848b2-ac77-4e72-a573-9ced06c2527f_1
f3ff37f7567d   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_backend-deployment-7ffbf79d9f-txhtf_default_9bb12e6b-a590-4cd5-9324-d4470570c0f6_1
ca68f23395b5   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_2
64accd335537   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_2
c89f9a4d4404   a9e7e6b294ba                 "etcd --advertise-cl…"   3 minutes ago    Up 3 minutes                           k8s_etcd_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_2
a36de72f8260   c2e17b8d0f4a                 "kube-apiserver --ad…"   3 minutes ago    Up 3 minutes                           k8s_kube-apiserver_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_2
cc552cee3d19   8cab3d2a8bd0                 "kube-controller-man…"   3 minutes ago    Up 3 minutes                           k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_2
9bcc0aa64500   a389e107f4ff                 "kube-scheduler --au…"   3 minutes ago    Up 3 minutes                           k8s_kube-scheduler_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_2
10bd92b6c239   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_2
7b6b35cb6488   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_2
decf891c582e   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_2
0f53d603cae2   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_2
6c8fc951068e   c69fa2e9cbf5                 "/coredns -conf /etc…"   43 minutes ago   Exited (0) 3 minutes ago               k8s_coredns_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_1
6ba9a1f051f3   040f9f8aac8c                 "/usr/local/bin/kube…"   43 minutes ago   Exited (2) 3 minutes ago               k8s_kube-proxy_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_1
aac5ad9a59a8   8cab3d2a8bd0                 "kube-controller-man…"   43 minutes ago   Exited (2) 3 minutes ago               k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_1
cb685c45e8c7   a389e107f4ff                 "kube-scheduler --au…"   43 minutes ago   Exited (1) 3 minutes ago               k8s_kube-scheduler_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_1
7e85c2241e3f   c2e17b8d0f4a                 "kube-apiserver --ad…"   43 minutes ago   Exited (137) 3 minutes ago             k8s_kube-apiserver_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_1
f5e8297d21a5   a9e7e6b294ba                 "etcd --advertise-cl…"   43 minutes ago   Exited (0) 3 minutes ago               k8s_etcd_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_1
fa6e9913d577   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_1
07ab3bfe6723   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_1
f8f1b17906fe   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_1
a0e8e725d2a0   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_1
5fb24687de7e   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_1
04ef65b86aaf   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_1
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   25 minutes ago   185MB
meu-frontend                              latest     899327028402   26 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago     97MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago     89.7MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago     69.6MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago     94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago     150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago     61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago     736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago      31.5MB
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ [dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   25 minutes ago   185MB
meu-frontend                              latest     899327028402   26 minutes ago   153MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago     97MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago     89.7MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago     69.6MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago     94MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago     150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago     61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago     736kB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago      31.5MB
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
deployment.apps/backend-deployment unchanged
service/backend-service unchanged
[dev2@localhost kubernetes]$ 

^C
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          13m
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          13m
[dev2@localhost kubernetes]$ kubectl describe pod backend-deployment-7ffbf79d9f-txhtf
Name:             backend-deployment-7ffbf79d9f-txhtf
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 00:51:52 -0300
Labels:           app=backend
                  pod-template-hash=7ffbf79d9f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.44
IPs:
  IP:           10.244.0.44
Controlled By:  ReplicaSet/backend-deployment-7ffbf79d9f
Containers:
  backend:
    Container ID:   
    Image:          meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      your_postgres_user
      POSTGRES_PASSWORD:  your_postgres_password
      POSTGRES_DB:        your_postgres_db
      POSTGRES_HOST:      postgres-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hdg5j (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-hdg5j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason          Age                    From               Message
  ----     ------          ----                   ----               -------
  Normal   Scheduled       13m                    default-scheduler  Successfully assigned default/backend-deployment-7ffbf79d9f-txhtf to minikube
  Normal   Pulling         10m (x5 over 13m)      kubelet            Pulling image "meu-backend:latest"
  Warning  Failed          10m (x5 over 13m)      kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed          10m (x5 over 13m)      kubelet            Error: ErrImagePull
  Warning  Failed          8m48s (x20 over 13m)   kubelet            Error: ImagePullBackOff
  Normal   BackOff         8m35s (x21 over 13m)   kubelet            Back-off pulling image "meu-backend:latest"
  Normal   SandboxChanged  6m57s                  kubelet            Pod sandbox changed, it will be killed and re-created.
  Normal   Pulling         3m54s (x5 over 6m56s)  kubelet            Pulling image "meu-backend:latest"
  Warning  Failed          3m52s (x5 over 6m52s)  kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed          3m52s (x5 over 6m52s)  kubelet            Error: ErrImagePull
  Normal   BackOff         106s (x20 over 6m51s)  kubelet            Back-off pulling image "meu-backend:latest"
  Warning  Failed          106s (x20 over 6m51s)  kubelet            Error: ImagePullBackOff
[dev2@localhost kubernetes]$ ls  -lsha
total 12K
   0 drwxr-xr-x 2 root root  93 Mar 19 00:30 .
   0 drwxr-xr-x 6 root root  67 Mar 18 23:42 ..
4.0K -rw-r--r-- 1 root root 959 Mar 19 00:30 deployment-backend.yaml
4.0K -rw-r--r-- 1 root root 681 Mar 19 00:29 deployment-frontend.yaml
4.0K -rw-r--r-- 1 root root 356 Mar 19 00:23 hpa-backend.yaml
[dev2@localhost kubernetes]$ docker pull meu-backend:latest
Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
[dev2@localhost kubernetes]$ exit
logout
Connection to 10.0.0.108 closed.
dev@Titan:~$ ssh dev2@10.0.0.108
dev2@10.0.0.108's password: 
Permission denied, please try again.
dev2@10.0.0.108's password: 
Permission denied, please try again.
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last failed login: Wed Mar 19 02:03:26 -03 2025 from 10.0.0.113 on ssh:notty
There were 2 failed login attempts since the last successful login.
Last login: Tue Mar 18 22:31:28 2025 from 10.0.0.113
[dev2@localhost ~]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-txhtf    0/1     ImagePullBackOff   0          71m
frontend-deployment-65bdb668df-6jnfc   0/1     ImagePullBackOff   0          71m
[dev2@localhost ~]$ minikube cache reload
[dev2@localhost ~]$ kubectl delete pod backend-deployment-7ffbf79d9f-txhtf
kubectl delete pod frontend-deployment-65bdb668df-6jnfc
pod "backend-deployment-7ffbf79d9f-txhtf" deleted
pod "frontend-deployment-65bdb668df-6jnfc" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ErrImagePull   0          9s
frontend-deployment-65bdb668df-lvrbp   0/1     ErrImagePull   0          8s
[dev2@localhost ~]$ kubectl apply -f deployment-backend.yaml
kubectl apply -f deployment-frontend.yaml
error: the path "deployment-backend.yaml" does not exist
error: the path "deployment-frontend.yaml" does not exist
[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd 
.cache/                    .config/                   devops-microservices-demo/ .docker/                   .kube/                     .minikube/                 
[dev2@localhost ~]$ cd 
.cache/                    .config/                   devops-microservices-demo/ .docker/                   .kube/                     .minikube/                 
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          35s
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          34s
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ErrImagePull       0          11m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          11m
[dev2@localhost devops-microservices-demo]$ [dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ docker ps -a
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS                       PORTS     NAMES
fd449b1a0743   6e38f40d628d                 "/storage-provisioner"   2 minutes ago    Up 2 minutes                           k8s_storage-provisioner_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_5
49fa4c29f675   c69fa2e9cbf5                 "/coredns -conf /etc…"   2 minutes ago    Up 2 minutes                           k8s_coredns_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_2
53a91a5a276c   6e38f40d628d                 "/storage-provisioner"   2 minutes ago    Exited (1) 2 minutes ago               k8s_storage-provisioner_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_4
7f3f22832b64   040f9f8aac8c                 "/usr/local/bin/kube…"   2 minutes ago    Up 2 minutes                           k8s_kube-proxy_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_2
58439cc29184   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_2
991de034d545   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_frontend-deployment-65bdb668df-6jnfc_default_13c848b2-ac77-4e72-a573-9ced06c2527f_1
f3ff37f7567d   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_backend-deployment-7ffbf79d9f-txhtf_default_9bb12e6b-a590-4cd5-9324-d4470570c0f6_1
ca68f23395b5   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_2
64accd335537   registry.k8s.io/pause:3.10   "/pause"                 2 minutes ago    Up 2 minutes                           k8s_POD_storage-provisioner_kube-system_9c9f035a-d472-4d28-b2e3-53d70223d002_2
c89f9a4d4404   a9e7e6b294ba                 "etcd --advertise-cl…"   3 minutes ago    Up 3 minutes                           k8s_etcd_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_2
a36de72f8260   c2e17b8d0f4a                 "kube-apiserver --ad…"   3 minutes ago    Up 3 minutes                           k8s_kube-apiserver_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_2
cc552cee3d19   8cab3d2a8bd0                 "kube-controller-man…"   3 minutes ago    Up 3 minutes                           k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_2
9bcc0aa64500   a389e107f4ff                 "kube-scheduler --au…"   3 minutes ago    Up 3 minutes                           k8s_kube-scheduler_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_2
10bd92b6c239   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_2
7b6b35cb6488   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_2
decf891c582e   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_2
0f53d603cae2   registry.k8s.io/pause:3.10   "/pause"                 3 minutes ago    Up 3 minutes                           k8s_POD_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_2
6c8fc951068e   c69fa2e9cbf5                 "/coredns -conf /etc…"   43 minutes ago   Exited (0) 3 minutes ago               k8s_coredns_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_1
6ba9a1f051f3   040f9f8aac8c                 "/usr/local/bin/kube…"   43 minutes ago   Exited (2) 3 minutes ago               k8s_kube-proxy_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_1
aac5ad9a59a8   8cab3d2a8bd0                 "kube-controller-man…"   43 minutes ago   Exited (2) 3 minutes ago               k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_1
cb685c45e8c7   a389e107f4ff                 "kube-scheduler --au…"   43 minutes ago   Exited (1) 3 minutes ago               k8s_kube-scheduler_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_1
7e85c2241e3f   c2e17b8d0f4a                 "kube-apiserver --ad…"   43 minutes ago   Exited (137) 3 minutes ago             k8s_kube-apiserver_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_1
f5e8297d21a5   a9e7e6b294ba                 "etcd --advertise-cl…"   43 minutes ago   Exited (0) 3 minutes ago               k8s_etcd_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_1
fa6e9913d577   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_coredns-668d6bf9bc-gbgf2_kube-system_11511471-663d-4458-b66e-c518978f683f_1
07ab3bfe6723   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-proxy-fbx68_kube-system_98d3f2e1-8075-4830-a1a5-7042c9912700_1
f8f1b17906fe   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-controller-manager-minikube_kube-system_843c74f7b3bc7d7040a05c31708a6a30_1
a0e8e725d2a0   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_etcd-minikube_kube-system_2b4b75c2a289008e0b381891e9683040_1
5fb24687de7e   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-apiserver-minikube_kube-system_d72d0a4cf4be077c9919d46b7358a5e8_1
04ef65b86aaf   registry.k8s.io/pause:3.10   "/pause"                 43 minutes ago   Exited (0) 3 minutes ago               k8s_POD_kube-scheduler-minikube_kube-system_d14ce008bee3a1f3bd7cf547688f9dfe_1
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ eval $(minikube -p minikube docker-env)
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED          SIZE
meu-backend                               latest     e30e4bd204a3   25 minutes ago   185MB
^Cev2@localhost devops-microservices-demo]$    ImagePullBackOff   0          34s .docker/                   .kube/                     .minikube/                 is deniedend, repository does not exist or 
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ ^C
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ vim deployment-frontend.yaml 
[dev2@localhost kubernetes]$ vim hpa-backend.yaml 


Press ENTER or type command to continue
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          3h16m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          3h16m
[dev2@localhost kubernetes]$ docker images | grep meu-backend
docker images | grep meu-frontend
meu-backend                   latest    78edb7c73a01   6 hours ago     185MB
meu-frontend                  latest    be7e78352791   6 hours ago     153MB
[dev2@localhost kubernetes]$ docker build -t meu-backend:latest .
docker build -t meu-frontend:latest .
[+] Building 0.4s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.1s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.0s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost kubernetes]$ pwd
/home/dev2/devops-microservices-demo/kubernetes
(failed reverse-i-search)`docke build -t ': ^Ccker build -t meu-frontend:latest .
[dev2@localhost kubernetes]$ ^C
[dev2@localhost kubernetes]$ 
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/Docker/
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cd /home/dev2/devops-microservices-demo/Backend/Docker/api
docker build -t meu-backend:latest -f Dockerfile.api .
[+] Building 4.3s (7/8)                                                                                                                                                                       docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.1s
 => => transferring dockerfile: 232B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      3.9s
 => [internal] load .dockerignore                                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => CACHED [2/4] WORKDIR /app                                                                                                                                                                           0.0s
 => ERROR [3/4] COPY backend_service.py .                                                                                                                                                               0.0s
------
 > [3/4] COPY backend_service.py .:
------
Dockerfile.api:5
--------------------
   3 |     WORKDIR /app
   4 |     
   5 | >>> COPY backend_service.py .
   6 |     
   7 |     RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref 0e0b5173-a2f2-4dd4-9f5f-adca7ed167e1::yp3u5cq0j0lmq8d7wkx27t8kp: "/backend_service.py": not found
[dev2@localhost api]$ ls /home/dev2/devops-microservices-demo/Backend/Docker/api
Dockerfile.api
[dev2@localhost api]$ cd /home/dev2/devops-microservices-demo/Backend
docker build -f Docker/api/Dockerfile.api -t meu-backend .
[+] Building 1.6s (9/9) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.0s
 => => transferring dockerfile: 232B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      1.1s
 => [internal] load .dockerignore                                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.1s
 => => transferring context: 40B                                                                                                                                                                        0.0s
 => CACHED [2/4] WORKDIR /app                                                                                                                                                                           0.0s
 => CACHED [3/4] COPY backend_service.py .                                                                                                                                                              0.0s
 => CACHED [4/4] RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary                                                                                                                             0.0s
 => exporting to image                                                                                                                                                                                  0.0s
 => => exporting layers                                                                                                                                                                                 0.0s
 => => writing image sha256:78edb7c73a01e0495e4b4dd87bf5c8de448c91850d958b95008a663da52bfee2                                                                                                            0.0s
 => => naming to docker.io/library/meu-backend                                                                                                                                                          0.0s
[dev2@localhost Backend]$ cd /home/dev2/devops-microservices-demo/Frontend
docker build -f Docker/Dockerfile -t meu-frontend .
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.0s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ docker build -f Docker/Dockerfile -t meu-frontend .
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.0s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ ls -lsha Docker/api/Dockerfile.api 
4.0K -rw-r--r-- 1 root root 211 Mar 18 23:24 Docker/api/Dockerfile.api
[dev2@localhost Frontend]$ docker build -f Docker/api/Dockerfile.api -t meu-frontend .
[+] Building 2.1s (10/10) FINISHED                                                                                                                                                            docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.0s
 => => transferring dockerfile: 254B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      1.7s
 => [internal] load .dockerignore                                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.0s
 => => transferring context: 110B                                                                                                                                                                       0.0s
 => CACHED [2/5] WORKDIR /app                                                                                                                                                                           0.0s
 => CACHED [3/5] COPY frontend_service.py .                                                                                                                                                             0.0s
 => CACHED [4/5] COPY ../templates/ templates/                                                                                                                                                          0.0s
 => CACHED [5/5] RUN pip install fastapi uvicorn requests jinja2                                                                                                                                        0.0s
 => exporting to image                                                                                                                                                                                  0.0s
 => => exporting layers                                                                                                                                                                                 0.0s
 => => writing image sha256:be7e7835279125c70804ca734c8856745e4343387be01f49c7cd6388a50c4781                                                                                                            0.0s
 => => naming to docker.io/library/meu-frontend                                                                                                                                                         0.0s
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd ..]
-bash: cd: ..]: No such file or directory
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd ..
[dev2@localhost devops-microservices-demo]$ l
-bash: l: command not found
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim hpa-backend.yaml 
[dev2@localhost kubernetes]$ cat deployment-frontend.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: meu-frontend:latest  # Verifique se a imagem está correta aqui
          env:
            - name: BACKEND_URL
              value: "http://backend-service:8080"
          ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP

[dev2@localhost kubernetes]$ cat deployment-frontend.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: meu-frontend:latest  # Verifique se a imagem está correta aqui
          env:
            - name: BACKEND_URL
              value: "http://backend-service:8080"
          ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP

[dev2@localhost kubernetes]$ cat hpa-backend.yaml 
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70

[dev2@localhost kubernetes]$ cat deployment-backend.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: meu-backend:latest  # Verifique se a imagem está correta aqui
          env:
            - name: POSTGRES_USER
              value: "your_postgres_user"
            - name: POSTGRES_PASSWORD
              value: "your_postgres_password"
            - name: POSTGRES_DB
              value: "your_postgres_db"
            - name: POSTGRES_HOST
              value: "postgres-service"
            - name: POSTGRES_PORT
              value: "5432"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP

[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          5h36m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          5h36m
[dev2@localhost kubernetes]$ podman ps -a
-bash: podman: command not found
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ sudo vim deployment-backend.yaml 
[sudo] password for dev2: 
[dev2@localhost kubernetes]$ minikube start
😄  minikube v1.35.0 on Oracle 9.2 (vbox/amd64)
✨  Using the docker driver based on existing profile
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.46 ...
🏃  Updating the running docker "minikube" container ...
🐳  Preparing Kubernetes v1.32.0 on Docker 27.4.1 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd ..
[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ ls -lsha
total 4.0K
   0 drwxr-xr-x 3 root root   46 Mar 18 22:06 .
   0 drwxr-xr-x 6 root root   67 Mar 18 23:42 ..
4.0K -rw-r--r-- 1 root root 1.5K Mar 18 22:06 backend_service.py
   0 drwxr-xr-x 3 root root   17 Mar 18 22:06 Docker
[dev2@localhost Backend]$ cat backend_service.py 
import os
import threading
import uvicorn
from fastapi import FastAPI
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

POSTGRES_USER = os.getenv("POSTGRES_USER", "default_user")
POSTGRES_PASSWORD = os.getenv("POSTGRES_PASSWORD", "default_password")
POSTGRES_DB = os.getenv("POSTGRES_DB", "default_db")
POSTGRES_HOST = os.getenv("POSTGRES_HOST", "localhost")
POSTGRES_PORT = os.getenv("POSTGRES_PORT", "5432")

DATABASE_URL = f"postgresql://{POSTGRES_USER}:{POSTGRES_PASSWORD}@{POSTGRES_HOST}:{POSTGRES_PORT}/{POSTGRES_DB}"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

backend_app = FastAPI(title="API de Processamento de Dados")

@backend_app.get("/")
def test_db_connection():
    try:
        with engine.connect() as connection:
            result = connection.execute(text("SELECT 1")).fetchone()
        return {"conexão realizada com sucesso! db_response": result[0]}
    except Exception as e:
        return {"error": str(e)}

backend_health_app = FastAPI(title="Health Check Backend")

@backend_health_app.get("/v1/health")
def health_check():
    return {"status": "ok"}

def run_backend():
    uvicorn.run(backend_app, host="0.0.0.0", port=8080)

def run_backend_health():
    uvicorn.run(backend_health_app, host="0.0.0.0", port=3030)

if __name__ == "__main__":
    threading.Thread(target=run_backend).start()
    threading.Thread(target=run_backend_health).start()
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd Docker/
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cat Dockerfile.api 
FROM python:3.9-slim

WORKDIR /app

COPY backend_service.py .

RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary

EXPOSE 8080 3030

CMD ["python", "backend_service.py"]
[dev2@localhost api]$ python --version
Python 3.9.16
[dev2@localhost api]$ ls -lsha
total 4.0K
   0 drwxr-xr-x 2 root root  28 Mar 18 22:06 .
   0 drwxr-xr-x 3 root root  17 Mar 18 22:06 ..
4.0K -rw-r--r-- 1 root root 189 Mar 18 22:06 Dockerfile.api
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Frontend/
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd Docker/
[dev2@localhost Docker]$ cat api/Dockerfile.api 
FROM python:3.9-slim

WORKDIR /app

COPY frontend_service.py .
COPY ../templates/ templates/

RUN pip install fastapi uvicorn requests jinja2

EXPOSE 8000 3030

CMD ["python", "frontend_service.py"]
[dev2@localhost Docker]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cat frontend_service.py 
import os
import threading
import uvicorn
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
import requests

BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:8080")

templates = Jinja2Templates(directory="templates")

frontend_app = FastAPI(title="Aplicação Frontend")

@frontend_app.get("/")
def home(request: Request):
    try:
        response = requests.get(BACKEND_URL)
        backend_data = response.json()
    except Exception as e:
        backend_data = {"error": str(e)}
    return templates.TemplateResponse("index.html", {"request": request, "backend_response": backend_data})

frontend_health_app = FastAPI(title="Health Check Frontend")

@frontend_health_app.get("/v1/health")
def health_check():
    return {"status": "ok"}

def run_frontend():
    uvicorn.run(frontend_app, host="0.0.0.0", port=8000)

def run_frontend_health():
    uvicorn.run(frontend_health_app, host="0.0.0.0", port=3030)

if __name__ == "__main__":
    threading.Thread(target=run_frontend).start()
    threading.Thread(target=run_frontend_health).start()
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd templates/
[dev2@localhost templates]$ ls
index.html
[dev2@localhost templates]$ cat index.html 
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Aplicação Frontend</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            margin: 0;
            padding: 20px;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Bem-vindo à Aplicação Frontend</h1>
        <p>Dados do backend:</p>
        <pre>{{ backend_response | tojson(indent=2) }}</pre>
    </div>
</body>
</html>
[dev2@localhost templates]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
meu-backend                   latest    78edb7c73a01   9 hours ago     185MB
meu-frontend                  latest    be7e78352791   9 hours ago     153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago    1.31GB
kindest/node                  <none>    36d37c652064   24 months ago   936MB
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          6h10m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          6h10m
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ vim deployment-backend.yaml 
[dev2@localhost kubernetes]$ vim deployment-frontend.yaml 
[dev2@localhost kubernetes]$ vim hpa-backend.yaml 
[dev2@localhost kubernetes]$ kubectl apply -f deployment-backend.yaml
kubectl apply -f deployment-frontend.yaml
kubectl apply -f hpa-backend.yaml
statefulset.apps/my-postgresql created
service/my-postgresql created
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
horizontalpodautoscaler.autoscaling/backend-hpa unchanged
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff    0          6h16m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff    0          6h16m
my-postgresql-0                        0/1     ContainerCreating   0          9s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff    0          6h16m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff    0          6h16m
my-postgresql-0                        0/1     ContainerCreating   0          34s
[dev2@localhost kubernetes]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          6h16m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          6h16m
my-postgresql-0                        1/1     Running            0          36s
[dev2@localhost kubernetes]$ vim hpa-backend.yaml 
[dev2@localhost kubernetes]$ kubectl logs backend-deployment-7ffbf79d9f-tl66s
Error from server (BadRequest): container "backend" in pod "backend-deployment-7ffbf79d9f-tl66s" is waiting to start: trying and failing to pull image
[dev2@localhost kubernetes]$ kubectl logs backend-deployment-7ffbf79d9f-tl66s
Error from server (BadRequest): container "backend" in pod "backend-deployment-7ffbf79d9f-tl66s" is waiting to start: trying and failing to pull image
[dev2@localhost kubernetes]$ kubectl describe pod backend-deployment-7ffbf79d9f-tl66s
Name:             backend-deployment-7ffbf79d9f-tl66s
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 02:05:12 -0300
Labels:           app=backend
                  pod-template-hash=7ffbf79d9f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.47
IPs:
  IP:           10.244.0.47
Controlled By:  ReplicaSet/backend-deployment-7ffbf79d9f
Containers:
  backend:
    Container ID:   
    Image:          meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      your_postgres_user
      POSTGRES_PASSWORD:  your_postgres_password
      POSTGRES_DB:        your_postgres_db
      POSTGRES_HOST:      postgres-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9plfx (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-9plfx:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                     From     Message
  ----     ------   ----                    ----     -------
  Normal   BackOff  42s (x1646 over 6h20m)  kubelet  Back-off pulling image "meu-backend:latest"
  Warning  Failed   42s (x1646 over 6h20m)  kubelet  Error: ImagePullBackOff
[dev2@localhost kubernetes]$ docker image
Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Download an image from a registry
  push        Upload an image to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

Run 'docker image COMMAND --help' for more information on a command.
[dev2@localhost kubernetes]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
meu-backend                   latest    78edb7c73a01   9 hours ago     185MB
meu-frontend                  latest    be7e78352791   9 hours ago     153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago    1.31GB
kindest/node                  <none>    36d37c652064   24 months ago   936MB
[dev2@localhost kubernetes]$ docker pull meu-backend:latest
Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ docker pull meu-backend:latest
Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd Docker/api/
[dev2@localhost api]$ docker pull meu-backend:latest
Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
[dev2@localhost api]$ docker login

USING WEB-BASED LOGIN

i Info → To sign in with credentials on the command line, use 'docker login -u <username>'
         

Your one-time device confirmation code is: KCFV-LDGP
Press ENTER to open your browser or submit your device code here: https://login.docker.com/activate

Waiting for authentication in the browser…
^Clogin canceled
[dev2@localhost api]$ docker login

USING WEB-BASED LOGIN

i Info → To sign in with credentials on the command line, use 'docker login -u <username>'
         

Your one-time device confirmation code is: FLMG-NKJM
Press ENTER to open your browser or submit your device code here: https://login.docker.com/activate

Waiting for authentication in the browser…
failed waiting for authentication: We found an existing Docker account for 'renan6rodrigues@outlook.com'.
      Sign-in using your password to complete your account verification process.
[dev2@localhost api]$ cd
[dev2@localhost ~]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-tl66s    0/1     ImagePullBackOff   0          6h41m
frontend-deployment-65bdb668df-lvrbp   0/1     ImagePullBackOff   0          6h41m
my-postgresql-0                        1/1     Running            0          25m
[dev2@localhost ~]$ kubectl logs backend-deployment-7ffbf79d9f-tl66s
kubectl logs frontend-deployment-65bdb668df-lvrbp
Error from server (BadRequest): container "backend" in pod "backend-deployment-7ffbf79d9f-tl66s" is waiting to start: trying and failing to pull image
Error from server (BadRequest): container "frontend" in pod "frontend-deployment-65bdb668df-lvrbp" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ kubectl delete pod backend-deployment-7ffbf79d9f-tl66s
kubectl delete pod frontend-deployment-65bdb668df-lvrbp
pod "backend-deployment-7ffbf79d9f-tl66s" deleted
pod "frontend-deployment-65bdb668df-lvrbp" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
backend-deployment-7ffbf79d9f-lvvtw    0/1     ErrImagePull        0          6s
frontend-deployment-65bdb668df-4hgh4   0/1     ContainerCreating   0          5s
my-postgresql-0                        1/1     Running             0          26m
[dev2@localhost ~]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-lvvtw    0/1     ErrImagePull   0          16s
frontend-deployment-65bdb668df-4hgh4   0/1     ErrImagePull   0          15s
my-postgresql-0                        1/1     Running        0          26m
[dev2@localhost ~]$ kubectl describe pod backend-deployment-7ffbf79d9f-lvvtw                
kubectl describe pod frontend-deployment-65bdb668df-4hgh4

Name:             backend-deployment-7ffbf79d9f-lvvtw
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 08:47:51 -0300
Labels:           app=backend
                  pod-template-hash=7ffbf79d9f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.51
IPs:
  IP:           10.244.0.51
Controlled By:  ReplicaSet/backend-deployment-7ffbf79d9f
Containers:
  backend:
    Container ID:   
    Image:          meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      your_postgres_user
      POSTGRES_PASSWORD:  your_postgres_password
      POSTGRES_DB:        your_postgres_db
      POSTGRES_HOST:      postgres-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-68w4j (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-68w4j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  76s                default-scheduler  Successfully assigned default/backend-deployment-7ffbf79d9f-lvvtw to minikube
  Normal   Pulling    30s (x3 over 74s)  kubelet            Pulling image "meu-backend:latest"
  Warning  Failed     28s (x3 over 72s)  kubelet            Failed to pull image "meu-backend:latest": Error response from daemon: pull access denied for meu-backend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     28s (x3 over 72s)  kubelet            Error: ErrImagePull
  Normal   BackOff    4s (x4 over 72s)   kubelet            Back-off pulling image "meu-backend:latest"
  Warning  Failed     4s (x4 over 72s)   kubelet            Error: ImagePullBackOff
Name:             frontend-deployment-65bdb668df-4hgh4
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 08:47:52 -0300
Labels:           app=frontend
                  pod-template-hash=65bdb668df
Annotations:      <none>
Status:           Pending
IP:               10.244.0.52
IPs:
  IP:           10.244.0.52
Controlled By:  ReplicaSet/frontend-deployment-65bdb668df
Containers:
  frontend:
    Container ID:   
    Image:          meu-frontend:latest
    Image ID:       
    Port:           8000/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      BACKEND_URL:  http://backend-service:8080
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-v45sx (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-v45sx:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  75s                default-scheduler  Successfully assigned default/frontend-deployment-65bdb668df-4hgh4 to minikube
  Normal   Pulling    30s (x3 over 74s)  kubelet            Pulling image "meu-frontend:latest"
  Warning  Failed     26s (x3 over 70s)  kubelet            Failed to pull image "meu-frontend:latest": Error response from daemon: pull access denied for meu-frontend, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     26s (x3 over 70s)  kubelet            Error: ErrImagePull
  Normal   BackOff    0s (x4 over 69s)   kubelet            Back-off pulling image "meu-frontend:latest"
  Warning  Failed     0s (x4 over 69s)   kubelet            Error: ImagePullBackOff
[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd Docker/
[dev2@localhost Docker]$ cd api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ vim Dockerfile.api 
[dev2@localhost api]$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dev2@localhost api]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-lvvtw    0/1     ImagePullBackOff   0          8m30s
frontend-deployment-65bdb668df-4hgh4   0/1     ImagePullBackOff   0          8m29s
my-postgresql-0                        1/1     Running            0          34m
[dev2@localhost api]$ [dev2@localhost api]$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
backend-deployment-7ffbf79d9f-lvvtw    0/1     ImagePullBackOff   0          8m30s
frontend-deployment-65bdb668df-4hgh4   0/1     ImagePullBackOff   0          8m29s
my-postgresql-0                        1/1     Running            0          34m
[dev2@localhost api]$ 
-bash: [dev2@localhost: command not found
-bash: NAME: command not found
-bash: backend-deployment-7ffbf79d9f-lvvtw: command not found
-bash: frontend-deployment-65bdb668df-4hgh4: command not found
-bash: my-postgresql-0: command not found
-bash: [dev2@localhost: command not found
[dev2@localhost api]$ kubectl delete pod backend-deployment-7ffbf79d9f-lvvtw
kubectl delete pod frontend-deployment-65bdb668df-4hgh4
pod "backend-deployment-7ffbf79d9f-lvvtw" deleted
pod "frontend-deployment-65bdb668df-4hgh4" deleted
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ kubectl get pods
NAME                                   READY   STATUS         RESTARTS   AGE
backend-deployment-7ffbf79d9f-w8njg    0/1     ErrImagePull   0          9s
frontend-deployment-65bdb668df-zvrn7   0/1     ErrImagePull   0          8s
my-postgresql-0                        1/1     Running        0          35m
[dev2@localhost api]$ kubectl delete deployment backend-deployment
kubectl delete deployment frontend-deployment
deployment.apps "backend-deployment" deleted
deployment.apps "frontend-deployment" deleted
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
my-postgresql-0   1/1     Running   0          36m
[dev2@localhost api]$ kubectl delete pod my-postgresql-0
pod "my-postgresql-0" deleted
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS              RESTARTS   AGE
my-postgresql-0   0/1     ContainerCreating   0          2s
[dev2@localhost api]$ kubectl delete statefulset my-postgresql
statefulset.apps "my-postgresql" deleted
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS        RESTARTS   AGE
my-postgresql-0   1/1     Terminating   0          8s
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS        RESTARTS   AGE
my-postgresql-0   1/1     Terminating   0          16s
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS        RESTARTS   AGE
my-postgresql-0   1/1     Terminating   0          17s
[dev2@localhost api]$ kubectl delete deployment my-postgresql
Error from server (NotFound): deployments.apps "my-postgresql" not found
[dev2@localhost api]$ kubectl get pods
NAME              READY   STATUS        RESTARTS   AGE
my-postgresql-0   1/1     Terminating   0          28s
[dev2@localhost api]$ kubectl delete pod my-postgresql-0 --force --grace-period=0
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
Error from server (NotFound): pods "my-postgresql-0" not found
[dev2@localhost api]$ kubectl get pods
No resources found in default namespace.
[dev2@localhost api]$ kubectl exec -it my-postgresql-0 -- bash
Error from server (NotFound): pods "my-postgresql-0" not found
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ l
-bash: l: command not found
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ ls -lsha
total 4.0K
   0 drwxr-xr-x  6 root root   67 Mar 18 23:42 .
4.0K drwx------. 8 dev2 dev2 4.0K Mar 19 08:51 ..
   0 drwxr-xr-x  3 root root   46 Mar 18 22:06 Backend
   0 drwxr-xr-x  4 root root   64 Mar 18 22:06 Frontend
   0 drwxr-xr-x  8 root root  163 Mar 18 22:06 .git
   0 drwxr-xr-x  2 root root   93 Mar 19 08:01 kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cat backend_service.py 
import os
import threading
import uvicorn
from fastapi import FastAPI
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

POSTGRES_USER = os.getenv("POSTGRES_USER", "default_user")
POSTGRES_PASSWORD = os.getenv("POSTGRES_PASSWORD", "default_password")
POSTGRES_DB = os.getenv("POSTGRES_DB", "default_db")
POSTGRES_HOST = os.getenv("POSTGRES_HOST", "localhost")
POSTGRES_PORT = os.getenv("POSTGRES_PORT", "5432")

DATABASE_URL = f"postgresql://{POSTGRES_USER}:{POSTGRES_PASSWORD}@{POSTGRES_HOST}:{POSTGRES_PORT}/{POSTGRES_DB}"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

backend_app = FastAPI(title="API de Processamento de Dados")

@backend_app.get("/")
def test_db_connection():
    try:
        with engine.connect() as connection:
            result = connection.execute(text("SELECT 1")).fetchone()
        return {"conexão realizada com sucesso! db_response": result[0]}
    except Exception as e:
        return {"error": str(e)}

backend_health_app = FastAPI(title="Health Check Backend")

@backend_health_app.get("/v1/health")
def health_check():
    return {"status": "ok"}

def run_backend():
    uvicorn.run(backend_app, host="0.0.0.0", port=8080)

def run_backend_health():
    uvicorn.run(backend_health_app, host="0.0.0.0", port=3030)

if __name__ == "__main__":
    threading.Thread(target=run_backend).start()
    threading.Thread(target=run_backend_health).start()
[dev2@localhost Backend]$ cd Docker/api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cat Dockerfile.api 
FROM python:3.9-slim

WORKDIR /app

COPY backend_service.py .

RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary

EXPOSE 8080 3030

CMD ["python", "backend_service.py"]
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ 
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd Docker/
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Frontend/
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cat frontend_service.py 
import os
import threading
import uvicorn
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
import requests

BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:8080")

templates = Jinja2Templates(directory="templates")

frontend_app = FastAPI(title="Aplicação Frontend")

@frontend_app.get("/")
def home(request: Request):
    try:
        response = requests.get(BACKEND_URL)
        backend_data = response.json()
    except Exception as e:
        backend_data = {"error": str(e)}
    return templates.TemplateResponse("index.html", {"request": request, "backend_response": backend_data})

frontend_health_app = FastAPI(title="Health Check Frontend")

@frontend_health_app.get("/v1/health")
def health_check():
    return {"status": "ok"}

def run_frontend():
    uvicorn.run(frontend_app, host="0.0.0.0", port=8000)

def run_frontend_health():
    uvicorn.run(frontend_health_app, host="0.0.0.0", port=3030)

if __name__ == "__main__":
    threading.Thread(target=run_frontend).start()
    threading.Thread(target=run_frontend_health).start()
[dev2@localhost Frontend]$ cd Docker/
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd api
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cat Dockerfile.api 
FROM python:3.9-slim

WORKDIR /app

COPY frontend_service.py .
COPY ../templates/ templates/

RUN pip install fastapi uvicorn requests jinja2

EXPOSE 8000 3030

CMD ["python", "frontend_service.py"]
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd templates/
[dev2@localhost templates]$ ls
index.html
[dev2@localhost templates]$ cat index.html 
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Aplicação Frontend</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            margin: 0;
            padding: 20px;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Bem-vindo à Aplicação Frontend</h1>
        <p>Dados do backend:</p>
        <pre>{{ backend_response | tojson(indent=2) }}</pre>
    </div>
</body>
</html>
[dev2@localhost templates]$ ls
index.html
[dev2@localhost templates]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd ..
[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd ..l
-bash: cd: ..l: No such file or directory
[dev2@localhost ~]$ sudo firewall-cmd --permanent --add-port=6443/tcp  # API Server
sudo firewall-cmd --permanent --add-port=10250/tcp # Kubelet
sudo firewall-cmd --permanent --add-port=10251/tcp # Controller
sudo firewall-cmd --permanent --add-port=10252/tcp # Scheduler
sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd
sudo firewall-cmd --permanent --add-port=30000-32767/tcp # NodePorts
sudo firewall-cmd --reload
[sudo] password for dev2: 
success
success
success
success
success
success
success
[dev2@localhost ~]$ docker tag meu-backend:latest localhost:5000/meu-backend
docker tag meu-frontend:latest localhost:5000/meu-frontend
docker push localhost:5000/meu-backend
docker push localhost:5000/meu-frontend
Using default tag: latest
The push refers to repository [localhost:5000/meu-backend]
Get "http://localhost:5000/v2/": dial tcp [::1]:5000: connect: connection refused
Using default tag: latest
The push refers to repository [localhost:5000/meu-frontend]
Get "http://localhost:5000/v2/": dial tcp [::1]:5000: connect: connection refused
[dev2@localhost ~]$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS        PORTS                                                                                                                                  NAMES
f72b26d496cd   kindest/node:v1.26.3                  "/usr/local/bin/entr…"   11 hours ago   Up 11 hours   127.0.0.1:43753->6443/tcp                                                                                                              kind-control-plane
04455b01de8e   gcr.io/k8s-minikube/kicbase:v0.0.46   "/usr/local/bin/entr…"   11 hours ago   Up 9 hours    127.0.0.1:32773->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32775->5000/tcp, 127.0.0.1:32776->8443/tcp, 127.0.0.1:32777->32443/tcp   minikube
[dev2@localhost ~]$ curl http://localhost:32775/v2/
curl: (56) Recv failure: Connection reset by peer
[dev2@localhost ~]$ minikube service registry --url -n kube-system

❌  Exiting due to SVC_NOT_FOUND: Service 'registry' was not found in 'kube-system' namespace.
You may select another namespace by using 'minikube service registry -n <namespace>'. Or list out all the services using 'minikube service list'

[dev2@localhost ~]$ minikube addons list | grep registry
| registry                    | minikube | disabled     | minikube                       |
| registry-aliases            | minikube | disabled     | 3rd party (unknown)            |
| registry-creds              | minikube | disabled     | 3rd party (UPMC Enterprises)   |
[dev2@localhost ~]$ minikube addons list | grep registry
| registry                    | minikube | disabled     | minikube                       |
| registry-aliases            | minikube | disabled     | 3rd party (unknown)            |
| registry-creds              | minikube | disabled     | 3rd party (UPMC Enterprises)   |
[dev2@localhost ~]$ kubectl get pods -n kube-system | grep registry
[dev2@localhost ~]$ minikube service registry --url -n kube-system

❌  Exiting due to SVC_NOT_FOUND: Service 'registry' was not found in 'kube-system' namespace.
You may select another namespace by using 'minikube service registry -n <namespace>'. Or list out all the services using 'minikube service list'

[dev2@localhost ~]$ minikube addons enable registry
💡  registry is an addon maintained by minikube. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image gcr.io/k8s-minikube/kube-registry-proxy:0.0.8
    ▪ Using image docker.io/registry:2.8.3
🔎  Verifying registry addon...
🌟  The 'registry' addon is enabled
[dev2@localhost ~]$ kubectl get pods -n kube-system | grep registry
registry-6c86875c6f-5xvdj          1/1     Running   0             2m1s
registry-proxy-llg8f               1/1     Running   0             2m1s
[dev2@localhost ~]$ minikube service registry --url -n kube-system
😿  service kube-system/registry has no node port
❗  Services [kube-system/registry] have type "ClusterIP" not meant to be exposed, however for local development minikube allows you to access this !
[dev2@localhost ~]$ http://192.168.49.2:5000
-bash: http://192.168.49.2:5000: No such file or directory
[dev2@localhost ~]$ kubectl get svc -n kube-system | grep registry
registry   ClusterIP   10.103.179.83   <none>        80/TCP,443/TCP           3m3s
[dev2@localhost ~]$ kubectl port-forward --namespace kube-system service/registry 5000:5000 &
[1] 357922
[dev2@localhost ~]$ error: Service registry does not have a service port 5000
kubectl port-forward --namespace kube-system service/registry 5000:80 &
^C
[1]+  Exit 1                  kubectl port-forward --namespace kube-system service/registry 5000:5000
[dev2@localhost ~]$ kubectl port-forward --namespace kube-system service/registry 5000:80 &
[1] 358328
[dev2@localhost ~]$ Forwarding from 127.0.0.1:5000 -> 5000
Forwarding from [::1]:5000 -> 5000
^C
[dev2@localhost ~]$ kubectl port-forward --namespace kube-system service/registry 5000:80 &
[2] 359185
[dev2@localhost ~]$ Unable to listen on port 5000: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:5000: bind: address already in use unable to create listener: Error listen tcp6 [::1]:5000: bind: address already in use]
error: unable to listen on any of the requested ports: [{5000 5000}]
^C
[2]+  Exit 1                  kubectl port-forward --namespace kube-system service/registry 5000:80
[dev2@localhost ~]$ ^C
[dev2@localhost ~]$ kubectl port-forward --namespace kube-system service/registry 5001:80 &
[2] 359442
[dev2@localhost ~]$ Forwarding from 127.0.0.1:5001 -> 5000
Forwarding from [::1]:5001 -> 5000
^C
[dev2@localhost ~]$ curl http://localhost:5001/v2/
Handling connection for 5001
{}[dev2@localhost ~]docker tag meu-backend:latest localhost:5001/meu-backendnd
docker tag meu-frontend:latest localhost:5001/meu-frontend
[dev2@localhost ~]$ docker push localhost:5001/meu-backend
docker push localhost:5001/meu-frontend
Using default tag: latest
The push refers to repository [localhost:5001/meu-backend]
Handling connection for 5001
Handling connection for 5001
56a9af7d23b7: Preparing 
d321655ddcdb: Preparing 
393195fdbe17: Preparing 
0796a33961ef: Preparing 
04f6e4cfc28e: Preparing 
140ec0aa8af0: Preparing 
140ec0aa8af0: Waiting 
1287fbecdfcc: Waiting 
Handling connection for 5001
Handling connection for 5001
d321655ddcdb: Pushing [================>                                  ]     512B/1.528kB
d321655ddcdb: Pushing [==================================================>]  3.584kB
56a9af7d23b7: Pushing [>                                                  ]  537.1kB/59.71MB
Handling connection for 5001
56a9af7d23b7: Pushing [=>                                                 ]  1.615MB/59.71MB
0796a33961ef: Pushing   5.12kB
04f6e4cfc28e: Pushing [>                                                  ]  447.5kB/41.52MB
Handling connection for 5001
393195fdbe17: Pushed 
56a9af7d23b7: Pushing [==>                                                ]  2.704MB/59.71MB
d321655ddcdb: Pushed 
Handling connection for 5001
56a9af7d23b7: Pushing [====>                                              ]  4.899MB/59.71MB
Handling connection for 5001
56a9af7d23b7: Pushing [============>                                      ]  15.32MB/59.71MB
Handling connection for 5001
56a9af7d23b7: Pushing [=====================================>             ]  45.21MB/59.71MB
56a9af7d23b7: Pushing [======================================>            ]  46.26MB/59.71MB
56a9af7d23b7: Pushing [==================================================>]  61.51MB
04f6e4cfc28e: Pushing [================>                                  ]   13.6MB/41.52MB
56a9af7d23b7: Pushed 
140ec0aa8af0: Pushed 
04f6e4cfc28e: Pushed 
1287fbecdfcc: Pushing [==================================>                ]   51.1MB/74.78MB
1287fbecdfcc: Pushing [============================================>      ]  66.73MB/74.78MB
Handling connection for 5001
1287fbecdfcc: Pushing [==================================================>]  77.84MB
Handling connection for 5001
1287fbecdfcc: Pushed 
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
latest: digest: sha256:d3b721cdf436cc199b57269dcd007fefac89817a42200a9c9d69fd802def4109 size: 1784
Using default tag: latest
Handling connection for 5001
The push refers to repository [localhost:5001/meu-frontend]
Handling connection for 5001
1e973fabd059: Preparing 
6d5262e89875: Preparing 
6e10d14e2005: Preparing 
393195fdbe17: Preparing 
0796a33961ef: Preparing 
04f6e4cfc28e: Preparing 
140ec0aa8af0: Waiting 
1287fbecdfcc: Preparing 
Handling connection for 5001
04f6e4cfc28e: Waiting 
Handling connection for 5001
1287fbecdfcc: Waiting 
6d5262e89875: Pushing [=============================>                     ]     512B/874B
1e973fabd059: Pushing [>                                                  ]    276kB/27.59MB
6e10d14e2005: Pushing [==================================================>]  3.584kB
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
1e973fabd059: Pushing [==>                                                ]  1.148MB/27.59MB
Handling connection for 5001
Handling connection for 5001
6d5262e89875: Pushed 
1e973fabd059: Pushing [==================================================>]  29.04MB
Handling connection for 5001
1e973fabd059: Pushed 
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
1287fbecdfcc: Mounted from meu-backend 
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
Handling connection for 5001
latest: digest: sha256:16595ebac7bf62d6d3ab67fe15e6162621b07fee51cc66d6fe070d920fa040d6 size: 1991
[dev2@localhost ~]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
localhost:5000/meu-backend    latest    78edb7c73a01   10 hours ago    185MB
localhost:5001/meu-backend    latest    78edb7c73a01   10 hours ago    185MB
meu-backend                   latest    78edb7c73a01   10 hours ago    185MB
localhost:5001/meu-frontend   latest    be7e78352791   10 hours ago    153MB
meu-frontend                  latest    be7e78352791   10 hours ago    153MB
localhost:5000/meu-frontend   latest    be7e78352791   10 hours ago    153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago    1.31GB
kindest/node                  <none>    36d37c652064   24 months ago   936MB
[dev2@localhost ~]$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[dev2@localhost ~]$ ^C
[dev2@localhost ~]$ curl http://localhost:5001/v2/
Handling connection for 5001
{}[dev2@localhost ~]$ ls
devops-microservices-demo
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml
[dev2@localhost kubernetes]$ rm -r deployment-backend.yaml 
rm: remove write-protected regular file 'deployment-backend.yaml'? y
rm: cannot remove 'deployment-backend.yaml': Permission denied
[dev2@localhost kubernetes]$ rm -r deployment-
deployment-backend.yaml   deployment-frontend.yaml  
[dev2@localhost kubernetes]$ rm -r deployment-
deployment-backend.yaml   deployment-frontend.yaml  
[dev2@localhost kubernetes]$ rm -r deployment-frontend.yaml 
rm: remove write-protected regular file 'deployment-frontend.yaml'? y
rm: cannot remove 'deployment-frontend.yaml': Permission denied
[dev2@localhost kubernetes]$ rm -r hpa-backend.yaml 
rm: remove write-protected regular file 'hpa-backend.yaml'? y
rm: cannot remove 'hpa-backend.yaml': Permission denied
[dev2@localhost kubernetes]$ vim postgresql-pvc.yaml
[dev2@localhost kubernetes]$ sudo vim postgresql-pvc.yaml
[sudo] password for dev2: 
[dev2@localhost kubernetes]$ kubectl apply -f postgresql-pvc.yaml
persistentvolumeclaim/postgres-pvc created
[dev2@localhost kubernetes]$ ls
deployment-backend.yaml  deployment-frontend.yaml  hpa-backend.yaml  postgresql-pvc.yaml
[dev2@localhost kubernetes]$ rm -r hpa-backend.yaml 
rm: remove write-protected regular file 'hpa-backend.yaml'? y
rm: cannot remove 'hpa-backend.yaml': Permission denied
[dev2@localhost kubernetes]$ sudo rm -r hpa-backend.yaml 
[dev2@localhost kubernetes]$ sudo rm -r deployment-backend.yaml 
[dev2@localhost kubernetes]$ sudo rm -r deployment-frontend.yaml 
[dev2@localhost kubernetes]$ docker ps -a
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS        PORTS                                                                                                                                  NAMES
f72b26d496cd   kindest/node:v1.26.3                  "/usr/local/bin/entr…"   11 hours ago   Up 11 hours   127.0.0.1:43753->6443/tcp                                                                                                              kind-control-plane
04455b01de8e   gcr.io/k8s-minikube/kicbase:v0.0.46   "/usr/local/bin/entr…"   11 hours ago   Up 9 hours    127.0.0.1:32773->22/tcp, 127.0.0.1:32774->2376/tcp, 127.0.0.1:32775->5000/tcp, 127.0.0.1:32776->8443/tcp, 127.0.0.1:32777->32443/tcp   minikube
[dev2@localhost kubernetes]$ ls
postgresql-pvc.yaml
[dev2@localhost kubernetes]$ kubectl get pods
No resources found in default namespace.
[dev2@localhost kubernetes]$ kubectl apply -f postgresql-pvc.yaml
persistentvolumeclaim/postgres-pvc unchanged
[dev2@localhost kubernetes]$ kubectl get pods
No resources found in default namespace.
[dev2@localhost kubernetes]$ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pvc   Bound    pvc-021732d5-163c-498e-9b84-0adfccb4323c   1Gi        RWO            standard       <unset>                 3m23s
[dev2@localhost kubernetes]$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
kube-system   coredns-668d6bf9bc-gbgf2           1/1     Running   3 (117m ago)   11h
kube-system   etcd-minikube                      1/1     Running   3 (117m ago)   11h
kube-system   kube-apiserver-minikube            1/1     Running   3 (117m ago)   11h
kube-system   kube-controller-manager-minikube   1/1     Running   3 (117m ago)   11h
kube-system   kube-proxy-fbx68                   1/1     Running   3 (117m ago)   11h
kube-system   kube-scheduler-minikube            1/1     Running   3 (117m ago)   11h
kube-system   registry-6c86875c6f-5xvdj          1/1     Running   0              24m
kube-system   registry-proxy-llg8f               1/1     Running   0              24m
kube-system   storage-provisioner                1/1     Running   7 (116m ago)   11h
[dev2@localhost kubernetes]$ cat postgresql-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

[dev2@localhost kubernetes]$ vim postgresql-deployment.yaml
[dev2@localhost kubernetes]$ vim postgresql-deployment.yaml
[dev2@localhost kubernetes]$ ls
postgresql-pvc.yaml
[dev2@localhost kubernetes]$ sudo vim postgresql-deployment.yaml
[sudo] password for dev2: 
[dev2@localhost kubernetes]$ kubectl apply -f postgresql-deployment.yaml
deployment.apps/postgresql created
[dev2@localhost kubernetes]$ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pvc   Bound    pvc-021732d5-163c-498e-9b84-0adfccb4323c   1Gi        RWO            standard       <unset>                 8m58s
[dev2@localhost kubernetes]$ ls
postgresql-deployment.yaml  postgresql-pvc.yaml
[dev2@localhost kubernetes]$ vim postgresql-service.yaml
[dev2@localhost kubernetes]$ sudo vim postgresql-service.yaml
[dev2@localhost kubernetes]$ kubectl apply -f postgresql-service.yaml

service/postgresql-service created
[dev2@localhost kubernetes]$ sudo vim postgresql-service.yaml
[sudo] password for dev2: 
Sorry, try again.
[sudo] password for dev2: 
[dev2@localhost kubernetes]$ sudo vim backend-deployment.yaml
[dev2@localhost kubernetes]$ kubectl apply -f backend-deployment.yaml
deployment.apps/meu-backend created
[dev2@localhost kubernetes]$ sudo vim frontend-deployment.yaml
[dev2@localhost kubernetes]$ apiVersion: apps/v1
kind: Deployment
metadata:
  name: meu-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: meu-frontend
  template:
    metadata:
      labels:
        app: meu-frontend
    spec:
      containers:
      - name: meu-frontend
        image: localhost:5001/meu-frontend:latest
        ports:
        - containerPort: 80  # Porta onde o frontend vai expor a aplicação
^C
[dev2@localhost kubernetes]$ ^C
[dev2@localhost kubernetes]$ kubectl apply -f frontend-deployment.yaml
deployment.apps/meu-frontend created
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  frontend-deployment.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ ls -lsha
total 20K
   0 drwxr-xr-x 2 root root 161 Mar 19 10:14 .
   0 drwxr-xr-x 6 root root  67 Mar 18 23:42 ..
4.0K -rw-r--r-- 1 root root 708 Mar 19 10:14 backend-deployment.yaml
4.0K -rw-r--r-- 1 root root 405 Mar 19 10:14 frontend-deployment.yaml
4.0K -rw-r--r-- 1 root root 720 Mar 19 10:04 postgresql-deployment.yaml
4.0K -rw-r--r-- 1 root root 162 Mar 19 09:56 postgresql-pvc.yaml
4.0K -rw-r--r-- 1 root root 206 Mar 19 10:09 postgresql-service.yaml
[dev2@localhost kubernetes]$ backend-service.yaml
-bash: backend-service.yaml: command not found
[dev2@localhost kubernetes]$ sudo vim backend-service.yaml
[dev2@localhost kubernetes]$ kubectl apply -f backend-service.yaml
service/backend-service configured
[dev2@localhost kubernetes]$ sudo vim frontend-service.yaml
[dev2@localhost kubernetes]$ kubectl apply -f frontend-service.yaml
service/frontend-service configured
[dev2@localhost kubernetes]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-7dffc9f5c4-7sbwc    0/1     ImagePullBackOff   0          5m56s
meu-frontend-74787cdf4f-bczjv   0/1     ImagePullBackOff   0          5m3s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          14m
[dev2@localhost kubernetes]$ kubectl exec postgresql-9865b8f7d-rlcfl
error: you must specify at least one command for the container
[dev2@localhost kubernetes]$ kubectl exec -it postgresql-9865b8f7d-rlcfl -- bash
root@postgresql-9865b8f7d-rlcfl:/# psql -U postgres -d meubanco
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "postgres" does not exist
root@postgresql-9865b8f7d-rlcfl:/# exit
exit
command terminated with exit code 2
[dev2@localhost kubernetes]$ sudo vim backend-deployment.yaml
[dev2@localhost kubernetes]$ kubectl apply -f backend-deployment.yaml
deployment.apps/meu-backend configured
[dev2@localhost kubernetes]$ kubectl exec -it postgresql-9865b8f7d-rlcfl -- psql -U teste -d meubanco
psql (17.4 (Debian 17.4-1.pgdg120+2))
Type "help" for help.

meubanco=# exit
[dev2@localhost kubernetes]$ exit
logout
Connection to 10.0.0.108 closed.
dev@Titan:~$ ssh dev2@10.0.0.108
dev2@10.0.0.108's password: 
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Wed Mar 19 02:03:33 2025 from 10.0.0.113
[dev2@localhost ~]$ kubectl exec -it postgresql-9865b8f7d-rlcfl -- psql -U teste -d meubanco
psql (17.4 (Debian 17.4-1.pgdg120+2))
Type "help" for help.

meubanco=# exit
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-wfdl2     0/1     ImagePullBackOff   0          30m
meu-backend-7dffc9f5c4-7sbwc    0/1     ImagePullBackOff   0          41m
meu-frontend-74787cdf4f-bczjv   0/1     ImagePullBackOff   0          40m
postgresql-9865b8f7d-rlcfl      1/1     Running            0          50m
[dev2@localhost ~]$ kubectl logs meu-backend-6b5655594-wfdl2
kubectl logs meu-backend-7dffc9f5c4-7sbwc
Error from server (BadRequest): container "meu-backend" in pod "meu-backend-6b5655594-wfdl2" is waiting to start: trying and failing to pull image
Error from server (BadRequest): container "meu-backend" in pod "meu-backend-7dffc9f5c4-7sbwc" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ kubectl logs meu-frontend-74787cdf4f-bczjv
Error from server (BadRequest): container "meu-frontend" in pod "meu-frontend-74787cdf4f-bczjv" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
meu-backend                   latest    78edb7c73a01   12 hours ago    185MB
localhost:5000/meu-backend    latest    78edb7c73a01   12 hours ago    185MB
localhost:5001/meu-backend    latest    78edb7c73a01   12 hours ago    185MB
localhost:5001/meu-frontend   latest    be7e78352791   12 hours ago    153MB
meu-frontend                  latest    be7e78352791   12 hours ago    153MB
localhost:5000/meu-frontend   latest    be7e78352791   12 hours ago    153MB
gcr.io/k8s-minikube/kicbase   v0.0.46   e72c4cbe9b29   2 months ago    1.31GB
kindest/node                  <none>    36d37c652064   24 months ago   936MB
[dev2@localhost ~]$ docker tag meu-backend localhost:5001/meu-backend
docker tag meu-frontend localhost:5001/meu-frontend
[dev2@localhost ~]$ docker push localhost:5001/meu-backend
docker push localhost:5001/meu-frontend
Using default tag: latest
The push refers to repository [localhost:5001/meu-backend]
56a9af7d23b7: Layer already exists 
d321655ddcdb: Layer already exists 
393195fdbe17: Layer already exists 
0796a33961ef: Layer already exists 
04f6e4cfc28e: Layer already exists 
140ec0aa8af0: Layer already exists 
1287fbecdfcc: Layer already exists 
latest: digest: sha256:d3b721cdf436cc199b57269dcd007fefac89817a42200a9c9d69fd802def4109 size: 1784
Using default tag: latest
The push refers to repository [localhost:5001/meu-frontend]
1e973fabd059: Layer already exists 
6d5262e89875: Layer already exists 
6e10d14e2005: Layer already exists 
393195fdbe17: Layer already exists 
0796a33961ef: Layer already exists 
04f6e4cfc28e: Layer already exists 
140ec0aa8af0: Layer already exists 
1287fbecdfcc: Layer already exists 
latest: digest: sha256:16595ebac7bf62d6d3ab67fe15e6162621b07fee51cc66d6fe070d920fa040d6 size: 1991
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-wfdl2
kubectl delete pod meu-backend-7dffc9f5c4-7sbwc
kubectl delete pod meu-frontend-74787cdf4f-bczjv
pod "meu-backend-6b5655594-wfdl2" deleted
pod "meu-backend-7dffc9f5c4-7sbwc" deleted
pod "meu-frontend-74787cdf4f-bczjv" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-bxr57     0/1     ImagePullBackOff   0          9s
meu-backend-7dffc9f5c4-wgpqr    0/1     ErrImagePull       0          8s
meu-frontend-74787cdf4f-l8lf8   0/1     ErrImagePull       0          7s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          55m
[dev2@localhost ~]$ [dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-bxr57     0/1     ImagePullBackOff   0          9s
meu-backend-7dffc9f5c4-wgpqr    0/1     ErrImagePull       0          8s
meu-frontend-74787cdf4f-l8lf8   0/1     ErrImagePull       0          7s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          55m
[dev2@localhost ~]$ 

^C
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
pod "meu-backend-6b5655594-bxr57" deleted
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
Error from server (NotFound): pods "meu-backend-6b5655594-bxr57" not found
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-q8dkg     0/1     ErrImagePull       0          10s
meu-backend-7dffc9f5c4-wgpqr    0/1     ImagePullBackOff   0          86s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          85s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          56m
[dev2@localhost ~]$ kubectl edit deployment meu-backend
Edit cancelled, no changes made.
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
kubectl delete pod meu-backend-7dffc9f5c4-wgpqr
Error from server (NotFound): pods "meu-backend-6b5655594-bxr57" not found
pod "meu-backend-7dffc9f5c4-wgpqr" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-q8dkg     0/1     ErrImagePull       0          65s
meu-backend-7dffc9f5c4-rpbd6    0/1     ErrImagePull       0          3s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          2m20s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          57m
[dev2@localhost ~]$ [dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
pod "meu-backend-6b5655594-bxr57" deleted
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
Error from server (NotFound): pods "meu-backend-6b5655594-bxr57" not found
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-q8dkg     0/1     ErrImagePull       0          10s
meu-backend-7dffc9f5c4-wgpqr    0/1     ImagePullBackOff   0          86s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          85s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          56m
[dev2@localhost ~]$ kubectl edit deployment meu-backend
Edit cancelled, no changes made.
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-bxr57
kubectl delete pod meu-backend-7dffc9f5c4-wgpqr
Error from server (NotFound): pods "meu-backend-6b5655594-bxr57" not found
pod "meu-backend-7dffc9f5c4-wgpqr" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-q8dkg     0/1     ErrImagePull       0          65s
meu-backend-7dffc9f5c4-rpbd6    0/1     ErrImagePull       0          3s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          2m20s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          57m
[dev2@localhost ~]$ 

^C
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-q8dkg
kubectl delete pod meu-backend-7dffc9f5c4-rpbd6
pod "meu-backend-6b5655594-q8dkg" deleted
pod "meu-backend-7dffc9f5c4-rpbd6" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-682qj     0/1     ErrImagePull       0          8s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          6s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          3m5s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          58m
[dev2@localhost ~]$ ^[[200~kubectl delete pod meu-backend-6b5655594-682qj
-bash: $'\E[200~kubectl': command not found
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-682qj
pod "meu-backend-6b5655594-682qj" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-mrdll     0/1     ImagePullBackOff   0          3s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          46s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          3m45s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          58m
[dev2@localhost ~]$ [dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-mrdll     0/1     ImagePullBackOff   0          3s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          46s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          3m45s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          58m
[dev2@localhost ~]$ 
^C
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-mrdll
             
pod "meu-backend-6b5655594-mrdll" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-gpjzn     0/1     ErrImagePull       0          4s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          107s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          4m46s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          59m
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-gpjzn
pod "meu-backend-6b5655594-gpjzn" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-69cgp     0/1     ErrImagePull       0          3s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          2m27s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          5m26s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          60m
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-69cgp
pod "meu-backend-6b5655594-69cgp" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-ghdxg     0/1     ErrImagePull       0          5s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          2m53s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          5m52s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          60m
[dev2@localhost ~]$ kubectl scale deployment meu-backend --replicas=1
deployment.apps/meu-backend scaled
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-ghdxg     0/1     ImagePullBackOff   0          41s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          3m29s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          6m28s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          61m
[dev2@localhost ~]$ kubectl describe pod meu-backend-6b5655594-ghdxg
Name:             meu-backend-6b5655594-ghdxg
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 11:06:05 -0300
Labels:           app=meu-backend
                  pod-template-hash=6b5655594
Annotations:      <none>
Status:           Pending
IP:               10.244.0.74
IPs:
  IP:           10.244.0.74
Controlled By:  ReplicaSet/meu-backend-6b5655594
Containers:
  meu-backend:
    Container ID:   
    Image:          localhost:5001/meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ErrImagePull
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      teste
      POSTGRES_PASSWORD:  teste
      POSTGRES_DB:        meubanco
      POSTGRES_HOST:      postgresql-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fngnh (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-fngnh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  62s                default-scheduler  Successfully assigned default/meu-backend-6b5655594-ghdxg to minikube
  Normal   Pulling    22s (x3 over 61s)  kubelet            Pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed     22s (x3 over 61s)  kubelet            Failed to pull image "localhost:5001/meu-backend:latest": Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
  Warning  Failed     22s (x3 over 61s)  kubelet            Error: ErrImagePull
  Normal   BackOff    10s (x3 over 61s)  kubelet            Back-off pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed     10s (x3 over 61s)  kubelet            Error: ImagePullBackOff
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-ghdxg     0/1     ErrImagePull       0          112s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          4m40s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          7m39s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          62m
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-ghdxg
pod "meu-backend-6b5655594-ghdxg" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-mgnwg     0/1     ErrImagePull       0          2s
meu-backend-7dffc9f5c4-pfngl    0/1     ImagePullBackOff   0          4m50s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          7m49s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          62m
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-mgnwg
kubectl delete pod meu-backend-7dffc9f5c4-pfngl
pod "meu-backend-6b5655594-mgnwg" deleted
pod "meu-backend-7dffc9f5c4-pfngl" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-dc9xr     0/1     ImagePullBackOff   0          5s
meu-backend-7dffc9f5c4-h2gck    0/1     ErrImagePull       0          4s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          8m45s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          63m
[dev2@localhost ~]$ kubectl delete pod meu-backend-6b5655594-dc9xr
kubectl delete pod meu-backend-7dffc9f5c4-h2gck
pod "meu-backend-6b5655594-dc9xr" deleted
pod "meu-backend-7dffc9f5c4-h2gck" deleted
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff   0          5s
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff   0          4s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff   0          10m
postgresql-9865b8f7d-rlcfl      1/1     Running            0          65m
[dev2@localhost ~]$ kubectl describe pod meu-backend-6b5655594-x46dh

Name:             meu-backend-6b5655594-x46dh
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 11:10:24 -0300
Labels:           app=meu-backend
                  pod-template-hash=6b5655594
Annotations:      <none>
Status:           Pending
IP:               10.244.0.78
IPs:
  IP:           10.244.0.78
Controlled By:  ReplicaSet/meu-backend-6b5655594
Containers:
  meu-backend:
    Container ID:   
    Image:          localhost:5001/meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      teste
      POSTGRES_PASSWORD:  teste
      POSTGRES_DB:        meubanco
      POSTGRES_HOST:      postgresql-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vzjdw (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-vzjdw:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  2m5s                default-scheduler  Successfully assigned default/meu-backend-6b5655594-x46dh to minikube
  Normal   Pulling    35s (x4 over 2m5s)  kubelet            Pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed     35s (x4 over 2m5s)  kubelet            Failed to pull image "localhost:5001/meu-backend:latest": Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
  Warning  Failed     35s (x4 over 2m5s)  kubelet            Error: ErrImagePull
  Normal   BackOff    8s (x8 over 2m4s)   kubelet            Back-off pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed     8s (x8 over 2m4s)   kubelet            Error: ImagePullBackOff
[dev2@localhost ~]$ kubectl describe pod meu-frontend-74787cdf4f-l8lf8
Name:             meu-frontend-74787cdf4f-l8lf8
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 11:00:18 -0300
Labels:           app=meu-frontend
                  pod-template-hash=74787cdf4f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.65
IPs:
  IP:           10.244.0.65
Controlled By:  ReplicaSet/meu-frontend-74787cdf4f
Containers:
  meu-frontend:
    Container ID:   
    Image:          localhost:5001/meu-frontend:latest
    Image ID:       
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bffjh (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-bffjh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  12m                  default-scheduler  Successfully assigned default/meu-frontend-74787cdf4f-l8lf8 to minikube
  Normal   Pulling    9m25s (x5 over 12m)  kubelet            Pulling image "localhost:5001/meu-frontend:latest"
  Warning  Failed     9m25s (x5 over 12m)  kubelet            Failed to pull image "localhost:5001/meu-frontend:latest": Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
  Warning  Failed     9m25s (x5 over 12m)  kubelet            Error: ErrImagePull
  Normal   BackOff    2m6s (x43 over 12m)  kubelet            Back-off pulling image "localhost:5001/meu-frontend:latest"
  Warning  Failed     2m6s (x43 over 12m)  kubelet            Error: ImagePullBackOff
[dev2@localhost ~]$ docker run -d -p 5001:5000 --name registry registry:2
Unable to find image 'registry:2' locally
2: Pulling from library/registry
44cf07d57ee4: Pull complete 
bbbdd6c6894b: Pull complete 
8e82f80af0de: Pull complete 
3493bf46cdec: Pull complete 
6d464ea18732: Pull complete 
Digest: sha256:a3d8aaa63ed8681a604f1dea0aa03f100d5895b6a58ace528858a7b332415373
Status: Downloaded newer image for registry:2
40b8e0ee6ae5ee341f6fcb480fa3d02890709eee47364da50de228a90fc64371
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint registry (d7a655c5c595e2bba68829866fcbe043a02d8bbefbf99ccb6e870201857cff49): failed to bind host port for 0.0.0.0:5001:172.17.0.2:5000/tcp: address already in use

Run 'docker run --help' for more information
[dev2@localhost ~]$ kubectl run meu-backend --image=localhost:5001/meu-backend:latest --image-pull-policy=Never
kubectl run meu-frontend --image=localhost:5001/meu-frontend:latest --image-pull-policy=Never
pod/meu-backend created
pod/meu-frontend created
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS              RESTARTS   AGE
meu-backend                     0/1     ErrImageNeverPull   0          5s
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff    0          3m55s
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff    0          3m54s
meu-frontend                    0/1     ErrImageNeverPull   0          5s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff    0          14m
postgresql-9865b8f7d-rlcfl      1/1     Running             0          69m
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS              RESTARTS   AGE
meu-backend                     0/1     ErrImageNeverPull   0          6m16s
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff    0          10m
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff    0          10m
meu-frontend                    0/1     ErrImageNeverPull   0          6m16s
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff    0          20m
postgresql-9865b8f7d-rlcfl      1/1     Running             0          75m
[dev2@localhost ~]$ kubectl logs meu-backend
Error from server (BadRequest): container "meu-backend" in pod "meu-backend" is waiting to start: ErrImageNeverPull
[dev2@localhost ~]$ kubectl get pods
NAME                            READY   STATUS              RESTARTS   AGE
meu-backend                     0/1     ErrImageNeverPull   0          20m
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff    0          24m
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff    0          23m
meu-frontend                    0/1     ErrImageNeverPull   0          20m
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff    0          34m
postgresql-9865b8f7d-rlcfl      1/1     Running             0          89m
[dev2@localhost ~]$ kubectl logs meu-backend-6b5655594-x46dh
Error from server (BadRequest): container "meu-backend" in pod "meu-backend-6b5655594-x46dh" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ kubectl logs meu-backend-7dffc9f5c4-55vgf
Error from server (BadRequest): container "meu-backend" in pod "meu-backend-7dffc9f5c4-55vgf" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ kubectl logs meu-frontend-74787cdf4f-l8lf8
Error from server (BadRequest): container "meu-frontend" in pod "meu-frontend-74787cdf4f-l8lf8" is waiting to start: trying and failing to pull image
[dev2@localhost ~]$ eval $(minikube docker-env)
[dev2@localhost ~]$ docker pull localhost:5001/meu-backend:latest
Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
[dev2@localhost ~]$ kubectl get pods --watch
NAME                            READY   STATUS              RESTARTS   AGE
meu-backend                     0/1     ErrImageNeverPull   0          24m
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff    0          28m
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff    0          28m
meu-frontend                    0/1     ErrImageNeverPull   0          24m
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff    0          38m
postgresql-9865b8f7d-rlcfl      1/1     Running             0          93m
cd
^C[dev2@localhost ~]ls
devops-microservices-demo
[dev2@localhost ~]$ cd devops-microservices-demo/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ vim backend-deployment.yaml 
[dev2@localhost kubernetes]$ docke images
-bash: docke: command not found
[dev2@localhost kubernetes]$ docker images
REPOSITORY                                TAG        IMAGE ID       CREATED         SIZE
meu-backend                               latest     e30e4bd204a3   11 hours ago    185MB
meu-frontend                              latest     899327028402   11 hours ago    153MB
postgres                                  latest     76e3e031d245   2 weeks ago     438MB
registry.k8s.io/kube-apiserver            v1.32.0    c2e17b8d0f4a   3 months ago    97MB
registry.k8s.io/kube-scheduler            v1.32.0    a389e107f4ff   3 months ago    69.6MB
registry.k8s.io/kube-controller-manager   v1.32.0    8cab3d2a8bd0   3 months ago    89.7MB
registry.k8s.io/kube-proxy                v1.32.0    040f9f8aac8c   3 months ago    94MB
gcr.io/k8s-minikube/kube-registry-proxy   <none>     9df718d81010   5 months ago    51.3MB
registry.k8s.io/etcd                      3.5.16-0   a9e7e6b294ba   6 months ago    150MB
registry.k8s.io/coredns/coredns           v1.11.3    c69fa2e9cbf5   7 months ago    61.8MB
registry.k8s.io/pause                     3.10       873ed7510279   9 months ago    736kB
registry                                  <none>     c18a86d35e98   17 months ago   25.4MB
gcr.io/k8s-minikube/storage-provisioner   v5         6e38f40d628d   3 years ago     31.5MB
[dev2@localhost kubernetes]$ vim backend-deployment.yaml 
[dev2@localhost kubernetes]$ docker build -t localhost:5001/meu-backend:latest .
docker build -t localhost:5001/meu-frontend:latest .
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.1s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[+] Building 0.4s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.1s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost kubernetes]$ eval $(minikube docker-env)
[dev2@localhost kubernetes]$ docker build -t localhost:5001/meu-backend:latest~
ERROR: docker: 'docker buildx build' requires 1 argument

Usage:  docker buildx build [OPTIONS] PATH | URL | -

Run 'docker buildx build --help' for more information
[dev2@localhost kubernetes]$ docker build -t localhost:5001/meu-backend:latest
ERROR: docker: 'docker buildx build' requires 1 argument

Usage:  docker buildx build [OPTIONS] PATH | URL | -

Run 'docker buildx build --help' for more information
[dev2@localhost kubernetes]$ docker build -t localhost:5001/meu-backend:latest .
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                    0.1s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/Docker/api/
[dev2@localhost api]$ docker build -t localhost:5001/meu-backend:latest
ERROR: docker: 'docker buildx build' requires 1 argument

Usage:  docker buildx build [OPTIONS] PATH | URL | -

Run 'docker buildx build --help' for more information
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ docker build -t localhost:5001/meu-backend:latest -f Dockerfile.api .
[+] Building 2.2s (7/8)                                                                                                                                                                       docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.1s
 => => transferring dockerfile: 232B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      1.5s
 => [internal] load .dockerignore                                                                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => CACHED [2/4] WORKDIR /app                                                                                                                                                                           0.0s
 => ERROR [3/4] COPY backend_service.py .                                                                                                                                                               0.0s
------
 > [3/4] COPY backend_service.py .:
------
Dockerfile.api:5
--------------------
   3 |     WORKDIR /app
   4 |     
   5 | >>> COPY backend_service.py .
   6 |     
   7 |     RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref c4b8adb6-bb0d-483d-ae6f-b1bd7aad903c::gi9ngj54rc0i4s6iebuy9wxyb: "/backend_service.py": not found
[dev2@localhost api]$ docker build -t localhost:5001/meu-backend:latest -f Dockerfile.api .
[+] Building 1.2s (7/8)                                                                                                                                                                       docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.1s
 => => transferring dockerfile: 232B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      0.6s
 => [internal] load .dockerignore                                                                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => CACHED [2/4] WORKDIR /app                                                                                                                                                                           0.0s
 => ERROR [3/4] COPY backend_service.py .                                                                                                                                                               0.0s
------
 > [3/4] COPY backend_service.py .:
------
Dockerfile.api:5
--------------------
   3 |     WORKDIR /app
   4 |     
   5 | >>> COPY backend_service.py .
   6 |     
   7 |     RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref c4b8adb6-bb0d-483d-ae6f-b1bd7aad903c::visjunq9o9jcyzm4salqbg076: "/backend_service.py": not found
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ vim kubernetes/
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ vim backend-service.yaml 
[dev2@localhost kubernetes]$ vim backend-deployment.yaml 
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ çs
-bash: çs: command not found
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ docker build -t localhost:5001/meu-backend:latest -f Dockerfile.api .
[+] Building 0.3s (1/1) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.0s
 => => transferring dockerfile: 2B                                                                                                                                                                      0.0s
ERROR: failed to solve: failed to read dockerfile: open Dockerfile.api: no such file or directory
[dev2@localhost Backend]$ docker build -t localhost:5001/meu-backend:latest -f Docker/api/Dockerfile.api .
[+] Building 1.3s (9/9) FINISHED                                                                                                                                                              docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.1s
 => => transferring dockerfile: 232B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      0.6s
 => [internal] load .dockerignore                                                                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/4] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.1s
 => => transferring context: 40B                                                                                                                                                                        0.0s
 => CACHED [2/4] WORKDIR /app                                                                                                                                                                           0.0s
 => CACHED [3/4] COPY backend_service.py .                                                                                                                                                              0.0s
 => CACHED [4/4] RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary                                                                                                                             0.0s
 => exporting to image                                                                                                                                                                                  0.1s
 => => exporting layers                                                                                                                                                                                 0.0s
 => => writing image sha256:e30e4bd204a30c4966663f7d5d6049b17e782f1a00623ab2a17e3e2f71f9f4dd                                                                                                            0.0s
 => => naming to localhost:5001/meu-backend:latest                                                                                                                                                      0.0s
[dev2@localhost Backend]$ kubectl apply -f kubernetes/backend-deployment.yaml
error: the path "kubernetes/backend-deployment.yaml" does not exist
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ kubectl apply -f kubernetes/backend-deployment.yaml
error: the path "kubernetes/backend-deployment.yaml" does not exist
[dev2@localhost kubernetes]$ kubectl apply -f backend-deployment.yaml
deployment.apps/meu-backend unchanged
[dev2@localhost kubernetes]$ kubectl get pods 
NAME                            READY   STATUS              RESTARTS   AGE
meu-backend                     1/1     Running             0          37m
meu-backend-6b5655594-x46dh     0/1     ImagePullBackOff    0          40m
meu-backend-7dffc9f5c4-55vgf    0/1     ImagePullBackOff    0          40m
meu-frontend                    0/1     ErrImageNeverPull   0          37m
meu-frontend-74787cdf4f-l8lf8   0/1     ImagePullBackOff    0          50m
postgresql-9865b8f7d-rlcfl      1/1     Running             0          106m
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ vim backend_service.py 
[dev2@localhost Backend]$ vim backend_service.py 
[dev2@localhost Backend]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ef:6e:d7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.108/24 brd 10.0.0.255 scope global dynamic noprefixroute enp0s3
       valid_lft 37854sec preferred_lft 37854sec
    inet6 fe80::a00:27ff:feef:6ed7/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether fa:ee:43:bd:62:ef brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::f8ee:43ff:febd:62ef/64 scope link 
       valid_lft forever preferred_lft forever
4: br-77625c8aafe0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 06:a4:d3:22:b6:a0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.49.1/24 brd 192.168.49.255 scope global br-77625c8aafe0
       valid_lft forever preferred_lft forever
    inet6 fe80::4a4:d3ff:fe22:b6a0/64 scope link 
       valid_lft forever preferred_lft forever
10: br-2302361ec6d8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether b2:11:7f:9c:be:24 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-2302361ec6d8
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::1/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::b011:7fff:fe9c:be24/64 scope link 
       valid_lft forever preferred_lft forever
11: veth9565e75@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-2302361ec6d8 state UP group default 
    link/ether 3e:de:7b:7c:39:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::3cde:7bff:fe7c:3903/64 scope link 
       valid_lft forever preferred_lft forever
17: veth47ad9e3@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-77625c8aafe0 state UP group default 
    link/ether d2:2d:ff:f1:cf:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::d02d:ffff:fef1:cf63/64 scope link 
       valid_lft forever preferred_lft forever
[dev2@localhost Backend]$ kubectl logs meu-backend
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:3030 (Press CTRL+C to quit)
[dev2@localhost Backend]$ kubectl port-forward pod/meu-backend 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
^C[dev2@localhost Backend]sudo firewall-cmd --list-allll
[sudo] password for dev2: 
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8080/tcp 8000/tcp 3030/tcp 5432/tcp 6443/tcp 10250/tcp 10251/tcp 10252/tcp 2379-2380/tcp 30000-32767/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[dev2@localhost Backend]$ kubectl get services
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
backend              ClusterIP   10.101.209.191   <none>        8080/TCP   11h
backend-service      ClusterIP   10.100.239.177   <none>        8080/TCP   11h
frontend             ClusterIP   10.107.44.137    <none>        8000/TCP   11h
frontend-service     ClusterIP   10.101.46.111    <none>        80/TCP     11h
kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP    13h
my-postgresql        ClusterIP   None             <none>        5432/TCP   3h43m
postgresql-service   ClusterIP   None             <none>        5432/TCP   112m
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ vim backend_service.py 
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ cat backend-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meu-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: meu-backend
  template:
    metadata:
      labels:
        app: meu-backend
    spec:
      containers:
      - name: meu-backend
        image: localhost:5001/meu-backend:latest
        env:
        - name: POSTGRES_USER
          value: "teste"
        - name: POSTGRES_PASSWORD
          value: "teste"
        - name: POSTGRES_DB
          value: "meubanco"
        - name: POSTGRES_HOST
          value: "postgresql-service"
        - name: POSTGRES_PORT
          value: "5432"
        ports:
        - containerPort: 8080  # Porta onde o backend vai expor a aplicação

[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Backend/Docker/api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cat Dockerfile.api 
FROM python:3.9-slim

WORKDIR /app

COPY backend_service.py .

RUN pip install fastapi uvicorn sqlalchemy psycopg2-binary

EXPOSE 8080 3030

CMD ["python", "backend_service.py"]
[dev2@localhost api]$ curl http://backend-service:3030/v1/health
curl: (6) Could not resolve host: backend-service
[dev2@localhost api]$ curl http://backend-service:8080/v1/health
curl: (6) Could not resolve host: backend-service
[dev2@localhost api]$ curl http://10.100.239.177:8080/v1/health
^C
[dev2@localhost api]$ curl http://10.100.239.177:3030/v1/health
^C
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ ls
api
[dev2@localhost Docker]$ cd ..
[dev2@localhost Backend]$ ls
backend_service.py  Docker
[dev2@localhost Backend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ cd Frontend/
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ docker build -t localhost:5001/meu-frontend:latest -f Docker/api/Dockerfile.api .
[+] Building 1.9s (10/10) FINISHED                                                                                                                                                            docker:default
 => [internal] load build definition from Dockerfile.api                                                                                                                                                0.1s
 => => transferring dockerfile: 254B                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                                                      1.3s
 => [internal] load .dockerignore                                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                                         0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:e52ca5f579cc58fed41efcbb55a0ed5dccf6c7a156cba76acfb4ab42fc19dd00                                                                                0.0s
 => [internal] load build context                                                                                                                                                                       0.1s
 => => transferring context: 110B                                                                                                                                                                       0.0s
 => CACHED [2/5] WORKDIR /app                                                                                                                                                                           0.0s
 => CACHED [3/5] COPY frontend_service.py .                                                                                                                                                             0.0s
 => CACHED [4/5] COPY ../templates/ templates/                                                                                                                                                          0.0s
 => CACHED [5/5] RUN pip install fastapi uvicorn requests jinja2                                                                                                                                        0.0s
 => exporting to image                                                                                                                                                                                  0.0s
 => => exporting layers                                                                                                                                                                                 0.0s
 => => writing image sha256:899327028402bf28057dc926f512a69da5b9272f105f150b5545d1fa55109065                                                                                                            0.0s
 => => naming to localhost:5001/meu-frontend:latest                                                                                                                                                     0.0s
[dev2@localhost Frontend]$ kubectl delete pod meu-backend-6b5655594-x46dh
kubectl delete pod meu-backend-7dffc9f5c4-55vgf
kubectl delete pod meu-frontend-74787cdf4f-l8lf8
pod "meu-backend-6b5655594-x46dh" deleted
pod "meu-backend-7dffc9f5c4-55vgf" deleted
pod "meu-frontend-74787cdf4f-l8lf8" deleted
[dev2@localhost Frontend]$ kubectl delete pod meu-backend-6b5655594-x46dh
kubectl delete pod meu-backend-7dffc9f5c4-55vgf
kubectl delete pod meu-frontend-74787cdf4f-l8lf8
Error from server (NotFound): pods "meu-backend-6b5655594-x46dh" not found
Error from server (NotFound): pods "meu-backend-7dffc9f5c4-55vgf" not found
Error from server (NotFound): pods "meu-frontend-74787cdf4f-l8lf8" not found
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          70m
meu-backend-6b5655594-wv9xl     0/1     ImagePullBackOff   0          13s
meu-backend-7dffc9f5c4-r6t2z    0/1     ImagePullBackOff   0          11s
meu-frontend                    1/1     Running            0          70m
meu-frontend-74787cdf4f-sgcjl   0/1     ImagePullBackOff   0          10s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          139m
[dev2@localhost Frontend]$ kubectl get pods | grep "ImagePullBackOff" | awk '{print $1}' | xargs kubectl delete pod
pod "meu-backend-6b5655594-wv9xl" deleted
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          76m
meu-backend-6b5655594-lsdbw     0/1     ImagePullBackOff   0          4s
meu-backend-7dffc9f5c4-r6t2z    0/1     ErrImagePull       0          6m6s
meu-frontend                    1/1     Running            0          76m
meu-frontend-74787cdf4f-sgcjl   0/1     ErrImagePull       0          6m5s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          145m
[dev2@localhost Frontend]$ kubectl delete pod meu-backend-6b5655594-wv9xl
kubectl delete pod meu-backend-7dffc9f5c4-r6t2z
kubectl delete pod meu-frontend-74787cdf4f-sgcjl
Error from server (NotFound): pods "meu-backend-6b5655594-wv9xl" not found
pod "meu-backend-7dffc9f5c4-r6t2z" deleted
pod "meu-frontend-74787cdf4f-sgcjl" deleted
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          76m
meu-backend-6b5655594-lsdbw     0/1     ImagePullBackOff   0          19s
meu-backend-7dffc9f5c4-lz4ch    0/1     ImagePullBackOff   0          5s
meu-frontend                    1/1     Running            0          76m
meu-frontend-74787cdf4f-dqbsq   0/1     ImagePullBackOff   0          4s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          145m
[dev2@localhost Frontend]$ kubectl delete pod meu-frontend-74787cdf4f-dqbsq
pod "meu-frontend-74787cdf4f-dqbsq" deleted
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          77m
meu-backend-6b5655594-lsdbw     0/1     ImagePullBackOff   0          59s
meu-backend-7dffc9f5c4-lz4ch    0/1     ImagePullBackOff   0          45s
meu-frontend                    1/1     Running            0          77m
meu-frontend-74787cdf4f-8wbqv   0/1     ErrImagePull       0          3s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          146m
[dev2@localhost Frontend]$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
meu-backend    0/1     1            0           138m
meu-frontend   0/1     1            0           137m
postgresql     1/1     1            1           147m
[dev2@localhost Frontend]$ kubectl get replicasets
NAME                      DESIRED   CURRENT   READY   AGE
meu-backend-6b5655594     1         1         0       127m
meu-backend-7dffc9f5c4    1         1         0       138m
meu-frontend-74787cdf4f   1         1         0       137m
postgresql-9865b8f7d      1         1         1       147m
[dev2@localhost Frontend]$ kubectl delete replicasets meu-backend-6b5655594
kubectl delete replicasets meu-backend-7dffc9f5c4
kubectl delete replicasets meu-frontend-74787cdf4f
replicaset.apps "meu-backend-6b5655594" deleted
replicaset.apps "meu-backend-7dffc9f5c4" deleted
replicaset.apps "meu-frontend-74787cdf4f" deleted
[dev2@localhost Frontend]$ kubectl delete replicasets meu-backend-6b5655594
kubectl delete replicasets meu-backend-7dffc9f5c4
kubectl delete replicasets meu-frontend-74787cdf4f
replicaset.apps "meu-backend-6b5655594" deleted
Error from server (NotFound): replicasets.apps "meu-backend-7dffc9f5c4" not found
replicaset.apps "meu-frontend-74787cdf4f" deleted
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          79m
meu-backend-6b5655594-d7lmh     0/1     ErrImagePull       0          4s
meu-frontend                    1/1     Running            0          79m
meu-frontend-74787cdf4f-dgkrc   0/1     ImagePullBackOff   0          4s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          148m
[dev2@localhost Frontend]$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
meu-backend    0/1     1            0           139m
meu-frontend   0/1     1            0           138m
postgresql     1/1     1            1           148m
[dev2@localhost Frontend]$ kubectl describe pod meu-backend-6b5655594-d7lmh
kubectl describe pod meu-frontend-74787cdf4f-dgkrc
Name:             meu-backend-6b5655594-d7lmh
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 12:33:31 -0300
Labels:           app=meu-backend
                  pod-template-hash=6b5655594
Annotations:      <none>
Status:           Pending
IP:               10.244.0.97
IPs:
  IP:           10.244.0.97
Controlled By:  ReplicaSet/meu-backend-6b5655594
Containers:
  meu-backend:
    Container ID:   
    Image:          localhost:5001/meu-backend:latest
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:
      POSTGRES_USER:      teste
      POSTGRES_PASSWORD:  teste
      POSTGRES_DB:        meubanco
      POSTGRES_HOST:      postgresql-service
      POSTGRES_PORT:      5432
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-m42rc (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-m42rc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason          Age                From               Message
  ----     ------          ----               ----               -------
  Normal   Scheduled       61s                default-scheduler  Successfully assigned default/meu-backend-6b5655594-d7lmh to minikube
  Normal   SandboxChanged  57s (x2 over 59s)  kubelet            Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff         29s (x4 over 58s)  kubelet            Back-off pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed          29s (x4 over 58s)  kubelet            Error: ImagePullBackOff
  Normal   Pulling         14s (x3 over 61s)  kubelet            Pulling image "localhost:5001/meu-backend:latest"
  Warning  Failed          14s (x3 over 60s)  kubelet            Failed to pull image "localhost:5001/meu-backend:latest": Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
  Warning  Failed          14s (x3 over 60s)  kubelet            Error: ErrImagePull
Name:             meu-frontend-74787cdf4f-dgkrc
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 12:33:31 -0300
Labels:           app=meu-frontend
                  pod-template-hash=74787cdf4f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.95
IPs:
  IP:           10.244.0.95
Controlled By:  ReplicaSet/meu-frontend-74787cdf4f
Containers:
  meu-frontend:
    Container ID:   
    Image:          localhost:5001/meu-frontend:latest
    Image ID:       
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ErrImagePull
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-6lj96 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-6lj96:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  61s                default-scheduler  Successfully assigned default/meu-frontend-74787cdf4f-dgkrc to minikube
  Normal   Pulling    17s (x3 over 60s)  kubelet            Pulling image "localhost:5001/meu-frontend:latest"
  Warning  Failed     17s (x3 over 60s)  kubelet            Failed to pull image "localhost:5001/meu-frontend:latest": Error response from daemon: Get "http://localhost:5001/v2/": dial tcp [::1]:5001: connect: connection refused
  Warning  Failed     17s (x3 over 60s)  kubelet            Error: ErrImagePull
  Normal   BackOff    5s (x4 over 59s)   kubelet            Back-off pulling image "localhost:5001/meu-frontend:latest"
  Warning  Failed     5s (x4 over 59s)   kubelet            Error: ImagePullBackOff
[dev2@localhost Frontend]$ kubectl delete pod meu-frontend-74787cdf4f-dgkrc
pod "meu-frontend-74787cdf4f-dgkrc" deleted
[dev2@localhost Frontend]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          80m
meu-backend-6b5655594-d7lmh     0/1     ImagePullBackOff   0          78s
meu-frontend                    1/1     Running            0          80m
meu-frontend-74787cdf4f-nnj42   0/1     ErrImagePull       0          4s
postgresql-9865b8f7d-rlcfl      1/1     Running            0          149m
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd Docker/api/
[dev2@localhost api]$ ls
Dockerfile.api
[dev2@localhost api]$ cd ..
[dev2@localhost Docker]$ cd ..
[dev2@localhost Frontend]$ ls
Docker  frontend_service.py  templates
[dev2@localhost Frontend]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ vim kubernetes/
[dev2@localhost devops-microservices-demo]$ cd kubernetes/
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ vim frontend-deployment.yaml 
[dev2@localhost kubernetes]$ ls
backend-deployment.yaml  backend-service.yaml  frontend-deployment.yaml  frontend-service.yaml  postgresql-deployment.yaml  postgresql-pvc.yaml  postgresql-service.yaml
[dev2@localhost kubernetes]$ cd ..
[dev2@localhost devops-microservices-demo]$ ls
Backend  Frontend  kubernetes
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --list-all
[sudo] password for dev2: 
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8080/tcp 8000/tcp 3030/tcp 5432/tcp 6443/tcp 10250/tcp 10251/tcp 10252/tcp 2379-2380/tcp 30000-32767/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
success
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8080/tcp 8000/tcp 3030/tcp 5432/tcp 6443/tcp 10250/tcp 10251/tcp 10252/tcp 2379-2380/tcp 30000-32767/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8080/tcp 8000/tcp 3030/tcp 5432/tcp 6443/tcp 10250/tcp 10251/tcp 10252/tcp 2379-2380/tcp 30000-32767/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --reload
success
[dev2@localhost devops-microservices-demo]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 8080/tcp 8000/tcp 3030/tcp 5432/tcp 6443/tcp 10250/tcp 10251/tcp 10252/tcp 2379-2380/tcp 30000-32767/tcp 80/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[dev2@localhost devops-microservices-demo]$ minikube status:

Error: unknown command "status:" for "minikube"

Did you mean this?
	status

Run 'minikube --help' for usage.
[dev2@localhost devops-microservices-demo]$ 
[dev2@localhost devops-microservices-demo]$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
docker-env: in-use

[dev2@localhost devops-microservices-demo]$ minikube ip
192.168.49.2
[dev2@localhost devops-microservices-demo]$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
meu-backend                     1/1     Running            0          99m
meu-backend-6b5655594-d7lmh     0/1     ImagePullBackOff   0          19m
meu-frontend                    1/1     Running            0          99m
meu-frontend-74787cdf4f-nnj42   0/1     ImagePullBackOff   0          18m
postgresql-9865b8f7d-rlcfl      1/1     Running            0          168m
[dev2@localhost devops-microservices-demo]$ kubectl port-forward pod/meu-backend 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
^C[dev2@localhost devops-microservices-demo]$ ^C
[dev2@localhost devops-microservices-demo]$ kubectl exec -it meu-frontend -- curl http://meu-backend:8080
OCI runtime exec failed: exec failed: unable to start container process: exec: "curl": executable file not found in $PATH: unknown
command terminated with exit code 126
[dev2@localhost devops-microservices-demo]$ kubectl logs meu-backend
kubectl logs meu-frontend
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:3030 (Press CTRL+C to quit)
INFO:     Started server process [1]
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:3030 (Press CTRL+C to quit)
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
[dev2@localhost devops-microservices-demo]$ kubectl logs meu-backend
kubectl logs meu-frontend
^C
[dev2@localhost devops-microservices-demo]$ kubectl exec -it meu-frontend -- bash
apt-get update && apt-get install curl                                 
root@meu-frontend:/app# kubectl exec -it meu-frontend -- wget -qO- http://meu-backend:8080
bash: kubectl: command not found
root@meu-frontend:/app# ^C
root@meu-frontend:/app# exit
exit
command terminated with exit code 130
-bash: apt-get: command not found
[dev2@localhost devops-microservices-demo]$ kubectl exec -it meu-frontend -- wget -qO- http://meu-backend:8080
OCI runtime exec failed: exec failed: unable to start container process: exec: "wget": executable file not found in $PATH: unknown
command terminated with exit code 126
[dev2@localhost devops-microservices-demo]$ kubectl describe pod meu-frontend
Name:             meu-frontend
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 19 Mar 2025 11:14:14 -0300
Labels:           run=meu-frontend
Annotations:      <none>
Status:           Running
IP:               10.244.0.81
IPs:
  IP:  10.244.0.81
Containers:
  meu-frontend:
    Container ID:   docker://29f53a649dd4ec211add74ed803078fbabe1b8d68a14d0a059cc6f8091f5d2e3
    Image:          localhost:5001/meu-frontend:latest
    Image ID:       docker://sha256:899327028402bf28057dc926f512a69da5b9272f105f150b5545d1fa55109065
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 19 Mar 2025 12:24:25 -0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kh8r5 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-kh8r5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason             Age                   From     Message
  ----     ------             ----                  ----     -------
  Warning  ErrImageNeverPull  39m (x308 over 104m)  kubelet  Container image "localhost:5001/meu-frontend:latest" is not present with pull policy of Never
  Normal   Pulled             34m                   kubelet  Container image "localhost:5001/meu-frontend:latest" already present on machine
  Normal   Created            34m                   kubelet  Created container: meu-frontend
  Normal   Started            34m                   kubelet  Started container meu-frontend
[dev2@localhost devops-microservices-demo]$ kubectl port-forward pod/meu-frontend 8081:8080
Forwarding from 127.0.0.1:8081 -> 8080
Forwarding from [::1]:8081 -> 8080

