+++
title = "Jupyter Notebooks"
description = "Using Jupyter notebooks in Kubeflow"
weight = 10
+++

## Bringing up a Jupyter Notebook

1. To connect to Jupyter follow the [instructions](/docs/other-guides/accessing-uis)
to access the Kubeflow UI. From there you will be able to navigate to JupyterHub   
   ![JupyterHub Link](/docs/images/jupyterlink.png)
1. Sign in 
   * On GCP you sign in using your Google Account
     * If you are already logged into your Google Account you may not 
       be prompted to login again
   * On all other platforms you can sign in using any username/password   
1. Click the "Start My Server" button, and you will be greeted by a dialog screen.
1. Select a CPU or GPU image from the Image dropdown menu depending on whether you are doing CPU or GPU training, or whether or not you have GPUs in your cluster. We currently offer a cpu and gpu image for each tensorflow minor version(eg: 1.4.1,1.5.1,1.6.0). Or you can type in the name of any TF image you want to run.
1. Allocate memory, CPU, GPU, or other resources according to your need (1 CPU and 2Gi of Memory are good starting points)
    * To allocate GPUs, make sure that you have GPUs available in your cluster
    * Run the following command to check if there are any nvidia gpus available:
    `kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"`
    * If you have GPUs available, you can schedule your server on a GPU node by specifying the following json in `Extra Resource Limits` section: `{"nvidia.com/gpu": "1"}`
  1. Click Spawn

      * The images are 10's of GBs in size and can take a long time to download
        depending on your network connection

      * You can check the status of your pod by doing

        ```
        kubectl -n ${NAMESPACE} describe pods jupyter-${USERNAME}
        ```

          * Where ${USERNAME} is the name you used to login
          * **GKE users** if you have IAP turned on the pod will be named differently

            * If you signed on as USER@DOMAIN.EXT the pod will be named

            ```
            jupyter-accounts-2egoogle-2ecom-3USER-40DOMAIN-2eEXT
            ```

1. You should now be greeted with a Jupyter Notebook interface.

