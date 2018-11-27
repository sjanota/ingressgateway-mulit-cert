# PoC of Istio ingressgateway serving multiple TLS server certificates

## Goal
The goal of this PoC is to find out how we can make istio ingressgateway serve different server certificates for different gateways. 
Acceptance criteria for each scenario are:
 * Application in namespace stage is served under certificate with CN *.stage.kyma.local
 * Application in namespace qa is served under certificate with CN *.qa.kyma.local
 * Application console is served under certificate with CN *.kyma.local

## Prerequisites
 * Run all commands from root of this repository.
 * Install Kyma prior to running those scenarios.
 * Do not tamper with istio-ingressgateway deployment in any way. Patch and cleanup depends on document structure.

## Scenarios

### Single certificate
In this scenario ingressgateway has one volume mounted. This volume contains several certificates.

#### Provision

1. Create secret with certificates
```
kubectl apply -f ./single-secret/cert-secret.yaml
```

2. Patch `istio-ingressgateway` to use this secret
```
kubectl patch deployment --type json -n istio-system istio-ingressgateway -p "$(cat ./single-secret/ingressgateway-patch.json)"
```

3. Create gateways
```
kubectl apply -f ./single-secret/gateway-qa.yaml
kubectl apply -f ./single-secret/gateway-stage.yaml
```

4. Create applications
```
kubectl apply -f ./deployment-qa.yaml
kubectl apply -f ./deployment-stage.yaml
```

5. Add service to your `/etc/hosts`
```
echo "$(minikube ip) http-db-service.qa.kyma.local" | sudo tee -a /etc/hosts > /dev/null
echo "$(minikube ip) http-db-service.stage.kyma.local" | sudo tee -a /etc/hosts > /dev/null
```

6. Verify certificate Common Name
> NOTE: You may need to wait couple of minutes, before ingressgateway is updated. 
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.qa.kyma.local:443 -servername http-db-service.qa.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.qa.kyma.local
```
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.stage.kyma.local:443 -servername http-db-service.stage.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.stage.kyma.local
```
```
echo "Q" | openssl s_client -showcerts -connect console.kyma.local:443 -servername console.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.kyma.local
```

#### Deprovision
```
sudo sed -i '' "/http-db-service.qa.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.stage.kyma.local/d" /etc/hosts
kubectl delete -f ./deployment-qa.yaml
kubectl delete -f ./deployment-stage.yaml
kubectl delete -f ./single-secret/gateway-qa.yaml
kubectl delete -f ./single-secret/gateway-stage.yaml
kubectl patch deployment --type json -n istio-system istio-ingressgateway -p "$(cat ./single-secret/ingressgateway-unpatch.json)"
kubectl delete -f ./single-secret/cert-secret.yaml
```