# Technical Challenge NG-Voice
Test Kubernetes by Juan Carlos Maturano Saldaña. 
# Overview
This doc contains step-by-step, how to deploy #GKE Standard zonal + Deployment of Kamailio including ConfigMap with kamailio.cfg and Service LoadBalancer for UDP/5060. 

The Kamailio SIP server is designed for scalability, targeting large deployments (e.g. for IP telephony operators or carriers, which have a large subscriber base or route a big volume of calls), but can be also used in enterprises or for personal needs to provide VoIP, Instant Messaging and Presence. Kamailio is well known for its flexibility, robustness, strong security and the extensive number of features

SSIPp is a free, open-source test tool and traffic generator for the Session Initiation Protocol (SIP). It is primarily used to perform performance, load, and stress testing on various SIP equipment like proxies, B2BUAs, and media servers.
Key Features

   1. Protocol Simulation: Can emulate thousands of SIP User Agents (UAC and UAS) simultaneously.
   2. Custom Scenarios: Uses XML-based files to define both simple and complex call flows (e.g., registration, call establishment, re-INVITE).
   3. Traffic Control: Features dynamically adjustable call rates and the ability to limit concurrent calls during a live test.
   4. Media Support: Capable of sending RTP (media) traffic through PCAP replay or RTP echo to test audio/video paths.
   5. Comprehensive Reporting: Provides real-time statistics on-screen and periodic CSV dumps for post-test analysis, including round-trip delay and message counts.
   6. Multi-Transport Support: Operates over UDP, TCP, TLS, and SCTP.


Kamailio supports defining `listen` and `advertise` per socket, which is very useful behind NAT or a load balancer. Its `request_route` also requires explicit actions such as `sl_send_reply()` or `forward()` to respond to or route SIP traffic. In addition, the `sl` module exists specifically for stateless replies, which is perfect for this minimal lab.

## Table of Contents

