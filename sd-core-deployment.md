# SD-Core 5G Deployment

This guide covers deploying SD-Core, the 5G core network, on your Kubernetes cluster.

## Prerequisites

- Kubernetes cluster running (see [Kubernetes Setup](kubernetes-setup.md))
- Helm 3 installed
- kubectl configured

## SD-Core Architecture

SD-Core implements 3GPP 5G Service Based Architecture (SBA):

- **AMF** (Access and Mobility Management Function): Handles UE registration and mobility
- **SMF** (Session Management Function): Manages PDU sessions
- **UPF** (User Plane Function): Handles user data forwarding
- **AUSF** (Authentication Server Function): Authenticates UEs
- **UDM** (Unified Data Management): Manages subscription data
- **UDR** (Unified Data Repository): Stores subscription data
- **PCF** (Policy Control Function): Provides policy rules
- **NRF** (Network Repository Function): Service discovery
- **NSSF** (Network Slice Selection Function): Selects network slices
- **WEBUI**: Web interface for subscriber management

## Step 1: Add SD-Core Helm Repository

```bash
# Add Aether Helm repository
helm repo add aether https://charts.aetherproject.org
helm repo add cord https://charts.opencord.org

# Update repositories
helm repo update
```

## Step 2: Create Configuration Values

Create a custom values file for SD-Core:

```bash
cat <<EOF > /tmp/sd-core-5g-values.yaml
# SD-Core 5G Configuration for AWS EC2 t3.large

# Global settings
global:
  projectName: aether-onramp
  
# 5G Control Plane
omec-control-plane:
  enable5G: true
  
  # AMF Configuration
  amf:
    cfgFiles:
      amfcfg.conf:
        configuration:
          amfName: AMF
          ngapIpList:
            - 0.0.0.0
          sbi:
            scheme: http
            registerIPv4: amf
            bindingIPv4: 0.0.0.0
            port: 29518
          serviceNameList:
            - namf-comm
            - namf-evts
            - namf-mt
            - namf-loc
            - namf-oam
          servedGuamiList:
            - plmnId:
                mcc: 208
                mnc: 93
              amfId: cafe00
          supportTaiList:
            - plmnId:
                mcc: 208
                mnc: 93
              tac: 1
          plmnSupportList:
            - plmnId:
                mcc: 208
                mnc: 93
              snssaiList:
                - sst: 1
                  sd: "010203"
          security:
            integrityOrder:
              - NIA2
            cipheringOrder:
              - NEA0
  
  # SMF Configuration
  smf:
    cfgFiles:
      smfcfg.conf:
        configuration:
          smfName: SMF
          sbi:
            scheme: http
            registerIPv4: smf
            bindingIPv4: 0.0.0.0
            port: 29502
          serviceNameList:
            - nsmf-pdusession
            - nsmf-event-exposure
          pfcp:
            addr: 0.0.0.0
          userplane_information:
            up_nodes:
              UPF:
                type: UPF
                node_id: upf
            links:
              - A: gNB
                B: UPF
          dnn_list:
            - dnn: internet
              cidr: 172.250.0.0/16
  
  # NRF Configuration
  nrf:
    cfgFiles:
      nrfcfg.conf:
        configuration:
          sbi:
            scheme: http
            registerIPv4: nrf
            bindingIPv4: 0.0.0.0
            port: 29510
  
  # AUSF Configuration
  ausf:
    cfgFiles:
      ausfcfg.conf:
        configuration:
          sbi:
            scheme: http
            registerIPv4: ausf
            bindingIPv4: 0.0.0.0
            port: 29509

# 5G User Plane
omec-user-plane:
  enable: true
  
  # UPF Configuration
  upf:
    cfgFiles:
      upf.json:
        mode: af_packet
        hwcksum: true
        gtppsc: true
        cpiface:
          dnn: internet
          hostname: upf
          http_port: "8080"
        access:
          ifname: eth0
        core:
          ifname: eth0
    resources:
      enabled: true
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1000m"
        memory: "2Gi"

# 5G Subscriber Database
omec-sub-provision:
  enable: true
  
  # Web UI Configuration
  webui:
    service:
      type: NodePort
      nodePort: 30080

# Network Slicing Configuration
network-slicing:
  enable: true
  slices:
    - name: slice1
      sst: 1
      sd: "010203"
      dnn: internet

# Monitoring (Optional)
monitoring:
  prometheus:
    enabled: false
  grafana:
    enabled: false
EOF
```

## Step 3: Install SD-Core

```bash
# Create namespace
kubectl create namespace omec || true

# Install SD-Core 5G
helm install sd-core aether/sd-core \
  --namespace omec \
  --values /tmp/sd-core-5g-values.yaml \
  --wait \
  --timeout 10m
```

## Step 4: Verify Deployment

```bash
# Check all pods
kubectl get pods -n omec

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all -n omec --timeout=300s

# Check services
kubectl get svc -n omec

# Check logs
kubectl logs -n omec -l app=amf --tail=50
kubectl logs -n omec -l app=smf --tail=50
kubectl logs -n omec -l app=upf --tail=50
```

## Step 5: Configure Subscribers

### Option 1: Using Web UI

Access the Web UI:

```bash
# Get WebUI service
kubectl get svc -n omec webui

# Port forward if using ClusterIP
kubectl port-forward -n omec svc/webui 5000:5000

# Access at http://localhost:5000
# Default credentials: admin / admin
```

