# easytravel for kubernetes

## Install easytravel on k3s rancher
Rollout the easytravel application on bare metal VM (VM on a cloud provider) with k3s and istio gateway.  
(tested with Azure VM Standard D2s v3 - 2 vCP, 8 GB)  

    #install k3s
    cd ~
    curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.19 K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--disable=traefik" sh -s -

    #install istio
    echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> .profile; source ~/.profile
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.1 sh -
    sudo mv istio-1.9.1/bin/istioctl /usr/local/bin/istioctl
    istioctl install -y
    kubectl label namespace default istio-injection=enabled

    #easytravel
    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easytravel_k3s/main/manifest_easytravel.yaml

    #waiting for easytravel_www pod report READY before installing istio gateway > 3 minutes
    while [[ `kubectl get pods -n sock-shop | grep easytravel_www | grep "0/"` ]];do kubectl get pods -n sock-shop;echo "==> waiting for easytravel_www pod ready";sleep 1; done
    kubectl apply -f https://raw.githubusercontent.com/JLLormeau/sock-shop/main/ingress-istio.yaml

Verify istio:

    istioctl analyze
    
Stop loadgen : 

    kubectl -n easytravel scale --replicas=0 deployment/loadgen
 
Restart k3s server:

    sudo systemctl restart k3s
    
Uninstall k3s :

    /usr/local/bin/k3s-uninstall.sh
