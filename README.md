## Instructions

### Build
1. Clone the gpins repo:
```
git clone https://sonic-preview.googlesource.com/alpine/sonic-buildimage
```
2. Init
```
export NOJESSIE=1 NOSTRETCH=1 NOBUSTER=1
make init
```
3. Configure
```
PLATFORM=alpinevs make configure
```
4. Build
```
make target/sonic-alpinevs.img.gz
```
5. Build alpinevs container
```
./alpine/build_alpinevs_container.sh
```
### Deploy

Pre-req:
A KVM enabled workstation (or VM) which can support VMs on it

1. Setup KNE cluster
```
kne deploy deploy/kne/kind-bridge.yaml
```
2. Load alpinevs container image in kne:
```
kind load docker-image alpine-vs:latest --name kne
```
3. Create 2 switch Alpine topology:
```
kne create twodut-alpine.pb.txt
```

4. Terminals

- [Terminal1] SSH to the AlpineVS DUT Switch VM inside the deployment:
```
ssh-keygen -f /tmp/id_rsa -N ""
Set IPDUT var to the EXTERNAL-IP of "kubectl get svc -n twodut-alpine service-alpine-dut"
ssh-copy-id -i /tmp/id_rsa.pub -oProxyCommand=none admin@$IPDUT
ssh -i /tmp/id_rsa -oProxyCommand=none admin@$IPDUT
```
- [Terminal2] SSH to the AlpineVS Control Switch VM inside the deployment:
```
ssh-keygen -f /tmp/id_rsa -N ""
Set IPCTL var to the EXTERNAL-IP of "kubectl get svc -n twodut-alpine service-alpine-ctl"
ssh-copy-id -i /tmp/id_rsa.pub -oProxyCommand=none admin@$IPCTL
ssh -i /tmp/id_rsa -oProxyCommand=none admin@$IPCTL
```

### Test
1. Copy `alpine_setup.sh` to `DUT` and run it
```
scp -i /tmp/id_rsa -oProxyCommand=none alpine_setup.sh admin@$IPDUT:~/
ssh -i /tmp/id_rsa -oProxyCommand=none admin@$IPDUT
sudo su
./alpine_setup.sh
```
2. Copy `alpine_setup.sh` to `CTL` and run it
```
scp -i /tmp/id_rsa -oProxyCommand=none alpine_setup.sh admin@$IPCTL:~/
ssh -i /tmp/id_rsa -oProxyCommand=none admin@$IPCTL
sudo su
./alpine_setup.sh
```
3. Copy `sai_entries` to `DUT` and run the following from p4rt container:
```
scp -i /tmp/id_rsa -oProxyCommand=none sai_entries admin@$IP:~/
docker exec -it p4rt /bin/bash
p4rt_program_table -cleanup=false -input_file=/tmp/sai_entries -push_config=true
```
4. From control switch (IPCTL) run scapy:
```
scapy
> packet=Ether(dst="02:fe:37:a0:13:c2",src="00:00:00:00:03:04", type=0x86dd)/IPv6(src="ac:bc::",dst="aa:bb::")/TCP(sport=66,dport=680)/("Alpine test.");
> sendp(packet,iface="Ethernet24",count=10)
```
5. Check counters on both side:
```
show interfaces counters -i Ethernet24
```


