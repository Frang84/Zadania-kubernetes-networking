# Zadania do prezentacji: Networking w Kubernetes 

## Narzędzia niezbędne do wykonania ćwiczeń 
1. Docker Desktop 
1. Lubectl 
1. Minikube 

**komendy do instalacji na Linux** 

komenda do zainstalowania najnowszej wersji Kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
   
#walidacja binarki w stosunku do pliku z checksuma 
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

```

komenda do zainstalowania najnowszej wersji minikube
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

## Zadanie 1 

### użyj flannel CNI do komunikacji między podami  

1. Włącz Docker Desktop na komputerze 
1. Stwórz nowy Klaster 
    ```bash
    minikube start -p flannelEx --cni=flannel
    ```
1. sprawdz jakiego klastra aktualnie używasz

    ```bash
    minikube profile
    ```

    w przypadku gdy używasz innego klastra niż ten który przed chwila stworzyłes użyj komendy do zmiany klastra

    ```bash
    minikube profile flannelEx
    ```
1. stwórz środowisko testowe - namespace 
    ```bash
    kubectl create ns networking-demo
    ```
    zobacz czy namespace został utworzony poprawnie, gdzieś na liście powinno istnieć namespace networking-demo
    ```bash
    kubectl get namespace
    ```
1. tworzymy pod w którym będzie działać nasz backend a następnie wystawiamy port 80
    ```bash
    kubectl run backend --image=nginxdemos/hello --port=80 \
    --labels="app=backend" -n networking-demo

    kubectl expose pod backend --port=80 --target-port=80 -n networking-demo
    ```

1. tworzymy frontend 
    ```bash
    kubectl run frontend --image=busybox --restart=Never -n networking-demo \
    --labels="app=frontend" --command -- sleep 3600

    ```
1. Test 

    ```bash
    kubectl exec -n networking-demo frontend -- wget -qO- backend
    ```
1. Dodanie polityk blokowania ruchu do poda backend
    zapisujemy w pliku "deny-all.yaml"
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: deny-all
    namespace: networking-demo
    spec:
    podSelector:
        matchLabels:
            app: backend
    ingress: []
    policyTypes:
    - Ingress
    ```
    wykonujemy komendę
    ```bash
    kubectl apply -f deny-all.yaml
    ```
1. Ponowny test 
    ```bash
    kubectl exec -n networking-demo frontend -- wget -qO- backend 
    ```
1. Czy zmiana CNI podczas korzystania z Klastra jest możliwa - tak, ale dużo rzeczy może pójść nie tak 
    1. Wszystkie istniejące połączenia mogą przestać działać 
    1. Pody mogą stracić łączność z siecią.
    1. Zostaną stare interfejsy sieciowe i reguły iptables

    nie jest to zalecane rozwiązanie. Zamiast tego wykonaj zadanie 2 

## Zadanie 2
### Użyj Calico CNI do komunikacji między podami 
1. Stwórz nowy Klaster 
    ```bash
    minikube start -p calicoEx --cni=calico
    ```
1. sprawdz jakiego klastra aktualnie używasz

    ```bash
    minikube profile
    ```

    w przypadku gdy używasz innego klastra niż ten który przed chwila stworzyłes użyj komendy do zmiany klastra

    ```bash
    minikube profile flannelEx
    ```
1. stwórz środowisko testowe - namespace 
    ```bash
    kubectl create ns networking-demo
    ```
    zobacz czy namespace został utworzony poprawnie, gdzieś na liście powinno istnieć namespace networking-demo
    ```bash
    kubectl get namespace
    ```
1. tworzymy pod w którym będzie działać nasz backend a następnie wystawiamy port 80
    ```bash
    kubectl run backend --image=nginxdemos/hello --port=80 \
    --labels="app=backend" -n networking-demo

    kubectl expose pod backend --port=80 --target-port=80 -n networking-demo
    ```

1. tworzymy frontend 
    ```bash
    kubectl run frontend --image=busybox --restart=Never -n networking-demo \
    --labels="app=frontend" --command -- sleep 3600

    ```
1. Test 

    ```bash
    kubectl exec -n networking-demo frontend -- wget -qO- backend
    ```
1. Dodanie polityk blokowania ruchu do poda backend
    zapisujemy w pliku "deny-all.yaml"
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: deny-all
    namespace: networking-demo
    spec:
    podSelector:
        matchLabels:
            app: backend
    ingress: []
    policyTypes:
    - Ingress
    ```
    wykonujemy komendę
    ```bash
    kubectl apply -f deny-all.yaml
    ```
1. Ponowny test 
    ```bash
    kubectl exec -n networking-demo frontend -- wget -qO- backend --timeout=3
    ```
1. Dodanie polityki która zezwala na ruch z frontendu do backendu

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: allow-frontend
    namespace: networking-demo
    spec:
    endpointSelector:
        matchLabels:
        app: backend
    ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
    ```
    zapisz ten plik pod nazwą  "allow-from-frontend.yaml"
1. Zastosuj polityke 
    ```bash
    kubectl apply -f allow-from-frontend.yaml
    ```
1. Ponowny test 
    ```bash
    kubectl exec -n networking-demo frontend -- wget -qO- backend --timeout=3
    ```
1. Próba połączenia się z backendem z innego poda 
    ```bash
    kubectl run outsider --image=busybox --restart=Never -n networking-demo --command -- sleep 3600

    kubectl exec -n networking-demo outsider -- wget -qO- --timeout=3 backend
    ```