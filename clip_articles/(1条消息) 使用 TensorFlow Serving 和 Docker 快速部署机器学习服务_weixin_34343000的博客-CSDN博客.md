# (1条消息) 使用 TensorFlow Serving 和 Docker 快速部署机器学习服务_weixin_34343000的博客-CSDN博客
从实验到生产，简单快速部署机器学习模型一直是一个挑战。这个过程要做的就是将训练好的模型对外提供预测服务。在生产中，这个过程需要可重现，隔离和安全。这里，我们使用基于 Docker 的 TensorFlow Serving 来简单地完成这个过程。TensorFlow 从 1.8 版本开始支持 Docker 部署，包括 CPU 和 GPU，非常方便。

## 获得训练好的模型

获取模型的第一步当然是训练一个模型，但是这不是本篇的重点，所以我们使用一个已经训练好的模型，比如 ResNet。TensorFlow Serving 使用 SavedModel 这种格式来保存其模型，SavedModel 是一种独立于语言的，可恢复，密集的序列化格式，支持使用更高级别的系统和工具来生成，使用和转换 TensorFlow 模型。这里我们直接下载一个预训练好的模型：

````null
$ curl -s https://storage.googleapis.com/download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v2_fp32_savedmodel_NHWC_jpg.tar.gz | tar --strip-components=2 -C /tmp/resnet -xvz```

如果是使用其他框架比如Keras生成的模型，则需要将模型转换为SavedModel格式，比如：

```py
from keras.models import Sequentialfrom keras import backend as Ksignature = tf.saved_model.signature_def_utils.predict_signature_def(    inputs={'input_param': model.input}, outputs={'type': model.output})builder = tf.saved_model.builder.SavedModelBuilder('/tmp/output_model_path/1/')builder.add_meta_graph_and_variables(    tags=[tf.saved_model.tag_constants.SERVING],        tf.saved_model.signature_constants.DEFAULT_SERVING_SIGNATURE_DEF_KEY:```

下载完成后，文件目录树为：

```null
        ├── variables.data-00000-of-00001```

部署模型
----

使用Docker部署模型服务：

```null
$ docker pull tensorflow/serving$ docker run -p 8500:8500 -p 8501:8501 --name tfserving_resnet \--mount type=bind,source=/tmp/resnet,target=/models/resnet \-e MODEL_NAME=resnet -t tensorflow/serving```

其中，`8500`端口对于TensorFlow Serving提供的gRPC端口，`8501`为REST API服务端口。`-e MODEL_NAME=resnet`指出TensorFlow Serving需要加载的模型名称，这里为`resnet`。上述命令输出为

```null
2019-03-04 02:52:26.610387: I tensorflow_serving/model_servers/server.cc:82] Building single TensorFlow model file config:  model_name: resnet model_base_path: /models/resnet2019-03-04 02:52:26.618200: I tensorflow_serving/model_servers/server_core.cc:461] Adding/updating models.2019-03-04 02:52:26.618628: I tensorflow_serving/model_servers/server_core.cc:558]  (Re-)adding model: resnet2019-03-04 02:52:26.745813: I tensorflow_serving/core/basic_manager.cc:739] Successfully reserved resources to load servable {name: resnet version: 1538687457}2019-03-04 02:52:26.745901: I tensorflow_serving/core/loader_harness.cc:66] Approving load for servable version {name: resnet version: 1538687457}2019-03-04 02:52:26.745935: I tensorflow_serving/core/loader_harness.cc:74] Loading servable version {name: resnet version: 1538687457}2019-03-04 02:52:26.747590: I external/org_tensorflow/tensorflow/contrib/session_bundle/bundle_shim.cc:363] Attempting to load native SavedModelBundle in bundle-shim from: /models/resnet/15386874572019-03-04 02:52:26.747705: I external/org_tensorflow/tensorflow/cc/saved_model/reader.cc:31] Reading SavedModel from: /models/resnet/15386874572019-03-04 02:52:26.795363: I external/org_tensorflow/tensorflow/cc/saved_model/reader.cc:54] Reading meta graph with tags { serve }2019-03-04 02:52:26.828614: I external/org_tensorflow/tensorflow/core/platform/cpu_feature_guard.cc:141] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA2019-03-04 02:52:26.923902: I external/org_tensorflow/tensorflow/cc/saved_model/loader.cc:162] Restoring SavedModel bundle.2019-03-04 02:52:28.098479: I external/org_tensorflow/tensorflow/cc/saved_model/loader.cc:138] Running MainOp with key saved_model_main_op on SavedModel bundle.2019-03-04 02:52:28.144510: I external/org_tensorflow/tensorflow/cc/saved_model/loader.cc:259] SavedModel load for tags { serve }; Status: success. Took 1396689 microseconds.2019-03-04 02:52:28.146646: I tensorflow_serving/servables/tensorflow/saved_model_warmup.cc:83] No warmup data file found at /models/resnet/1538687457/assets.extra/tf_serving_warmup_requests2019-03-04 02:52:28.168063: I tensorflow_serving/core/loader_harness.cc:86] Successfully loaded servable version {name: resnet version: 1538687457}2019-03-04 02:52:28.174902: I tensorflow_serving/model_servers/server.cc:286] Running gRPC ModelServer at 0.0.0.0:8500 ...[warn] getaddrinfo: address family for nodename not supported2019-03-04 02:52:28.186724: I tensorflow_serving/model_servers/server.cc:302] Exporting HTTP/REST API at:localhost:8501 ...[evhttp_server.cc : 237] RAW: Entering the event loop ...```

