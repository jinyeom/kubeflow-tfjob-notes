# Kubeflow TensorFlow Training (TFJobs)

## Requirements
- Kubernetes cluster (AWS EKS)
- `kubectl`: Kubernetes CLI
- `ks`: Ksonnet CLI

## Key concepts
- Each job is represented with a single component (`${APP}/components/____.jsonnet` file)
- Each of these components can be created with `ks generate ...` which utilizes imported prototypes

## Key steps
1. Create an app and make components from prototypes
2. Set up environment (`default` for now)
3. Set up storage (PVC) for data
4. Set parameters for each job (`ks param set ...`)

## Notes from the tutorial
Tutorial page: https://github.com/kubeflow/examples/tree/master/object_detection

### Setting up
First, we're going to set up a new app via `ksonnet`.

```
$ APP=ks-app
$ ks init ${APP} --context=`kubectl config current-context`
```

For [this tutorial](https://github.com/kubeflow/examples/tree/master/object_detection) I was following, this wasn't necessary as the app
directory was provided in its github repository. Simply copy over the `ks-app` directory from the `examples` repo. Then, create a new
environment by executing `ks env add ...` (we're going to use `default` as our environment, which may or may not already exist in your setup)
and set its namespace to `kubeflow` with `ks env set default --namespace kubeflow`. In summation,

```
$ cd ${APP}
$ ENV=default
$ ks env add ${ENV} --context=`kubectl config current-context`
$ ks env set ${ENV} --namespace kubeflow
```

Now, we move onto preparing training data; to do so, we have to set up a file system (PersistentVolumeClaim, or PVC). The tutorial we're
following already has a component called `pets-pvc`, so we're going to use it; we may have to create a new one via `ks generate ...` if
we were to set this up from scratch. We'll address that later.

```
ks param set pets-pvc accessMode "ReadWriteMany"
ks param set pets-pvc storage "20Gi"
ks apply ${ENV} -c pets-pvc
```

**Note**: `accessMode "ReadWriteMany"` is not supported; use `accessMode "ReadWriteOnce"` instead.


We then retrieve data as shown below:

```
PVC="pets-pvc"
MOUNT_PATH="/pets_data"
DATASET_URL="http://www.robots.ox.ac.uk/~vgg/data/pets/data/images.tar.gz"
ANNOTATIONS_URL="http://www.robots.ox.ac.uk/~vgg/data/pets/data/annotations.tar.gz"
MODEL_URL="http://download.tensorflow.org/models/object_detection/faster_rcnn_resnet101_coco_2018_01_28.tar.gz"
PIPELINE_CONFIG_URL="https://raw.githubusercontent.com/kubeflow/examples/master/object_detection/conf/faster_rcnn_resnet101_pets.config"

ks param set get-data-job mounthPath ${MOUNT_PATH}
ks param set get-data-job pvc ${PVC}
ks param set get-data-job urlData ${DATASET_URL}
ks param set get-data-job urlAnnotations ${ANNOTATIONS_URL}
ks param set get-data-job urlModel ${MODEL_URL}
ks param set get-data-job urlPipelineConfig ${PIPELINE_CONFIG_URL}

ks apply ${ENV} -c get-data-job
```

Note that we're simply setting paths and URLs as parameters as parts of the data pipeline. In our case, this can be replaced with Quilt API.
We should pay special attention to `PIPELINE_CONFIG_URL` as we probably won't have this as a URL, but a path.

Since our data/annotations/model are downloaded as `.tar.gz` files, we should include a job for decompressing those files once downloaded.

```
ANNOTATIONS_PATH="${MOUNT_PATH}/annotations.tar.gz"
DATASET_PATH="${MOUNT_PATH}/images.tar.gz"
PRE_TRAINED_MODEL_PATH="${MOUNT_PATH}/faster_rcnn_resnet101_coco_2018_01_28.tar.gz"

ks param set decompress-data-job mountPath ${MOUNT_PATH}
ks param set decompress-data-job pvc ${PVC}
ks param set decompress-data-job pathToAnnotations ${ANNOTATIONS_PATH}
ks param set decompress-data-job pathToDataset ${DATASET_PATH}
ks param set decompress-data-job pathToModel ${PRE_TRAINED_MODEL_PATH}

ks apply ${ENV} -c decompress-data-job
```
 
The tutorial also includes a step where a job for creating `TFRecords`. Fortunately for us, we're going to be using the same format. 
Let's keep this in mind. This is done as follows,

```
OBJ_DETECTION_IMAGE="lcastell/pets_object_detection"
DATA_DIR_PATH="${MOUNT_PATH}"
OUTPUT_DIR_PATH="${MOUNT_PATH}"

ks param set create-pet-record-job image ${OBJ_DETECTION_IMAGE}
ks param set create-pet-record-job dataDirPath ${DATA_DIR_PATH}
ks param set create-pet-record-job outputDirPath ${OUTPUT_DIR_PATH}
ks param set create-pet-record-job mountPath ${MOUNT_PATH}
ks param set create-pet-record-job pvc ${PVC}

ks apply ${ENV} -c create-pet-record-job
```

Note that all these changes to parameters can be seen in `${APP}/components/params.libsonnet`.

### Submitting a training job
In this section, we deploy and launch a TensorFlow training job. You could build a Docker image for training yourself, but to save time we'll
use an existing image `lcastell/pets_object_detection`. Once again, we start with setting parameters for the training job.

```
PIPELINE_CONFIG_PATH="${MOUNT_PATH}/faster_rcnn_resnet101_pets.config"
TRAINING_DIR="${MOUNT_PATH}/train"

ks param set tf-training-job image ${OBJ_DETECTION_IMAGE}
ks param set tf-training-job mountPath ${MOUNT_PATH}
ks param set tf-training-job pvc ${PVC}
ks param set tf-training-job numPs 1
ks param set tf-training-job numWorkers 1
ks param set tf-training-job pipelineConfigPath ${PIPELINE_CONFIG_PATH}
ks param set tf-training-job trainDir ${TRAINING_DIR}

ks apply ${ENV} -c tf-training-job
```

**Note**: if you get an error: `ERROR handle object: patching object from cluster: merging object with existing state: unable to recognize 
"/tmp/ksonnet-mergepatch230068099": no matches for kind "TFJob" in version "kubeflow.org/v1alpha2"`, go into `${APP}/components/tf-training-job.jsonnet` 
and change `apiVersion: "kubeflow.org/v1alpha2"` to `apiVersion: "kubeflow.org/v1beta1"`.

Additionally, add GPU support with the following:

```
ks param set tf-training-job numGpu 1
```


