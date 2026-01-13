此前已经部署了GPUStack的v0.7.1版本,由于最近GPUStack版本有较大升级，从官网看性能有较大提升，最重要的是能支持比较新的模型，所以决定将GPUStack从v0.7.1升级到v2.0.2版本。

## 一、原部署情况说明
此前已经部署了3个虚拟机器已经部署了GPUStack集群，版本是0.7.1，每个虚拟机挂了一块5090D的显卡，每个显存为32G。

| 虚拟机   | IP           | 操作系统               | 显卡                            | 驱动版本       | cuda版本 |
| ----- | ------------ | ------------------ | ----------------------------- | ---------- | ------ |
| GPU-1 | 172.16.1.111 | Ubuntu 24.04.2 LTS | NVIDIA GeForce RTX 5090 D 32G | 570.153.02 | 12.8   |
| GPU-3 | 172.16.1.113 | Ubuntu 24.04.2 LTS | NVIDIA GeForce RTX 5090 D 32G | 570.153.02 | 12.8   |
| GPU-5 | 172.16.1.115 | Ubuntu 24.04.2 LTS | NVIDIA GeForce RTX 5090 D 32G | 570.153.02 | 12.8   |
三个虚拟机组成集群，其中113作为主节点。

原来集群三个节点存储数据的地方，通过NFS挂在到同一个磁盘的不同目录下，gpustack-data这个是主节点挂在数据目录，gpustack-worker-gpu1是GPU1这个节点上挂在数据的目录，gpustack-worker-gpu5是GPU5这个节点上挂在数据的目录。如下图所示：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108101651341.png)

## 二、迁移准备
从0.7.1版本升级到2.0版本，从版本号就可以看出应该是有很大的变动，所以一定要先看官方升级文档。确保原来的数据不会丢失。
#### 1.先备份数据
一定要做数据备份，如下图所示，我直接通过`cp -a`来备份原来数据目录。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108111855635.png)


#### 2.目录授权
这个主要是考虑2.0以后架构变更，引入postgresSQL后，防止使用非root用户时，没有权限。root用户的话可以不执行，不过谨慎考虑还是执行下比较好。

```
# ${your-data-dir}是指原来存放数据的目录
dir="${your-data-dir}"
chmod a+rx "${dir}/log" || true
chmod a+rx $dir
while [ "$dir" != "/" ]; do
  chmod a+x "$dir"
  dir=$(dirname "$dir")
done

```

新版本以docker等容器为主的部署环境。由于我们之前就是docker部署的，后面的操作就比较跟官方的操作一致。

#### 3.数据迁移
由于2.0以前是通过内置嵌入式的**SQLite**来存储数据，但是2.0以后数据库改成了**PostgreSQL**。所以需要将sqlite的数据迁移到Postgresql中。需要再启动容器是添加数据迁移的参数。

```
# 这是主节点
sudo docker run -d --name gpustack \
  --restart=unless-stopped \
  --privileged \
  --network=host \
  --env GPUSTACK_DATA_MIGRATION=true \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --volume ${your-data-dir}:/var/lib/gpustack \
  --runtime nvidia \
  gpustack/gpustack:latest

```
重点是--env GPUSTACK_DATA_MIGRATION=true 参数，添加启动后将会自动将原来sqlite的数据迁移到Postgresql的数据库中。
***特别注意***，${your-data-dir}一定要换成原来挂载数据的目录。**该目录千万要先备份，防止中途意外导致数据损坏**。我就进行中遇到停电，数据损坏，不得不重新从备份中复制数据，万幸提前备份了数据。
通过`--runtime nvidia` 参数来指定运行环境，不需要像以前一样选择带-cuda12.8这样后缀的镜像了。
也可以使用外部数据库，如果已经有数据库，可以通过`--database-url ${your-database-url}`来连接到外部的数据库。
latest最好换成确定的版本号，我就直接换成2.0.2版本。

## 三、执行升级
如果原来容器就叫gpustack，还得先把原来的容器删除。
#### 1.升级主节点
准备就绪就用上面的脚本启动主节点，需要等待一会儿，因为需要初始化和进行数据迁移。可以通过`docker logs -f gpustack`命令实时查看日志，了解具体进度。一会儿以后，主节点服务已经启动好了，如下图所示：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108153754473.png)


启动完成即可打开页面入口，原来的账号密码登录，可以看到显示1个集群，3个节点和3个GPU
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108153941475.png)

进入到节点页面显示如下：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108154151413.png)
**奇怪，怎么显示有3个节点呢？但是实际上另外两个节点是原来升级前的信息，应该是记录到数据库了，显示也是2个节点还没有起来，是Not Ready状态，这些只是原来的数据信息显示而已**。我们直接删除另外两个状态为Not Ready的节点的信息，留着也没有什么用。我们等下再将两个节点加进来。

#### 2.升级worker节点
页面上提供了添加节点按钮，我们可以直接从这里开始添加。
直接在节点页面点【添加节点】，在弹出的添加页面按引导步骤操作即可。
第1步选择集群，我们只有唯一一个启动的主节点，就一个选择。选择【下一步】到第2步选择厂商。支持的厂商还挺多的，我的是NVIDIA选第一个即可，如下图
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108162524781.png)

如第3步验证环境是否准备好了。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108155342509.png)
验证结果如下：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108155310797.png)
全是OK说明驱动和cuda以及container toolkit已经正确安装和配置了。
继续添加节点信息，这里是比较重要的，这里的数据卷直接配置原来挂载的数据卷路径即可。IP也指定为要部署worker机器的IP即可，实际上不填写，默认也是该节点ip，只是如果有多网卡的话避免识别错误，所以最好配置。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108155856785.png)

