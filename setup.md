# Hálózat setup a hosztokon

Először beállítottam a node-okon az interfészeket.
Történeti okok miatt a workerek hosztnevei w2, w3 és w4

a masteren

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: yes
    enp4s0:
      addresses: [10.158.0.2/16]
      #gateway4: 10.158.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

w2-n

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: yes
    enp4s0:
      addresses: [10.158.0.3/16]
      gateway4: 10.158.0.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  vlans:
    vlan100:
      id: 100
      link: enp4s0
      addresses: [10.100.0.2/24]
      gateway4: 10.100.0.254
    vlan200:
      id: 200
      link: enp4s0
      addresses: [10.200.0.2/24]
      gateway4: 10.200.0.254
```

w3-n

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: yes
    enp4s0:
      addresses: [10.158.0.4/16]
      gateway4: 10.158.0.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  vlans:
    vlan100:
      id: 100
      link: enp4s0
      addresses: [10.100.0.3/16]
      gateway4: 10.100.0.254
    vlan200:
      id: 200
      link: enp4s0
      addresses: [10.200.0.3/16]
      gateway4: 10.200.0.254
```

w4-en

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s10:
      dhcp4: yes
    enp5s0:
      addresses: [10.158.0.5/16]
      gateway4: 10.158.0.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  vlans:
    vlan100:
      id: 100
      link: enp5s0
      addresses: [10.100.0.4/16]
      gateway4: 10.100.0.254
    vlan200:
      id: 200
      link: enp5s0
      addresses: [10.200.0.4/16]
      gateway4: 10.200.0.254
```
# Multus NetworkAttachmentDefinition-ök hozzáadása

macvlan100

```yaml
# This net-attach-def defines macvlan-conf with 
#   + ips capabilities to specify ip in pod annotation and 
#   + mac capabilities to specify mac address in pod annotation
# default gateway is defined as well
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan100
spec: 
  config: '{
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "macvlan",
          "capabilities": { "ips": true },
          "master": "vlan100",
          "mode": "bridge",
          "ipam": { 
            "type": "static",
            "routes": [
              {
                "dst": "0.0.0.0/0",
                "gw": "10.100.0.254"
              },
              {
                "dst": "192.168.100.0/24",
                "gw": "10.100.0.254"
              },
              {
                "dst": "10.158.0.0/16",
                "gw": "10.100.0.254"
              }
            ] 
          }
        }, {
          "capabilities": { "mac": true },
          "type": "tuning"
        }
      ]
    }'
```

macvlan200

```yaml
# This net-attach-def defines macvlan-conf with 
#   + ips capabilities to specify ip in pod annotation and 
#   + mac capabilities to specify mac address in pod annotation
# default gateway is defined as well
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan200
spec: 
  config: '{
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "macvlan",
          "capabilities": { "ips": true },
          "master": "vlan200",
          "mode": "bridge",
          "ipam": { 
            "type": "static"
          }
        }, {
          "capabilities": { "mac": true },
          "type": "tuning"
        }
      ]
    }'
```

# Frontendek installálása

Frontendek installálása mindhárom worker node-on

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fe1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan100",
              "ips": [ "10.100.0.12/24" ],
              "mac": "c2:b0:57:49:47:f1",
              "gateway": [ "10.100.0.254" ],
              "default-route": ["10.100.0.254" ]
            },
            { "name": "macvlan200",
              "ips": [ "10.200.0.12/24" ],
              "mac": "c2:b0:57:49:47:f2"
            }
            ]'
spec:
  nodeName: w2
  hostAliases:
  - ip: "10.200.0.253"
    hostnames:
    - "hello"
  containers:
  - name: teszt
    image: remenyi/myubuntu:latest
    imagePullPolicy: Always
    ports:
    - containerPort: 80
#    lifecycle:
#      preStop:
#        exec:
#          command: ["/usr/sbin/nginx","-s","quit"]
    securityContext:
      capabilities:
        add:
          - ALL
      privileged: true

---
apiVersion: v1
kind: Pod
metadata:
  name: fe2
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan100",
              "ips": [ "10.100.0.13/24" ],
              "mac": "c2:b0:57:49:47:f3",
              "gateway": [ "10.100.0.254" ],
              "default-route": ["10.100.0.254" ]
            },
            { "name": "macvlan200",
              "ips": [ "10.200.0.13/24" ],
              "mac": "c2:b0:57:49:47:f4"
            }
            ]'
spec:
  nodeName: w3
  hostAliases:
  - ip: "10.200.0.253"
    hostnames:
    - "hello"
  containers:
  - name: teszt
    image: remenyi/myubuntu:latest
    imagePullPolicy: Always
    ports:
    - containerPort: 80
#    lifecycle:
#      preStop:
#        exec:
#          command: ["/usr/sbin/nginx","-s","quit"]
    securityContext:
      capabilities:
        add:
          - ALL
      privileged: true

---
apiVersion: v1
kind: Pod
metadata:
  name: fe3
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan100",
             "ips": [ "10.100.0.14/24" ],
              "mac": "c2:b0:57:49:47:f5",
              "gateway": [ "10.100.0.254" ],
              "default-route": ["10.100.0.254" ]
            },
            { "name": "macvlan200",
              "ips": [ "10.200.0.14/24" ],
              "mac": "c2:b0:57:49:47:f6"
            }
            ]'
spec:
  nodeName: w4
  hostAliases:
  - ip: "10.200.0.253"
    hostnames:
    - "hello"
  containers:
  - name: teszt
    image: remenyi/myubuntu:latest
    imagePullPolicy: Always
    ports:
    - containerPort: 80
#    lifecycle:
#      preStop:
#        exec:
#          command: ["/usr/sbin/nginx","-s","quit"]
    securityContext:
      capabilities:
        add:
          - ALL
      privileged: true
```

