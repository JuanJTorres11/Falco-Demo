# Falco-Demo

Este demo se hará sobre Minikube, pero puedes seguirlo en cualquier implementación de Kubernetes. Igualmente, se asume que ya tienes instalado helm en tu entorno de desarrollo.

1. Luego de instalar minikube, lo inicias y pruebas con 
```
minikube start --driver=docker --cpus=4 --memory=8192
kubectl cluster-info
kubectl get nodes -o wide
```
2. Instalas Falco usando helm
```
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# Importante: las claves de Helm mapean al falco.yaml (underscores). 
# Si tu versión del chart difiere, valida con: helm show values falcosecurity/falco | less
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.driver.kind=ebpf \
  --set falco.json_output=true \
  --set falco.http_output.enabled=true \
  --set falco.http_output.url="http://falcosidekick.falco.svc.cluster.local:2801/"

kubectl -n falco get pods -o wide
kubectl -n falco logs daemonset/falco | head -n 80
```
3. Debes desplegar y configurar Falcosidekick para conectarlo con Slack
```
kubectl -n falco create secret generic falcosidekick-secrets \
  --from-literal=SLACK_WEBHOOKURL='https://hooks.slack.com/services/XXX/YYY/ZZZ'
```
4. Aplicas el values de Falcosidekick para configurar el Deployment
```
kubectl apply -f falcosidekick.yaml
kubectl -n falco get deploy,svc,pods -l app=falcosidekick
kubectl -n falco logs deploy/falcosidekick | tail -n 50
```
5. PUedes generar eventos de prueba para generar alertas
```
# Shell interactivo
kubectl run demo --image=alpine -it -- sh
# Dentro del contenedor:
id; uname -a; ps aux | head

# Gestor de paquetes (drift)
apk update || true
apk add curl || true

# Escritura sensible
echo "# test kcd" >> /etc/hosts || true
```
6. Puedes debuggear con los siguientes comandos
```
kubectl -n falco logs ds/falco | sed -n '1,120p'
kubectl -n falco logs deploy/falcosidekick | tail -n 100
kubectl -n falco get svc falcosidekick -o yaml | grep -E 'clusterIP|port'
```
