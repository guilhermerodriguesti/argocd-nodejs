

# Argo CD Node JS


Create application
Connect with ArgoCD CLI using our cluster context:

CONTEXT_NAME=`kubectl config view -o jsonpath='{.current-context}'`
argocd cluster add $CONTEXT_NAME
ArgoCD provides multicluster deployment functionalities. For the purpose of this workshop we will only deploy on the local cluster.

Configure the application and link to your fork (replace the GITHUB_USERNAME):

kubectl create namespace argocd-nodejs
argocd app create argocd-nodejs --repo https://github.com/guilhermerodriguesti/argocd-nodejs.git --path kubernetes --dest-server https://kubernetes.default.svc --dest-namespace argocd-nodejs

Application is now setup, let’s have a look at the deployed application state:

argocd app get argocd-nodejs
You should have this output:

Health Status:      Missing

GROUP       KIND              NAMESPACE         NAME              STATUS     HEALTH   HOOK  MESSAGE
_           Service           argocd-nodejs    argocd-nodejs    OutOfSync  Missing        
apps        Deployment        default          argocd-nodejs   OutOfSync  Missing        
We can see that the application is in an OutOfSync status since the application has not been deployed yet. We are now going to sync our application:

argocd app sync argocd-nodejs
After a couple of minutes our application should be synchronized.

GROUP  KIND        NAMESPACE       NAME            STATUS  HEALTH   HOOK  MESSAGE
_      Service     argocd-nodejs  argocd-nodejs  Synced  Healthy        service/argocd-nodejs created
apps   Deployment  default         argocd-nodejs  Synced  Healthy        deployment.apps/argocd-nodejs created

Update your application
Go to your Github fork repository:

Update spec.replicas: 2 in argocd-nodejs/kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-nodejs
  labels:
    app: argocd-nodejs
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: argocd-nodejs
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: argocd-nodejs
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: argocd-nodejs
        ports:
        - containerPort: 3000
          protocol: TCP
Add a commit message and click on Commit changes
Access ArgoCD Web Interface
To deploy our change we can access to ArgoCD UI. Open your web browser and go to the Load Balancer url:

echo $ARGOCD_SERVER
Login using admin / $ARGO_PWD. You now have access to the ecsdemo-nodejds application. After clicking to refresh button status should be OutOfSync:

Fork url

This means our Github repository is not synchronised with the deployed application. To fix this and deploy the new version (with 2 replicas) click on the sync button, and select the APPS/DEPLOYMENT/DEFAULT/ECSDEMO-NODEJS and SYNCHRONIZE:

Fork url

After the sync completed our application should have the Synced status with 2 pods:

Fork url

All those actions could have been made with the Argo CLI also.

CLEANUP
Congratulations on completing the Continuous Deployment with ArgoCD module.

This module is not used in subsequent steps, so you can remove the resources now, or at the end of the workshop:

argocd app delete ecsdemo-nodejs -y
watch argocd app get ecsdemo-nodejs
Wait until all ressources are cleared with this message:

FATA[0000] rpc error: code = NotFound desc = applications.argoproj.io "ecsdemo-nodejs" not found 
And then delete ArgoCD from your cluster:

kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
Delete namespaces created for this chapter:

kubectl delete ns argocd
kubectl delete ns ecsdemo-nodejs
You may also delete the cloned repository ecsdemo-nodejs within your GitHub account.