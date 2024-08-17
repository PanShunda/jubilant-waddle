###  	简单CNN创建流程

首先需要安装CUDA conda等组件，并在自己的虚拟环境中安装troch

先创建一个模型

```python
# -*- coding: utf-8 -*-
# 作者：小土堆
# 公众号：土堆碎念
import torch
from torch import nn

# 搭建神经网络
class Tudui(nn.Module):
    def __init__(self):
        super(Tudui, self).__init__()
        #Sequential函数的意思是将以下层打包，在forward函数中便可以只写一行
        self.model = nn.Sequential(
            #conv2d 二维卷积层 输入通道 输出通道 卷积核尺寸5*5 步长为1 填充为2
            nn.Conv2d(3, 32, 5, 1, 2),
            #maxpool 池化层 大小为2
            nn.MaxPool2d(2),
            nn.Conv2d(32, 32, 5, 1, 2),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 5, 1, 2),
            nn.MaxPool2d(2),
            # flatten 将多维的输入张量展平成一维张量 为之后的两个全连接层做准备
            nn.Flatten(),
            #全连接层 输入 输出
            nn.Linear(64*4*4, 64),
            nn.Linear(64, 10)
        )
	#forward函数 字面意思 前进
    def forward(self, x):
        x = self.model(x)
        return x

#为这个类设计方法
if __name__ == '__main__':
    tudui = Tudui()
    input = torch.ones((64, 3, 32, 32))
    output = tudui(input)
    print(output.shape)
```



导入数据集CIFAR10

```python
# root:下载根目录 
# train:为true则代表训练集 为false则代表测试集
# transform=torchvision.transforms.ToTensor():将下载的数据集转换为tensor格式
# download=True开启就可以了

train_data=torchvision.datasets.CIFAR10(root="../data",train=True,transform=torchvision.transforms.ToTensor(),download=True)
test_data=torchvision.datasets.CIFAR10(root="../data",train=False,transform=torchvision.transforms.ToTensor(),download=True)
```



读取数据集长度

```python
# length 长度
train_data_size = len(train_data)
test_data_size = len(test_data)
# 如果train_data_size=10, 训练数据集的长度为：10
print("训练数据集的长度为：{}".format(train_data_size))
print("测试数据集的长度为：{}".format(test_data_size))
```



利用 DataLoader 来加载数据集

```python
train_dataloader = DataLoader(train_data, batch_size=64)
test_dataloader = DataLoader(test_data, batch_size=64)
```



创建名为tudui的网络模型 并且创建损失函数（torch.nn库函数自带 交叉熵损失函数）

```python
# 创建网络模型
tudui = Tudui()
# 利用GPU进行加速
mynet = tudui.cuda()
# 损失函数
loss_fn = nn.CrossEntropyLoss()
# 利用GPU进行加速
loss_fn = loss_fn.cuda()
```



创建优化器

```python
#学习率 10^-2
learning_rate = 1e-2
#设定SGD优化器 参数tudui.parameters()为网络模型tudui的参数(全随机 未被优化过)
optimizer = torch.optim.SGD(tudui.parameters(), lr=learning_rate)

```



训练前准备

```python
# 设置训练网络的一些参数
# 记录训练的次数
total_train_step = 0
# 记录测试的次数
total_test_step = 0
# 训练的轮数
epoch = 10
```



添加tensorboard

```python
writer = SummaryWriter("../logs_train")
```



训练开始

```python
for i in range(epoch):
    print("-------第 {} 轮训练开始-------".format(i+1))

    # 训练步骤开始
    tudui.train()
    for data in train_dataloader:
        imgs, targets = data
        #利用GPU加速
        img = img.cuda()
        targets = targets.cuda()
        
        outputs = tudui(imgs)
        loss = loss_fn(outputs, targets)

        # 优化器优化模型
        # 梯度清零
        optimizer.zero_grad()
        # 反向传播(获取梯度)
        loss.backward()
        # 优化
        optimizer.step()

        total_train_step = total_train_step + 1
        
        #每100步记录一次损失函数 并输出训练次数
        if total_train_step % 100 == 0:
            print("训练次数：{}, Loss: {}".format(total_train_step, loss.item()))
            writer.add_scalar("train_loss", loss.item(), total_train_step)

    # 测试步骤开始
    # 将模型设置为评估模式
    # 通常在模型训练结束后，进行模型评估或测试之前，调用eval()方法。这样可以确保评估或测试时使用的是模型的最佳状态。
    tudui.eval()
    total_test_loss = 0
    total_accuracy = 0
    # 临时禁用梯度计算
    with torch.no_grad():
        for data in test_dataloader:
            # 获取已经被转换成tensor的 imgs 和标签 targets
            imgs, targets = data
            #利用GPU加速
            img = img.cuda()
        	targets = targets.cuda()
        
            outputs = tudui(imgs)
            #计算损失值并求和
            loss = loss_fn(outputs, targets)
            total_test_loss = total_test_loss + loss.item()
            #计算精确度并求和
            accuracy = (outputs.argmax(1) == targets).sum()
            total_accuracy = total_accuracy + accuracy

    print("整体测试集上的Loss: {}".format(total_test_loss))
    print("整体测试集上的正确率: {}".format(total_accuracy/test_data_size))
    writer.add_scalar("test_loss", total_test_loss, total_test_step)
    writer.add_scalar("test_accuracy", total_accuracy/test_data_size, total_test_step)
    total_test_step = total_test_step + 1
	# 训练完成后保存模型
    torch.save(tudui, "tudui_{}.pth".format(i))
    print("模型已保存")
```



关闭writer 之后可以在tb中查看图表

```python
writer.close()
```



### CNN训练完毕后的验证

假设保存的网络模型为my_model_14.pth

```python
import torch
import torchvision
from PIL import Image
from torch import nn

# 在网络上找到的截图命名为qw.png 并将其放入imgs文件夹
image_path = "../imgs/qw.png"
image = Image.open(image_path)
print(image)
# 将qw转换为RGB三通道，并将其转换为tensor模式
image = image.convert('RGB')
transform = torchvision.transforms.Compose([torchvision.transforms.Resize((32, 32)),
                                            torchvision.transforms.ToTensor()])

image = transform(image)
print(image.shape)

# 读取网络模型，并对其进行验证
model = torch.load("my_model_14.pth", map_location=torch.device('cpu'))
print(model)
image = torch.reshape(image, (1, 3, 32, 32))
model.eval()
with torch.no_grad():
    output = model(image)
print(output)

print(output.argmax(1))

```

