Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"
    # config.vm.disk :disk, size: "50GB"
    config.vm.provision :docker
    config.vm.network "private_network", ip: "192.168.56.11"
    config.vm.synced_folder './', '/vagrant', type: 'rsync'
    config.ssh.extra_args = ["-t", "cd /home/vagrant/; bash --login"]
    config.vm.provider "virtualbox" do |v|
      v.memory = 8192
      v.cpus = 2
    end
  
    # Mostly copied from .github/workflows/gotests.yml to install dependencies
    config.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install -y build-essential clang conntrack libcap-dev libelf-dev net-tools docker-compose curl apt-transport-https golang gnupg2 ca-certificates lsb-release software-properties-common syslog-ng
        git clone https://github.com/mikkel0x/ebpf-sandbox.git
        
        # Install kind
        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64
        chmod +x ./kind
        mv ./kind /usr/local/bin/
        
        # Install k8s
        snap install kubectl --classic
        snap install kubelet --classic
        snap install kubeadm --classic
        
        # Install helm
        curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
        chmod 700 get_helm.sh
        ./get_helm.sh
        
        # Install unzip
        apt install unzip
        
        # Install portainer
        docker volume create portainer_data
        docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
        
        # Install ELK and Filebeat
        git clone https://github.com/deviantony/docker-elk.git
        docker-compose -f docker-elk/docker-compose.yml up -d
        wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
        echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
        curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.2.2-linux-x86_64.tar.gz
        tar xzvf filebeat-8.2.2-linux-x86_64.tar.gz
        sudo ./filebeat-8.2.2-linux-x86_64/filebeat -c /configs/filebeat.yml -e
        
        # Install Tetragon
        echo "[$(date)] Starting kind-start" > /var/log/kind-start.log
        kind create cluster --config "./ebpf-sandbox/config/nocni_1worker.yaml"
        echo "[$(date)] kind cluster created" >> /var/log/kind-start.log
        kubectl apply -f ./ebpf-sandbox/config/tetragon.yml
        wait $!
        kubectl apply -f ./ebpf-sandbox/config/sys-write-etc-kubernetes-manifests.yaml

        # Install vulnerable clusters
        # Kube-goat
        kind create cluster --name=kube-goat --config=./config/clusters/kube-goat.yaml

        #BishopFox badPods
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/everything-allowed/pod/everything-allowed-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/priv-and-hostpid/pod/priv-and-hostpid-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/priv/pod/priv-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/hostpath/pod/hostpath-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/hostpid/pod/hostpid-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/hostnetwork/pod/hostnetwork-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/hostipc/pod/hostipc-exec-pod.yaml
        kubectl apply -f https://raw.githubusercontent.com/BishopFox/badPods/main/manifests/nothing-allowed/pod/nothing-allowed-exec-pod.yaml 
       
        # Privledged Pod from CCNF
        kubectl apply -f https://gist.githubusercontent.com/orkamara/ea5e1d317e733744315c439eb2ab7b33/raw/2227a674bc517f2ff2632ea23814c5cfbd74fa1d/privileged-nginx-deployment.yaml
        kubectl expose deployment nginx-deployment --type=NodePort --name=nginx-service

        # Isovalent Privleged Pod
        kubectl apply -f isovalent-privileged.yaml
    SHELL
  end