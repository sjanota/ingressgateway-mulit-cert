# PoC of Istio ingressgateway serving multiple TLS server certificates

## Goal
The goal of this PoC is to find out how we can make istio ingressgateway serve different server certificates for different gateways. PoC is split into three scenarios:
 * [Single volume with many certificates](#single-volume-with-many-certificates)
 * [Volume per certificate](#volume-per-certificate)
 * [Ingressgateway per namespace](#ingressgateway-per-namespace)

Acceptance criteria for each scenario are:
 * Application in namespace stage is served under certificate with CN *.stage.kyma.local
 * Application in namespace qa is served under certificate with CN *.qa.kyma.local
 * Application console is served under certificate with CN *.kyma.local
 * `production` Namespace can be added and served under CN *.production.kyma.local 

## Prerequisites
 * Run all commands from root of this repository.
 * Install Kyma prior to running those scenarios.
 * Do not tamper with istio-ingressgateway deployment in any way. Patch and cleanup depends on document structure.

## Scenarios

### Single volume with many certificates
In this scenario ingressgateway has one volume mounted. This volume contains several certificates, which can be added dynamically to this volume.

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

### Add `production`
1. Add production cert to secret
```
kubectl patch secret -n istio-system istio-ingressgateway-certs-namespaces -p '
data:
  "production.key": '$(cat ./production.key | base64)'
  "production.crt": '$(cat ./production.crt | base64)'
'
```

2. Create production gateway
```
kubectl apply -f ./single-secret/gateway-production.yaml
```

3. Create application in production
```
kubectl apply -f ./deployment-production.yaml
```

4. Add service to your `/etc/hosts/`
```
echo "$(minikube ip) http-db-service.production.kyma.local" | sudo tee -a /etc/hosts > /dev/null
```

5. Verify certificate Common Name
> NOTE: You may need to wait couple of minutes, before ingressgateway is updated. 
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.production.kyma.local:443 -servername http-db-service.production.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.production.kyma.local
```

#### Deprovision
```
sudo sed -i '' "/http-db-service.qa.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.stage.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.production.kyma.local/d" /etc/hosts
kubectl delete -f ./deployment-qa.yaml
kubectl delete -f ./deployment-stage.yaml
kubectl delete -f ./deployment-production.yaml
kubectl delete -f ./single-secret/gateway-qa.yaml
kubectl delete -f ./single-secret/gateway-stage.yaml
kubectl delete -f ./single-secret/gateway-production.yaml
kubectl patch deployment --type json -n istio-system istio-ingressgateway -p "$(cat ./single-secret/ingressgateway-unpatch.json)"
kubectl delete -f ./single-secret/cert-secret.yaml
```

### Volume per certificate
In this scenario ingressgateway has many volumes mounted. Every volume contains one certificate. Adding new certificate requires 
modification in ingressgateway deployment.

#### Provision

1. Create secrets with certificates
```
kubectl apply -f ./many-secrets/secret-cert-qa.yaml
kubectl apply -f ./many-secrets/secret-cert-stage.yaml
```

2. Patch `istio-ingressgateway` to use this secret
```
kubectl patch deployment --type json -n istio-system istio-ingressgateway -p "$(cat ./many-secrets/ingressgateway-patch.json)"
```

3. Create gateways
```
kubectl apply -f ./many-secrets/gateway-qa.yaml
kubectl apply -f ./many-secrets/gateway-stage.yaml
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

### Add `production`
1. Add production cert to secret
```
kubectl apply -f ./many-secrets/secret-cert-production.yaml
```

2. Add cert to ingressgateway
```
kubectl patch --type json -n istio-system deployment istio-ingressgateway -p "$(cat ./many-secrets/ingressgateway-patch-prod.json)"
```

3. Create production gateway
```
kubectl apply -f ./many-secrets/gateway-production.yaml
```

4. Create application in production
```
kubectl apply -f ./deployment-production.yaml
```

5. Add service to your `/etc/hosts/`
```
echo "$(minikube ip) http-db-service.production.kyma.local" | sudo tee -a /etc/hosts > /dev/null
```

6. Verify certificate Common Name
> NOTE: You may need to wait couple of minutes, before ingressgateway is updated. 
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.production.kyma.local:443 -servername http-db-service.production.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.production.kyma.local
```

#### Deprovision
```
sudo sed -i '' "/http-db-service.qa.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.stage.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.production.kyma.local/d" /etc/hosts
kubectl delete -f ./deployment-qa.yaml
kubectl delete -f ./deployment-stage.yaml
kubectl delete -f ./deployment-production.yaml
kubectl delete -f ./many-secrets/gateway-qa.yaml
kubectl delete -f ./many-secrets/gateway-stage.yaml
kubectl delete -f ./many-secrets/gateway-production.yaml
kubectl patch deployment --type json -n istio-system istio-ingressgateway -p "$(cat ./many-secrets/ingressgateway-unpatch.json)"
kubectl delete -f ./many-secrets/secret-cert-qa.yaml
kubectl delete -f ./many-secrets/secret-cert-stage.yaml
kubectl delete -f ./many-secrets/secret-cert-production.yaml
```

### Ingressgateway per namespace
In this scenario eveyr namespace have its own ingressgateway with certificate matching namespace domain.

#### Provision

1. Create secrets with certificates
```
kubectl apply -f ./many-ingressgateways/secret-cert-qa.yaml
kubectl apply -f ./many-ingressgateways/secret-cert-stage.yaml
```

2. Create `ingressgateway`s for namespaces
```
kubectl apply -f ./many-ingressgateways/ingressgateway-qa.yaml
kubectl apply -f ./many-ingressgateways/ingressgateway-stage.yaml
```

3. Create gateways
```
kubectl apply -f ./many-ingressgateways/gateway-qa.yaml
kubectl apply -f ./many-ingressgateways/gateway-stage.yaml
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
echo "Q" | openssl s_client -showcerts -connect http-db-service.qa.kyma.local:31391 -servername http-db-service.qa.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.qa.kyma.local
```
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.stage.kyma.local:31392 -servername http-db-service.stage.kyma.local 2>/dev/null | grep subject
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

### Add `production`
1. Add production cert to secret
```
kubectl apply -f ./many-secrets/secret-cert-production.yaml
```

2. Add cert to ingressgateway
```
kubectl patch --type json -n istio-system deployment istio-ingressgateway -p "$(cat ./many-secrets/ingressgateway-patch-prod.json)"
```

3. Create production gateway
```
kubectl apply -f ./many-secrets/gateway-production.yaml
```

4. Create application in production
```
kubectl apply -f ./deployment-production.yaml
```

5. Add service to your `/etc/hosts/`
```
echo "$(minikube ip) http-db-service.production.kyma.local" | sudo tee -a /etc/hosts > /dev/null
```

6. Verify certificate Common Name
> NOTE: You may need to wait couple of minutes, before ingressgateway is updated. 
```
echo "Q" | openssl s_client -showcerts -connect http-db-service.production.kyma.local:443 -servername http-db-service.production.kyma.local 2>/dev/null | grep subject
```
Expected output:
```
subject=/CN=*.production.kyma.local
```

#### Deprovision
```
sudo sed -i '' "/http-db-service.qa.kyma.local/d" /etc/hosts
sudo sed -i '' "/http-db-service.stage.kyma.local/d" /etc/hosts
kubectl delete -f ./deployment-qa.yaml
kubectl delete -f ./deployment-stage.yaml
kubectl delete -f ./many-ingressgateways/gateway-qa.yaml
kubectl delete -f ./many-ingressgateways/gateway-stage.yaml
kubectl delete -f ./many-ingressgateways/ingressgateway-qa.yaml
kubectl delete -f ./many-ingressgateways/ingressgateway-stage.yaml
kubectl delete -f ./many-ingressgateways/secret-cert-qa.yaml
kubectl delete -f ./many-ingressgateways/secret-cert-stage.yaml
```
