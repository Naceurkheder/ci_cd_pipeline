# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Use the same base box for all machines to save storage space
  config.vm.box = "ubuntu/focal64"

  # =========================================================
  # 1. Jenkins Server (Your Existing Machine)
  # =========================================================
  config.vm.define "default" do |jenkins|
    jenkins.vm.hostname = "jenkins-server"
    
    # Port forwarding & Private IP for internal communication
    jenkins.vm.network "forwarded_port", guest: 8080, host: 8080
    jenkins.vm.network "private_network", ip: "192.168.56.32"

    jenkins.vm.provider "virtualbox" do |vb|
      vb.linked_clone = true
      vb.memory = "2048"
      vb.cpus = 2
      vb.name = "jenkins-focal" 
    end

    # INTERNET ACTIVE: Downloads and installs Jenkins + Java automatically
    jenkins.vm.provision "shell", inline: <<-SHELL
      set -e
      apt_install() {
        local retries=3
        local count=0
        until [ $count -ge $retries ]; do
          apt-get install -y --fix-missing "$@" && return 0
          count=$((count + 1))
          echo "==> Retry $count/$retries in 10s..."
          sleep 10
        done
        echo "==> Failed after $retries attempts"
        return 1
      }

      echo "==> Refreshing package lists..."
      apt-get update -y

      echo "==> Installing Java (headless)..."
      apt_install fontconfig openjdk-21-jre-headless
      echo "==> Adding Jenkins repository..."
      curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
      echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | tee /etc/apt/sources.list.d/jenkins.list > /dev/null

      echo "==> Updating Jenkins repo..."
      apt-get update -y -o Dir::Etc::sourcelist="sources.list.d/jenkins.list" -o Dir::Etc::sourceparts="-" -o APT::Get::List-Cleanup="0"

      echo "==> Installing Jenkins..."
      apt_install jenkins

      echo "==> Starting Jenkins..."
      systemctl enable jenkins
      systemctl start jenkins
      sleep 30

      echo "============================================="
      echo "  Jenkins initial admin password:"
      cat /var/lib/jenkins/secrets/initialAdminPassword
      echo "============================================="
    SHELL
  end

  # =========================================================
  # 2. Gitea Server 
  # =========================================================
  config.vm.define "gitea" do |gitea|
    gitea.vm.hostname = "git-server"
    gitea.vm.network "forwarded_port", guest: 3000, host: 3000
    gitea.vm.network "forwarded_port", guest: 8081, host: 8081 
    gitea.vm.network "private_network", ip: "192.168.56.31"

    gitea.vm.provider "virtualbox" do |vb|
      vb.linked_clone = true  
      vb.memory = "4096"      
      vb.cpus = 2             
      vb.name = "gitea-focal" 
    end

    # INTERNET ACTIVE: Downloads the lightweight Gitea binary directly
    gitea.vm.provision "shell", inline: <<-SHELL
      set -e
      echo "==> Installing Git & wget..."
      apt-get update -y
      apt-get install -y git wget

      echo "==> Downloading Gitea..."
      wget -qO /usr/local/bin/gitea https://dl.gitea.com/gitea/1.21.11/gitea-1.21.11-linux-amd64
      chmod +x /usr/local/bin/gitea

      echo "==> Setting up Gitea users and directories..."
      useradd -m -d /home/git -s /bin/bash git
      mkdir -p /var/lib/gitea/{custom,data,log}
      chown -R git:git /var/lib/gitea/
      mkdir -p /etc/gitea
      chown root:git /etc/gitea
      chmod 770 /etc/gitea

      echo "==> Creating systemd service for Gitea..."
      cat << 'EOF' > /etc/systemd/system/gitea.service
[Unit]
Description=Gitea (Git with a cup of tea)
After=network.target

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOF

      echo "==> Starting Gitea..."
      systemctl daemon-reload
      systemctl enable gitea
      systemctl start gitea
      
      echo "============================================="
      echo "  Gitea is running! Access at http://localhost:3000"
      echo "============================================="
    SHELL
  end

  # =========================================================
  # 3. Dev Server 
  # =========================================================
  config.vm.define "dev-server" do |dev|
    dev.vm.hostname = "dev-server"
    dev.vm.network "private_network", ip: "192.168.56.11"

    dev.vm.provider "virtualbox" do |vb|
      vb.linked_clone = true  
      vb.memory = "1024"      
      vb.cpus = 1             
      vb.name = "dev-focal" 
    end
  end
end