配置好后直接点下一步就会自动生成worker节点加入集群的命令，执行该命令就会自动加入集群。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108161115120.png)
我直接在我的GPU5的节点中执行上述提供的命令
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108161447512.png)
第一次由于我已经有了同名的容器，要先删除原来的容器，删除后在执行便成功了。这里我把镜像的地址前缀quay.io/去掉了，因为我本地不能访问quay.io，并且我本地已经导入了镜像，不需要这个前缀。接着我们到页面看下集群节点是不是已经加入了。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108161952642.png)
看到节点已经加入了，继续依葫芦画瓢，加入111节点。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108162727021.png)
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108163000811.png)

最终效果，三个节点都已准备就绪。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108163036581.png)


#### 3.升级完成效果
概览：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108163126115.png)
模型库：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108163246890.png)
部署：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108163656591.png)
**但是这里不行。显示原来部署的模型没有正常启动，而是显示Error，重试几次也是如此。**
看日志是后端的llama-box不存在或者没有指定。原来的小参数模型都是GGUF格式的模型，需要基于llama.cpp的llama-box部署。
干脆不用原来的模型算了，看下新的模型库有哪些模型可用。我们看下qwen3和deepseek有没有合适大小的模型可用。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260112094244449.png)
deepseek没有可用参数规模的，这种动不动几百G显存的我们想都不用想了。看看qwen3的模型。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260112094402974.png)
只能看红色圈出来的参数小一些的模型，看看有没有可能。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260112104300741.png)
基本只有30B内的模型勉强能用，但是占用资源太高了，其他的嵌入模型和排序模型可能就没资源用了。还是看看能不能以其他方式支持原来的模型文件部署。

#### 4.新增自定义推理后端
遇到事情先别慌，我们找啊找，在菜单栏看到一个之前的版本没有的菜单，这是本次升级新增的菜单项【推理后端】，我们看下能不找到原因。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108164001513.png)

发现支持的后端有4个，但是没有看到报错提示的llama-box这个推理后端。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108164305050.png)

这是**因为2.0版本已经内置没有llama-box这个推理后端了**，看下面两个图对比。在0.7时的GGUF格式是基于llama-box推理后端部署的，但是2.0以后支持的后端没有了llama-box了。而是新增了SGLang这个推理后端。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108165953286.png)
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108170524984.png)


由于之前基本全是gguf格式的部署文件，要不重新下载能支持的文件格式重新部署，要不就添加llama-box后端推理引擎，在2.0看起来是是可以添加自定义推理后端引擎。

果然在文档中找到了添加在定义推理后端运行GGUF格式的文件，我们按这个添加试试：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108172621553.png)

添加的推理引擎如下：
![](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108172901797.png)
回到部署这里，注意看这些失败后端还是llama-box，我们要先改成我们自定义的这个后端。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108174148532.png)
后端选自定义的llama.cpp，版本选v1-cuda
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108174416633.png)
又失败了，不过这次是依赖镜像访问下载不下来，我们把这个搞通。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108174733229.png)
我在本地的私服把ghcr.io和gcr.io代理上。代理完以后，镜像这里要改一下，就把地址前缀去掉，就会从本地镜像仓库获取，由于我们本地仓库配置了镜像代理，就能把镜像下载到仓库了，服务器从仓库能够获取到镜像。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260109091557054.png)

重新启动就好了，注意看，后端是就是我们新增加的自定义后端。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260109091334143.png)
运行起来了，我们打开页面测试下效果。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260109092146822.png)
OK，大功告成。
后面我们就可以直接用GPUStack部署的大模型为我们服务了，后面我们讲解如何利用GPUStack打造我们自己的RAG知识库。

## 四、踩坑回顾

**此次升级最重要的事是提前做了数据备份，避免了中间出现问题时可以重新来过。** 

如下图，直接将原来的数据目录完整的备份为_backup的目录。这是我的后悔药。
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108153350849.png)

此次升级其实并没有那么顺利。
#### 1.突然的断电
第一次在升级主节点时出现了突然由于市政电压不稳定，出现了突然断电，刚好在数据迁移过程中，导致重启后数据故障，无法完成主节点升级，且数据乱了，不得已把数据目录删除，重新从备份目录中重新复制一份完整的新数据重新进行主节点升级。
#### 2.配错了token
第二次是在升级worker节点时，没有从页面开始添加节点，而是直接跟之前一样，用命令行的方式添加worker节点，官方给出了添加命令脚本，但是token从哪里来，没有说清楚。导致我错误的取了连接到主节点的token。

```
sudo docker run -d --name gpustack-worker \
  --restart=unless-stopped \
  --privileged \
  --network=host \
  --volume ${your-data-dir}:/var/lib/gpustack \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --runtime nvidia \
  gpustack/gpustack \
  --server-url ${server-url} \
  --token ${token}

```

于是我直接到主节点数据目录下找关于token的文件，如下图所示，有一个token的文件，还有一个worker_token的文件，我就想当然得一位是worker_token，但是实际应该是token文件里的token。如下图所示：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260109182135278.png)

启动节点也没有报错，但是一会儿也没看到该worker显示在集群中，于是`docker logs -f gpustack`查看日志。日志信息如下：
![image.png](https://cdn.jsdelivr.net/gh/chlk1234/zhpic@image/20260108140110886.png)
大量的异常报错信息。尝试过数据修复，但是数据已经被污染，找到不根源。只好删除数据，重新从备份的数据中复制重新开始。后来才发现页面添加节点有明确的步骤指引了，还包含了数据迁移，于是直接从页面开始按照指引操作，还省了不少事。


