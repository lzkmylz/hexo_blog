---
title: AWS托管静态网站
date: 2019-10-27 21:56:02
tags: ['AWS']
---

 最近两天通过AWS部署了个人项目的前端，感觉十分的方便，于是写篇博客将整个部署流程总结一下。

# 基于AWS开发工具的CI/CD

平时版本管理都是通过Git进行，而在基于AWS进行CI/CD时，比较推荐的做法是将项目代码添加一个remote分支，关联到AWS CodeCommit，然后通过AWS CodeBuild和AWS CodePipeline来构建CI/CD。

在项目主文件夹下添加buildspec.yml文件，来定义Build阶段与各阶段执行的命令。然后通过设置CodePipeline，就可以在向CodeCommit push代码时，自动触发CI/CD Pipeline。

示例buildspec.yml如下：

```
version: 0.2

phases:
  install:
    runtime-versions: 
      nodejs: 10 
    commands:
      - echo install awscli
      - pip install awscli
  pre_build:
    commands:
      - echo Deployment started on 'date'
      - echo Syncing S3 Content
      - aws s3 sync ./build/ YOUR_S3_BUCKET
  build:
    commands:
      - echo Invalidating CloudFront Cache
      - aws cloudfront create-invalidation --distribution-id XXXX --paths "/*"
  post_build:
    commands:
      - echo Deployment completed on 'date'
```

# 通过S3和CloudFront进行CDN分发

可以在本地先build好，也可以在CodeBuild中进行build，随后通过AWSCLI将build文件夹下的资源文件上传到对应到S3 Bucket，再触发CloudFront的Invalidation来更新CDN分发。

# 通过Route 53管理域名

可以直接从Route 53完成域名购买与注册等，然后由AWS Certificate Manager来注册域名的SSL证书。这里注意，域名的SSL证书必须注册在us-east-1区域，才能正确的绑定到域名中。