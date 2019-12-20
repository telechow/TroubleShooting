**com.github.penggle的kaptcha图片验证码，打包后在宿主机上运行没问题，在docker容器中会抛无字体空指针异常**
==
  - 原因：
    - docker容器使用的是java:8-alpine作为基础镜像，此镜像中没有字体文件
    - 而kaptcha是生成图片文件Image对象的时候需要使用字体文件（没有字体，怎么生成验证码的图片呢）
  - 解决方法：
    - 在build docker容器时，往容器中添加字体文件
    - 在Dockerfile添加jar包之前加上一句：
    - ```RUN set -xe && apk --no-cache add ttf-dejavu fontconfig```