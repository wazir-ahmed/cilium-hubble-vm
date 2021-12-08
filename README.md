# Cilium/Hubble in Virtual Machines
Using the following instructions, you can setup Cilium in VMs and enable hubble flows.

## Dependencies
1) **A k8s cluster** -  Cilium needs a k8s cluster (it can be a single node cluster) to act as a control plane and serve information about labels and IPs of the VMs.

2) **Docker >= 20.10** should be installed in all the VMs.

## Setup Cilium on k8s cluster
Setup Cilium `clustermesh-apiserver` in the k8s cluster which will act as a control plane for the VMs.

1) Install Cilium CLI
```
./install-cilium-cil.sh
```

2) Deploy Cilium
```
cilium install --config tunnel=vxlan
```

3) Check the Cilium status. There should not be any errors.
```
cilium status
```

4) Enable Cilium `clustermesh`
```
cilium clustermesh enable
```
- If this fails indicating that `--service-type` needs to be given, add `--service-type NodePort` to the command above, i.e. `cilium clustermesh enable --service-type NodePort`.
- If it fails indicating that `cilium-ca` is not found, add `--create-ca` to the command above, i.e. `cilium clustermesh enable --create-ca`.

## Add VMs to the control plane
1) Create an entry for each VM and assign labels to them.
```
cilium clustermesh vm create <hostname> --labels=<key1=value1,key2=value2...keyN=valueN>
```
> - `hostname` - VM's hostname
> - `key1=value1,key2=value2...keyN=valueN` - Labels of the VM (similar to pod labels).

2) Repeat the above command for each VM you wish to add to the control plane.

3) Once all the VMs are added, verify it.
```
cilium clustermesh vm status
```

## Generate Cilium VM installation script
1) Generate the shell script to install Cilium in VMs.
```
cilium clustermesh vm install <file-name> --config devices=<interfaces>,enable-host-firewall,enable-hubble=true,hubble-listen-address=:4244,hubble-disable-tls=true,external-workload
```
> - `file-name` - script name (example, `cilium-vm-install.sh`)
> - `interfaces` - one or more, comma separated list of VM's physical interfaces (example - `eth0,eth1`).

2) Edit the generated script and change the value of `CILIUM_IMAGE` to `${1:-docker.io/wazirak/cilium-dev:hubble-vm}`

3) **Note:**
    - If the interface name differs from one VM to another VM, then you have to generate a script of each VM by configuring the `interface` parameter appropriately.
    - If the interface name is same for all the VMs, then you could generate the script once and use it across all the VMs.

## Install Cilium in the VMs
1) Copy the installation script to the VMs in which you wish to install Cilium and execute it in the VM's shell.


2) Once the installation is successful in the VM, check the status by executing the following command in VM's shell.
```
cilium status
```

## Hubble Flows
To get the network traffic flows between VMs, you can either use Hubble CLI or Feeder-service in the VMs. 

**Note:** Execute the following commands in VM's shell.

### Hubble CLI
1) Install Hubble CLI
```
./install-hubble-cli.sh
```

2) To check the network flow in a VM.
```
hubble observe --server=localhost:4244 -f
```
### Feeder Service
1) Install Feeder service (requires `git`, `make` and `go`)
```
git clone https://github.com/accuknox/feeder-service.git
cd feeder-service
make binary
sudo cp bin/feeder-service /usr/local/bin
cd ..
```

2) To run feeder-service in VMs
```
HUBBLE_ENABLED=true  \
  HUBBLE_URL=localhost \
  HUBBLE_PORT=4244 \
  feeder-service
```
If you want to push feeder-service events to Kafka, then use the below command and set `KAFKA_*` env variables appropriately.
```
KAFKA_ENABLED=true \
  KAFKA_URL=<kafka-ip> \
  KAFKA_PORT=<kafka-port> \
  HUBBLE_ENABLED=true  \
  HUBBLE_URL=localhost \
  HUBBLE_PORT=4244 \
  feeder-service
```
