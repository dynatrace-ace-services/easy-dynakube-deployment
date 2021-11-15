# Monitor easytravel on kubernetes with DynaKube

## Install easytravel on k3s rancher
Rollout the easytravel application on bare metal VM (VM on a cloud provider) with k3s and istio gateway.  
(tested with Azure VM Standard D2s v3 - 2 vCPU, 8 GB)  

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
    kubectl create namespace easytravel
    kubectl create -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/manifest_easytravel.yaml

    #waiting for easytravel_www pod report READY before installing istio gateway > 3 minutes
    while [[ `kubectl get pods -n easytravel | grep easytravel-www | grep "0/"` ]];do kubectl get pods -n easytravel;echo "==> waiting for easytravel_www pod ready";sleep 1; done
    kubectl apply -f https://raw.githubusercontent.com/dynatrace-ace-services/easy-dynakube-deployment/main/istio_easytravel.yaml


## Deploy Dynakube

From Dynatrace > Deploy Dynatrace > Install OneAgent > Kubernetes
Generate DynaKube script installation 

    export TENANT=<YYYY>.live.dynatrace.com
    export API_TOKEN=<API_TOKEN>
    export PAAS_TOKEN=<PAAS_TOKEN>
    
    wget https://github.com/dynatrace/dynatrace-operator/releases/latest/download/install.sh -O install.sh && sh ./install.sh --api-url "https://$TENANT/api" --api-token $API_TOKEN --paas-token $PAAS_TOKEN --skip-ssl-verification --cluster-name "k3s"


## Kubernetes Monitoring for K3S

Why I don't have workload with k3s ?  
it's because with "K3s" Kubernetes master is running at https://127.0.0.1:6443 ... and can't be changed on k3s   
You can verify it with this command    
    
    kubectl cluster-info 

As a workaround, you have to manually configured it here `Settings > Cloud and virtualization > Kubernetes`:   

   1) Delete "k3s" on `https://127.0.0.1:6443`
   2) Create new one with
        Connection name : `k3s`
        Kubernetes API URL Target : `https://<server_name>:6443`
        Verify hostname in certificate against Kubernetes API URL : disable
        Kubernetes Bearer Token : => use this command 

    kubectl get secret $(kubectl get sa dynatrace-kubernetes-monitoring -o jsonpath='{.secrets[0].name}' -n dynatrace) -o jsonpath='{.data.token}' -n dynatrace | base64 --decode;echo
    
Now you have your workload with k3s :)
            

## Usefull command
Verify istio:

    istioctl analyze
    
Stop loadgen : 

    kubectl -n easytravel scale --replicas=0 deployment/loadgen
 
Restart k3s server:

    sudo systemctl restart k3s
    
Uninstall k3s :

    /usr/local/bin/k3s-uninstall.sh
