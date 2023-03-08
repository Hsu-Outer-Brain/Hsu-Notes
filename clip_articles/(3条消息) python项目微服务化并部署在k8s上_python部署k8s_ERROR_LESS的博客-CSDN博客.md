# (3条消息) python项目微服务化并部署在k8s上_python部署k8s_ERROR_LESS的博客-CSDN博客
[(3 条消息) python 项目微服务化并部署在 k8s 上\_python 部署 k8s_ERROR_LESS 的博客 - CSDN 博客](https://blog.csdn.net/qq_47058489/article/details/122111549) 

 创建 tensorflow-gpu 虚拟环境，参考[这篇博客](https://editor.csdn.net/md/?articleId=122069956)

```python

root@master:/home/hqc


(tf) root@master:/home/hqc
(tf) root@master:/home/hqc/自然基金项目/Federated


(tf) root@master:/home/hqc/自然基金项目/Federated



(tf) root@master:/home/hqc/自然基金项目/Federated
(tf) root@master:/home/hqc/自然基金项目/Federated


(tf) root@master:/home/hqc/自然基金项目/Federated
	...
	...
	32/32 [==============================] - 0s 805us/step - loss: 0.0610 - accuracy: 0.9840
	Sever: 轮次: 99,准确率: 0.9840，共测试了10000张图片 
	Epoch 1/3
	7/7 [==============================] - 0s 1ms/step - loss: 0.0515 - accuracy: 0.9799
	Epoch 2/3
	7/7 [==============================] - 0s 2ms/step - loss: 0.0153 - accuracy: 1.0000
	Epoch 3/3
	7/7 [==============================] - 0s 1ms/step - loss: 0.0073 - accuracy: 1.0000
	Epoch 1/3
	6/6 [==============================] - 0s 1ms/step - loss: 0.0122 - accuracy: 1.0000
	Epoch 2/3
	6/6 [==============================] - 0s 2ms/step - loss: 0.0053 - accuracy: 1.0000
	Epoch 3/3
	6/6 [==============================] - 0s 1ms/step - loss: 0.0038 - accuracy: 1.0000
	Epoch 1/3
	16/16 [==============================] - 0s 1ms/step - loss: 0.0101 - accuracy: 0.9980
	Epoch 2/3
	16/16 [==============================] - 0s 1ms/step - loss: 0.0037 - accuracy: 1.0000
	Epoch 3/3
	16/16 [==============================] - 0s 1ms/step - loss: 0.0017 - accuracy: 1.0000
	Epoch 1/3
	1/1 [==============================] - 0s 2ms/step - loss: 0.0122 - accuracy: 1.0000
	Epoch 2/3
	1/1 [==============================] - 0s 2ms/step - loss: 0.0065 - accuracy: 1.0000
	Epoch 3/3
	1/1 [==============================] - 0s 1ms/step - loss: 0.0038 - accuracy: 1.0000
	32/32 [==============================] - 0s 775us/step - loss: 0.0595 - accuracy: 0.9800
	Sever: 轮次: 100,准确率: 0.9800，共测试了10000张图片 
	QStandardPaths: wrong ownership on runtime directory /run/user/1000, 1000 instead of 0



```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/eb9b0199-4ca2-4d3e-abe4-94f5dc9c7235.png?raw=true)

```python
from flask import Flask,render_template
from flask import jsonify
import random

import matplotlib.pyplot as plt




import numpy as np


from tensorflow.keras import datasets, layers, models





app = Flask(__name__)


BATCH = 100


class DataSource(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()

        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))

        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.train_images, self.train_labels = train_images[0:15000], train_labels[0:15000]
        self.test_images, self.test_labels = test_images[0:10000], test_labels[0:10000]


def random_num_with_fix_total(maxvalue, num):
    """生成总和固定的整数序列
    maxvalue: 序列总和
    num：要生成的整数个数"""
    a = random.sample(range(1, maxvalue), k=num - 1)  
    a.append(0)  
    a.append(maxvalue)
    a = sorted(a)
    b = [a[count] - a[count - 1] for count in range(1, len(a))]  
    return b


class DataSource1(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[0:15000], train_labels[0:15000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]
        self.test_images, self.test_labels = test_images[0:10000], test_labels[0:10000]


class DataSource2(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]


class DataSource3(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]


class DataSource4(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]



class CNN(object):
    def __init__(self):
        model = models.Sequential()
        model.add(layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)))
        model.add(layers.MaxPool2D((2, 2)))
        model.add(layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(layers.MaxPool2D((2, 2)))
        model.add(layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(layers.Flatten())
        model.add(layers.Dense(64, activation='relu'))
        model.add(layers.Dense(10, activation='softmax'))
        
        self.model = model



def FedAvg():
    weight_CNN_1 = np.load("Client1Weight.npy", allow_pickle=True)
    weight_CNN_2 = np.load("Client2Weight.npy", allow_pickle=True)
    weight_CNN_3 = np.load("Client3Weight.npy", allow_pickle=True)
    weight_CNN_4 = np.load("Client4Weight.npy", allow_pickle=True)
    weight_array = (weight_CNN_1 + weight_CNN_2 + weight_CNN_3 + weight_CNN_4) / 4
    weight_out = np.array(weight_array)
    return weight_out



def EKF(cnn, weight_in):
    cnn.model.set_weights(weight_in)
    return cnn



cnn_sever = CNN()
cnn1 = CNN()
cnn2 = CNN()
cnn3 = CNN()
cnn4 = CNN()

data_sever = DataSource()
data1 = DataSource1()
data2 = DataSource2()
data3 = DataSource3()
data4 = DataSource4()


cnn_sever.model.compile(optimizer='adam',
                        loss='sparse_categorical_crossentropy',
                        metrics=['accuracy'])
cnn1.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn2.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn3.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn4.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
storage_acc = []
weight = cnn_sever.model.get_weights()
np.save("SeverWeight", weight)

for i in range(BATCH):
    
    weight = np.load("SeverWeight.npy", allow_pickle=True)
    
    
    
    
    cnn1 = EKF(cnn1, weight)
    cnn2 = EKF(cnn2, weight)
    cnn3 = EKF(cnn3, weight)
    cnn4 = EKF(cnn4, weight)
    
    cnn1.model.fit(data1.train_images[i], data1.train_labels[i], epochs=3)
    cnn2.model.fit(data2.train_images[i], data2.train_labels[i], epochs=3)
    cnn3.model.fit(data3.train_images[i], data3.train_labels[i], epochs=3)
    cnn4.model.fit(data4.train_images[i], data4.train_labels[i], epochs=3)
    

    weight_CNN1 = np.array(cnn1.model.get_weights())
    weight_CNN2 = np.array(cnn2.model.get_weights())
    weight_CNN3 = np.array(cnn3.model.get_weights())
    weight_CNN4 = np.array(cnn4.model.get_weights())


    np.save("Client1Weight", weight_CNN1)
    np.save("Client2Weight", weight_CNN2)
    np.save("Client3Weight", weight_CNN3)
    np.save("Client4Weight", weight_CNN4)
    weight = FedAvg()
    
    cnn_sever.model.set_weights(weight)
    np.save("SeverWeight", weight)
    test_loss, test_acc = cnn_sever.model.evaluate(data_sever.test_images[0:1000], data_sever.test_labels[0:1000])
    print("Sever: 轮次: %d,准确率: %.4f，共测试了%d张图片 " % (i + 1, test_acc, len(data_sever.test_labels)))
    storage_acc = np.append(storage_acc, test_acc)

x = np.array(range(100))
plt.plot(x, storage_acc)
plt.savefig('./static/acc.png')

@app.route('/')
def index():
    return render_template('index.html', weight = str(weight))

if __name__ == '__main__':
    app.run(host = '0.0.0.0')

```

关联的 HTML 文件：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>RESULT</title>
</head>
<body>
    the picture of accuracy is here :
    <br>
    <br>
    <img src="/static/acc.png" width="1080px" height="720px">
    <br>
    <br>
    the sever's weight is :
    <br>
    <br>
    {{weight}}

</body>
</html>

```

**注意：** 

1.  导入 Flask,render_template 模块
2.  一定要注意文件结构：图片要放在 static 文件夹下，html 文件要放在 templates 文件夹下，**不能都放在一个文件夹**下（至于为啥，不知道，反正是不能运行）
3.  最后的 host 必须为`host = '0.0.0.0'`否则 docker run 之后不能在公网进行访问

Dockerfile 文件：

```go
FROM python
RUN mkdir -p /Federated \
    && mkdir -p /Federated/templates \
    && mkdir -p /Federated/static \
    && pip install tensorflow \
    && pip install matplotlib \
    && pip install flask
COPY ./templates/index.html /Federated/templates
COPY main.py /Federated/
WORKDIR /Federated
EXPOSE 5000
RUN /bin/bash -c 'echo init ok'
CMD ["python", "main.py"]

```

注意：得自己 pip 安装所需要的包

`federated-deployment.yaml`文件：

```go
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: federated-deployment
  name: federated-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: federated-deployment
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: federated-deployment
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/hqc-k8s/federated:v1.0
        name: federated
        resources: {}
        ports:
        - containerPort: 5000
        imagePullPolicy: IfNotPresent
status: {}

```

`federated-service.yaml`文件：

```go
apiVersion: v1 # 注意此处不能和deployment一样为‘apps/v1’
kind: Service
metadata:
  name: federated-deployment
  labels:
    app: federated-deployment
spec:
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30001
    protocol: TCP
  selector:
    app: federated-deployment
  type: NodePort

```

## 1 命令行结果

```go
root@master:/home/hqc/自然基金项目/Federated# kubectl get all
NAME                                         READY   STATUS        RESTARTS   AGE
pod/federated-deployment-5d5cfb4c7c-5bq9r    1/1     Running       0          12m
pod/federated-deployment-5d5cfb4c7c-mcfg2    1/1     Running       0          12m

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/federated-deployment    NodePort    10.107.96.50     <none>        80:30001/TCP   8m41s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/federated-deployment    2/2     2            2           12m

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/federated-deployment-5d5cfb4c7c    2         2         2       12m


```

## 2 dashboard 结果

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/937a6a75-de1b-40e3-a985-5b648a61e1f8.png?raw=true)
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/cbb88b2d-aea4-4b4e-a51a-5ba2f47493a9.png?raw=true)
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/c80d219a-1b4b-4f30-9f5f-ed2a469dbb5f.png?raw=true)
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/a990451e-e4fd-421b-ae09-29fdcd0659b1.png?raw=true)

## 3 运行结果

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/23de8f6a-4ba8-49d7-9380-39c8d0e2b1e5.png?raw=true)

## 1 main.py

1.  添加了下载功能
2.  bootstrap 优化 html 模块
3.  flash 闪现消息模块

```python
from flask import Flask,render_template
from flask import jsonify
import random

import matplotlib.pyplot as plt




import numpy as np


from tensorflow.keras import datasets, layers, models





from flask_bootstrap import Bootstrap 
import os 

from flask import flash 
from flask import request 


from flask import send_from_directory 



app = Flask(__name__)
bootstrap = Bootstrap(app)


app.secret_key = 'sdafdsdfdasaf'



BATCH = 100


class DataSource(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()

        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))

        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.train_images, self.train_labels = train_images[0:15000], train_labels[0:15000]
        self.test_images, self.test_labels = test_images[0:10000], test_labels[0:10000]


