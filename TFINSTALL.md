title: Tensorflow 在CentOS 7下安装
author: qing
date: 2018-06-06
description: Tensorflow 在CentOS 7下安装
tags:
category:
acl: 00

# Tensorflow 在CentOS 7下安装

## 安装和配置

安装virtualenv，用于创建virtual的python运行时。

    pip install virtualenv

创建virtual环境，下面的命令会创建一个新的目录tensorflow，里面包含了virtual环境下的python运行时。

    virtualenv tensorflow

在新的虚拟环境中，安装tensorflow

    pip install tensorflow
    pip install ipykernel
    pip install jupyter
    pip install matplotlib

生成jupyter配置文件


    jupyter notebook --generate-config

修改jupyter配置文件，设置base_url可以使用nginx做reverse proxy。

    c.NotebookApp.notebook_dir = u'/path/to/jupyter/workdir'
    c.NotebookApp.password = u'...'
    c.NotebookApp.base_url = 'notebook'

生成密码

    :::python
    In [1]: from notebook.auth import passwd
    In [2]: passwd()
    Enter password:
    Verify password:
    Out[2]: 'sha1:67c9e60bb8b6:9ffede0825894254b2e042ea597d771089e11aed'

将Out[2]中的内容复制到`c.NotebookApp.password`后面，密码才能生效。

安装supervisor

    yum install -y supervisor

在`/etc/supervisord.d`下添加jupyter.ini

    [program:jupyter]
    command=/bin/bash -c "source /path/to/activate; jupyter notebook --no-browser --allow-root --config=/path/to/jupyter/config"
    directory=/path/to/jupyter/workdir

使用下面命令启动

    supervisorctl reread
    supervisorctl add jupyter

nginx 添加反向代理，然后重启服务。

    location /notebook {
        proxy_pass http://127.0.0.1:8888;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_http_version 1.1;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
