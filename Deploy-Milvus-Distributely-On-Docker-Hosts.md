# Milvus分布式部署到多台Docker Host
本篇文档将介绍如何创建Milvus分布式部署，并且提供Ansible Playbook创建所需的Docker Host，以及Docker Container来运行分布式Milvus.
### 前置条件：
    Docker Host. 准备3台虚拟机，并保护网络畅通。建议资源配置：4CPU, 8GB内存，100GB磁盘空间。用户可以根据自身备具的条件向上或向下调整资源配置，但最低保持2CPU, 4GB内存。
    虚拟机操作系统，Ubuntu 20.04 LTS
    Ansible admin controller，任何能够运行Python与Ansible的设备都可以。用户如果需要新建Ansible admin controller，建议选择Ubuntu操作系统，系统资源保证最低限度能够流畅地运行Ansible任务即可。
    其它的依赖项将会在Playbook中安装与设置，在后面的内容会逐步说明。
### 开始安装Docker
#### Ansible主机清单Inventory文件可以将主机分组，在执行相同任务时可以按组分配。

    [dockernodes] #方括号表示组名，作用为创建主机组。在Playbook中引用组名，会部署到该组下所包含的主机。"dockernodes"组适合于Docker安装的任务，所有的Docker主机安装任务都相同。
    dockernode01 #根据实际Hostname替换此处默认值
    dockernode02
    dockernode03

    [admin]
    ansible-controller

    [coords]
    dockernode01

    [nodes]
    dockernode02

    [dependencies]
    dockernode03

    [docker:children] #"docker"组是为了定义变量而创建的组。
    dockernodes
    coords
    nodes
    dependencies

    [docker:vars] #定义变量，这里定义的变量此组下所有的成员都可以引用。
    ansible_python_interpreter=/usr/bin/python3
    StrictHostKeyChecking=no
    ```
#### Ansible配置文件Ansible.cfg文件可以控制Playbook中的行为，例如ssh key等
    ```
    [defaults]
    host_key_checking = False
    inventory = inventory.ini #定义Inventory引用文件，如不定义则需要在Ansible-Playbook命令中使用 "-i" 参数再加入文件地址。
    private_key_file=~/.my_ssh_keys/gpc_sshkey #Ansible访问Docker主机的SSH钥匙，如主机上不需要SSH则可以删除此处。
    ```
#### Ansible运行脚本deploy-docker.yml中详细定义了安装Docker的任务。
    ```
    - name: setup pre-requisites #安装前置条件
    hosts: all #指定执行该任务的主机，在Inventory下定义的组在此可以引用
    become: yes #提升执行任务的权限
    become_user: root
    roles:
        - install-modules #预配置的任务，安装curl, wget, python, ntp, python-pip等工具。详细任务参考文件 .\roles\install-modules\main.yml
        - configure-hosts-file #预配置的任务，添加Host记录。详细任务参考文件 .\roles\configure-hosts-file\tasks\main.yml
    - name: install docker #安装Docker
    become: yes
    become_user: root
    hosts: dockernodes
    roles:
        - docker-installation #安装Docker。详细任务参考文件 .\roles\docker-installation\tasks\main.yml
    ```
#### 运行Ansible任务脚本。
###### 在系统terminal中进入脚本的目录下，运行ansible all -m ping，如果未在指定ansible.cfg中指定inventory，则需要加入"-i"并指定路径，否则ansible将引用/etc/ansible/hosts的主机地址。返回的结果如下：
    ```
    dockernode01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
    }
    ansible-controller | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python3"
        },
        "changed": false,
        "ping": "pong"
    }
    dockernode03 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    dockernode02 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    ```
##### 运行ansible-playbook deploy-docker.yml --syntax-check检查脚本是否有语法错误，返回的正常结果如下：playbook: deploy-docker.yml
##### 运行ansible-playbook deploy-docker.yml，部分返回结果如下：
    TASK [docker-installation : Install Docker-CE] *******************************************************************
    ok: [dockernode01]
    ok: [dockernode03]
    ok: [dockernode02]

    TASK [docker-installation : Install python3-docker] **************************************************************
    ok: [dockernode01]
    ok: [dockernode02]
    ok: [dockernode03]

    TASK [docker-installation : Install docker-compose python3 library] ******************************************************************************************************************
    changed: [dockernode01]
    changed: [dockernode03]
    changed: [dockernode02]

    PLAY RECAP ********************************************************************************************************
    ansible-controller         : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    dockernode01               : ok=10   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    dockernode02               : ok=10   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    dockernode03               : ok=10   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
##### 到这里Docker就已经成功地安装到了3台主机上，接下来我们检查Docker安装是否成功。
    SSH分别登录到3台主机，运行docker -v，root以外的帐户运行sudo docker -v，返回结果如下：
    root@swarm-manager:~$ docker -v
    Docker version 20.10.14, build a224086
    运行docker ps，初始状态下，返回结果没有运行的container。
### 创建Milvus
##### 检查deploy-milvus.yml，在进入目录后运行ansible-playbook deploy-milvus.yml --syntax-check来检查语法错误，正常的返回结果为：
``` playbook: deploy-milvus.yml ```

##### 运行 ansible-playbook deploy-milvus.yml，创建Milvus的任务已在deploy-milvus.yml中定义，在脚本中有详细说明。
###### 返回结果如下：
    PLAY [Create milvus-etcd, minio, pulsar, network] *****************************************************************

    TASK [Gathering Facts] ********************************************************************************************
    ok: [dockernode03]

    TASK [etcd] *******************************************************************************************************
    changed: [dockernode03]

    TASK [pulsar] *****************************************************************************************************
    changed: [dockernode03]

    TASK [minio] ******************************************************************************************************
    changed: [dockernode03]

    PLAY [Create milvus nodes] ****************************************************************************************

    TASK [Gathering Facts] ********************************************************************************************
    ok: [dockernode02]

    TASK [querynode] **************************************************************************************************
    changed: [dockernode02]

    TASK [datanode] ***************************************************************************************************
    changed: [dockernode02]

    TASK [indexnode] **************************************************************************************************
    changed: [dockernode02]

    PLAY [Create milvus coords] ***************************************************************************************

    TASK [Gathering Facts] ********************************************************************************************
    ok: [dockernode01]

    TASK [rootcoord] **************************************************************************************************
    changed: [dockernode01]

    TASK [datacoord] **************************************************************************************************
    changed: [dockernode01]

    TASK [querycoord] *************************************************************************************************
    changed: [dockernode01]

    TASK [indexcoord] *************************************************************************************************
    changed: [dockernode01]

    TASK [proxy] ******************************************************************************************************
    changed: [dockernode01]

    PLAY RECAP ********************************************************************************************************
    dockernode01               : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    dockernode02               : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    dockernode03               : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

    到这里Milvus已部署到3台Docker主机上，接下来可以参考[Hello Milvus](https://milvus.io/docs/v2.0.x/example_code.md)进行一个hello_milvus.py的测试。