Add subscribers via Web UI:
1. Navigate to Subscribers
2. Click "Add Subscriber"
3. Fill in details:
   - IMSI: 208930000000001
   - Key: 000102030405060708090A0B0C0D0E0F
   - OPC: 69d5c2eb2e2e624750541d3bbc692ba5
   - PLMN: 20893

### Option 2: Using simapp (Recommended)

```bash
# Deploy simapp for subscriber provisioning
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: simapp
  namespace: omec
spec:
  containers:
  - name: simapp
    image: omecproject/simapp:latest
    command: ["/bin/bash", "-c", "tail -f /dev/null"]
EOF

# Wait for simapp to be ready
kubectl wait --for=condition=ready pod simapp -n omec --timeout=60s

# Add subscribers
kubectl exec -it simapp -n omec -- /simapp/bin/simapp \
  subscriber add \
  imsi-208930000000001 \
  key-000102030405060708090A0B0C0D0E0F \
  opc-69d5c2eb2e2e624750541d3bbc692ba5 \
  plmn-20893 \
  slice-010203
```

## Step 6: Configure Network Interfaces

For proper N2/N3 interface connectivity:

```bash
# Create NetworkAttachmentDefinition for N2
cat <<EOF | kubectl apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n2-network
  namespace: omec
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "eth0",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.252.0/24",
      "rangeStart": "192.168.252.10",
      "rangeEnd": "192.168.252.50",
      "gateway": "192.168.252.1"
    }
  }'
EOF

# Create NetworkAttachmentDefinition for N3
cat <<EOF | kubectl apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n3-network
  namespace: omec
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "eth0",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.250.0/24",
      "rangeStart": "192.168.250.10",
      "rangeEnd": "192.168.250.50",
      "gateway": "192.168.250.1"
    }
  }'
EOF
```

## Step 7: Test Core Network

```bash
# Check NRF service discovery
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=nrf -o jsonpath='{.items[0].metadata.name}') -- \
  curl -X GET http://nrf:29510/nnrf-nfm/v1/nf-instances

# Check AMF status
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  curl -X GET http://localhost:29518/namf-comm/v1/ue-contexts

# Check SMF status
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=smf -o jsonpath='{.items[0].metadata.name}') -- \
  curl -X GET http://localhost:29502/nsmf-pdusession/v1/sm-contexts
```

## Monitoring and Logging

### View Component Logs

```bash
# AMF logs
kubectl logs -n omec -l app=amf -f

# SMF logs
kubectl logs -n omec -l app=smf -f

# UPF logs
kubectl logs -n omec -l app=upf -f

# All logs
kubectl logs -n omec --all-containers=true --tail=100
```

### Enable Prometheus Monitoring (Optional)

```bash
# Update values to enable monitoring
helm upgrade sd-core aether/sd-core \
  --namespace omec \
  --values /tmp/sd-core-5g-values.yaml \
  --set monitoring.prometheus.enabled=true \
  --set monitoring.grafana.enabled=true
```

## Troubleshooting

### Pods Not Starting

```bash
# Describe pod
kubectl describe pod <pod-name> -n omec

# Check events
kubectl get events -n omec --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -n omec
```

### Network Function Registration Issues

```bash
# Check NRF logs
kubectl logs -n omec -l app=nrf

# Verify NRF service
kubectl get svc -n omec nrf

# Test NRF connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://nrf.omec.svc.cluster.local:29510/nnrf-nfm/v1/nf-instances
```

### UPF Connectivity Issues

```bash
# Check UPF logs
kubectl logs -n omec -l app=upf

# Check UPF interfaces
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- ip addr

# Check routing
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- ip route
```

### Subscriber Not Found

```bash
# List all subscribers
kubectl exec -it simapp -n omec -- /simapp/bin/simapp subscriber list

# Check UDM/UDR logs
kubectl logs -n omec -l app=udm
kubectl logs -n omec -l app=udr
```

## Advanced Configuration

### Custom PLMN Configuration

Edit the values file to use your PLMN:

```yaml
amf:
  cfgFiles:
    amfcfg.conf:
      configuration:
        servedGuamiList:
          - plmnId:
              mcc: <your-mcc>
              mnc: <your-mnc>
```

### Network Slicing

Configure multiple slices:

```yaml
network-slicing:
  slices:
    - name: embb
      sst: 1
      sd: "010203"
      dnn: internet
    - name: urllc
      sst: 2
      sd: "112233"
      dnn: iot
```

### QoS Configuration

Configure QoS profiles in SMF:

```yaml
smf:
  cfgFiles:
    smfcfg.conf:
      configuration:
        qos_profile:
          - 5qi: 9
            priority: 8
            arp: 1
            bitrate:
              uplink: 100000000
              downlink: 200000000
```

## Upgrading SD-Core

```bash
# Update Helm repository
helm repo update

# Upgrade SD-Core
helm upgrade sd-core aether/sd-core \
  --namespace omec \
  --values /tmp/sd-core-5g-values.yaml \
  --wait
```

## Uninstalling SD-Core

```bash
# Uninstall SD-Core
helm uninstall sd-core -n omec

# Delete namespace (optional)
kubectl delete namespace omec
```

## Next Steps
