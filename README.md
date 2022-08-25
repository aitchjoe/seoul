# Mirror Mirror on the Wall

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

这种做法当然没有直接使用现成的国内镜像方便，但是在目前并没有一个稳定可靠、足够信赖的大厂镜像服务的情况下，实在无法放心使用小众的二级制成品，因此利用以上基本算一目了然的 Shell 脚本来自给自足，不失为一种选择。但是**注意**：

* 这种做法仍然是在确认 [Amazon ECR Public Gallery](https://gallery.ecr.aws/)、[Quay.io](https://quay.io/)、[Red Hat](https://catalog.redhat.com/software/containers/explore) 等没有可信替代品之后的**无奈之举**。
  * 所以我们想实现的也不是 `gcr.io` 全站的自动 Mirror，如果真的有某个 Registry 网站无法访问、有大量的独占制品需要使用、版本还频繁升级，那么需要的是另外的方案了。
* 以上只是一个最简化的可用版本，最终还是会复杂不少也因此可能会利用现成的 GitHub Actions，但注意尽量使用官方或认证的第三方 Actions。
  * 实际上 Docker 在 [Copy images between registries](https://github.com/docker/build-push-action/blob/master/docs/advanced/copy-between-registries.md) 提及了使用 [tag-push-action](https://github.com/akhilerm/tag-push-action) 完成以上功能且 "_without changing the image SHA_"，但因为相对小众还使用了自己的二进制 [akhilerm/repo-copy:latest](https://github.com/akhilerm/tag-push-action/blob/v2.0.0/src/main.ts#L32) 容器制品，因此舍弃。
