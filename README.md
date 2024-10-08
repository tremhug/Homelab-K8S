# Homelab-OKD
This repo is a set of scripts needed to deploy an K8S cluster on homelab. The cluster will be composed of 3 Virtualbox VMs.

##Background information
I run this cluster on a refurbished HP Proliant ML350p Gen8 on which I have installed Ubuntu server 24.04.1 LTS. This server has the following description: HP Proliant ML 350p Gen8

- CPU - 2x E5-2670 @ 2.6 GHz (16 Cores)
- RAM - 128GB DDR3 ECC RAM (16x 8GB)
- Hard Drives - 2x 900GB 10K SAS Drives
  
This does not mean that the script won't run on anything else, but it has been deployed on that environment.

## Objective
The goal of this scripts is to automate the deployment of 3 VM such that one will be the K8S Master while the other two will be used as workers.

I'm sure this has been done many times. I just want to play with it and share my experience through this journey. And if it helps someone, then Yeah! :)

## Prerequesite Before being able to build the cluster, we need to validate some items are present on the host.

The host OS is Ubuntuserver 24.04.1 LTS
VirtualBox: 7.0.16
Vagrant: 2.4.1-1
Once again, I write the version number here because this is what I have tested but it will likely work with other permutation of versions.

# The scripts
In my few tryal and errors, I've ended up using 3 scripts (so far). Is it the best way to do it? Meh! It's never going to prod and it works for me. 

The first script id a Vagrantfile that allocates 3 VMs. The first one is named KubeMaster and the other 2 are named KubeWorker1 and 2.