The image supplied above can be used for training Tensorflow models with Jupyter. The images include all the requisite plugins, including [Tensorboard](https://www.tensorflow.org/get_started/summaries_and_tensorboard) that you can use for rich visualizations and insights into your models.

To test the install, we can run a basic hello world (adapted from [mnist_softmax.py](https://github.com/tensorflow/tensorflow/blob/r1.4/tensorflow/examples/tutorials/mnist/mnist_softmax.py) )

```
from tensorflow.examples.tutorials.mnist import input_data
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)

import tensorflow as tf

x = tf.placeholder(tf.float32, [None, 784])

W = tf.Variable(tf.zeros([784, 10]))
b = tf.Variable(tf.zeros([10]))

y = tf.nn.softmax(tf.matmul(x, W) + b)

y_ = tf.placeholder(tf.float32, [None, 10])
cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(y), reduction_indices=[1]))

train_step = tf.train.GradientDescentOptimizer(0.05).minimize(cross_entropy)

sess = tf.InteractiveSession()
tf.global_variables_initializer().run()

for _ in range(1000):
  batch_xs, batch_ys = mnist.train.next_batch(100)
  sess.run(train_step, feed_dict={x: batch_xs, y_: batch_ys})

correct_prediction = tf.equal(tf.argmax(y,1), tf.argmax(y_,1))
accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
print(sess.run(accuracy, feed_dict={x: mnist.test.images, y_: mnist.test.labels}))
```

Paste the example into a new Python 3 Jupyter notebook and execute the code. This should result in a 0.9014 accuracy result against the test data.

Please note that when running on most cloud providers, the public IP address will be exposed to the internet and is an
unsecured endpoint by default. For a production deployment with SSL and authentication, refer to the [documentation](https://github.com/kubeflow/kubeflow/tree/{{< params "githubbranch" >}}/components/jupyterhub).


## Submitting k8s resources from Jupyter notebooks in `kubeflow` namespace

The Jupyter Notebook pods are assigned the `jupyter-notebook` service account in `kubeflow` namespace by default. This service account is bound to `jupyter-notebook` role which has namespace-scoped permissions to the following k8s resources:

* pods
* deployments
* services
* jobs
* tfjobs
* pytorchjobs

This means that you can directly create these k8s resources directly from your jupyter notebook. kubectl is already installed in the notebook, so you can create k8s resources running the following command in a jupyter notebook cell


```
!kubectl create -f myspec.yaml
```
## Accessing GCP services from Jupyter notebooks in `kubeflow` namespace
GCP credential secret is currently created in `kubeflow` namespace and is injected to Jupyter Notebook pods by default. Secretes in k8s are all namespace-scoped.  GCP client services are already installed in Kubeflow notebooks. 
This means that you can use GCP services available for your account in your jupyter notebooks. 
For instance, if your account has permission for using BigQuery service, you can use it in your notebooks (follow instructions available in section Querying and visualizing BigQuery data [here](https://cloud.google.com/bigquery/docs/visualize-jupyter)
to learn how to use BigQuery in jupyter notebooks).   

## Creating a Jupyter notebook in user-defined namespaces
You can also create jupyter notebooks in other namespaces than `kubeflow`. However, before creating any notebook you have to make sure that you create GCP secret and service account required by Kubeflow notebooks in your namespace. 
In specific, kubeflow expects a service account named `jupyter-notebook`  and a secret named `user-gcp-sa` to exist in your namespace.  
To this end, you can simply copy the above from `kubeflow` namespace to your desired namespace as follows:
``` 
kubectl get secret user-gcp-sa  --namespace=kubeflow --export -o yaml |\
 kubectl apply --namespace=<your desired namespace> -f -
```
```
kubectl get serviceaccount jupyter-notebook  --namespace=kubeflow --export -o yaml |\
 kubectl apply --namespace=<your desired namespace> -f -
``` 

Alternatively, if you would like to leverage GCP services and kubeflow fairing functionality using your account you can  manually create 
them in your namespace as follows:
* Create a GCP service account:
```
export PROJECT_ID=<your-project-id>
export SA_NAME=<your-sa-name>
export NAMESPACE=<your-desired-namespace>
gcloud iam service-accounts create $SA_NAME
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/editor
```
* Create a key from your GCP service account:
```
gcloud iam service-accounts keys create <path-to-key>/key.json \
    --iam-account $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com
```
* Create a secret from your key JSON file and name it `user-gcp-sa`

```
kubectl create secret generic user-gcp-sa -n $NAMESPACE\
  --from-file=user-gcp-sa.json=<path-to-key>/key.json
```
* Create a service account bounding to a role which has permission for k8s resources in your namespace. 
To this end, you need to create three manifests to declare role, service account and rolebinding. Then you need to deploy them in your cluster and desired namespace.
  * Create notebook-role using the following manifest in a file named  jupyterRole.yaml, and then deploy it:
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: jupyter-notebook-role
rules:
- apiGroups: [ ""]
  resources: ["pods","pods/log","secrets","services"]
  verbs: ["*",]
- apiGroups: ["","apps","extensions"]
  resources: ["deployments","replicasets"]
  verbs: ["*"]
- apiGroups: ["kubeflow.org"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["*"]
```

```
   kubectl apply -n $NAMESPACE -f jupyterRole.yaml
```


  * Write the service account `jupyter-notebook` from the following manifest in a file named jupyterNotebook.yaml and then deploy it:
```
apiVersion: v1
kind: ServiceAccount
metadata: 
  name: jupyter-notebook
```

```
kubectl apply -n $NAMESPACE -f jupyterNotebook.yaml
```

  * Write a `RoleBinding` to bound `jupyter-notebook` to the role from the following manifest in a file named `jupyterRoleBinding.yaml`, and then deploy it.
``` 
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: jupyter-notebook-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jupyter-notebook-role
subjects:
- kind: ServiceAccount
  name: jupyter-notebook  
```
```
kubectl apply -n $NAMESPACE -f jupyterRoleBinding.yaml
```

Finally, if you are not interested in using GCP services and/or Kubeflow Fairing, the workaround is to create  
the aforementioned secret (i.e., `user-gcp-sa`) and service account (i.e., `jupyter-notebook`) without necessarily having them to point to an actual account as follows: 
* Write a secret manifest in a file named `gcpSecret.yaml`:
``` 
apiVersion: v1
kind: Secret
metadata:
  name: user-gcp-sa
type: Opaque
```
* Write a serviceaccount manifest without binding it to a role and store it in a file named `jupyterNotebook.yaml`: 
```
 apiVersion: v1
 kind: ServiceAccount
 metadata: 
  name: jupyter-notebook
```
* Apply the above manifests in your namespace
```
kubectl apply -n $NAMESPACE -f gcpSecret.yaml
```
```
kubectl apply -n $NAMESPACE -f jupytetNotebook.yaml
```


## Creating a custom Jupyter image
You can create your own Jupyter image and use it in your Kubeflow cluster.
Your custom image needs to meet the requirements created by Kubeflow Notebook Controller. Kubeflow Notebook Controller  manages the life-cycle of notebooks.
 Kubeflow Web UI expects the Jupyer to be launched upon running the docker image with only `docker run`. For that you need to set the default command of your image to launch Jupyter. The Jupyter launch command needs to be set as follows:

* Set the working directory: `--notebook-dir=/home/jovyan`. This is because the folder `/home/jovyan` is backed by Kubernetes Persistent Volume (PV)
* Allow Jupyter to listen on all IPs: `--ip=0.0.0.0`
* Allow the user to run the notebook as root: `--allow-root`
* Set port: `--port=8888`
* Disable authentication. Kubeflow takes care of authentication. Use the following to allow passwordless access to Jupyter: `--NotebookApp.token=''  --NotebookApp.password=''`
* Allow any origin to access your Jupyter: `--NotebookApp.allow_origin='*'`
* Set base_url: Kubeflow Notebook Controller manages the base URL for the notebook server using the environment variable called `NB_PREFIX`. Your should define the variable in your image and set the value of `base_url` as follows: `--NotebookApp.base_url=NB_PREFIX`

As an example your Dockerfile should contain the following:


```
ENV NB_PREFIX /

CMD ["sh","-c", "jupyter notebook --notebook-dir=/home/jovyan --ip=0.0.0.0 --no-browser --allow-root --port=8888 --NotebookApp.token='' --NotebookApp.password='' --NotebookApp.allow_origin='*' --NotebookApp.base_url=${NB_PREFIX}"]
```

## Building docker images from Jupyter Notebook on GCP

If using Jupyter Notebooks on GKE, you can submit docker image builds to Cloud Build which builds your docker images and pushes them to Google Container Registry.

Activate the attached service account using

```
!gcloud auth activate-service-account --key-file=${GOOGLE_APPLICATION_CREDENTIALS}
```

If you have a Dockerfile in your current directory, you can submit a build using

```
!gcloud container builds submit --tag gcr.io/myproject/myimage:tag .
```

Advanced build documentation for docker images is available [here](https://cloud.google.com/cloud-build/docs/quickstart-docker)