Ezután minden frontend podba belépve kiadtam az alábbi parancsot, amely a podokhoz hozzáadott egy dummy interfészt és beállította az ipvs-t

```bash
#!/bin/bash

VIP=10.201.0.1
MASK=24

echo `hostname`

ip link add dummy1 type dummy
ip a add $VIP/$MASK dev dummy1

ipvsadm -A -t $VIP:80 -s mh
for i in {248..253}
do
	ipvsadm -a -t 10.201.0.1:80 -r 10.200.0.$i
done 
```

# Backendek installálása

Mindhárom worker node-ra 2-2 backend podot telepítettem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: be1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.253/24" ],
              "mac": "c2:b0:57:49:47:aa",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w2
  containers:
    - name: be1
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN

---

apiVersion: v1
kind: Pod
metadata:
  name: be2
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.252/24" ],
              "mac": "c2:b0:57:49:47:ab",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w2
  containers:
    - name: be2
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN

---

apiVersion: v1
kind: Pod
metadata:
  name: be3
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.251/24" ],
              "mac": "c2:b0:57:49:47:ac",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w3
  containers:
    - name: be3
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
---

apiVersion: v1
kind: Pod
metadata:
  name: be4
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.250/24" ],
              "mac": "c2:b0:57:49:47:ad",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w3
  containers:
    - name: be4
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
---

apiVersion: v1
kind: Pod
metadata:
  name: be5
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.249/24" ],
              "mac": "c2:b0:57:49:47:ae",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w4
  containers:
    - name: be5
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
---

apiVersion: v1
kind: Pod
metadata:
  name: be6
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "macvlan200",
              "ips": [ "10.200.0.248/24" ],
              "mac": "c2:b0:57:49:47:af",
              "gateway": [ "10.200.0.254" ],
              "default-route": ["10.200.0.254" ]
            }]'
spec:
  nodeName: w4
  containers:
    - name: be6
      image: "remenyi/hello:latest"
      imagePullPolicy: Always
      ports:
        - name: http
          containerPort: 80
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP 
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
```

Lefuttatam az alábbi scriptet

```bash
#!/bin/bash

kubectl exec --tty be1 -- /interfacesetup.sh 10.201.0.1 10.200.0.253 24
kubectl exec --tty be2 -- /interfacesetup.sh 10.201.0.1 10.200.0.252 24
kubectl exec --tty be3 -- /interfacesetup.sh 10.201.0.1 10.200.0.251 24
kubectl exec --tty be4 -- /interfacesetup.sh 10.201.0.1 10.200.0.250 24
kubectl exec --tty be5 -- /interfacesetup.sh 10.201.0.1 10.200.0.249 24
kubectl exec --tty be6 -- /interfacesetup.sh 10.201.0.1 10.200.0.248 24
```

Az interfacesetup.sh szkript:

```bash
#!/bin/ash

VIP=$1
RIP=$2
MASK=$3

arptables -F
arptables -A INPUT -d $VIP -j DROP
arptables -A OUTPUT -s $VIP -j mangle --mangle-ip-s $RIP

ip link add dummy1 type dummy
ip a add $VIP/$MASK dev dummy1
```