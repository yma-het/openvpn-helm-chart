# Helm chart for OpenVPN
This chart will install an openvpn server inside a kubernetes cluster.  New certificates are generated on install, and a script is provided to generate client keys as needed.  The chart will automatically configure dns to use kube-dns and route all network traffic to kubernetes pods and services through the vpn.  By connecting to this vpn a host is effectively inside a cluster's network.

###Uses
The primary purpose of this chart was to make it easy to access kubernetes services during development.  It could also be used for any service that only needs to be accessed through a vpn or as a standard vpn.

##Usage

```bash
git clone https://github.com/whalescorp/openvpn-helm-chart.git
cd openvpn-helm-chart
helm install release-name charts/openvpn
```

Wait for the external load balancer IP to become available.  Check service status via: kubectl get svckubectl get svc
 
Please be aware that certificate generation is variable and may take some time (minutes).
Check pod status via:
POD_NAME=`kubectl get pods -l type=openvpn | awk END'{ print $1 }'` \
&& kubectl log $POD_NAME --follow

When ready generate a client key as follows:

```bash
POD_NAME=`kubectl get pods --namespace {{ .Release.Namespace }} -l type=openvpn | awk END'{ print $1 }'` \
&& SERVICE_NAME=`kubectl get svc --namespace {{ .Release.Namespace }} -l type=openvpn | awk END'{ print $1 }'` \
&& SERVICE_IP=`kubectl get svc $SERVICE_NAME --namespace {{ .Release.Namespace }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` \
&& KEY_NAME=kubeVPN \
&& kubectl exec --namespace {{ .Release.Namespace }} -it $POD_NAME /etc/openvpn/setup/newClientCert.sh $KEY_NAME $SERVICE_IP \
&& kubectl exec --namespace {{ .Release.Namespace }} -it $POD_NAME cat /etc/openvpn/certs/pki/$KEY_NAME.ovpn > $KEY_NAME.ovpn

```

Be sure to change KEY_NAME if generating additional keys.  Import the .ovpn file into your favorite openvpn tool like tunnelblick and verify connectivity.

##Configuration

All configuration is in the values.yaml file, and can be overwritten via the helm --set flag.

###Certificates

New certificates are generated with each deployment.  If persistence is enabled certificate data will be persisted across pod restarts.  Otherwise new client certs will be needed after each deployment or pod restart.

###Important values
* service.port: 32767 - openVPN process, k8s service and ingress port
* openvpn.OVPN_NETWORK: 10.240.0.0 - Network allocated for openvpn clients (default: 10.240.0.0).
* openvpn.OVPN_SUBNET:  255.255.0.0 - Network subnet allocated for openvpn client (default: 255.255.0.0).
* openvpn.OVPN_PROTO: tcp - Protocol used by openvpn tcp or udp (default: tcp).
* openvpn.OVPN_K8S_POD_NETWORK: "10.0.0.0" - Kubernetes pod network (optional).
* openvpn.OVPN_K8S_POD_SUBNET: "255.0.0.0" - Kubernetes pod network subnet (optional).

###Exposing service via ingress
The following values are mandatory to be passed to nginx chart to expose openvpn service via ingress
constroller:
  globalConfiguration:
    create: true
  spec:
    listeners:
    - name: openvpn-tcp
      port: {{ .Values.service.port }}
      protocol: {{ .Values.openvpn.OVPN_PROTO | upper }}
  service:
    customPorts:
    - name: openvpn
      port: {{ .Values.service.port }}

Example on how to set those variables via terraform:
  set {
    name = "controller.globalConfiguration.create"
    value = "true"
  }

  set {
    name = "controller.globalConfiguration.spec.listeners[0].name"
    value = "openvpn-tcp"
  }

  set {
    name = "controller.globalConfiguration.spec.listeners[0].port"
    value = "32767"
  }

  set {
    name = "controller.globalConfiguration.spec.listeners[0].protocol"
    value = "TCP"
  }

  set {
    name = "controller.service.customPorts[0].name"
    value = "openvpn"
  }

  set {
    name = "controller.service.customPorts[0].port"
    value = "32767"
  }

After doing it, please supply nginxChartIsPrepared variable with value "true" to this chart. This is done to prevent misunderstanding of how this chart works.

###Using pre-genarated keys
There is an option to use pre-generated openvpn keys via hashicorp vault agent injector. Correscponding values are defined in values.vault.yaml.
Also there is script to generate those keys called generate_ovpn_keys.sh It is generates keys and ensrypts them with aws kms, so they can be used just right after this operation to be stored in vault via terraform.
Script requirements:
- golang
- openssl
- initialized aws util
- initialized kubectl
- gnu-compatible sed
Also, golang tool for generating keys is git submodule, so before using it, you must initialize submodules in this repositiory.

####Note: As configured the chart will create a route for a large 10.0.0.0/8 network that may cause issues if that is your local network.  If so tweak this value to something more restrictive.  This route is added, because GKE generates pods with IPs in this range.
