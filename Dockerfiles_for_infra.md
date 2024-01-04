We are using few docker files which needs used as executers. for that we use below docker files 

```sh
ARG DOCKER_VERSION
FROM docker:${DOCKER_VERSION}
LABEL org.opencontainers.image.vendor="example"
LABEL org.opencontainers.image.title="example and docker-compose"

RUN apk update && \
    apk add bash py-pip python3-dev libffi-dev openssl-dev gcc libc-dev make cargo git aws-cli git curl yq jq openjdk17 maven && \
    pip install --upgrade pip && \
    pip install "cython<3.0.0" wheel && pip install pyyaml==5.4.1 --no-build-isolation && \
    pip install setuptools && \
    pip install --no-use-pep517 bcrypt && \
    pip install docker-compose && \
    mkdir /app

COPY --from=docker/buildx-bin /buildx /usr/libexec/docker/cli-plugins/docker-buildx
WORKDIR /app
```

Another docker files looks like this. 
This docker image provides the following common tools for infrastructure related tasks:
- Terraform
- AWS CLI
- Kubectl
- Helm
- yq
- jq
- curl

```sh
FROM registry.gitlab.com/gitlab-org/terraform-images/stable:latest
# TODO: Use a better base image?
LABEL org.opencontainers.image.vendor="example"
LABEL org.opencontainers.image.title="example Infrastructure Tools"
ARG TARGETARCH
# Upgrade yq from parent image to >4.25
# AWS API v2
# hcledit for editing Terraform scripts.
# Pin boto3 and botocore to avoid breaking changes with AWS API v2
RUN apk -Uuv add binutils py3-pip ansible bash git curl jq openssl && \
    apk add --upgrade yq --repository=http://dl-cdn.alpinelinux.org/alpine/edge/community && \
    pip install --upgrade pip && \
    pip install --no-cache-dir pre-commit && \
    pip install --no-cache-dir awscli && \
    pip install --no-cache-dir boto3==1.24.82 botocore==1.27.82 && \
    wget -q https://github.com/minamijoyo/hcledit/releases/download/v0.2.5/hcledit_0.2.5_linux_${TARGETARCH}.tar.gz -O - | tar -xzO hcledit > /usr/local/bin/hcledit && \
    chmod +x /usr/local/bin/hcledit && \
    apk --purge -v del py-pip && \
    rm -rf /var/cache/apk/*
    
# Install kubectl & Helm
RUN curl -L -O -s https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/${TARGETARCH}/kubectl && \
    chmod +x ./kubectl && \
    mv ./kubectl /usr/local/bin/kubectl && \
    wget -q https://get.helm.sh/helm-v3.6.3-linux-${TARGETARCH}.tar.gz -O - | tar -xzO linux-${TARGETARCH}/helm > /usr/local/bin/helm && \
    chmod +x /usr/local/bin/helm && \
    helm repo add "stable" "https://charts.helm.sh/stable" --force-update

# Install jfrog cli (the one present on alpine repository is old)
RUN wget -O /usr/bin/jf https://releases.jfrog.io/artifactory/jfrog-cli/v2-jf/2.25.0/jfrog-cli-linux-${TARGETARCH}/jf && \
    chmod +x /usr/bin/jf
RUN mkdir -p /app/example_infra/common/
COPY ./scripts /app/example_infra/common/
RUN chmod +x /app/example_infra/common/*.sh

CMD bash
```
