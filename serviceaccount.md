
## ServiceAccounts

A service account provides an identity for processes that run in a Pod, and maps to a ServiceAccount object. 
When you authenticate to the API server, you identify yourself as a particular user. Kubernetes recognises the concept of a user, however, Kubernetes itself does not have a User API. 
When Pods contact the API server, Pods authenticate as a particular ServiceAccount and access the resources in the cluster. 

Whenever we create a namespace, by default a service account will be created. We can view the contents of default serviceaccount(sa)

``` bash
kubectl describe sa default

```

### How to use ServiceAccounts

1. Create a service account using kubectl or helm charts
2. Grant permissions to ServiceAccount object using RBAC authorization
3. Assign the serviceaccount object to deployment



An RBAC Role or ClusterRole contains rules that represent a set of permissions. Permissions are purely additive (there are no "deny" rules). 
A Role always sets permissions within a particular namespace; when you create a Role, you have to specify the namespace it belongs in.


## Create Role and ServiceAccout

```yaml
---
# Service account with broader set of permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubernetes.io/enforce-mountable-secrets: "true"
  name: open-service-account
  namespace: default

---

# Create a Role that has the permissions to  get, list and watch secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: open-secret-reader
rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]

---

```


Now lets create a roleBinding between the serviceAccount and the Role.
A rolebinding grants the permissions defined in a role to a user or set of users or service accounts.
Similar to Role, RoleBinding also is a namescaped resource and it is applicable only in the namespace it is created.


```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: open-secrets-binding
  namespace: default
roleRef: # points to the Role
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: open-secret-reader # name of Role
subjects: # points to the ServiceAccount
  - kind: ServiceAccount
    name: open-service-account # service account to bind to
    namespace: default # ns of service account

```

Here’s how you can create a Go HTTP server that uses the Kubernetes API to list secrets.

<details>
<summary> main.go</summary>

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"

	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	k8s "k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

const (
	KUBE_CONFIG_PATH = <path-to-kube-config>
)

var (
	client *k8s.Clientset
)

type ListSecretsResponse struct {
	SecretsList []string
}

// checks whether the file exists in the system
func exists(path string) (bool, error) {
	_, err := os.Stat(path)
	if err == nil {
		return true, nil
	}
	if os.IsNotExist(err) {
		return false, nil
	}
	return false, err
}

func getKubeClient() error {
	var config *rest.Config

	fmt.Println("Strarting the server")
	checkKubeConfig, err := exists(KUBE_CONFIG_PATH)
	if err != nil {
		fmt.Println("Error while locating kube config with error:", err)
		return err
	}

	if checkKubeConfig {
		fmt.Println("Kube config exists")
		config, _ = clientcmd.BuildConfigFromFlags("", KUBE_CONFIG_PATH)
	} else {
		config, err = rest.InClusterConfig()
		if err != nil {
			fmt.Println("Unable to generate in cluster config with error:", err.Error())
		}
		// creates the clientset

	}
	client, err = k8s.NewForConfig(config)
	if err != nil {
		fmt.Println("Error in deciding kube client with error:", err)
		return err

	}
	return nil

}

func listSecrets(w http.ResponseWriter, r *http.Request) {

	listSecretsResponse := ListSecretsResponse{}

	fmt.Println("got /listSecrets request")
	listResponse, err := client.CoreV1().Secrets("default").List(context.Background(), v1.ListOptions{})
	if err != nil {
		fmt.Println("Unable to list Secrets with error:", err)
		w.Header().Set("Content-Type", "application/json")
	}

	for _, item := range listResponse.Items {
		listSecretsResponse.SecretsList = append(listSecretsResponse.SecretsList, item.ObjectMeta.Name)
	}

	// fmt.Println("Successfully returned the list of secrets")
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(&listSecretsResponse)
}

