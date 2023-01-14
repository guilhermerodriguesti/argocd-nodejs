

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



cat <<EoF > crd/resourcedefinition.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
EoF

kubectl apply -f crd/resourcedefinition.yaml

❯ kubectl get crd crontabs.stable.example.com
NAME                          CREATED AT
crontabs.stable.example.com   2023-01-14T02:20:08Z

kubectl describe crd crontabs.stable.example.com


kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true

curl -i 127.0.0.1:8080/apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/crontabs.stable.example.com


cat <<EoF > crontab/my-crontab.yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
EoF


kubectl apply -f crontab/my-crontab.yaml


kubectl get ct -o yaml

kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true

curl -i 127.0.0.1:8080/apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object

kubectl get nodes -o wide

cat << EOF > kube-bench/job-eks.yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench", "--benchmark", "eks-1.0.1"]
          volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-kubelet
          hostPath:
            path: "/var/lib/kubelet"
        - name: etc-systemd
          hostPath:
            path: "/etc/systemd"
        - name: etc-kubernetes
          hostPath:
            path: "/etc/kubernetes"
EOF


kubectl apply -f kube-bench/job-eks.yaml

kubectl get pods --all-namespaces

kubectl logs kube-bench-6j5m8

cat << EOF > kube-bench/job-debug-eks.yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-debug
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench", "-v", "3", "--logtostderr", "--benchmark", "eks-1.0.1"]
          volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-kubelet
          hostPath:
            path: "/var/lib/kubelet"
        - name: etc-systemd
          hostPath:
            path: "/etc/systemd"
        - name: etc-kubernetes
          hostPath:
            path: "/etc/kubernetes"
EOF

CREATE DJ APP
Let’s create the DJ App!

dj app 1

The application repo has all of the configuration YAML required to deploy the DJ App into its own prod namespace on your Kubernetes cluster.

kubectl apply -f 1_base_application/base_app.yaml
This will create the prod namespace as well as the Deployments and Services for the application. The output should be similar to:


namespace/prod created
deployment.apps/dj created
deployment.apps/metal-v1 created
deployment.apps/jazz-v1 created
service/dj created
service/metal-v1 created
service/jazz-v1 created
You can now verify that the objects were all created successfully and the application is up and running.

kubectl -n prod get all

NAME                            READY   STATUS    RESTARTS   AGE
pod/dj-6bf5fb7f45-qkhv7         1/1     Running   0          42s
pod/jazz-v1-6f688dcbf9-djb9h    1/1     Running   0          41s
pod/metal-v1-566756fbd6-8k2rs   1/1     Running   0          41s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/dj         ClusterIP   10.100.42.60     <none>        9080/TCP   41s
service/jazz-v1    ClusterIP   10.100.192.113   <none>        9080/TCP   40s
service/metal-v1   ClusterIP   10.100.238.120   <none>        9080/TCP   40s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dj         1/1     1            1           42s
deployment.apps/jazz-v1    1/1     1            1           41s
deployment.apps/metal-v1   1/1     1            1           41s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/dj-6bf5fb7f45         1         1         1       43s
replicaset.apps/jazz-v1-6f688dcbf9    1         1         1       42s
replicaset.apps/metal-v1-566756fbd6   1         1         1       42s
Once you’ve verified everything is looking good in the prod namespace, you’re ready to test out this initial version of the DJ App.
======================================================================================================================================================

kubectl get pods -n prod -l app=dj -o jsonpath='{.items[].metadata.name}'


App Mesh

kubectl apply -f djapp/base_app.yml

kubectl -n prod get all


NAME                            READY   STATUS    RESTARTS   AGE
pod/dj-75d87f676-67tln          1/1     Running   0          47s
pod/jazz-v1-b7587d5f4-ml2xc     1/1     Running   0          46s
pod/metal-v1-654c88c679-bvnfd   1/1     Running   0          46s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/dj         ClusterIP   172.20.177.58    <none>        9080/TCP   46s
service/jazz-v1    ClusterIP   172.20.117.170   <none>        9080/TCP   44s
service/metal-v1   ClusterIP   172.20.176.52    <none>        9080/TCP   45s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dj         1/1     1            1           49s
deployment.apps/jazz-v1    1/1     1            1           48s
deployment.apps/metal-v1   1/1     1            1           48s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/dj-75d87f676          1         1         1       50s
replicaset.apps/jazz-v1-b7587d5f4     1         1         1       49s
replicaset.apps/metal-v1-654c88c679   1         1         1       49s

export DJ_POD_NAME=$(kubectl get pods -n prod -l app=dj -o jsonpath='{.items[].metadata.name}')

kubectl exec -n prod -it ${DJ_POD_NAME} bash

kubectl exec -n prod -it dj-75d87f676-67tln bash

======================================================================================================================================================
TEST DJ APP
To test what we’ve just created, we will:

Exec into the dj pod
curl out to the jazz-v1 and metal-v1 backends
First we will exec into the dj pod

export DJ_POD_NAME=$(kubectl get pods -n prod -l app=dj -o jsonpath='{.items[].metadata.name}')

kubectl exec -n prod -it ${DJ_POD_NAME} bash
You will see a prompt from within the dj container.


root@dj-5b445fbdf4-8xkwp:/usr/src/app#
Now from the dj container, we’ll issue a curl request to the jazz-v1 backend service:

curl -s jazz-v1:9080 | json_pp
Output should be similar to:


[
   "Astrud Gilberto",
   "Miles Davis"
]
Try it again, but issue the command to the metal-v1 backend:

curl -s metal-v1:9080 | json_pp
You should get a list of heavy metal bands back:


[
   "Megadeth",
   "Judas Priest"
]
When you’re done exploring this vast world of music, hit CTRL-D, or type exit to quit the container’s shell.