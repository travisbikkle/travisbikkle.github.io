# Centos Docker 入門


快速上手
========

**install(offline).**

    # 在有網絡的機器上，執行以下命令，獲取安裝所需的包
    $ yum install --downloadonly --downloaddir=/opt/utils yum-utils device-mapper-persistent-data lvm2
    $ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    $ yum install --downloadonly --downloaddir=/opt/all_packages docker-ce docker-ce-cli containerd.io
    root@192.168.31.201:/opt/all_packages #0$ l
    audit-libs-python-2.8.4-4.el7.x86_64.rpm  libcgroup-0.41-20.el7.x86_64.rpm
    checkpolicy-2.5-8.el7.x86_64.rpm          libsemanage-python-2.5-14.el7.x86_64.rpm
    containerd.io-1.2.2-3.3.el7.x86_64.rpm    policycoreutils-2.5-29.el7_6.1.x86_64.rpm
    container-selinux-2.74-1.el7.noarch.rpm   policycoreutils-python-2.5-29.el7_6.1.x86_64.rpm
    docker-ce-18.09.2-3.el7.x86_64.rpm        python-IPy-0.75-6.el7.noarch.rpm
    docker-ce-cli-18.09.2-3.el7.x86_64.rpm    setools-libs-3.3.8-4.el7.x86_64.rpm
    root@192.168.31.201:/opt/all_packages #0$ l ../utils/
    device-mapper-1.02.149-10.el7_6.3.x86_64.rpm             lvm2-2.02.180-10.el7_6.3.x86_64.rpm
    device-mapper-event-1.02.149-10.el7_6.3.x86_64.rpm       lvm2-libs-2.02.180-10.el7_6.3.x86_64.rpm
    device-mapper-event-libs-1.02.149-10.el7_6.3.x86_64.rpm  python-chardet-2.2.1-1.el7_1.noarch.rpm
    device-mapper-libs-1.02.149-10.el7_6.3.x86_64.rpm        python-kitchen-1.1.1-5.el7.noarch.rpm
    libxml2-python-2.9.1-6.el7_2.3.x86_64.rpm                yum-utils-1.1.31-50.el7.noarch.rpm

    # 在離線機器上， 執行以下命令以安裝
    $ yum localinstall /opt/utils/*.rpm
    $ yum localinstall /opt/all_packages/*.rpm

**install docker-compose.**

    $ curl -L &#34;https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)&#34; \
    -o /usr/local/bin/docker-compose
    chmod &#43;x /usr/local/bin/docker-compose
    $ docker-compose --version

**install docker-machine.**

    $ base=https://github.com/docker/machine/releases/download/v0.16.1 &amp;&amp;
      curl -L $base/docker-machine-$(uname -s)-$(uname -m) &gt;/tmp/docker-machine &amp;&amp;
      install /tmp/docker-machine /usr/local/bin/docker-machine

**基本命令.**

    $ service docker start或者systemctl start docker
    $ docker run helloworld
    $ docker version
    $ docker info
    $ docker image ls
    $ docker container ls
    $ docker container ls -a
    $ docker container ls -aq

**鏡像製作、分發.**

    $ docker build --tag=zhaoyu/helloworld:0.0.1 .
    $ docker save zhaoyu/helloworld -o zhaoyu_helloworld_0.0.1.tar
    $ docker load &lt; zhaoyu_helloworld_0.0.1.tar
    $ docker tag zhaoyu/helloworld:0.0.1 zhaoyu/helloworld:0.0.2
    $ docker image ls
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    zhaoyu/helloworld   0.0.1               8a2ec1b2d523        14 hours ago        131MB
    zhaoyu/helloworld   0.0.2               8a2ec1b2d523        14 hours ago        131MB
    python              2.7-slim            99079b24ed51        4 days ago          120MB

**運行，查看，停止，刪除.**

    $ docker run -p 4000:80 zhaoyu/helloworld
    $ docker run -p -d 4000:80 zhaoyu/helloworld
    $ docker run -it --name=zhaoyu zhaoyu/helloworld:0.0.1 /bin/bash
    $ docker attach zhaoyu
    $ docker container ls
    $ docker container top &lt;container_id&gt;
    $ docker container stop &lt;container_id&gt;
    $ docker rm &lt;container_id&gt;
    $ docker rmi &lt;image_id&gt;
    $ docker container stop $(docker ps -a -q)
    $ docker rm $(docker ps -a -q)

**swarm/stack.**

    $ docker swarm init
    $ docker node ls
    $ docker stack deploy -c docker-compose.yml helloworld_swarm
    $ docker stack ls
    $ docker stack ps helloworld_swarm
    ID                  NAME                     IMAGE                     NODE                DESIRED STATE       CURRENT STATE              ERROR               PORTS
    l8c3a4haeccd        helloworld_swarm_web.1   zhaoyu/helloworld:0.0.1   host1               Running             Running 8 seconds ago
    z8pz8s0zh6b1        helloworld_swarm_web.2   zhaoyu/helloworld:0.0.1   host1               Running             Preparing 17 seconds ago
    yd5qyb7q146x        helloworld_swarm_web.3   zhaoyu/helloworld:0.0.1   host1               Running             Running 1 second ago
    82o2in6wudci        helloworld_swarm_web.4   zhaoyu/helloworld:0.0.1   host1               Running             Preparing 17 seconds ago
    lidmd9n70wnz        helloworld_swarm_web.5   zhaoyu/helloworld:0.0.1   host1               Running             Running 2 seconds ago

    $ docker service ps helloworld_swarm
    $ docker stack rm helloworld_swarm

    $ docker swarm leave --force

概覽
====

1.  docker daemon

2.  docker client

3.  docker registries

4.  docker objects

    -   images

    -   containers

    -   services

5.  underlying technology

    -   namespaces

    -   control group

    -   union file systems

    -   container format

開始入門
========


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/centos7-docker-getstart/  