func main() {

	//Create kubernetes client
	kubeClientErr := getKubeClient()
	if kubeClientErr != nil {
		fmt.Println("Error in generating kube client with error:", kubeClientErr)
	}

	http.HandleFunc("/listSecrets", listSecrets)

	err := http.ListenAndServe(":8400", nil)

	if err != nil {
		fmt.Errorf("Unable to start the server with error: ", err)
	}

}

```
</details>

Create a dockerfile to containerize the above go file.


<details>
<summary>DockerFile</summary>

```bash
FROM golang:1.22.6

WORKDIR /app

COPY go.mod ./
COPY go.sum ./


RUN go mod download && go mod verify

COPY *.go ./

RUN go build -o main

CMD [ "./main" ]
```

</details>


Here’s how to create a Kubernetes Deployment for the Go HTTP server:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-http-server
  labels:
    app: "open-http-server"
spec:
  selector:
    matchLabels:
      app: "open-http-server"
  replicas: 1
  template:
    metadata:
      labels:
        app: "open-http-server"
    spec:
      serviceAccount: open-service-account
      automountServiceAccountToken: true
      containers:
        - name: demoserver
          image: local/demoghttpserver:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8400
```


To expose the Pod, use port forwarding:

```bash
kubectl port-forward pod/<pod-name> 8400:8400
```

You can then access the endpoint:

```bash
curl http://localhost:8400/listSecrets
{"SecretsList":["db-user-pass","db-user-pass2","db-user-pass3"]}

```

You can verify the output using the following kubectl command

```bash
kubectl get secrets
NAME            TYPE     DATA   AGE
db-user-pass    Opaque   2      2d12h
db-user-pass2   Opaque   2      2d12h
db-user-pass3   Opaque   2      2d12h
```


Lets edit the role and update the permissions so that role "open-secret-reader" can only get secrets and deploy the changes.

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: open-secret-reader
rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["secrets"]
    verbs: ["get"] # removed list and watch

```

```bash
restart the  deployment
kubectl rollout restart deployment/open-http-server
```

Lets use portforward and hit the endpoint

```bash
curl http://localhost:8400/listSecrets
{"SecretsList":null}
```

We can check the logs to see what is the error. 
```
Unable to list Secrets with error: secrets is forbidden: User "system:serviceaccount:default:open-service-account" cannot list resource "secrets" in API group "" in the namespace "default"
```

As you can see that the new service account does not have access to list the secrets.



### AutoMount ServiceAccount Token

If you don't want the kubelet to automatically mount a ServiceAccount's API credentials, you can opt out of the default behavior. 
You can opt out of automounting API credentials on /var/run/secrets/kubernetes.io/serviceaccount/token for a service account by setting automountServiceAccountToken: false on the ServiceAccount or deployment.

Also another approach is to use projected volumes to inject the service account token directy into the pod. 


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-http-server
  labels:
    app: "open-http-server"
spec:
  selector:
    matchLabels:
      app: "open-http-server"
  replicas: 1
  template:
    metadata:
      labels:
        app: "open-http-server"
    spec:
      serviceAccount: open-service-account
      volumes:
      - name : token-vol
        projected:
          sources:
          - serviceAccountToken:
              audience: api
              expirationSeconds: 3600
              path: token
      containers:
        - name: demoserver
          image: local/demoghttpserver:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8400
          volumeMounts:
          - name: token-vol
            mountPath: "/var/run/secrets/kubernetes.io/serviceaccount/"
            readOnly: true
```


Basically now the pod will have a volume created with the injected service token. Containers in the pod can make use of this service account to access the kubernetes sever. 
Also this token has an expirationSeconds which is the duration of the validity of service account token. The kubelet willrequest and store the token on behalf of the Pod.
Also kubelet refresh the token as it approaches expiration. The application is responsible for reloading the token when it rotates
The audience field contains the intended audience of the token, if nothing is specified, it defaults to API server.



References:

1. [Configuring Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 
2. [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
3. [Configuring Service Accounts in pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
