# Azure IoT Edge Self-signed Root CA

Create the IoT Hub:

```sh
# IoT Hub
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"

# Upgrade the root to V2 if required
az iot hub certificate root-authority set --hub-name iothub789 --certificate-authority v2
```

Create the DPS:

```sh
# Create the DPS
az iot dps create -n dps789 -g IoTEdgeResources -l westus2

# Link with the IoT Hub
hubConnectionString=$(az iot hub connection-string show -n iothub789 --kt primary --query connectionString -o tsv)
az iot dps linked-hub create --dps-name dps789 --resource-group IoTEdgeResources --connection-string $hubConnectionString

# Verify
az iot dps show -n dps789
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
sudo bash certGen.sh create_edge_device_identity_certificate EdgeDevice
```

Upload the root CA certificate to DPS:

```
az iot dps certificate create -n "Test-Only-Root" --dps-name dps789 -g IoTEdgeResources -p certs/azure-iot-test-only.root.ca.cert.pem -v true
```

Create the enrollment group:

```sh
az iot dps enrollment-group create -n dps789 -g IoTEdgeResources\
    --root-ca-name "Test-Only-Root" \
    --secondary-root-ca-name "Test-Only-Root" \
    --enrollment-id "EdgeDevicesGroup" \
    --provisioning-status "enabled" \
    --reprovision-policy "reprovisionandmigratedata" \
    --iot-hubs "iothub789.azure-devices.net" \
    --allocation-policy "hashed" \
    --edge-enabled true \
    --tags '{ "Environment": "Staging" }' \
    --props '{ "Debug": "false" }'
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

Install the Edge runtime:

```
sudo apt-get update; \
   sudo apt-get install aziot-edge
```

Copy the certificates:

```sh
sudo mkdir /var/secrets
sudo mkdir /var/secrets/aziot
sudo mv edgedevice.full-chain.cert.pem /var/secrets/aziot/
#sudo mv edgedevice.cert.pem /var/secrets/aziot/
sudo mv edgedevice.key.pem /var/secrets/aziot/
```

Certificate directory:

```sh
# Give the IoT Edge user permission
#sudo chown -R iotedge: /tmp/iotedge
```

Create the configuration file:

```sh
sudo cp /etc/aziot/config.toml.edge.template /etc/aziot/config.toml
```

Set the configuration:

```
sudo vim /etc/aziot/config.toml
```

Edit the provisioning section. You'll need to change the `id_scope` value:

```toml
# DPS provisioning with X.509 certificate
[provisioning]
source = "dps"
global_endpoint = "https://global.azure-devices-provisioning.net"
id_scope = "SCOPE_ID_HERE"

[provisioning.attestation]
method = "x509"
registration_id = "EdgeDevice"

identity_cert = "file:///var/secrets/aziot/edgedevice.full-chain.cert.pem"

identity_pk = "file:///var/secrets/aziot/edgedevice.key.pem"
```

Apply the config:

```
sudo iotedge config apply
```

Run the verification commands:

```
sudo iotedge system status
sudo iotedge system logs
sudo iotedge check
```

Using the Portal, add a marketplace Edge module, then check again:

```
sudo iotedge list
```
