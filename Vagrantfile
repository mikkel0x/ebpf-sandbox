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
        
        # Enable and start Rsyslog
        systemctl enable rsyslog
        systemctl start rsyslog
        
        # Install kind
        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-linux-amd64
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
        sudo apt-get update && sudo apt-get install filebeat
        sudo systemctl enable filebeat
        
        # Install Tetragon
        git clone https://github.com/mikkel0x/ebpf-sandbox.git
        kind create cluster --config "./ebpf-sandbox/config/nocni_1worker.yaml"
        kubectl apply -f ./ebpf-sandbox/config/tetragon.yml
        kubectl apply -f ./ebpf-sandbox/config/sys-write-etc-kubernetes-manifests.yaml
    SHELL
  end