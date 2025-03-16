## Demo

### 1. Prerequisites verification

#### Install KinD

KinD (Kubernetes in Docker) is a tool that lets you run a Kubernetes cluster inside Docker containers on your local machine. Think of it as a mini Kubernetes 
playground that runs on your computer. Instead of needing actual servers or cloud resources, KinD creates virtual Kubernetes nodes as Docker containers.
Why use KinD? It's perfect for testing and development because:

It's lightweight and runs on your laptop
It closely mimics a real Kubernetes environment
You can create and destroy clusters in seconds
Great for learning Kubernetes without cloud costs

Local Docker Registry is like having your own private version of DockerHub running on your computer. When you build container images for your applications, 
instead of pushing them to the public internet, you store them locally. This is:

Faster (no internet upload/download needed)
Works offline
Keeps your development code private
Perfect for testing before sharing publicly

The script mentioned (kind-with-registry.sh) automatically sets up both a KinD cluster AND a local registry that work together. This means you can build container 
images on your machine, store them locally, and deploy them to your local Kubernetes cluster without ever needing an internet connection. It's basically setting up 
a complete, self-contained Kubernetes development environment right on your Windows machine with Docker Desktop.

*Show script in vscode*

you can run the script to create your local Kubernetes cluster with local Docker registry.
```bash
./kind-with-registry.sh
```

To make sure the kubectl context is set to the newly created cluster; run:
```bash
kubectl config current-context
```

if the result is not kind-kind; then run:
```bash
kubectl config use kind-kind
```
Now make sure docker is running:

```bash
docker ps -l
```

#### Deploying OpenFaas to KinD cluster
Now that we have our local Kubernetes cluster up and running lets deploy our OpenFaas services to support our functions.

```bash
arkade install openfaas
```

```bash
kubectl get pods -n openfaas
```

#### Templates
Before we move to creating our function, it is essential to understand the concept of templates in OpenFaas.

What are templates?

Templates are basically a wrapper for your functions. Same template can be used for multiple functions.

OpenFaas already provides a variety of templates for different programming languages to start with.

There are two type of templates based on the webserver :

classic watchdog which uses template format from https://github.com/openfaas/templates

of-watchdog which uses template format from https://github.com/openfaas-incubator?q=template&type=&language=

Watchdogs are basically webservers to proxy request to our functions.

We will be using the templates that supports the latest of-watchdog. You can read more about OpenFaas watchdog.

The most important piece in the templates directory is the Dockerfile, it drives everything from your watchdog to how and where your function gets placed 
in the template. You can also create your own template based on the need of your function.

Pull the template from store:
```bash
faas-cli template store list
```

```bash
faas-cli template store pull python3-flask
```
#### Template structure explanation:

index.py is an entrypoint to our Flask application.

requirements.txt on the root project directory contains dependencies for our Flask application. It contains Flask and waitress (WSGI).

template.yml contains the deployment specific configurations.

requirements.txt inside of the function directory is for the function specific dependencies.

Dockerfile contains the instructions to build image for our function.

### Demo Part 1: Build our function with openFaaS

####  1. Create a simple serverless application

Now it’s time to create our Serverless function.

We’ll keep the scope really small and build a simple dictionary function that returns the meaning of words you query. For this we will be using the PyDictionary 
module.Since we have our template in place, we can utilize it to scaffold our function. You may have noticed the function directory and template.yml file inside 
our python3-flask-debian template folder earlier. We can leverage that to create our new function using the faas-cli.
```
faas-cli new pydict --lang python3-flask-debian
```
This will create a folder with the function name(pydict) along with the contents 
we saw earlier inside the templates function folder, plus a yaml file with the function name(pydict).

Let’s add our dependency to the requirements.txt file of function folder.
```Python
nltk==3.8.1
```
Update the handler.py with the following code.

```Python
import nltk
from nltk.corpus import wordnet
import sys

def handle(req):
    word = req.strip()
    
    try:
        wordnet.synsets('test')
    except LookupError:
        sys.stderr.write("Downloading WordNet data...\n")
        nltk.download('wordnet', quiet=True)
    
    sys.stderr.write(f"Looking up: {word}\n")
    
    try:
        synsets = wordnet.synsets(word)
        
        if not synsets:
            sys.stderr.write("No definitions found\n")
            return {}
        
        result = {}
        for syn in synsets:
            pos = syn.pos()
            if pos == 'n':
                pos_full = 'Noun'
            elif pos == 'v':
                pos_full = 'Verb'
            elif pos == 'a' or pos == 's':
                pos_full = 'Adjective'
            elif pos == 'r':
                pos_full = 'Adverb'
            else:
                pos_full = pos
                
            if pos_full not in result:
                result[pos_full] = []
                
            result[pos_full].append(syn.definition())
        
        sys.stderr.write(f"Found definitions: {result}\n")
        return result
    except Exception as e:
        sys.stderr.write(f"Error: {str(e)}\n")
        return {"error": str(e)}
```
Our function is complete

### 2. Openfaas YAML config

A Kubernetes demo won’t be complete without talking about YAML files. They are necessary to describe the state of our deployments. 
faas-cli has already created a yaml file for our function, namely, stack.yaml

This file will tell the faas-cli about the functions and images we want to deploy, 
language templates we want to use, scaling factor, and other configurations.

#### YAML file breakdown
gateway: determines the IP address to connect to the OpenFaas service. Since we are testing it locally t
he gateway provided by default is fine. Note: if we were deploying on managed cloud: we will take the IP address of gateway-external

Under functions: we can register different Serverless functions we want to deploy along with their configuration.

lang: name of the language template the function uses.

handler: location of our Serverless function.

image: fully qualified docker registry URL along with image name and tag.

Since we are using Local registry we have to change the image tag to incorporate it.
Change value of image to the following:

```yaml
image: localhost:5000/pydict:latest
```

#### Port-forward to Localhost 

Now, we need to port-forward the OpenFaas gateway service to our localhost port. Remember the gateway on our yaml file?

To check the gateway of OpenFaas service; run:
```
kubectl get service -n openfaas
```

```
kubectl port-forward -n openfaas svc/gateway 8080:8080
```
Although the the cluster is deployed locally, internally Kubernetes manages it’s own IP addresses. We are port forwarding 
to access the service inside the Kubernetes cluster from our local machine. Once that is done, prepare to deploy our environment

#### Build ➡Push ➡Deploy 🚀

Build our image
```
faas-cli build -f stack.yaml
```

push our image to the local repo
```
faas-cli push -f stack.yaml
```

Now to deploy the function to the cluster; we need to login to faas-cli Let’s generate credentials for authentication.
```
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
```

Check if password exists:
```
env | grep PASSWORD
```

now lets login:
```
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

Time for deployment:
```
faas-cli deploy -f stack.yaml -g http://127.0.0.1:8080
```

### Testing our function 

#### Testing using cli

We can invoke our function from CL I to get the result.
```
echo "brevity" | faas-cli invoke pydict
``` 

#### Test using Openfaas Portal
OpenFaas also comes with a neat UI portal to invoke our function.

```
open a second tab
localhost:8080
```
A prompt will ask you for username and password. 
Use admin for username and for password check the password we generated earlier. 
In case you forgot, you can always repeat the earlier command, i.e.

```
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo) && echo $PASSWORD
```

paradigm