- [1. Prepare your environment](#1-prepare-your-environment)
- [2. Create a small cluster](#2-create-a-small-cluster)
- [3. Reserve a static public IP](#3-reserve-a-static-public-ip)
- [4. Create the namespace](#4-create-the-namespace)
- [5. Create the Kamailio ConfigMap](#5-create-the-kamailio-configmap)
- [6. Create the Deployment](#6-create-the-deployment)
- [7. Expose Kamailio with a LoadBalancer](#7-expose-kamailio-with-a-loadbalancer)
- [8. Verify that Kamailio is up](#8-verify-that-kamailio-is-up)

## Global Architecture. 

User → SIP client / SIPp → GCP public IP → GKE external UDP LoadBalancer Service → single GKE node → single Kamailio pod ← ConfigMap-provided kamailio.cfg
with sngrep/SIPp VM used externally for traffic generation and troubleshooting.


## 1. Prepare your environment

Enable the required APIs and authenticate:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

gcloud services enable container.googleapis.com compute.googleapis.com
```

Install `kubectl` and the GKE authentication plugin. 

```bash
gcloud components install kubectl
gcloud components install gke-gcloud-auth-plugin

kubectl version --client
gke-gcloud-auth-plugin --version
```

## 2. Create a small cluster

This uses a zonal cluster with 1 node.  This is valid for lab/demo use, but not for high availability.

```bash
gcloud container clusters create kamailio-lab \
  --zone us-central1-a \
  --machine-type e2-standard-2 \
  --num-nodes 1 \
  --release-channel regular
```

Load the credentials into `kubectl`:

```bash
gcloud container clusters get-credentials kamailio-lab \
  --zone us-central1-a

kubectl get nodes
```

## 3. Reserve a static public IP

A static IP helps keep Kamailio's `advertise` value from changing when you recreate the Service. 

```bash
gcloud compute addresses create kamailio-sip-ip \
  --region us-central1
```

Get the IP:

```bash
gcloud compute addresses describe kamailio-sip-ip \
  --region us-central1 \
  --format='get(address)'
```

Save it. In this guide, it will be referred to as:

```text
YOUR_PUBLIC_IP
```

## 4. Create the namespace

```bash
kubectl create namespace telecom
```

## 5. Create the Kamailio ConfigMap

Save the following as `kamailio-configmap.yaml` and replace `YOUR_PUBLIC_IP`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kamailio-config
  namespace: telecom
data:
  kamailio.cfg: |
   #!KAMAILIO

    debug=2
    log_stderror=yes
    children=2
    disable_tcp=yes

    listen=udp:0.0.0.0:5060 advertise 34.59.213.59:5060

    loadmodule "sl.so"
    loadmodule "maxfwd.so"
    loadmodule "textops.so"
    loadmodule "pv.so"
    loadmodule "xlog.so"
    
    
    request_route {
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    xlog("L_INFO", "[lab] $rm from $si:$sp to $ru callid=$ci\n");

    if (is_method("OPTIONS")) {
        sl_send_reply("200", "OK");
        exit;
    }

    if (is_method("INVITE")) {
        sl_send_reply("480", "Temporarily Unavailable");
        exit;
    }

    sl_send_reply("501", "Not Implemented");
    exit;
    }
```

Apply the ConfigMap:

```bash
kubectl apply -f kamailio-configmap.yaml
```

## 6. Create the Deployment

Save the following as `kamailio-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kamailio
  namespace: telecom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kamailio
  template:
    metadata:
      labels:
        app: kamailio
    spec:
      containers:
      - name: kamailio
        image: ghcr.io/kamailio/kamailio-ci:6.0.0-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5060
          protocol: UDP
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        volumeMounts:
        - name: kamailio-config
          mountPath: /etc/kamailio/kamailio.cfg
          subPath: kamailio.cfg
      volumes:
      - name: kamailio-config
        configMap:
          name: kamailio-config
```

Apply the Deployment:

```bash
kubectl apply -f kamailio-deployment.yaml
kubectl get pods -n telecom
kubectl logs -n telecom deploy/kamailio
```

## 7. Expose Kamailio with a LoadBalancer

Save the following as `kamailio-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kamailio-udp
  namespace: telecom
  annotations:
    cloud.google.com/l4-rbs: "enabled"
spec:
  type: LoadBalancer
  loadBalancerIP: 
  externalTrafficPolicy: Local
  selector:
    app: kamailio
  ports:
  - name: sip-udp
    protocol: UDP
    port: 5060
    targetPort: 5060
```

Apply the Service:

```bash
kubectl apply -f kamailio-service.yaml
kubectl get svc -n telecom -w
```

Once the external IP is assigned, SIP/UDP 5060 should already be exposed.

## 8. Verify that Kamailio is up

```bash
kubectl get all -n telecom
kubectl logs -n telecom deploy/kamailio -f
```

If the pod does not start, first check:

```bash
kubectl describe pod -n telecom -l app=kamailio
kubectl logs -n telecom deploy/kamailio --previous
```

# Guide to Build a Second VM for `sngrep` + `SIPp`

This guide explains how to create a second VM in Google Cloud Platform (GCP) to run **sngrep** and **SIPp**.

## Table of Contents

- [1. Create the VM in GCP](#1-create-the-vm-in-gcp)
- [2. Open Only the Access You Really Need](#2-open-only-the-access-you-really-need)
- [3. Connect via SSH](#3-connect-via-ssh)
- [4. Install sngrep](#4-install-sngrep)
- [5. Install SIPp](#5-install-sipp)
- [6. Run a Quick Validation Test](#6-run-a-quick-validation-test)
- [7. Use It Against Your Kamailio in GKE](#7-use-it-against-your-kamailio-in-gke)

---

## 1. Create the VM in GCP

First, make sure you are using the correct project and that the **Compute Engine API** is enabled. Google’s official documentation indicates that the API must be enabled before creating the instance.

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud services enable compute.googleapis.com
```

Now create a simple VM. For this use case, an `e2-micro` or `e2-small` instance is enough. I recommend assigning it the `sip-tools` tag so that you can later apply firewall rules only to that VM.

```bash
gcloud compute instances create sip-tools-vm \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=sip-tools
```



---

## 2. Open Only the Access You Really Need

For SSH, Google recommends creating **specific firewall rules** and using **restricted IP ranges** instead of opening access to the whole Internet.

If you want to connect using **IAP**, Google uses the range `35.235.240.0/20` for SSH over IAP.

If you will connect directly from your home or office, replace `YOUR_PUBLIC_IP/32` with your actual public IP.

### SSH from Your Public IP

```bash
gcloud compute firewall-rules create allow-ssh-sip-tools \
  --direction=INGRESS \
  --action=ALLOW \
  --network=default \
  --priority=1000 \
  --rules=tcp:2024\
  --source-ranges=YOUR_PUBLIC_IP/32 \
  --target-tags=sip-tools
```

---

## 3. Connect via SSH

Google documents two simple ways to connect:

- The **SSH** button in the Google Cloud Console
- The `gcloud compute ssh` command

```bash
gcloud compute ssh --project=YOUR_PROJECT_ID --zone=us-central1-a sip-tools-vm
```

---

## 4. Install `sngrep`

On Ubuntu, `sngrep` is available as a terminal tool to:

- group SIP messages by **Call-ID**
- display the **ladder view** on screen
- apply **BPF-style filters** similar to `tcpdump` or `ngrep`

Install it with:

```bash
sudo apt update
sudo apt install -y sngrep
sngrep -V
```

---

## 5. Install `SIPp`

I recommend the most robust method: **compile SIPp from source**.

The official SIPp documentation for Linux presents it as source code and documents the use of `cmake` and `make`, as well as the options `-DUSE_SSL=1` and `-DUSE_PCAP=1` if you want TLS or PCAP/media support.

The official repository also indicates that it is commonly built with `cmake .` and `make`.

### Install Build Dependencies

These packages cover the C++ compiler, `ncurses`, and—if you want it ready for TLS/PCAP—OpenSSL, `libpcap`, and `libnet`.

```bash
sudo apt install -y \
  git \
  build-essential \
  cmake \
  libncurses-dev \
  libssl-dev \
  libpcap-dev \
  libnet1-dev
```

### Download and Compile

```bash
git clone https://github.com/SIPp/sipp.git
cd sipp
cmake . -DUSE_SSL=1 -DUSE_PCAP=1
make -j"$(nproc)"
sudo install -m 0755 sipp /usr/local/bin/sipp
```

### Verify the Installation

```bash
sipp -v
```

---

## 6. Run a Quick Validation Test

The official SIPp documentation shows a very simple test using an embedded **UAS**, followed by a **UAC** pointing to it. This is useful to validate the installation even before interacting with Kamailio.

### In one terminal on the VM

```bash
sipp -sn uas -i 0.0.0.0 -p 5062
```
<img width="730" height="433" alt="imagen" src="https://github.com/user-attachments/assets/e23e603e-a63d-4891-ba5e-d01f976f2377" />


### In another terminal on the same VM

```bash
sipp -sn uac 127.0.0.1:5062 -m 1
```

<img width="785" height="791" alt="imagen" src="https://github.com/user-attachments/assets/b35de418-62a5-42f7-9a51-59b3e6d25cf1" />


### In a third terminal, to view the traffic in `sngrep`

```bash
sudo sngrep port 5062
```

<img width="1482" height="246" alt="imagen" src="https://github.com/user-attachments/assets/a7eaecf0-2cbc-4920-9cd1-95ddf9bfcf10" />

<img width="1617" height="321" alt="imagen" src="https://github.com/user-attachments/assets/4f54ac98-c199-461f-8c2c-0d06f9d36c86" />



`sngrep` supports exactly this kind of BPF filter by port or host.

---

## 7. Use It Against Your Kamailio in GKE

When you want to use this VM to test your Kamailio, the most practical approach is to run `sngrep` filtered by Kamailio’s public IP, and then launch `SIPp` or your XML scenarios from the same VM.

`sngrep` recognizes SIP over **UDP** and **TCP** and groups dialogs by **Call-ID**. `SIPp` can run both built-in scenarios and custom XML scenarios.

### Example: monitor all traffic going to Kamailio on port 5060

```bash
sudo sngrep host KAMAILIO_PUBLIC_IP and port 5060
```

### In another terminal, launch your XML-based test

```bash
sipp KAMAILIO_PUBLIC_IP:5060 -sf options.xml -t u1 -m 1 -trace_msg -trace_err -trace_logs
```

---

## Notes

- Replace placeholders such as `YOUR_PROJECT_ID`, `YOUR_PUBLIC_IP`, and `KAMAILIO_PUBLIC_IP` with your actual values.
- If you plan to test TLS, make sure the target port and certificates are configured correctly.
- Keeping the firewall scope limited to your IP is safer than exposing SSH broadly.
- 
## Testing Scenarios. 

 OPTIONS to Kamailio
```bash
sipp kamailio_IP:5060 -sf options.xml -t u1 -s kamailio -m 1 -trace_msg -trace_err -trace_logs
```
 Failed Invite with 480. 
This is used to get 480 Temporarily Unavailable from an INVITE.
sipp kamailio_IP:5060 -sf invite_480.xml -t u1 -s 1001 -m 1 -trace_msg -trace_err -trace_logs

 Successful INVITE to the SIPp UAS
This is the server-side for a successful call. You run it on another VM, another pod, or even your laptop. The UAS scenarios start with recv request="INVITE" and respond using the [last_*] headers.
```bash
sipp -sf uas_200.xml -i 0.0.0.0 -p 5062 -t u1 -trace_msg -trace_err -trace_logs
```
This is the client-side for testing against uas_200.xml. The flow matches SIPp’s built-in UAC scenario: INVITE, optional 100/180, 200, ACK, pause, BYE, 200.
```bash
sipp IP_DEL_UAS:5062 -sf uac_200.xml -t u1 -s 1001 -m 1 -trace_msg -trace_err -trace_logs
```
401/407 with authentication

-This works when peer reply  407 Proxy Authentication Required or 401 Unauthorized. SIPp support both bu the official way it´s <recv ... auth="true"> following next message as authentication
```bash
sipp IP_DEL_PROXY_O_UAS:5060 -sf invite_auth_407.xml -t u1 -s 1001 -m 1 -trace_msg -trace_err -trace_logs
```
How to catch in wireshark. 

Run this in sipp VM

sudo tcpdump -i any -w sipp-demo.pcap udp port 5060

## Deployment issues . 

-During sipp installation got bellow error. 

matu@sip-tools-vm:~/sipp$ cmake . -DUSE_SSL=1 -DUSE_PCAP=1 -- The C compiler identification is GNU 13.3.0 
-- The CXX compiler identification is GNU 13.3.0 
-- Detecting C compiler ABI info 
-- Detecting C compiler ABI info - done 
-- Check for working C compiler: /usr/bin/cc - skipped 
-- Detecting C compile features
-- Detecting C compile features - done 
-- Detecting CXX compiler ABI info 
-- Detecting CXX compiler ABI info - done 
-- Check for working CXX compiler: /usr/bin/c++ - skipped 
-- Detecting CXX compile features 
-- Detecting CXX compile features - done -- SCTP is disabled -- GSL is disabled 
-- Checking for module 'pugixml' -- Package 'pugixml', required by 'virtual:world', not found CMake Error at CMakeLists.txt:151 (message): pugixml is required. Either install the library or run 'git submodule update --init'

had to modyfy the Cmake part, as bellow. 

```bash
git submodule update --init
rm -rf CMakeCache.txt CMakeFiles
cmake . -DUSE_SSL=1 -DUSE_PCAP=1
make -j"$(nproc)"
sudo install -m 0755 sipp /usr/local/bin/sipp**
```


## References & Useful Resources

- https://www.kamailio.org/w/documentation/
- https://sipp.sourceforge.net/doc/reference.html
- https://manpages.ubuntu.com/manpages/focal/man8/sngrep.8.html
- https://sipp.readthedocs.io/en/latest/installation.html
- https://sipp.readthedocs.io/en/latest/sipp.html
