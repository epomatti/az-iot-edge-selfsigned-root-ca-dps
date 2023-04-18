# Azure IoT Edge Self-signed Root CA

```sh
# IoT Hub
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"

# Upgrade the root to V2 if required
az iot hub certificate root-authority set --hub-name iothub789 --certificate-authority v2
```

Create the Iot Edge (Ubuntu Server):

```sh
az vm create \
  --resource-group IoTEdgeResources \
  --name vmiotedge \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --public-ip-sku Standard
```

Connect to the VM:

```sh
ssh azureuser@<public-ip>
```

Update the packages:

```sh
sudo apt update
sudo apt upgrade -y
```

Restart the VM and connect again:

```
az vm restart -n vmiotedge -g IoTEdgeResources
```

Generate the certificates:

```sh
mkdir wrkdir
cd wrkdir

# Download the raw files for testing
curl "https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/certGen.sh" --output certGen.sh
curl "https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/openssl_root_ca.cnf" --output openssl_root_ca.cnf

# Root
sudo bash certGen.sh create_root_and_intermediate

# IoT Edge
sudo bash certGen.sh create_edge_device_identity_certificate myEdgeDevice
```

After creating your certificates, upload the Root CA to the IoT Hub:

```sh
az iot hub certificate create --name "Test-Only-Root" \
    --hub-name iothub789 \
    --resource-group IoTEdgeResources \
    --path certs/azure-iot-test-only.root.ca.cert.pem \
    --verified true
```

Register the device with x509_ca authentication method:

```
az iot hub device-identity create \
    --device-id "device-01" \
    --hub-name iothub789 \
    --auth-method x509_ca \
    --edge-enabled
```

Add Microsoft packages:

```
sudo wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo rm packages-microsoft-prod.deb
```

Install Docker:

```
sudo apt-get update; \
  sudo apt-get install moby-engine
```

Create `/etc/docker/daemon.json` and add:

```json
{
  "log-driver": "local"
}
```

Then restart:

```
sudo systemctl restart docker
sudo systemctl status docker
```
