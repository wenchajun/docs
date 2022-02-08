### 开启buildx 功能

默认情况下，buildx已经在安装包里面了
 在 ~/.docker/config.json增加，是家目录的client端的配置不是/etc下的配置
 "experimental": "enabled"
 即可永久开启buildx命令
 为了良好的支持性，如果是centos版本需要升级内核到5.12.9才能正常使用
 内核升级过程（略）

### 使用buildx 功能

在 Docker 19.03+ 版本中可以使用 docker buildx build 命令使用 BuildKit 构建镜像。该命令支持--platform 参数可以同时构建支持多种系统架构的 Docker 镜像，大大简化了构建步骤。

1、由于 Docker 默认的 builder 实例不支持同时指定多个 --platform ，我们必须首先创建一个新的 builder 实例。
 `$ docker buildx create --name mybuilder --driver docker-container`
 2、使用新创建好的builder实例
 `$ docker buildx use mybuilder`
 3、查看已有的builder实例
 `$ docker buildx ls`
 4、安装模拟器（用于多平台镜像构建）
 `$ docker run --privileged --rm tonistiigi/binfmt --install all`
 5、本地构建镜像并推送
 `$ docker buildx build --platform linux/arm,linux/arm64,linux/amd64 -t test/arch --push -f ./dockerfile .`



作者：大鹏_5a67
链接：https://www.jianshu.com/p/f4417f2b58c2
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
