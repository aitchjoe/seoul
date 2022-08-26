# Mirror Mirror on the Wall

_注意由于镜像有容器 Image 和 Mirror 两种含义，因此以下内容统一用容器制品表示 Image、镜像表示 Mirror。_

## 说明

针对无法访问 `gcr.io` 等容器制品的限制，利用 GitHub / [GitLab SaaS](https://docs.gitlab.com/ee/subscriptions/gitlab_com/) 能够被国内访问（？）、且运行其上的 GitHub Actions / GitLab CI 可以访问外网的这种情况，就能轻松配置 Job 来制作国内可直接访问的镜像，一个最简单的 GitLab Actions 示例如下：

```
name: Mirror container image
on: workflow_dispatch
jobs:
  mirror-image:
    runs-on: ubuntu-22.04
    steps:
      - name: Mirror image
        run: |
          IMAGE_TAG=v1.8.0
          SOURCE_IMAGE="gcr.io/kaniko-project/executor:$IMAGE_TAG"
          TARGET_REGISTRY="ghcr.io"
          TARGET_IMAGE="$TARGET_REGISTRY/$GITHUB_REPOSITORY/kaniko-executor:$IMAGE_TAG"
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login $TARGET_REGISTRY -u $ --password-stdin
          docker pull $SOURCE_IMAGE
          docker tag $SOURCE_IMAGE $TARGET_IMAGE
          docker push $TARGET_IMAGE
```

其中利用了 [Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)（`GITHUB_TOKEN`）及 [Default environment variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables#default-environment-variables)（`GITHUB_REPOSITORY`）进一步简化配置操作（GitLab 也有相应的 [Predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)），实际生成的制品见 GitHub [Packages](https://github.com/aitchjoe?tab=packages&repo_name=seoul) 界面。

这种做法当然没有直接使用现成的国内镜像方便，但是在目前并没有一个稳定可靠、足够信赖的大厂镜像服务的情况下，实在无法放心使用小众的二进制成品，因此利用以上基本算一目了然的 Shell 脚本来自给自足，不失为一种选择。但是**注意**：

* 这种做法仍然是在确认 [Amazon ECR Public Gallery](https://gallery.ecr.aws/)、[Quay.io](https://quay.io/)、[Red Hat](https://catalog.redhat.com/software/containers/explore) 等没有可信替代品之后的**无奈之举**。
  * 所以我们想实现的也不是 `gcr.io` 全站的自动镜像，如果真的有某个 Registry 网站无法访问、有大量的独占制品需要使用、版本还频繁升级，那么需要的是另外的方案了。
* 以上只是一个最简化的可用版本，最终还是会复杂不少也因此可能会利用现成的 GitHub Actions，但注意尽量使用官方或认证的第三方 Actions。
  * 实际上 Docker 在 [Copy images between registries](https://github.com/docker/build-push-action/blob/master/docs/advanced/copy-between-registries.md) 提及了使用 [tag-push-action](https://github.com/akhilerm/tag-push-action) 完成以上功能且 "_without changing the image SHA_"，但因为相对小众还使用了自己的 [akhilerm/repo-copy:latest](https://github.com/akhilerm/tag-push-action/blob/v2.0.0/src/main.ts#L32) 容器制品，因此舍弃。

## Without changing the image SHA

虽然我们没有使用 [tag-push-action](https://github.com/akhilerm/tag-push-action) 来复制容器制品，但研究后发现它最终也是通过 Google 的 [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane) 工具来实现的，因此我们直接在脚本中使用了该工具，基本就是：

```
crane auth login -u $ -p ${{ secrets.GITHUB_TOKEN }} $TARGET_REGISTRY
crane cp $SOURCE_IMAGE $TARGET_IMAGE
```

首先验证 "_without changing the image SHA_"，我们对比两种不同方式的复制结果：

* [ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.0](https://github.com/aitchjoe/seoul/pkgs/container/seoul%2Fkaniko-executor/39249057?tag=v1.8.0)：使用 `docker pull/tag/push` 生成。
* [ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.1](https://github.com/aitchjoe/seoul/pkgs/container/seoul%2Fkaniko-executor/39249752?tag=v1.8.1)：使用 `crane cp` 生成。

然后使用以下命令检查结果：

```
# docker pull ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.0 >/dev/null
# docker pull ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.1 >/dev/null
# docker pull gcr.io/kaniko-project/executor:v1.8.0 >/dev/null
# docker pull gcr.io/kaniko-project/executor:v1.8.1 >/dev/null
# docker images --digests|grep kaniko
gcr.io/kaniko-project/executor           v1.8.1      sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650   a2a981eb8745   4 months ago   63.4MB
ghcr.io/aitchjoe/seoul/kaniko-executor   v1.8.1      sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650   a2a981eb8745   4 months ago   63.4MB
gcr.io/kaniko-project/executor           v1.8.0      sha256:ff98af876169a488df4d70418f2a60e68f9e304b2e68d5d3db4c59e7fdc3da3c   62345e1f31a9   5 months ago   63.4MB
ghcr.io/aitchjoe/seoul/kaniko-executor   v1.8.0      sha256:18caa2d67954f03fc7dba94d1ff8ef953bf70ab8a9816f08d9ffe82de163db93   62345e1f31a9   5 months ago   63.4MB
```

可以发现不同 Registry 上的 v1.8.1 的 SHA 是一样的、而 v1.8.0 不同。进一步验证使用源 SHA 拉取生成的容器制品：

```
# docker pull ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.1@sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650 >/dev/null && echo v1.8.1 found
v1.8.1 found
# docker pull ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.0@sha256:ff98af876169a488df4d70418f2a60e68f9e304b2e68d5d3db4c59e7fdc3da3c >/dev/null || echo v1.8.0@source-sha256 not found
Error response from daemon: manifest unknown
v1.8.0@source-sha256 not found
# docker pull ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.0@sha256:18caa2d67954f03fc7dba94d1ff8ef953bf70ab8a9816f08d9ffe82de163db93 >/dev/null && echo v1.8.0@new-sha256 found
v1.8.0@new-sha256 found
```

而镜像保持和源制品相同的 SHA 能提升一定的安全性，参见以下[使用帮助](#使用帮助)。另外使用 `crane cp` 还可以同时创建多平台版本，从 [kaniko-executor](https://github.com/aitchjoe/seoul/pkgs/container/seoul%2Fkaniko-executor) 制品的 OS / Arch  栏可以发现只有 crane 生成的 v1.8.1 才有多个版本，当然这对我们现在的场景帮助不大。

## 使用帮助

无论是自己创建、还是其他的小众镜像，可以使用如下方式相对提升安全性（无法篡改？）。从**其他可信渠道**获取官方容器制品相应版本的 SHA，一个笨办法就是自己运行 GitHub Actions 如 [inspect.yml](.github/workflows/inspect.yml)，然后从 Job 输出中确认如：

* `gcr.io/kaniko-project/executor:v1.8.1` 的 SHA：`sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650`

再按照以下方式使用镜像：

* `ghcr.io/aitchjoe/seoul/kaniko-executor:v1.8.1@sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650`
* 如果容器运行时不支持以上格式则：`ghcr.io/aitchjoe/seoul/kaniko-executor@sha256:b44b0744b450e731b5a5213058792cd8d3a6a14c119cf6b1f143704f22a7c650`
