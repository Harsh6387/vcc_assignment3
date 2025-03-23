# vcc_assignment3


## 1. Objective

The goal of this project is to:

- Set up a **local Virtual Machine (VM)** using VirtualBox.
- Install and configure **Prometheus and Node Exporter** to monitor the VM’s resource usage.
- Configure **Google Cloud Platform (GCP)** to automatically scale instances when CPU usage exceeds 75%.
- Deploy a simple web application on the auto-scaled GCP instances.

---

## 2. Prerequisites

Ensure you have the following:

- **GCP account** with Compute Engine and IAM permissions.
- **VirtualBox** installed on your local machine.
- **SSH** and **gcloud CLI** installed.
- **Ubuntu ISO** file for VM installation.

---

## 3. Setting Up the Local VM

### 3.1. Install VirtualBox

If VirtualBox is not already installed, install it using the following commands:

```bash
sudo apt update
sudo apt install -y virtualbox
```

### 3.2. Create the VM

1. Open **VirtualBox**.
2. Click on **New**.
3. Configure the VM:
   - **Name:** `local-vm`
   - **Type:** Linux
   - **Version:** Ubuntu (64-bit)
   - **Memory:** 4 GB
   - **Disk Size:** 20 GB
4. Go to **Network settings → Bridged Adapter** for internet access.
5. Click **Create**.

### 3.3. Install Ubuntu on the VM

1. Download the Ubuntu ISO: [Ubuntu Downloads](https://ubuntu.com/download/desktop).
2. In VirtualBox, go to **Settings → Storage**.
3. Click on the **Empty disk icon → Select the Ubuntu ISO**.
4. Start the VM and follow the installation instructions.
5. Once installed, update the VM:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. Installing and Configuring Prometheus

### 4.1. Install Prometheus and Node Exporter

Run the following commands on the VM:

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository -y universe
sudo apt update
sudo apt install -y prometheus prometheus-node-exporter
```

### 4.2. Verify Installation

Check the status of both services:

```bash
systemctl status prometheus
systemctl status prometheus-node-exporter
```

Ensure they are active and running.

### 4.3. Configure Prometheus

Edit the configuration file:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the following:

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'local_vm'
    static_configs:
      - targets: ['localhost:9100']
```

### 4.4. Restart the Services

Apply the changes by restarting the services:

```bash
sudo systemctl restart prometheus
sudo systemctl restart prometheus-node-exporter
```

---

## 5. Accessing the Prometheus Web UI

### 5.1. Get the VM IP Address

Run the following command to get the local VM IP address:

```bash
ip a
```

### 5.2. Access the Prometheus Dashboard

Open your browser and access the Prometheus UI:

```http
http://<LOCAL_VM_IP>:9090
```

---

## 6. Deploying a Simple Web Application

### 6.1. Install Apache Web Server

Run the following commands to install Apache:

```bash
sudo apt install -y apache2
```

Create a sample web page:

```bash
echo "<h1>Local VM Web App</h1>" | sudo tee /var/www/html/index.html
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

Access the web app in your browser:

```http
http://<LOCAL_VM_IP>
```

---

## 7. GCP Configuration

### 7.1. Authenticate GCP

Log in to GCP:

```bash
gcloud auth login
```

Set your project:

```bash
gcloud config set project <YOUR_PROJECT_ID>
```

---

## 8. Creating GCP Auto-Scaling Infrastructure

### 8.1. Create a Startup Script

Create a startup script to install Apache and deploy a web app:

```bash
nano startup-script.sh
```

Add the following content:

```bash
#!/bin/bash
sudo apt update -y
sudo apt install -y apache2 stress

echo "<h1>Auto-scaled GCP Instance</h1>" > /var/www/html/index.html
sudo systemctl start apache2
```

### 8.2. Create an Instance Template

Run the following command to create an instance template:

```bash
gcloud compute instance-templates create auto-scale-template \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server \
  --metadata-from-file startup-script=./startup-script.sh
```

### 8.3. Create the Instance Group

Create a managed instance group:

```bash
gcloud compute instance-groups managed create auto-scale-group \
  --base-instance-name auto-scale-vm \
  --size 1 \
  --template auto-scale-template \
  --zone us-central1-a
```

### 8.4. Set Auto-scaling Policy

Enable auto-scaling based on CPU usage:

```bash
gcloud compute instance-groups managed set-autoscaling auto-scale-group \
  --zone us-central1-a \
  --min-num-replicas 1 \
  --max-num-replicas 5 \
  --target-cpu-utilization 0.75 \
  --cool-down-period 60
```

---

## 9. Triggering Auto-scaling

### 9.1. Simulate High CPU Usage

Install the stress testing tool:

```bash
sudo apt install -y stress-ng
```

Simulate CPU load:

```bash
stress-ng --cpu 4 --timeout 60s
```

### 9.2. Verify Auto-scaling

Check the instance group status:

```bash
gcloud compute instance-groups list
```

List the running instances:

```bash
gcloud compute instances list
```

You should see new instances being automatically created.

---

## 10. Deploying the Web App on GCP

### 10.1. Connect to GCP Instance

SSH into a GCP instance:

```bash
gcloud compute ssh <INSTANCE_NAME> --zone us-central1-a
```

### 10.2. Deploy the Web App

Run the following commands to deploy the web app:

```bash
echo "<h1>Auto-Scaled GCP Instance</h1>" | sudo tee /var/www/html/index.html
sudo systemctl restart apache2
```

The web app should now be accessible on the auto-scaled GCP instances.