# Vagrantfile
The content of my Vagrantfile goes like this (note: comments are in french for now. I might eventually ask Chatgpt to translate them to english eventually.
```
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"
    
    # Configuration de la première VM : `KubeMaster`
    config.vm.define "KubeMaster" do |master|
      master.vm.hostname = "KubeMaster"
      master.vm.network "public_network", bridge: "eno1", ip: "192.168.0.210", mac: "080027996558"
      master.vm.provider "virtualbox" do |vb|
        vb.name = 'KubeMaster-VM'
        vb.memory = 4096
        vb.cpus = 4
      end
      # Provisionnement du script d'installation pour le maître
      master.vm.provision :docker
      master.vm.provision "shell", path: "master_install.sh"
    end
  
    # Configuration simplifiée pour `KubeWorker1`
    config.vm.define "KubeWorker1" do |worker1|
      worker1.vm.hostname = "KubeWorker1"
      worker1.vm.network "public_network", bridge: "eno1", ip: "192.168.0.211", mac: "0800273B70C3"
      worker1.vm.provider "virtualbox" do |vb|
        vb.name = 'KubeWorker1-VM'
        vb.memory = 4096
        vb.cpus = 4
      end
      worker1.vm.provision :docker
      worker1.vm.provision "shell", path: "worker_install.sh"
    end
  
    # Configuration simplifiée pour `KubeWorker2`
    config.vm.define "KubeWorker2" do |worker2|
      worker2.vm.hostname = "KubeWorker2"
      worker2.vm.network "public_network", bridge: "eno1", ip: "192.168.0.212", mac: "080027D09E53"
      worker2.vm.provider "virtualbox" do |vb|
        vb.name = 'KubeWorker2-VM'
        vb.memory = 4096
        vb.cpus = 4
      end
      worker2.vm.provision :docker
      worker2.vm.provision "shell", path: "worker_install.sh"
    end
  end
  
```

# The master_install.sh script
Once KubeMaster is created, master_install.sh is invoked.
```
#!/bin/bash

echo "***** Mettre à jour le système"
sudo apt-get update && sudo apt-get upgrade -y

echo "***** Désactiver le swap (requis pour Kubernetes)"
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "***** Charger les modules nécessaires pour Kubernetes"
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

echo "***** Attendre un petit délai pour s'assurer que les modules sont bien chargés"
sleep 2

echo "***** Définir les paramètres sysctl requis par Kubernetes"
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

echo "***** Appliquer les paramètres sysctl"
sudo sysctl --system

echo "***** Vérifier la présence d'anciens dépôts Kubernetes et les supprimer s'ils existent"
if [ -f /etc/apt/sources.list.d/kubernetes.list ]; then
  echo "***** Ancien dépôt Kubernetes trouvé. Suppression..."
  sudo rm -f /etc/apt/sources.list.d/kubernetes.list
fi

echo "***** Créer le dossier /etc/apt/keyrings s'il n'existe pas (spécifique aux versions récentes)"
sudo mkdir -p /etc/apt/keyrings

echo "***** Supprimer la clé existante si elle est présente pour éviter les conflits"
if [ -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg ]; then
  echo "***** Clé existante trouvée. Suppression..."
  sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
fi

echo "***** Télécharger la clé de signature et ajouter le dépôt Kubernetes communautaire avec la version correcte"
wget -O Release.key https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key

echo "***** Vérifier si le fichier est bien téléchargé"
if [ $? -ne 0 ]; then
  echo "***** Erreur lors du téléchargement de la clé de signature."
  exit 1
fi

echo "***** Ajouter la clé de signature téléchargée au système"
sudo gpg --batch --no-tty --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg Release.key

echo "***** Ajouter le dépôt avec le chemin complet incluant la version"
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

echo "***** Mettre à jour les dépôts pour refléter le changement"
sudo apt-get update

echo "***** Installer kubeadm, kubelet et kubectl"
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "***** Installer et configurer containerd"
sudo apt-get install -y containerd

echo "***** Créer le fichier de configuration par défaut pour containerd"
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

echo "***** Activer le support CRI pour Kubernetes dans la configuration containerd"
sudo sed -i '/SystemdCgroup = false/c\            SystemdCgroup = true' /etc/containerd/config.toml

echo "***** Redémarrer containerd pour appliquer la configuration"
sudo systemctl restart containerd

echo "***** Vérifier le statut de containerd"
if ! sudo systemctl is-active --quiet containerd; then
    echo "Erreur : containerd n'a pas démarré correctement."
    sudo systemctl status containerd
    exit 1
fi

echo "***** Installer socat (nécessaire pour certaines fonctionnalités de Kubernetes)"
sudo apt-get install -y socat

echo "***** Initialiser le nœud maître Kubernetes"
sudo kubeadm init --apiserver-advertise-address=192.168.0.210 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU,CRI

echo "***** Configurer kubectl pour l'utilisateur vagrant"
mkdir -p /home/vagrant/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
sudo chown vagrant:vagrant /home/vagrant/.kube/config

echo "***** Configurer kubectl pour root pour s'assurer de l'accès complet"
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "***** Vérifier le statut des nœuds et des pods"
kubectl get nodes
kubectl get pods -n kube-system

echo "***** Installer un réseau de pods pour permettre la communication entre les conteneurs"
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

echo "***** Afficher la commande join pour les nœuds workers"
kubeadm token create --print-join-command


echo "***** Mise à jour du système"
sudo apt-get update && sudo apt-get upgrade -y

echo "***** Générer la clé SSH pour le maître"
sudo -u vagrant ssh-keygen -t rsa -b 4096 -f /home/vagrant/.ssh/id_rsa -q -N ""

echo "***** Ajouter la clé publique au fichier authorized_keys du maître"
cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
chmod 600 /home/vagrant/.ssh/authorized_keys

```

# The worker_install.sh script
Note that, when you execute the script, you will have to pause after KubeMaster is created. That will allow you to capture the line with the token and discovery-token-ca-cert-hash from the logs
ex: `kubeadm join 192.168.0.210:6443 --token miwhbb.or9olk0arwcbgczb --discovery-token-ca-cert-hash sha256:a8ac3f4d1042742ece12bfb5f84f0694c7cbbdc0bddfe3cec9eb4aba65068fa7`

And use it in the worker_install.sh (last line) before creating the 2 worker nodes. (I guess I could automate this too...but my scripting knowledge aren't great so this works for me.
```
#!/bin/bash

# Variables de connexion SSH
MASTER_IP="192.168.0.210"
SSH_USER="vagrant"
SSH_KEY="/home/vagrant/.ssh/id_rsa"

echo "***** Mise à jour du système"
sudo apt-get update && sudo apt-get upgrade -y

echo "***** Installer socat (nécessaire pour certaines fonctionnalités de Kubernetes)"
sudo apt-get install -y socat

echo "***** Désactiver le swap"
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "***** Charger les modules requis par Kubernetes"
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

echo "***** Configurer les paramètres sysctl requis par Kubernetes"
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

echo "***** Installer containerd"
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i '/SystemdCgroup = false/c\            SystemdCgroup = true' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

echo "***** Ajouter le dépôt Kubernetes"
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

echo "***** Installer kubeadm, kubelet et kubectl"
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


echo "***** Exécution de la commande kubeadm join sur le nœud worker *****"
kubeadm join 192.168.0.210:6443 --token miwhbb.or9olk0arwcbgczb --discovery-token-ca-cert-hash sha256:a8ac3f4d1042742ece12bfb5f84f0694c7cbbdc0bddfe3cec9eb4aba65068fa7

```

