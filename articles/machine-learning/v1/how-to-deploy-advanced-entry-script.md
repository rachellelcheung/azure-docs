---
title: Entry script authoring for advanced scenarios
titleSuffix: Azure Machine Learning
description: Learn how to write Azure Machine Learning entry scripts for pre- and post-processing during deployment.
services: machine-learning
ms.service: machine-learning
ms.subservice: mlops
ms.topic: how-to
ms.date: 03/12/2024
author: msakande
ms.author: mopeakande
ms.reviewer: sehan
ms.custom: UpdateFrequency5, deploy, sdkv1
---

# Advanced entry script authoring

[!INCLUDE [sdk v1](../includes/machine-learning-sdk-v1.md)]

This article explains how to write entry scripts for specialized use cases.

## Prerequisites

This article assumes you already have a trained machine learning model that you intend to deploy with Azure Machine Learning. To learn more about model deployment, see [Deploy machine learning models to Azure](how-to-deploy-and-where.md).

## Automatically generate a Swagger schema

To automatically generate a schema for your web service, provide a sample of the input and/or output in the constructor for one of the defined type objects. The type and sample are used to automatically create the schema. Azure Machine Learning then creates an [OpenAPI specification](https://swagger.io/docs/specification/about/) (formerly, Swagger specification) for the web service during deployment.

> [!WARNING]
> You must not use sensitive or private data for sample input or output. The Swagger page for AML-hosted inferencing exposes the sample data.

These types are currently supported:

* `pandas`
* `numpy`
* `pyspark`
* Standard Python object

To use schema generation, include the open-source `inference-schema` package version 1.1.0 or above in your dependencies file. For more information on this package, see [InferenceSchema on GitHub](https://github.com/Azure/InferenceSchema). In order to generate conforming Swagger for automated web service consumption, scoring script run() function must have API shape of:
* A first parameter of type `StandardPythonParameterType`, named *Inputs* and nested
* An optional second parameter of type `StandardPythonParameterType`, named *GlobalParameters*
* Return a dictionary of type `StandardPythonParameterType`, named *Results* and nested

Define the input and output sample formats in the `input_sample` and `output_sample` variables, which represent the request and response formats for the web service. Use these samples in the input and output function decorators on the `run()` function. The following scikit-learn example uses schema generation.

## Power BI compatible endpoint

The following example demonstrates how to define API shape according to preceding instruction. This method is supported for consuming the deployed web service from Power BI.

```python
import json
import pickle
import numpy as np
import pandas as pd
import azureml.train.automl
from sklearn.externals import joblib
from sklearn.linear_model import Ridge

from inference_schema.schema_decorators import input_schema, output_schema
from inference_schema.parameter_types.standard_py_parameter_type import StandardPythonParameterType
from inference_schema.parameter_types.numpy_parameter_type import NumpyParameterType
from inference_schema.parameter_types.pandas_parameter_type import PandasParameterType


def init():
    global model
    # Replace filename if needed.
    model_path = os.path.join(os.getenv('AZUREML_MODEL_DIR'), 'sklearn_regression_model.pkl')
    # Deserialize the model file back into a sklearn model.
    model = joblib.load(model_path)


# providing 3 sample inputs for schema generation
numpy_sample_input = NumpyParameterType(np.array([[1,2,3,4,5,6,7,8,9,10],[10,9,8,7,6,5,4,3,2,1]],dtype='float64'))
pandas_sample_input = PandasParameterType(pd.DataFrame({'name': ['Sarah', 'John'], 'age': [25, 26]}))
standard_sample_input = StandardPythonParameterType(0.0)

# This is a nested input sample, any item wrapped by `ParameterType` will be described by schema
sample_input = StandardPythonParameterType({'input1': numpy_sample_input, 
                                        'input2': pandas_sample_input, 
                                        'input3': standard_sample_input})

sample_global_parameters = StandardPythonParameterType(1.0) # this is optional
sample_output = StandardPythonParameterType([1.0, 1.0])
outputs = StandardPythonParameterType({'Results':sample_output}) # 'Results' is case sensitive

@input_schema('Inputs', sample_input) 
# 'Inputs' is case sensitive

@input_schema('GlobalParameters', sample_global_parameters) 
# this is optional, 'GlobalParameters' is case sensitive

@output_schema(outputs)

def run(Inputs, GlobalParameters): 
    # the parameters here have to match those in decorator, both 'Inputs' and 
    # 'GlobalParameters' here are case sensitive
    try:
        data = Inputs['input1']
        # data will be convert to target format
        assert isinstance(data, np.ndarray)
        result = model.predict(data)
        return result.tolist()
    except Exception as e:
        error = str(e)
        return error
```

> [!TIP]
> The return value from the script can be any Python object that is serializable to JSON. For example, if your model returns a Pandas dataframe that contains multiple columns, you might use an output decorator similar to the following code:
> 
> ```python
> output_sample = pd.DataFrame(data=[{"a1": 5, "a2": 6}])
> @output_schema(PandasParameterType(output_sample))
> ...
> result = model.predict(data)
> return result
> ```

## <a id="binary-data"></a> Binary (that is, image) data

If your model accepts binary data, like an image, you must modify the *score.py* file used for your deployment to accept raw HTTP requests. To accept raw data, use the `AMLRequest` class in your entry script and add the `@rawhttp` decorator to the `run()` function.

Here's an example of a `score.py` that accepts binary data:

```python
from azureml.contrib.services.aml_request import AMLRequest, rawhttp
from azureml.contrib.services.aml_response import AMLResponse
from PIL import Image
import json


def init():
    print("This is init()")
    

@rawhttp
def run(request):
    print("This is run()")
    
    if request.method == 'GET':
        # For this example, just return the URL for GETs.
        respBody = str.encode(request.full_path)
        return AMLResponse(respBody, 200)
    elif request.method == 'POST':
        file_bytes = request.files["image"]
        image = Image.open(file_bytes).convert('RGB')
        # For a real-world solution, you would load the data from reqBody
        # and send it to the model. Then return the response.

        # For demonstration purposes, this example just returns the size of the image as the response.
        return AMLResponse(json.dumps(image.size), 200)
    else:
        return AMLResponse("bad request", 500)
```

> [!IMPORTANT]
> The `AMLRequest` class is in the `azureml.contrib` namespace. Entities in this namespace change frequently as we work to improve the service. Anything in this namespace should be considered a preview that's not fully supported by Microsoft.
>
> If you need to test this in your local development environment, you can install the components by using the following command:
>
> ```shell
> pip install azureml-contrib-services
> ```

> [!NOTE]
> *500* is not recommended as a customed status code, as at azureml-fe side, the status code will be rewritten to *502*.
>   * The status code is passed through the azureml-fe, then sent to client.
>   * The azureml-fe only rewrites the 500 returned from the model side to be 502, the client receives 502.
>   * But if the azureml-fe itself returns 500, client side still receives 500.

The `AMLRequest` class only allows you to access the raw posted data in the *score.py* file, there's no client-side component. From a client, you post data as normal. For example, the following Python code reads an image file and posts the data:

```python
import requests

uri = service.scoring_uri
image_path = 'test.jpg'
files = {'image': open(image_path, 'rb').read()}
response = requests.post(uri, files=files)

print(response.json)
```

<a id="cors"></a>

## Cross-origin resource sharing (CORS)

Cross-origin resource sharing is a way to allow resources on a webpage to be requested from another domain. CORS works via HTTP headers sent with the client request and returned with the service response. For more information on CORS and valid headers, see [Cross-origin resource sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) in Wikipedia.

To configure your model deployment to support CORS, use the `AMLResponse` class in your entry script. This class allows you to set the headers on the response object.

The following example sets the `Access-Control-Allow-Origin` header for the response from the entry script:

```python
from azureml.contrib.services.aml_request import AMLRequest, rawhttp
from azureml.contrib.services.aml_response import AMLResponse


def init():
    print("This is init()")

@rawhttp
def run(request):
    print("This is run()")
    print("Request: [{0}]".format(request))
    if request.method == 'GET':
        # For this example, just return the URL for GET.
        # For a real-world solution, you would load the data from URL params or headers
        # and send it to the model. Then return the response.
        respBody = str.encode(request.full_path)
        resp = AMLResponse(respBody, 200)
        resp.headers["Allow"] = "OPTIONS, GET, POST"
        resp.headers["Access-Control-Allow-Methods"] = "OPTIONS, GET, POST"
        resp.headers['Access-Control-Allow-Origin'] = "http://www.example.com"
        resp.headers['Access-Control-Allow-Headers'] = "*"
        return resp
    elif request.method == 'POST':
        reqBody = request.get_data(False)
        # For a real-world solution, you would load the data from reqBody
        # and send it to the model. Then return the response.
        resp = AMLResponse(reqBody, 200)
        resp.headers["Allow"] = "OPTIONS, GET, POST"
        resp.headers["Access-Control-Allow-Methods"] = "OPTIONS, GET, POST"
        resp.headers['Access-Control-Allow-Origin'] = "http://www.example.com"
        resp.headers['Access-Control-Allow-Headers'] = "*"
        return resp
    elif request.method == 'OPTIONS':
        resp = AMLResponse("", 200)
        resp.headers["Allow"] = "OPTIONS, GET, POST"
        resp.headers["Access-Control-Allow-Methods"] = "OPTIONS, GET, POST"
        resp.headers['Access-Control-Allow-Origin'] = "http://www.example.com"
        resp.headers['Access-Control-Allow-Headers'] = "*"
        return resp
    else:
        return AMLResponse("bad request", 400)
```

> [!IMPORTANT]
> The `AMLResponse` class is in the `azureml.contrib` namespace. Entities in this namespace change frequently as we work to improve the service. Anything in this namespace should be considered a preview that's not fully supported by Microsoft.
>
> If you need to test this in your local development environment, you can install the components by using the following command:
>
> ```shell
> pip install azureml-contrib-services
> ```

> [!WARNING]
> Azure Machine Learning only routes POST and GET requests to the containers running the scoring service. This can cause errors due to browsers using OPTIONS requests to pre-flight CORS requests.
>

## Load registered models

There are two ways to locate models in your entry script:
* `AZUREML_MODEL_DIR`: An environment variable containing the path to the model location
* `Model.get_model_path`: An API that returns the path to model file using the registered model name

#### AZUREML_MODEL_DIR

`AZUREML_MODEL_DIR` is an environment variable created during service deployment. You can use this environment variable to find the location of the deployed model(s).

The following table describes the value of `AZUREML_MODEL_DIR` depending on the number of models deployed:

| Deployment | Environment variable value |
| ----- | ----- |
| Single model | The path to the folder containing the model. |
| Multiple models | The path to the folder containing all models. Models are located by name and version in this folder (`$MODEL_NAME/$VERSION`) |

During model registration and deployment, Models are placed in the AZUREML_MODEL_DIR path, and their original filenames are preserved.

To get the path to a model file in your entry script, combine the environment variable with the file path you're looking for.

**Single model example**
```python
# Example when the model is a file
model_path = os.path.join(os.getenv('AZUREML_MODEL_DIR'), 'sklearn_regression_model.pkl')

# Example when the model is a folder containing a file
file_path = os.path.join(os.getenv('AZUREML_MODEL_DIR'), 'my_model_folder', 'sklearn_regression_model.pkl')
```

**Multiple model example**

In this scenario, two models are registered with the workspace:

* `my_first_model`: Contains one file (`my_first_model.pkl`) and there's only one version, `1`
* `my_second_model`: Contains one file (`my_second_model.pkl`) and there are two versions, `1` and `2`

When the service was deployed, both models are provided in the deploy operation:

```python
first_model = Model(ws, name="my_first_model", version=1)
second_model = Model(ws, name="my_second_model", version=2)
service = Model.deploy(ws, "myservice", [first_model, second_model], inference_config, deployment_config)
```

In the Docker image that hosts the service, the `AZUREML_MODEL_DIR` environment variable contains the directory where the models are located. In this directory, each of the models is located in a directory path of `MODEL_NAME/VERSION`. Where `MODEL_NAME` is the name of the registered model, and `VERSION` is the version of the model. The files that make up the registered model are stored in these directories.

In this example, the paths would be `$AZUREML_MODEL_DIR/my_first_model/1/my_first_model.pkl` and `$AZUREML_MODEL_DIR/my_second_model/2/my_second_model.pkl`.

```python
# Example when the model is a file, and the deployment contains multiple models
first_model_name = 'my_first_model'
first_model_version = '1'
first_model_path = os.path.join(os.getenv('AZUREML_MODEL_DIR'), first_model_name, first_model_version, 'my_first_model.pkl')
second_model_name = 'my_second_model'
second_model_version = '2'
second_model_path = os.path.join(os.getenv('AZUREML_MODEL_DIR'), second_model_name, second_model_version, 'my_second_model.pkl')
```

### get_model_path

When you register a model, you provide a model name that's used for managing the model in the registry. You use this name with the [Model.get_model_path()](/python/api/azureml-core/azureml.core.model.model#azureml-core-model-model-get-model-path) method to retrieve the path of the model file or files on the local file system. If you register a folder or a collection of files, this API returns the path of the directory that contains those files.

When you register a model, you give it a name. The name corresponds to where the model is placed, either locally or during service deployment.

## Framework-specific examples

See the following articles for more entry script examples for specific machine learning use cases:

* [PyTorch](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/ml-frameworks/pytorch)
* [TensorFlow](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/ml-frameworks/tensorflow)
* [Keras](https://github.com/Azure/MachineLearningNotebooks/blob/master/how-to-use-azureml/ml-frameworks/keras/train-hyperparameter-tune-deploy-with-keras/train-hyperparameter-tune-deploy-with-keras.ipynb)
* [AutoML](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/automated-machine-learning/classification-bank-marketing-all-features)

## Related content

* [Troubleshooting remote model deployment](how-to-troubleshoot-deployment.md)
* [Deploy a model to an Azure Kubernetes Service cluster with v1](how-to-deploy-azure-kubernetes-service.md)
* [Consume an Azure Machine Learning model deployed as a web service](how-to-consume-web-service.md)
* [Update a deployed web service (v1)](how-to-deploy-update-web-service.md)
* [Use a custom container to deploy a model to an online endpoint](../how-to-deploy-custom-container.md)
* [Use TLS to secure a web service through Azure Machine Learning](how-to-secure-web-service.md)
* [Monitor and collect data from ML web service endpoints](how-to-enable-app-insights.md)
* [Collect data from models in production](how-to-enable-data-collection.md)
* [Trigger applications, processes, or CI/CD workflows based on Azure Machine Learning events](../how-to-use-event-grid.md)
