# kubernetes-minikube-onpremise

In this repo, you will create 2 pods in the default namespace to deploy a Flask application and a MySQL server containers. Also we are going to use istio-system and kubernetes-dashboard namespaces for tooling. 

## Requirements 

Minikube installed (it will be on-premises, that's why). You can install it using `choco install minikube`

kubectl installed. Run kubectl --version, Minikube installs it by default, but ensure you got it.
Docker Desktop running. Go to [Docker webpage](https://docs.docker.com/get-started/get-docker/).
Install Istio in demo mode. You can go [here](https://istio.io/latest/docs/setup/install/) to search how to install it. I am on Windows, this helped me doing it on Powershell:

```
$IstioVersion = "1.19.0"  # Replace with the desired Istio version
Invoke-WebRequest -Uri "https://github.com/istio/istio/releases/download/$IstioVersion/istio-$IstioVersion-win.zip" -OutFile "istio.zip"
Expand-Archive -Path "istio.zip" -DestinationPath "."
cd "istio-$IstioVersion"
$Env:PATH += ";$PWD\bin"

.\bin\istioctl.exe install --set profile=demo -y

# To ensure Istio can access the namespace, tag it:
kubectl label namespace default istio-injection=enabled

# Also you can use kubectl to scale certain deployment, but better do changes in the YAML files.
# It is commented, but you can do it if want to test it.
# kubectl scale deployment flask-app --replicas=3

```

## Notes

For this you will need to edit the variable for some files in order to point to your volume folder. Change them according to your needs.

Check `mysql-deployment.yml`:
Line 12: `path: {{ vars.LOCAL_PATH }}  # Adjust for your Windows/Linux environment`
Line 49: `value: {{ secrets.MYSQL_PASSWORD }}  # MySQL root password`
Line 51: `value: {{ vars.MYSQL_DB }}  # Database name`

Check `flask-app-deployment.yml`:

* Line 25: `value: {{ vars.MYSQL_HOST }}  # Kubernetes internal service name for MySQL`
* Line 27: `{{ vars.MYSQL_PORT }}`
* Line 29: `{{ vars.MYSQL_USER }}`
* Line 31: `{{ secrets.MYSQL_PASSWORD }}`
* Line 33: `{{ vars.MYSQL_DB }}`

Check `flask-app-deployment.yml`:
* Line 23: `value: {{ secrets.GRAFANA_PASSWORD }}  # Set admin password for Grafana`

---

Now that you have the reqs, do this:

```
# To use Docker instead of HyperV and a VM
minikube start --driver=docker

# To create the pods
# The image that is pulling is public, but the code is on an owned private GitHub repo.
kubectl apply -f flask-app-deployment.yml 

# These are public.
kubectl apply -f mysql-deployment.yml
kubectl apply -f istio-mesh.yml
kubectl apply -f prometheus-deployment.yml
kubectl apply -f grafana-deployment.yml

# This is to install the Kubernetes Dashboard (for poc-testing purposes only, not recommended for production):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# To create a service account that will be the bridge between your cluster and the dashboard
kubectl create serviceaccount dashboard-admin -n default

#Create a role with enough permissions to list your Kubernetes info
kubectl create clusterrolebinding dashboard-admin-binding --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin

# For Istio, you will need to tunnel Minikube, because Minikube doesn't enable
# External IPs on-premises, so Istio will not be accessible without it.
minikube tunnel

# Note: You will need to open many bash/powershell windows (for on-premises only)
# When you run locally Minikube it doesn't add an external IP.
# You need to do the port binding into your localhost,
# so you can access to it using http://127.0.0.1:5000 at your browser.
kubectl port-forward svc/flask-app 5000:5000
kubectl port-forward svc/grafana -n istio-system 3000:3000

# Now in another shell, open the Dashboard
kubectl proxy

# Use this url to login: 
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
# Create token to log into the Dashboard
kubectl -n default create token dashboard-admin

# For this PoC was used 2 manual queries once thet you logged into Grafana, which follows PromQL.
# You can add a ClusterMap to do so, but for this time, we are going to add them using the cluster's info.

```