def random_num_with_fix_total(maxvalue, num):
    """生成总和固定的整数序列
    maxvalue: 序列总和
    num：要生成的整数个数"""
    a = random.sample(range(1, maxvalue), k=num - 1)  
    a.append(0)  
    a.append(maxvalue)
    a = sorted(a)
    b = [a[count] - a[count - 1] for count in range(1, len(a))]  
    return b


class DataSource1(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[0:15000], train_labels[0:15000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]
        self.test_images, self.test_labels = test_images[0:10000], test_labels[0:10000]


class DataSource2(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]


class DataSource3(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]


class DataSource4(object):
    def __init__(self):
        (train_images, train_labels), (test_images, test_labels) = datasets.mnist.load_data()
        
        train_images = train_images.reshape((60000, 28, 28, 1))
        test_images = test_images.reshape((10000, 28, 28, 1))
        
        train_images, test_images = train_images / 255.0, test_images / 255.0
        self.TI, self.TL = train_images[15000:30000], train_labels[15000:30000]
        self.train_images = np.empty(BATCH, dtype=object)
        self.train_labels = np.empty(BATCH, dtype=object)
        begin = 0
        rand_count = random_num_with_fix_total(15000, BATCH)
        for count in range(100):
            self.train_images[count] = self.TI[begin:(begin + rand_count[count])]
            self.train_labels[count] = self.TL[begin:(begin + rand_count[count])]
            begin = begin + rand_count[count]



class CNN(object):
    def __init__(self):
        model = models.Sequential()
        model.add(layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)))
        model.add(layers.MaxPool2D((2, 2)))
        model.add(layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(layers.MaxPool2D((2, 2)))
        model.add(layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(layers.Flatten())
        model.add(layers.Dense(64, activation='relu'))
        model.add(layers.Dense(10, activation='softmax'))
        
        self.model = model



def FedAvg():
    weight_CNN_1 = np.load("Client1Weight.npy", allow_pickle=True)
    weight_CNN_2 = np.load("Client2Weight.npy", allow_pickle=True)
    weight_CNN_3 = np.load("Client3Weight.npy", allow_pickle=True)
    weight_CNN_4 = np.load("Client4Weight.npy", allow_pickle=True)
    weight_array = (weight_CNN_1 + weight_CNN_2 + weight_CNN_3 + weight_CNN_4) / 4
    weight_out = np.array(weight_array)
    return weight_out



def EKF(cnn, weight_in):
    cnn.model.set_weights(weight_in)
    return cnn



cnn_sever = CNN()
cnn1 = CNN()
cnn2 = CNN()
cnn3 = CNN()
cnn4 = CNN()

data_sever = DataSource()
data1 = DataSource1()
data2 = DataSource2()
data3 = DataSource3()
data4 = DataSource4()


cnn_sever.model.compile(optimizer='adam',
                        loss='sparse_categorical_crossentropy',
                        metrics=['accuracy'])
cnn1.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn2.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn3.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
cnn4.model.compile(optimizer='adam',
                   loss='sparse_categorical_crossentropy',
                   metrics=['accuracy'])
storage_acc = []
weight = cnn_sever.model.get_weights()
np.save("SeverWeight", weight)

for i in range(BATCH):
    
    weight = np.load("SeverWeight.npy", allow_pickle=True)
    
    
    
    
    cnn1 = EKF(cnn1, weight)
    cnn2 = EKF(cnn2, weight)
    cnn3 = EKF(cnn3, weight)
    cnn4 = EKF(cnn4, weight)
    
    cnn1.model.fit(data1.train_images[i], data1.train_labels[i], epochs=3)
    cnn2.model.fit(data2.train_images[i], data2.train_labels[i], epochs=3)
    cnn3.model.fit(data3.train_images[i], data3.train_labels[i], epochs=3)
    cnn4.model.fit(data4.train_images[i], data4.train_labels[i], epochs=3)
    

    weight_CNN1 = np.array(cnn1.model.get_weights())
    weight_CNN2 = np.array(cnn2.model.get_weights())
    weight_CNN3 = np.array(cnn3.model.get_weights())
    weight_CNN4 = np.array(cnn4.model.get_weights())


    np.save("Client1Weight", weight_CNN1)
    np.save("Client2Weight", weight_CNN2)
    np.save("Client3Weight", weight_CNN3)
    np.save("Client4Weight", weight_CNN4)
    weight = FedAvg()
    
    cnn_sever.model.set_weights(weight)
    np.save("SeverWeight", weight)
    test_loss, test_acc = cnn_sever.model.evaluate(data_sever.test_images[0:1000], data_sever.test_labels[0:1000])
    print("Sever: 轮次: %d,准确率: %.4f，共测试了%d张图片 " % (i + 1, test_acc, len(data_sever.test_labels)))
    storage_acc = np.append(storage_acc, test_acc)


















x = np.array(range(100))
plt.plot(x, storage_acc)
plt.savefig('./acc.png')





@app.route('/', methods=['GET', 'POST'])



def download_file():
    Path = os.listdir('.') 
    

		
    entries = []
    for path in Path:
        if os.path.splitext(path)[1] == '.png' or os.path.splitext(path)[1] == '.npy':
            entries.append(path)
            
            
    return render_template('download.html',entries = entries)


@app.route('/downloads/<filename>', methods=['GET', 'POST'])
def downloaded_file(filename):
    
        
        
    flash('Download Successfully ! ! !', 'success')
    return send_from_directory('./',filename,as_attachment = True)



if __name__ == '__main__':
    app.run(host = '0.0.0.0')
    
    
    
    
    

```

## 2 download.html

```
`{% extends "bootstrap/base.html" %}
{% block title %}DOWNLOAD PAGE{% endblock %}

{% block content %}
    <div class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">
                        <button type="button" class="close" data-dismiss="alert" aria-hidden="true">×</button>
                        {{ message }}
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        <h2>You Can Download The Following Files</h2>
        <h5 style="color:red">(just click it ☝ ~)</h5>
        <ol>
        {% for entry in entries %}
        <li><a href="{{url_for('downloaded_file',filename = entry)}}">{{entry}}</a>
        {% endfor %}
        </ol>
    </div>
{% endblock %}` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/ed2c53af-0901-453a-a393-348c9bc63662.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24


```

## 3 Dockerfile 和 requirement.txt

```bash
FROM federated:v1.2
COPY . . 
WORKDIR . 
RUN pip install -r requirement.txt
EXPOSE 5000
RUN /bin/bash -c 'echo init ok'
CMD ["python", "main.py"]

```

**requirement.txt**：  
可用`tensorflow==2.4.1`的形式指定版本进行`pip install`

```bash
Flask
tensorflow
matplotlib
flask_bootstrap
numpy

```

## 4 deployment 和 service

**deployment.yaml**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: federated-deployment
  name: federated-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: federated-deployment
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: federated-deployment
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/hqc-k8s/federated:v1.3
        name: federated
        resources: {}
        ports:
        - containerPort: 5000
        imagePullPolicy: IfNotPresent
status: {}

```

**service.yaml**:

```yaml
apiVersion: v1 
kind: Service
metadata:
  name: federated-service
  labels:
    app: federated-service
spec:
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30000
    protocol: TCP
  selector:
    app: federated-service
  type: NodePort


```

## 5 结果

控制器和服务等全部正常  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/9fa70287-d1aa-4213-8773-d6df2f103449.png?raw=true)

### 5.1 出错

但是发现只是一瞬间 running，过一会就一直重启。`ip+port`的方式更是没法成功。  
查看日志发现：

```yaml
root@master:/home/hqc/自然基金项目/Federated


```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/ec962bd9-a0b2-4c12-9cf2-508ab9c1c04c.png?raw=true)
发现是内部程序的原因，**不是集群的问题**，程序无法访问网址下载所需数据集。  
为啥本地 docker run 可以成功运行，创建 deployment 后不能访问呢？？  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/fa8f58de-743d-45d9-8faa-bb97bc03f91b.png?raw=true)

以为的原因：

1.  看这报错好像是外网的问题，但实际上不是。
2.  肯定是 service.yaml 文件出错了，修改后仍然不行。

### 5.2 解决

最后发现不知为啥 master 的 IP 地址变成了`192.168.43.49`，好离谱。  
想设置静态 IP，按照网上大家都行的方法尝试发现没法固定 IP，遂作罢。这个工作以后再弄。  
直接修改 hosts 文件为`192.168.43.49`。  
`vim /etc/hosts`  
重启节点，惊喜发现所有都 running。  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/36ea93f4-d07b-456c-8f3f-efb831600bba.png?raw=true)

创建 deployment 和 service 之后要等一段时间运行结束，再进行`ip+port`方式访问。  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2014-21-34/bafc48f8-cf4c-43ba-af46-42421ca88027.png?raw=true)

**成功！！！**