我们可以看到，TensorFlow Serving使用`1538687457`作为模型的版本号。我们使用curl命令来查看一下启动的服务状态，也可以看到提供服务的模型版本以及模型状态。

```null
$ curl http://localhost:8501/v1/models/resnet "model_version_status": [```

查看模型输入输出
--------

很多时候我们需要查看模型的输出和输出参数的具体形式，TensorFlow提供了一个`saved_model_cli`命令来查看模型的输入和输出参数：

```null
$ saved_model_cli show --dir /tmp/resnet/1538687457/ --allMetaGraphDef with tag-set: 'serve' contains the following SignatureDefs:signature_def['predict']:  The given SavedModel SignatureDef contains the following input(s):    inputs['image_bytes'] tensor_info:  The given SavedModel SignatureDef contains the following output(s):    outputs['classes'] tensor_info:    outputs['probabilities'] tensor_info:  Method name is: tensorflow/serving/predictsignature_def['serving_default']:  The given SavedModel SignatureDef contains the following input(s):    inputs['image_bytes'] tensor_info:  The given SavedModel SignatureDef contains the following output(s):    outputs['classes'] tensor_info:    outputs['probabilities'] tensor_info:  Method name is: tensorflow/serving/predict```

注意到`signature_def`，`inputs`的名称，类型和输出，这些参数在接下来的模型预测请求中需要。

使用模型接口预测：REST和gRPC
------------------

TensorFlow Serving提供REST API和gRPC两种请求方式，接下来将具体这两种方式。

### REST

我们下载一个客户端脚本，这个脚本会下载一张猫的图片，同时使用这张图片来计算服务请求时间。

```null
$ curl -o /tmp/resnet/resnet_client.py https://raw.githubusercontent.com/tensorflow/serving/master/tensorflow_serving/example/resnet_client.py```

以下脚本使用`requests`库来请求接口，使用图片的base64编码字符串作为请求内容，返回图片分类，并计算了平均处理时间。

```py
from __future__ import print_functionSERVER_URL = 'http://localhost:8501/v1/models/resnet:predict'IMAGE_URL = 'https://tensorflow.org/images/blogs/serving/cat.jpg'  dl_request = requests.get(IMAGE_URL, stream=True)  dl_request.raise_for_status()  jpeg_bytes = base64.b64encode(dl_request.content).decode('utf-8')  predict_request = '{"instances" : [{"b64": "%s"}]}' % jpeg_bytes    response = requests.post(SERVER_URL, data=predict_request)    response.raise_for_status()for _ in range(num_requests):    response = requests.post(SERVER_URL, data=predict_request)    response.raise_for_status()    total_time += response.elapsed.total_seconds()    prediction = response.json()['predictions'][0]  print('Prediction class: {}, avg latency: {} ms'.format(      prediction['classes'], (total_time*1000)/num_requests))if __name__ == '__main__':```

输出结果为

