
在上一篇已经介绍过如何搭建Nexus作为docker的镜像仓库。接下来基于已经搭建好的Nexus配置Maven仓库。

**各位彦祖们，别忘了用您发财的小手点赞、关注和收藏**

如下图是已经配置好的docker镜像库的Nexus，接下来在此基础上继续配置Maven仓库即可。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225113539256.png)


在docker配置中其实已经讲过，Nexus是提供三种docker类型仓库配置，在这里类似：
> 1.hosted 本地仓库，可以用于存储内部库包，如公司或者部门内部生成的jar包，可以上传到这种hosted仓库里，可以配置一个或者多个。
> 2.proxy 代理仓库，代理外部存储库包。
> 3.group 分组聚合。一般是将多个hosted和proxy聚合对外提供统一使用入口，也方面进行权限控制。

![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225115126462.png)


## 一、配置hosted仓库
### 1.配置maven-release
先配置一个hosted的类型Maven仓库用存储Release版本的库包,直接命名maven-release。如下图
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225120403748.png)
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225120835769.png)
在下面位置有个deployment policy是什么意思呢？就是说该maven包在groupId、artifactId、version一样的情况下是否允许覆盖更新。一般Release是不允许覆盖更新的，因为已经代表稳定发型版了就不能乱改，容易出问题。如果实在有问题，可以通过发行版本的方式处理，新的版本就不是覆盖了。

### 2.配置maven-snapshot
再来先配置一个hosted的类型Maven仓库用存储Snapshot版本的库包,命名maven-snapshot。如下图：
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225122010625.png)
这里deployment policy我们选择允许更新，因为Snapshot我们认为本身就代表本稳定版本。为了避免被库包使用者揍的鼻青脸肿的情况最好还是通过版本号更新的方式来更新会更好。不然本地库包一清理，刚才还好好的代码就又问题了，或者问题一会好一会有问题，万一查到时你们在库包中动手脚那就生死难料了。
![00BBB681.jpg](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/00BBB681.jpg)

## 二、配置proxy仓库
再来配置一个proxy的类型Maven仓库用存储Release版本的库包。如下图
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225140212320.png)
这里有个Layout Policy选项选择Permissive。是指要不要跳过对仓库存储路径是否严格符合 Maven 官方格式的强制验证。为什么要选择这个？因为这是全球最大最全的仓库，可能会有很老的库由于早期验证规则和格式不够完善的，如果严格模式的话可能就会不通过验证，出现用不了的情况。

可以多配置一两个镜像仓库备用，考虑到速度和稳定性。可以配置国内的镜像仓库，这样既能确保速度也能确保稳定。国内速度比较快的，如阿里的仓库 https://maven.aliyun.com/repository/public 聚合了central仓和jcenter仓的聚合仓。

还有一些也可以配置备用
华为的仓库 https://repo.huaweicloud.com/repository/maven/ 
腾讯云 https://mirrors.cloud.tencent.com/nexus/repository/maven-public/
开源中国 https://repo.oschina.net/content/groups/public/ 
推荐优先阿里的镜像源。

## 三、配置group仓库
最后来配置一个group的类型Maven仓库用存储Mixed版本的库包。如下图
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225142354566.png)

这里由于是分组聚合对外的，所以版本选择混合模式，就是包括了Release和Snapshot。
由于中心库是Permissive模式，所以这里还是要选择Permissive(宽松)模式。聚合的原则是hosted优先，Release优先，速度快的优先。

下面这个maven-public的地址就是可以提供给内部使用了
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225143020028.png)

## 四、Maven配置使用
上面已经配置好仓库了，想要使用这些本地仓库还需要再Maven端做个配置。
下面是配置maven的setting.xml，即$MAVEN_HOME/conf/setting.xml文件
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225144208297.png)
配置注意事项：
> 1.id是唯一的镜像名称
> 2.name是对该镜像的描述说明，简单说明即可。
> 3.url配置group仓库的URL地址
> 4.mirrorOf是指要映射到哪个镜像，直接\*即可，表示全部都走该镜像。

最后使用该maven配置，例如我的IDEA配置如下，指定我刚才配置的setting.xml文件
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225144823523.png)
maven的setting.xml配置是全局的。除了maven的conf下的setting还有用户目录下setting文件也可以配置(差不多可以参考)，以及pom.xml文件中也可以配置。生效的顺序默认是pom.xml、用户目录下setting.xml，以及maven的conf下的setting.xml。

我们在IDEA中指定配置为maven的conf下的setting.xml。去掉pom.xml的repository部分(或者直接repository改成我们配置的仓库地址，即上面的URL)。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225155929319.png)


<font color="#548dd4">验证一下</font>
目前maven仓库里面没有任何库包组件
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225152006225.png)

重新载入依赖
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225145145628.png)
**如果这样不行，可以点击clean，然后执行package。**
已经从仓库下载了
![](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225151406436.png)

可已经可以看到仓库中下载的包了
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251225151742971.png)

OK，至此大功告成！
**看完别忘了用您发财的小手点赞、关注和收藏**
