​本文将手把手教你搭建Nexus作为docker的镜像仓库，【**建议点赞收藏**】

# 一、Nexus是什么

 Sonatype Nexus，又叫Nexus Repository Manager。是一个管理和托管软件构件的仓库管理工具。支持多种格式，如Maven、Docker、npm、PyPI等，能帮助团队建立私有仓库，实现构件的集中存储、分发和管理，从而提升开发效率和构建可靠性。

主要功能包括创建托管仓库(即存放内部构件)、代理仓库（作为内部公共仓库构件缓存仓库）和组仓库(统一作为内部访问入口)。主要用于企业内网环境，以加速依赖下载、统一管理私有构件、组件、依赖库、容器镜像等，而且可以提供权限控制和实现安全策略。

我们之前通常是将其用作Maven的依赖仓库以及go的依赖库。实际上其支持的组件格式非常的多，我们可以看下下面几个图。

Nexus支持的组件存储格式：
![](https://i-blog.csdnimg.cn/direct/0e61ef20e4624005b939fb6d33d949f0.png)
支持格式广泛，具体表格如下:

![](https://i-blog.csdnimg.cn/direct/42857957af6244e79d70f84fc1822f57.png)

![](https://i-blog.csdnimg.cn/direct/dd83760b556a4fe1ab7d8432d9f841b1.png)

社区版本几乎该有的功能都有，可谓是非常良心，其功能特征：

![](https://i-blog.csdnimg.cn/direct/58e68e86c1ff4974a1bc01d7c24f9055.png)

# 二、Nexus下载

可以直接下载安装包，下载地址是  [Nexus下载地址](https://www.sonatype.com/products/nexus-community-edition-download "Nexus下载地址") 。

不过需要填写登记信息才能下载，登记信息没有验证，填写即可,一般不接受公共邮箱，比如126邮箱、QQ邮箱都会提示不被接受。要求填写企业邮箱(不会验证，可以随便填)

![](https://i-blog.csdnimg.cn/direct/d18b3f7c231c4d12b7e75a5e36cf0c16.png)

登记表填写没问题后点击登录，验证码验证通过后会进入下面这个下载选择页面，选择对应的需要的安装包，这里由于我是安装在Linux Ubuntu 22.04版本的系统下，Intel的CPU，所以我选Unix x86的。

![](https://i-blog.csdnimg.cn/direct/db9c4a0d00fd4dcda5798e73739bd320.png)

如果大家不能下载或者下载缓慢可以选择我已经下载好的，大家可以直接在我的资源中下载。

# 三、Nexus安装运行

本次安装Nexus是在Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-161-generic x86_64)的服务器中，

机器IP是172.16.1.180。

## 1.使用安装包安装运行

将下载的nexus安装包放到/opt/nexus目录下，操作如下：

```bash
# 在/opt目录下建nexus目录存放安装包
sudo mkdir -p /opt/nexus
cd /opt/nexus
# 如果能下载则直接下载安装包,如果不能直接下载则到官方页面下载放到该目录
wget https://cdn.download.sonatype.com/repository/downloads-prod-group/3/nexus-3.87.1-01-linux-x86_64.tar.gz
# 执行解压
tar xvz --keep-directory-symlink -f ./nexus-3.87.1-01-linux-x86_64.tar.gz
```

解压以后会看到两个目录，一个是nexus-3.87.1-01，另一个是sonatype-work

![](https://i-blog.csdnimg.cn/direct/66ea0959baa44c09a319f1ec862f64a0.png)

执行如下命令，进到nexus-3.87.1-01的bin目录下

```bash
cd nexus-3.87.1-01/bin/
# 查看目录文件
ls -l
```

显示如下图所示，有一个nexus的可执行文件(绿色后面带星号的)

![](https://i-blog.csdnimg.cn/direct/da97d083c6d4479a96d58a4507e2abf3.png)

可以通过这个程序来执行启动服务，可用的命令有：

> start, stop, run, restart, force-reload

启动 Nexus 存储库服务：

```bash
./nexus start
```

日志会输出到应用程序日志文件中。

要停止在后台运行的 Nexus 存储库服务：

```bash
./nexus stop
```

也可以使用 run 命令运行应用程序并在当前 shell 中显示日志，以便进行测试：

```bash
./nexus run
```

所以用run一般是用于测试观察日志的情况。使用该命令启动应用程序后，应用程序将在当前 shell 中运行。可以在控制台中使用`run,`该命令停止应用程序 `CTRL+C。`

如果已经启动的服务需要重启可以用下列命令

```bash
./nexus restart
```

当有配置变化时，需要使变更生效，则可以使用force-reload命令时期强制加载生效。

```bash
./nexus force-reload
```

**不过，在生产环境中安装软件时，需要将 Nexus Repository 配置为以服务形式运行。这样才能确保服务器重启后服务能够正常重启。**

## 2.以docker容器的方式运行

 docker容器运行首先必须确保已经安装docker。具体安装过程暂不在此讲解，大家可以自行搜索方法安装。

运行容器前大家可以先在一个拥有较大空间的目录下建一个准备用于存储nexus数据文件的目录，例如我这里在/data目录下创建share/nexus/nexus-data目录，用于存放数据，将会被挂在到nexus容器下

```bash
# 因为容器内是200的用户和用户组访问，为了能够写入外部的挂载目录，新建200的用户和用户组
sudo useradd -u 200 -g 200 -s /bin/false nexus 2>/dev/null || echo "User exists"

# 创建/data/share/nexus/nexus-data用来存储nexus的数据文件目录
sudo mkdir -p /data/share/nexus/nexus-data && chown -R 200 /data/share/nexus/nexus-data

# 运行容器
docker run -d -p 8081:8081 --name nexus -v /data/share/nexus/nexus-data:/nexus-data sonatype/nexus3
```

**这里注意几点：**

1.由于容器内的用户和用户组是200，所以必须给挂载目录授权200；

2.参数-d表示后台运行；

3.参数-p 8081:8081是指将容器内的8081端口映射到宿主机的8081端口，这样访问外部的ip才能访问到这个服务；

4.参数--name nexus 是指给这个容器命名为nexus，这样方便访问和管理，但是docker只支持但容器，容器名不能重复。

5.参数-v /data/share/nexus/nexus-data:/nexus-data是指将宿主机的/data/share/nexus/nexus-data目录挂在到容器内的/nexus-data目录，这样容器内的数据实际上是在外部宿主机的的/data/share/nexus/nexus-data目录中，这样方便数据备份和管理，也避免容器销毁以后数据也随之销毁的问题。

# 四、访问用户界面

Nexus Repository 启动后，使用 Web 浏览器访问服务 URL，即可访问 Web 应用程序用户界面：

```bash
http://<主机IP>:<端口>
```

Nexus Repository 包含一个拥有完全访问权限的管理员用户。用户名是__admin__`admin.password` ，初始密码位于目录中一个名为 `<temporary_file_name>` 的临时文件中`$data-dir`。确实不知道怎么找，也可以先直接打开登录界面。比如我这里在浏览器输入地址：http://172.16.1.180:8081

首次登录显示如下图所示登录界面，提示如何找到admin的密码：

![](https://i-blog.csdnimg.cn/direct/165f743e75ce4f5aab048a02beda8a02.png)

我们根据提示进入/opt/nexus/sonatype-work/nexus3/目录查看(**一定要根据提示去找**)

![](https://i-blog.csdnimg.cn/direct/e4a7649bcca046d3b51aedb8d2d4eed4.png)

用admin.password文件中的字符串作为密码登录，如下图：

![](https://i-blog.csdnimg.cn/direct/e2083163c3e54b40ab3581e860cdc03f.png)

admin首次登录必须修改密码：

![](https://i-blog.csdnimg.cn/direct/52bd3f5ba6464e21bd41ac906adefc90.png)

修改密码后会问你是否允许匿名访问系统，允许匿名访问的话主要是指不需要密码就可以搜索、浏览和下载仓库中的组件。如果你是自己内网使用或者公司内部局域网可以根据情况设置匿名访问。如果不允许匿名访问的话就是所有这些操作必须有账号密码才能允许使用。我这里是内部访问就设置允许。

![](https://i-blog.csdnimg.cn/direct/9f0ee4374f174aa7940f90cf97895a9d.png)

匿名访问页面如下图所示，搜索到的组件也可以直接下载使用。

![](https://i-blog.csdnimg.cn/direct/ec5280d91a564f3ca78b3faa4c00d2ca.png)

# 五、配置docker镜像库

上面我们已经搭建好了Nexus仓库了，下面将带大家设置docker镜像仓库。

我们重新从登录开始：

![](https://i-blog.csdnimg.cn/direct/934869c51ff04d12bfb9075308d40b07.png)

![](https://i-blog.csdnimg.cn/direct/9667fef85a2840ad84661819093131d3.png)

登录后我们可以看到多了一个设置的菜单，这就是我们需要的

![](https://i-blog.csdnimg.cn/direct/3cdd1524648447c58ef56fe340f06884.png)

点击【settings】进行设置，主要就是设置Repositories

![](https://i-blog.csdnimg.cn/direct/88e591bbe0104595812077dd5893b50d.png)

![](https://i-blog.csdnimg.cn/direct/4e6210989b6a4f68b54bcdc6021111e4.png)

如下图，选择一个添加，一般先添加一proxy的。

![](https://i-blog.csdnimg.cn/direct/25d59d62dbbf41a3a31bf9e7784ef985.png)

一般有hosted、proxy、group这三种，这三者有什么区别呢？

hosted一般是本地自主所有仓库，一般用于**内部上传**的组件和镜像等。

proxy 代理外部仓库，比如代理镜像仓库docker.io(Docker Hub)的镜像。

group 是一个分组聚合。

由于国内公开镜像大部分不能用了，我添加一个能用的演示下添加proxy仓库：

![](https://i-blog.csdnimg.cn/direct/b2833ef68ecb4f15bacde7917f041c20.png)

![](https://i-blog.csdnimg.cn/direct/56c45183f81743f2a7aae3ce6e41087d.png)

添加hosted仓库，这个主要用于内部组件上传存储到这里。

![](https://i-blog.csdnimg.cn/direct/0b973e8879134d8587435b2f14d4f4d4.png)

添加group仓库，这里要注意下：

![](https://i-blog.csdnimg.cn/direct/63f2b523d23d4e10a23e094703ace561.png)

![](https://i-blog.csdnimg.cn/direct/a51c5da65939462091d6b3b26993b97d.png)

**这里有两点需要注意的：**

1.这里我选择了端口访问，如果是容器部署，则需要把这个端口映射出来。

```bash
# 先停掉并删掉原来的
docker stop nexus
docker rm nexus
# 在重新运行新的
docker run -d -p 8081:8081 -p 16888:16888 --name nexus -v /data/share/nexus/nexus-data:/nexus-data sonatype/nexus3
```

2.设置聚合是要注意顺序，一般来说hosted在前，因为一般是优先公司内部的，proxy在后，多个proxy时，选择稳定和访问速度快的在前，不稳定或者速度不理想的在后。

**至此，docker镜像仓库已经配置完成了。**

# **六、配置内部仓库访问**

上述配置了内部仓库，但是还需要配置docker如何使用该内部仓库。这部分很简单,只要在需要使用的docker配置/etc/docker/daemon.json中添加上两行即可：

>   "insecure-registries": ["172.16.1.180:16888"],  
>   "registry-mirrors": ["http://172.16.1.180:16888"]

脚本如下：

```bash
sudo vim /etc/docker/daemon.json
# 添加两行
{
  "insecure-registries": ["172.16.1.180:16888"],
  "registry-mirrors": ["http://172.16.1.180:16888"]
}
```

注意：上面由于我改成了端口16888访问即可这样配置。

但是如果不是改成端口访问，而是路径访问，则应该换成原端口+路径的方式。

**最后大家如果有什么问题和疑问或者大家还想了解那些内容，请在评论区反馈，我看到了会尽快回复。**

​