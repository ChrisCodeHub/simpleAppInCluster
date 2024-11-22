This document covers the basics of deploying a simple python app into k8's
The app is a simple hellow world flask app that just responds to any access on "/"  with hello world

The point of this code is that it 

puts a simple web server into a Dockerfile
then puts that docker file into a deployment file
also defines a service so that the end point inside the cluster is available
finally creates a helm chart to assist in that deployment

so its simple - and is just meant to show the basics of deploying a web-server into k8s
It has no authentications etc... thats in other repos.
Start small, build up to a full featured beastie

the dockerfile 
```Dockerfile
FROM python:3.7

RUN mkdir /app
WORKDIR /app
ADD . /app/
RUN pip install -r requirements.txt

EXPOSE 5000
CMD ["python", "/app/main.py"]
```

requirements.txt is just flash
```
Flask
```

main.py is as simple a flask app as can be
```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello from Python!"

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

deployment.yaml - note its two files in one using the `---` to join
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-python-service
spec:
  selector:
    app: hello-python
  ports:
  - protocol: "TCP"
    port: 6000
    targetPort: 5000
  type: LoadBalancer


---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: hello-python
spec:
  replicas: 4
  template:
    metadata:
      labels:
        app: hello-python
    spec:
      containers:
      - name: hello-python
        image: hello-python:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
```

enter all that... then 
```bash
docker build -t flask-hello .
docker run -p 5000:5000 flask-hello
```
and you should be able to connect a web browser as you've mapped the port to the outside


then spin up a cluster

```
kind create cluster
```

The gotcha here is that the kind cluster is runing its own container registry, so you'll need to copy 
your docker image into the cluster
```bash
kind load docker-image flask-hello:latest
```

then deploy the app into it

```
kubectl cluster-info
kubectl apply -f deploymemnt.yaml
```

then you just need to get the port out of the cluster - since we are playing locally
```bash
kubectl port-forward service/flask-hello 6000:6000
```


OK - lets get this into Helm. For a simple application like this the benefits are small, but
as the number of pods and containers grows, or the deployment needs to be localised (ie variables per environment)
then helm and its templating support are useful. Starting small, convert this to helm

Helm will create a blank directory tree
```bash
helm create simple-app-helm
```
that created 

```
simple-app-helm/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

edit the values.yaml 
```
# This sets the container image more information can be found here: https://kubernetes.io/docs/concepts/containers/images/
image:
  repository: flask-hello
  # This sets the pull policy for images.
  pullPolicy: Never
  # Overrides the image tag whose default is the chart appVersion.
  tag: latest

serviceAccount:
  # Specifies whether a service account should be created
  create: false

service:
  # This sets the service type more information can be found here: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
  type: NodePort
  # This sets the ports more information can be found here: https://kubernetes.io/docs/concepts/services-networking/service/#field-spec-ports
  port: 5000  
```

```
helm install quick-test --debug ./simple-app-helm
helm list --all-namespaces
```


once done 
```
helm uninstall quick-test
```