```null
$ python resnet_client.pyPrediction class: 286, avg latency: 210.12310000000002 ms```

### gRPC

让我们下载另一个客户端脚本，这个脚本使用gRPC作为服务，传入图片并获取输出结果。这个脚本需要安装`tensorflow-serving-api`这个库。

```null
$ curl -o /tmp/resnet/resnet_client_grpc.py https://raw.githubusercontent.com/tensorflow/serving/master/tensorflow_serving/example/resnet_client_grpc.py$ pip install tensorflow-serving-api```

脚本内容：

```py
from __future__ import print_functionfrom tensorflow_serving.apis import predict_pb2from tensorflow_serving.apis import prediction_service_pb2_grpcIMAGE_URL = 'https://tensorflow.org/images/blogs/serving/cat.jpg'tf.app.flags.DEFINE_string('server', 'localhost:8500','PredictionService host:port')tf.app.flags.DEFINE_string('image', '', 'path to image in JPEG format')FLAGS = tf.app.flags.FLAGSwith open(FLAGS.image, 'rb') as f:    dl_request = requests.get(IMAGE_URL, stream=True)    dl_request.raise_for_status()    data = dl_request.content  channel = grpc.insecure_channel(FLAGS.server)  stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)  request = predict_pb2.PredictRequest()  request.model_spec.name = 'resnet'  request.model_spec.signature_name = 'serving_default'  request.inputs['image_bytes'].CopyFrom(      tf.contrib.util.make_tensor_proto(data, shape=[1]))  result = stub.Predict(request, 10.0)  if __name__ == '__main__':```

输出的结果可以看到图片的分类，概率和使用的模型信息：

```null
$ python resnet_client_grpc.py    float_val: 2.4162832232832443e-06    float_val: 1.9012182974620373e-06    float_val: 2.7247710022493266e-05    float_val: 4.426385658007348e-07    float_val: 1.4636580090154894e-05    float_val: 5.812107133351674e-07    float_val: 6.599806511076167e-05    float_val: 0.0012952701654285192  signature_name: "serving_default"```

性能
--

### 通过编译优化的TensorFlow Serving二进制来提高性能

TensorFlows serving有时会有输出如下的日志：

```null
Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA```

TensorFlow Serving已发布Docker镜像旨在尽可能多地使用CPU架构，因此省略了一些优化以最大限度地提高兼容性。如果你没有看到此消息，则你的二进制文件可能已针对你的CPU进行了优化。根据你的模型执行的操作，这些优化可能会对你的服务性能产生重大影响。幸运的是，编译优化的TensorFlow Serving二进制非常简单。官方已经提供了自动化脚本，分以下两部进行：

```null
$ docker build -t $USER/tensorflow-serving-devel -f Dockerfile.devel https://github.com/tensorflow/serving.git$ docker build -t $USER/tensorflow-serving --build-arg TF_SERVING_BUILD_IMAGE=$USER/tensorflow-serving-devel https://github.com/tensorflow/serving.git```

之后，使用新编译的`$USER/tensorflow-serving`重新启动服务即可。

总结
--

上面我们快速实践了使用TensorFlow Serving和Docker部署机器学习服务的过程，可以看到，TensorFlow Serving提供了非常方便和高效的模型管理，配合Docker，可以快速搭建起机器学习服务。

参考
--

*   [Serving ML Quickly with TensorFlow Serving and Docker](https://link.juejin.im/?target=https%3A%2F%2Fmedium.com%2Ftensorflow%2Fserving-ml-quickly-with-tensorflow-serving-and-docker-7df7094aa008)
*   [Train and serve a TensorFlow model with TensorFlow Serving](https://link.juejin.im/?target=https%3A%2F%2Fwww.tensorflow.org%2Ftfx%2Fserving%2Ftutorials%2FServing_REST_simple)

> GitHub repo: [qiwihui/blog](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fqiwihui%2Fblog)
> 
> Follow me: [@qiwihui](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fqiwihui)
> 
> Site: [QIWIHUI](https://link.juejin.im/?target=https%3A%2F%2Fqiwihui.com) 
 [https://blog.csdn.net/weixin_34343000/article/details/88118667](https://blog.csdn.net/weixin_34343000/article/details/88118667)
````
