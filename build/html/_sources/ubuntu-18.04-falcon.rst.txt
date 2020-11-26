.. contents::
   :depth: 3
..

ubuntu 18.04 falcon安装使用
===========================

**1、安装mysql数据库redis**

.. code:: bash

   apt-get install -y redis
   apt-get install -y mysql-server

**2、安装go语言环境**

.. code:: bash

   apt-get install -y go

**3、创建安装目录下载源代码**

.. code:: bash

   mkdir -p $GOPATH/src/github.com/open-falcon
   cd $GOPATH/src/github.com/open-falcon
   git clone https://github.com/open-falcon/falcon-plus.git
   #$GOPATH 安装路径就可以
   export GOPATH=/data/go
   # make all modules
   make all

   # make specified module
   make agent

   # pack all modules
   make pack

**4、初始化数据库**

.. code:: bash

   mysql -h 127.0.0.1 -u root -p < 1_uic-db-schema.sql
   mysql -h 127.0.0.1 -u root -p < 2_portal-db-schema.sql
   mysql -h 127.0.0.1 -u root -p < 3_dashboard-db-schema.sql
   mysql -h 127.0.0.1 -u root -p < 4_graph-db-schema.sql
   mysql -h 127.0.0.1 -u root -p < 5_alarms-db-schema.sql

**5、修改配置文件中的数据库连接用户和密码**

.. code:: bash

   grep -Ilr 3306  ./ | xargs -n1 -- sed -i 's/root:/real_user:real_password/g'

**6、启动服务**

.. code:: bash

   ./open-falcon start
   ./open-falcon check

7、前端安装

.. code:: bash

   #创建安装路径
   mkdir /data/falcon-dashboard
   git clone https://github.com/open-falcon/dashboard.git
   #安装依赖包
   apt-get install -y python-virtualenv
   apt-get install -y slapd ldap-utils
   apt-get install -y libmysqld-dev
   apt-get install -y build-essential
   apt-get install -y python-dev libldap2-dev libsasl2-dev libssl-dev

   #设置虚拟python环境，安装python包
   virtualenv ./env
   ./env/bin/pip install -r pip_requirements.txt

8、修改dashboard配置

.. code:: bash

   vim rrd/config.py
   # portal database
   # TODO: read from api instead of db
   PORTAL_DB_HOST = os.environ.get("PORTAL_DB_HOST","127.0.0.1")
   PORTAL_DB_PORT = int(os.environ.get("PORTAL_DB_PORT",3306))
   PORTAL_DB_USER = os.environ.get("PORTAL_DB_USER","root")
   PORTAL_DB_PASS = os.environ.get("PORTAL_DB_PASS","123456")
   PORTAL_DB_NAME = os.environ.get("PORTAL_DB_NAME","falcon_portal")

   # alarm database
   # TODO: read from api instead of db
   ALARM_DB_HOST = os.environ.get("ALARM_DB_HOST","127.0.0.1")
   ALARM_DB_PORT = int(os.environ.get("ALARM_DB_PORT",3306))
   ALARM_DB_USER = os.environ.get("ALARM_DB_USER","root")
   ALARM_DB_PASS = os.environ.get("ALARM_DB_PASS","123456")
   ALARM_DB_NAME = os.environ.get("ALARM_DB_NAME","alarms")

9、启动服务

.. code:: bash

   ./control start 

测试falcon报警
==============

*1、自定义监控项*

.. code:: bash

   #!/bin/bash
   source ./test
   metric="mysql.status.down"
   hostname=`hostname`

   ts=`date -d "1 minutes ago" +%s`;

   curl --connect-timeout 2 -m 5 -X POST -d "[{\"metric\": \"${metric}\", \"endpoint\": \"${hostname}\", \"timestamp\": $ts,\"step\": 60,\"value\": ${value},\"counterType\": \"GAUGE\",\"tags\": \"idc=bj,project=falcon\"}]" http://127.0.0.1:1988/v1/push

*2、创建模板设置报警策略*

.. figure:: D:\typora图片保存\1601195897554.png
   :alt: 1601195897554

   1601195897554

*3、设置hostgroups*

.. figure:: D:\typora图片保存\1601195928887.png
   :alt: 1601195928887

   1601195928887

-  绑定模板，添加hosts。

.. figure:: D:\typora图片保存\1601195966985.png
   :alt: 1601195966985

   1601195966985

-  测试告警成功

设置邮件告警
------------

*1、安装 mail-provider*

.. code:: bash

   cd /data/open-falcon/
   git clone https://github.com/open-falcon/mail-provider.git
   cd mail-provider
   go get ./...
   ./control build
   #也可以下载编译好的包
   #wget https://dl.cactifans.com/open-falcon/falcon-mail-provider.tar.gz

*2、配置配置文件*

.. code:: bash

   root@falcon:/data/open-falcon/mail-provider# cat cfg.json 
   {
       "debug": true,
       "http": {
           "listen": "0.0.0.0:4000",
           "token": ""
       },
       "smtp": {
           "addr": "smtp.163.com:25",
           "username": "xxx@163.com",
           "password": "xxx",   #授权码
           "from": "xxx", #与username相同
           "tls":false,
           "anonymous":false,
           "skipVerify":true
       }
   }

*3、测试发送邮件*

.. code:: bash

   curl http://127.0.0.1:4000/sender/mail -d "tos=你的邮箱&subject=报警测试&content=这是一封测试邮件"
   #注意修改邮箱
   #返回success说明配置成功

*4、修改alarm配置文件*

.. code:: bash

   "mail": "http://127.0.0.1:4000/sender/mail",

.. figure:: D:\typora图片保存\1601200050230.png
   :alt: 1601200050230

   1601200050230

邮件报警配置成功
----------------
