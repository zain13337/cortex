# Handler implementation

Realtime APIs respond to requests in real-time and autoscale based on in-flight request volumes. They can be used for realtime inference or data processing workloads.

If you plan on deploying ML models and run realtime inferences, check out the [Models](models.md) page. Cortex provides out-of-the-box support for a variety of frameworks such as: PyTorch, ONNX, scikit-learn, XGBoost, TensorFlow, etc.

The response type of the handler can vary depending on your requirements, see [HTTP API responses](#http-responses) and [gRPC API responses](#grpc-responses) below.

## Project files

Cortex makes all files in the project directory (i.e. the directory which contains `cortex.yaml`) available for use in your Handler class implementation. Python bytecode files (`*.pyc`, `*.pyo`, `*.pyd`), files or folders that start with `.`, and the api configuration file (e.g. `cortex.yaml`) are excluded.

The following files can also be added at the root of the project's directory:

* `.cortexignore` file, which follows the same syntax and behavior as a [.gitignore file](https://git-scm.com/docs/gitignore).
* `.env` file, which exports environment variables that can be used in the handler. Each line of this file must follow the `VARIABLE=value` format.

For example, if your directory looks like this:

```text
./my-classifier/
├── cortex.yaml
├── values.json
├── handler.py
├── ...
└── requirements.txt
```

You can access `values.json` in your handler class like this:

```python
# handler.py

import json

class Handler:
    def __init__(self, config):
        with open('values.json', 'r') as values_file:
            values = json.load(values_file)
        self.values = values
```

## HTTP

### Handler

```python
# initialization code and variables can be declared here in global scope

class Handler:
    def __init__(self, config):
        """(Required) Called once before the API becomes available. Performs
        setup such as downloading/initializing the model or downloading a
        vocabulary.

        Args:
            config (required): Dictionary passed from API configuration (if
                specified). This may contain information on where to download
                the model and/or metadata.
        """
        pass

    def handle_<HTTP-VERB>(self, payload, query_params, headers):
        """(Required) Called once per request. Preprocesses the request payload
        (if necessary), runs workload, and postprocesses the workload output
        (if necessary).

        Args:
            payload (optional): The request payload (see below for the possible
                payload types).
            query_params (optional): A dictionary of the query parameters used
                in the request.
            headers (optional): A dictionary of the headers sent in the request.

        Returns:
            Result or a batch of results.
        """
        pass
```

Your `Handler` class can implement methods for each of the following HTTP methods: POST, GET, PUT, PATCH, DELETE. Therefore, the respective methods in the `Handler` definition can be `handle_post`, `handle_get`, `handle_put`, `handle_patch`, and `handle_delete`.

For proper separation of concerns, it is recommended to use the constructor's `config` parameter for information such as from where to download the model and initialization files, or any configurable model parameters. You define `config` in your API configuration, and it is passed through to your Handler's constructor.

Your API can accept requests with different types of payloads such as `JSON`-parseable, `bytes` or `starlette.datastructures.FormData` data. See [HTTP API requests](#http-requests) to learn about how headers can be used to change the type of `payload` that is passed into your handler method.

Your handler method can return different types of objects such as `JSON`-parseable, `string`, and `bytes` objects. See [HTTP API responses](#http-responses) to learn about how to configure your handler method to respond with different response codes and content-types.

### Callbacks

A callback is a function that starts running in the background after the results have been sent back to the client. They are meant to be short-lived.

Each handler method of your class can implement callbacks. To do this, when returning the result(s) from your handler method, also make sure to return a 2-element tuple in which the first element are your results that you want to return and the second element is a callable object that takes no arguments.

You can implement a callback like in the following example:

```python
def handle_post(self, payload):
    def _callback():
        print("message that gets printed after the response is sent back to the user")
    return "results", _callback
```

### HTTP requests

The type of the `payload` parameter in `handle_<HTTP-VERB>(self, payload)` can vary based on the content type of the request. The `payload` parameter is parsed according to the `Content-Type` header in the request. Here are the parsing rules (see below for examples):

1. For `Content-Type: application/json`, `payload` will be the parsed JSON body.
1. For `Content-Type: multipart/form-data` / `Content-Type: application/x-www-form-urlencoded`, `payload` will be `starlette.datastructures.FormData` (key-value pairs where the values are strings for text data, or `starlette.datastructures.UploadFile` for file uploads; see [Starlette's documentation](https://www.starlette.io/requests/#request-files)).
1. For `Content-Type: text/plain`, `payload` will be a string. `utf-8` encoding is assumed, unless specified otherwise (e.g. via `Content-Type: text/plain; charset=us-ascii`)
1. For all other `Content-Type` values, `payload` will be the raw `bytes` of the request body.

Here are some examples:

#### JSON data

##### Making the request

```bash
curl http://***.amazonaws.com/my-api \
    -X POST -H "Content-Type: application/json" \
    -d '{"key": "value"}'
```

##### Reading the payload

When sending a JSON payload, the `payload` parameter will be a Python object:

```python
class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload):
        print(payload["key"])  # prints "value"
```

#### Binary data

##### Making the request

```bash
curl http://***.amazonaws.com/my-api \
    -X POST -H "Content-Type: application/octet-stream" \
    --data-binary @object.pkl
```

##### Reading the payload

Since the `Content-Type: application/octet-stream` header is used, the `payload` parameter will be a `bytes` object:

```python
import pickle

class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload):
        obj = pickle.loads(payload)
        print(obj["key"])  # prints "value"
```

Here's an example if the binary data is an image:

```python
from PIL import Image
import io

class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload, headers):
        img = Image.open(io.BytesIO(payload))  # read the payload bytes as an image
        print(img.size)
```

#### Form data (files)

##### Making the request

```bash
curl http://***.amazonaws.com/my-api \
    -X POST \
    -F "text=@text.txt" \
    -F "object=@object.pkl" \
    -F "image=@image.png"
```

##### Reading the payload

When sending files via form data, the `payload` parameter will be `starlette.datastructures.FormData` (key-value pairs where the values are `starlette.datastructures.UploadFile`, see [Starlette's documentation](https://www.starlette.io/requests/#request-files)). Either `Content-Type: multipart/form-data` or `Content-Type: application/x-www-form-urlencoded` can be used (typically `Content-Type: multipart/form-data` is used for files, and is the default in the examples above).

```python
from PIL import Image
import pickle

class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload):
        text = payload["text"].file.read()
        print(text.decode("utf-8"))  # prints the contents of text.txt

        obj = pickle.load(payload["object"].file)
        print(obj["key"])  # prints "value" assuming `object.pkl` is a pickled dictionary {"key": "value"}

        img = Image.open(payload["image"].file)
        print(img.size)  # prints the dimensions of image.png
```

#### Form data (text)

##### Making the request

```bash
curl http://***.amazonaws.com/my-api \
    -X POST \
    -d "key=value"
```

##### Reading the payload

When sending text via form data, the `payload` parameter will be `starlette.datastructures.FormData` (key-value pairs where the values are strings, see [Starlette's documentation](https://www.starlette.io/requests/#request-files)). Either `Content-Type: multipart/form-data` or `Content-Type: application/x-www-form-urlencoded` can be used (typically `Content-Type: application/x-www-form-urlencoded` is used for text, and is the default in the examples above).

```python
class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload):
        print(payload["key"])  # will print "value"
```

#### Text data

##### Making the request

```bash
curl http://***.amazonaws.com/my-api \
    -X POST -H "Content-Type: text/plain" \
    -d "hello world"
```

##### Reading the payload

Since the `Content-Type: text/plain` header is used, the `payload` parameter will be a `string` object:

```python
class Handler:
    def __init__(self, config):
        pass

    def handle_post(self, payload):
        print(payload)  # prints "hello world"
```

### HTTP responses

The response of your `handle_<HTTP-VERB>()` method may be:

1. A JSON-serializable object (*lists*, *dictionaries*, *numbers*, etc.)

1. A `string` object (e.g. `"class 1"`)

1. A `bytes` object (e.g. `bytes(4)` or `pickle.dumps(obj)`)

1. An instance of [starlette.responses.Response](https://www.starlette.io/responses/#response)

## gRPC

To serve your API using the gRPC protocol, make sure the `handler.protobuf_path` field in your API configuration is pointing to a protobuf file. When the API gets deployed, Cortex will compile the protobuf file for its use when serving the API.

### Python Handler

#### Interface

```python
# initialization code and variables can be declared here in global scope

class Handler:
    def __init__(self, config, module_proto_pb2):
        """(Required) Called once before the API becomes available. Performs
        setup such as downloading/initializing the model or downloading a
        vocabulary.

        Args:
            config (required): Dictionary passed from API configuration (if
                specified). This may contain information on where to download
                the model and/or metadata.
            module_proto_pb2 (required): Loaded Python module containing the
                class definitions of the messages defined in the protobuf
                file (`handler.protobuf_path`).
        """
        self.module_proto_pb2 = module_proto_pb2

    def <RPC-METHOD-NAME>(self, payload, context):
        """(Required) Called once per request. Preprocesses the request payload
        (if necessary), runs workload, and postprocesses the workload output
        (if necessary).

        Args:
            payload (optional): The request payload (see below for the possible
                payload types).
            context (optional): gRPC context.

        Returns:
            Result (when streaming is not used).

        Yield:
            Result (when streaming is used).
        """
        pass
```

Your `Handler` class must implement the RPC methods found in the protobuf. Your protobuf must have a single service defined, which can have any name. If your service has 2 RPC methods called `Info` and `Predict` methods, then your `Handler` class must also implement these methods like in the above `Handler` template.

For proper separation of concerns, it is recommended to use the constructor's `config` parameter for information such as from where to download the model and initialization files, or any configurable model parameters. You define `config` in your API configuration, and it is passed through to your Handler class' constructor.

Your API can only accept the type that has been specified in the protobuf definition of your service's method. See [gRPC API requests](#grpc-requests) for how to construct gRPC requests.

Your handler method(s) can only return the type that has been specified in the protobuf definition of your service's method(s). See [gRPC API responses](#grpc-responses) for how to handle gRPC responses.

### gRPC requests

Assuming the following service:

```protobuf
# handler.proto

syntax = "proto3";
package sample_service;

service Handler {
    rpc Predict (Sample) returns (Response);
}

message Sample {
    string a = 1;
}

message Response {
    string b = 1;
}
```

The handler implementation will also have a corresponding `Predict` method defined that represents the RPC method in the above protobuf service. The name(s) of the RPC method(s) is not enforced by Cortex.

The type of the `payload` parameter passed into `Predict(self, payload)` will match that of the `Sample` message defined in the `handler.protobuf_path` file. For this example, we'll assume that the above protobuf file was specified for the API.

#### Simple request

The service method must look like this:

```protobuf
...
rpc Predict (Sample) returns (Response);
...
```

##### Making the request

```python
import grpc, handler_pb2, handler_pb2_grpc

stub = handler_pb2_grpc.HandlerStub(grpc.insecure_channel("***.amazonaws.com:80"))
stub.Predict(handler_pb2.Sample(a="text"))
```

##### Reading the payload

In the `Predict` method, you'll read the value like this:

```python
...
def Predict(self, payload):
    print(payload.a)
...
```

#### Streaming request

The service method must look like this:

```protobuf
...
rpc Predict (stream Sample) returns (Response);
...
```

##### Making the request

```python
import grpc, handler_pb2, handler_pb2_grpc

def generate_iterator(sample_list):
    for sample in sample_list:
        yield sample

stub = handler_pb2_grpc.HandlerStub(grpc.insecure_channel("***.amazonaws.com:80"))
stub.Predict(handler_pb2.Sample(generate_iterator(["a", "b", "c", "d"])))
```

##### Reading the payload

In the `Predict` method, you'll read the streamed values like this:

```python
...
def Predict(self, payload):
    for item in payload:
        print(item.a)
...
```

### gRPC responses

Assuming the following service:

```protobuf
# handler.proto

syntax = "proto3";
package sample_service;

service Handler {
    rpc Predict (Sample) returns (Response);
}

message Sample {
    string a = 1;
}

message Response {
    string b = 1;
}
```

The handler implementation will also have a corresponding `Predict` method defined that represents the RPC method in the above protobuf service. The name(s) of the RPC method(s) is not enforced by Cortex.

The type of the value that you return in your `Predict()` method must match the `Response` message defined in the `handler.protobuf_path` file. For this example, we'll assume that the above protobuf file was specified for the API.

#### Simple response

The service method must look like this:

```protobuf
...
rpc Predict (Sample) returns (Response);
...
```

##### Making the request

```python
import grpc, handler_pb2, handler_pb2_grpc

stub = handler_pb2_grpc.HandlerStub(grpc.insecure_channel("***.amazonaws.com:80"))
r = stub.Predict(handler_pb2.Sample())
```

##### Returning the response

In the `Predict` method, you'll return the value like this:

```python
...
def Predict(self, payload):
    return self.proto_module_pb2.Response(b="text")
...
```

#### Streaming response

The service method must look like this:

```protobuf
...
rpc Predict (Sample) returns (stream Response);
...
```

##### Making the request

```python
import grpc, handler_pb2, handler_pb2_grpc

def generate_iterator(sample_list):
    for sample in sample_list:
        yield sample

stub = handler_pb2_grpc.HandlerStub(grpc.insecure_channel("***.amazonaws.com:80"))
for r in stub.Predict(handler_pb2.Sample())):
    print(r.b)
```

##### Returning the response

In the `Predict` method, you'll return the streamed values like this:

```python
...
def Predict(self, payload):
    for text in ["a", "b", "c", "d"]:
        yield self.proto_module_pb2.Response(b=text)
...
```

## Chaining APIs

It is possible to make requests from one API to another within a Cortex cluster. All running APIs are accessible from within the handler implementation at `http://api-<api_name>:8888/`, where `<api_name>` is the name of the API you are making a request to.

For example, if there is an api named `text-generator` running in the cluster, you could make a request to it from a different API by using:

```python
import requests

class Handler:
    def handle_post(self, payload):
        response = requests.post("http://api-text-generator:8888/", json={"text": "machine learning is"})
        # ...
```

Note that the autoscaling configuration (i.e. `target_replica_concurrency`) for the API that is making the request should be modified with the understanding that requests will still be considered "in-flight" with the first API as the request is being fulfilled in the second API (during which it will also be considered "in-flight" with the second API).

## Structured logging

You can use Cortex's logger in your handler implemention to log in JSON. This will enrich your logs with Cortex's metadata, and you can add custom metadata to the logs by adding key value pairs to the `extra` key when using the logger. For example:

```python
...
from cortex_internal.lib.log import logger as cortex_logger

class Handler:
    def handle_post(self, payload):
        cortex_logger.info("received payload", extra={"payload": payload})
```

The dictionary passed in via the `extra` will be flattened by one level. e.g.

```text
{"asctime": "2021-01-19 15:14:05,291", "levelname": "INFO", "message": "received payload", "process": 235, "payload": "this movie is awesome"}
```

To avoid overriding essential Cortex metadata, please refrain from specifying the following extra keys: `asctime`, `levelname`, `message`, `labels`, and `process`. Log lines greater than 5 MB in size will be ignored.

## Cortex Python client

A default [Cortex Python client](../../clients/python.md#cortex.client.client) environment has been configured for your API. This can be used for deploying/deleting/updating or submitting jobs to your running cluster based on the execution flow of your handler. For example:

```python
import cortex

class Handler:
    def __init__(self, config):
        ...
        # get client pointing to the default environment
        client = cortex.client()
        # get the existing apis in the cluster for something important to you
        existing_apis = client.list_apis()
```