从测试看storlet的部署过程
=========================

storlet的测试入口`.unittest`如下：

``` shell
#! /bin/bash

SRC_DIR=$(python -c "import os; print(os.path.dirname(os.path.realpath('$0')))")    #获取当前目录绝对地址
cd ${SRC_DIR}/tests/unit    #进入测试目录
nosetests --exe -v $@   #执行测试，返回结果
rvalue=$?   #返回上次的返回值
cd -    #返回上一次工作目录

exit $rvalue
```

nosetest是一个python测试模块，它能够检测当前目录下所有的测试文件、方法并执行测试。然后我们的目的是为了理解storlet的运行流程，那么可以先找到一个一个用于测试某个storlet功能的用例，比如`tests/functional/java/test_compress_storlet.py`的部分代码如下:

``` python
from swiftclient import client as c
from tests.functional.java import StorletJavaFunctionalTest
import unittest


class TestCompressStorlet(StorletJavaFunctionalTest):
    def setUp(self):
        self.storlet_log = ''
        self.additional_headers = {}
        main_class = 'org.openstack.storlet.compress.CompressStorlet'
        super(TestCompressStorlet, self).setUp('CompressStorlet',
                                               'compressstorlet-1.0.jar',
                                               main_class,
                                               'input.txt')

    def test_put(self):
        headers = {'X-Run-Storlet': self.storlet_name}
        headers.update(self.additional_headers)
        querystring = "action=compress"

        # simply set 1KB string data to compress
        data = 'A' * 1024

··· ···
```

先去看一下它的父类`StorletJavaFunctionalTest`,它的代码如下：

``` python
import os
from tests.functional import StorletFunctionalTest, PATH_TO_STORLETS

BIN_DIR = 'bin'


class StorletJavaFunctionalTest(StorletFunctionalTest):
    def setUp(self, storlet_dir, storlet_name, storlet_main,
              storlet_file, dep_names=None, headers=None):
        storlet_dir = os.path.join('java', storlet_dir)
        path_to_bundle = os.path.join(PATH_TO_STORLETS, storlet_dir,
                                      BIN_DIR)
        super(StorletJavaFunctionalTest, self).setUp('Java',
                                                     path_to_bundle,
                                                     storlet_dir,
                                                     storlet_name,
                                                     storlet_main,
                                                     storlet_file,
                                                     dep_names,
                                                     headers)
```

可以看到它的逻辑很简单，只是在语言选项那里换成了java，想必另外一个用python写的storlet也是类似。再看它的父类`StorletFunctionalTest`:

``` python
import unittest
import uuid

from swiftclient import client as swiftclient
from storlets.tools.cluster_config_parser import ClusterConfig
from storlets.tools.utils import deploy_storlet, get_admin_auth, put_local_file
import os

CONFIG_DIR = os.environ.get('CLUSTER_CONF_DIR', os.getcwd())
CONFIG_FILE = os.path.join(CONFIG_DIR, 'cluster_config.json')
PATH_TO_STORLETS = os.environ.get(
    'STORLET_SAMPLE_PATH',
    # assuming, current working dir is at top of storlet repo
    os.path.join(os.getcwd(), 'StorletSamples'))
CONSOLE_TIMEOUT = 2


class StorletBaseFunctionalTest(unittest.TestCase):
    def setUp(self):
        self.conf_file = CONFIG_FILE
        try:
            self.conf = ClusterConfig(CONFIG_FILE)
        except IOError:
            self.fail('cluster_config.json not found in %s. '
                      'Please check your testing environment.' % CONFIG_DIR)

        self.url, self.token = get_admin_auth(self.conf)
        # TODO(kota_): do we need to call setUp() when inheriting TestCase
        # directly? AFAIK, no setUp method in the class...
        super(StorletBaseFunctionalTest, self).setUp()


class StorletFunctionalTest(StorletBaseFunctionalTest):

    def create_container(self, container):
        response = dict()
        swiftclient.put_container(self.url, self.token,
                                  container, headers=None,
                                  response_dict=response)
        status = response.get('status')
        assert (status >= 200 or status < 300)

    def cleanup_container(self, container):
        # list all objects inside the container
        _, objects = swiftclient.get_container(
            self.url, self.token, container, full_listing=True)

        # delete all objects inside the container
        # N.B. this cleanup could run in parallel but currently we have a few
        # objects in the user testing container so that, currently this does
        # as sequential simply
        for obj_dict in objects:
            swiftclient.delete_object(
                self.url, self.token, container, obj_dict['name'])
        swiftclient.get_container(self.url, self.token, container)

        # delete the container
        swiftclient.delete_container(self.url, self.token, container)

    def setUp(self, language, path_to_bundle,
              storlet_dir,
              storlet_name, storlet_main, storlet_file,
              dep_names, headers):
        super(StorletFunctionalTest, self).setUp()
        self.storlet_dir = storlet_dir
        self.storlet_name = storlet_name
        self.storlet_main = storlet_main
        self.dep_names = dep_names
        self.path_to_bundle = path_to_bundle
        self.container = 'container-%s' % uuid.uuid4()
        self.storlet_file = storlet_file
        self.headers = headers or {}
        self.acct = self.url.split('/')[4]
        self.deps = []
        if dep_names:
            for d in self.dep_names:
                self.deps.append('%s/%s' % (self.path_to_bundle, d))
        storlet = '%s/%s' % (self.path_to_bundle, self.storlet_name)

        deploy_storlet(self.url, self.token,
                       storlet, self.storlet_main,
                       self.deps, language)

        self.create_container(self.container)
        if self.storlet_file:
            put_local_file(self.url, self.token,
                           self.container,
                           self.path_to_bundle,
                           self.storlet_file,
                           self.headers)

    def tearDown(self):
        self.cleanup_container(self.container)
```

可以看到这个类是我们想找的类，它封装了storlet的部署流程。下面就来分析一下这个类。

他首先定义了一些依赖，比如docker和编译好的storlet(在这里是jar,python那边有所不同)。先看一下docker的配置文件`cluster_config.json`。我的配置文件如下：

``` json
{
    "groups" : {
        "storlet-mgmt": ["127.0.0.1"],
        "storlet-proxy": ["127.0.0.1"],
        "storlet-storage": ["127.0.0.1"],
        "docker": ["127.0.0.1"]
    },
    "all" : {
        "ansible_ssh_user" : "daqu",
        "docker_device": "/home/docker_device",
        "storlet_source_dir": "~/storlets/",
        "python_dist_packages_dir": "usr/local/lib/python2.7/dist-packages",
        "storlet_gateway_conf_file": "/etc/swift/storlet_docker_gateway.conf",
        "keystone_endpoint_host": "127.0.0.1",
        "keystone_public_url": "http://127.0.0.1/identity/v3",
        "keystone_admin_url": "http://127.0.0.1/identity_admin",
        "keystone_admin_password": "admin",
        "keystone_admin_user": "admin",
        "keystone_admin_project": "admin",
        "keystone_default_domain": "default",
        "keystone_auth_version": "3",
        "swift_endpoint_host": "127.0.0.1",
        "swift_run_time_user" : "daqu",
        "swift_run_time_group" : "daqu",
        "swift_run_time_dir": "/opt/stack/data/swift/run",
        "storlets_management_user": "daqu",
        "storlet_management_account": "storlet_management",
        "storlet_management_admin_username": "storlet_manager",
        "storlet_manager_admin_password": "storlet_manager",
        "storlet_management_swift_topology_container": "swift_cluster",
        "storlet_management_swift_topology_object": "cluster_config.json",
        "storlet_management_ansible_dir": "/opt/ibm/ansible/playbook",
        "storlet_management_install_dir": "/opt/ibm",
        "storlets_enabled_attribute_name": "storlet-enabled",
        "docker_registry_random_string": "ABCDEFGHIJABCDEFGHIJABCDEFGHIJABCDEFGHIJABCDEFGHIJABCDEFGHIJ1234",
        "docker_registry_port": "5001",
        "container_install_dir": "/opt/storlets",
        "storlets_default_project_name": "test",
        "storlets_default_project_user_name": "tester",
        "storlets_default_project_user_password": "testing",
        "storlets_default_project_member_user" : "tester_member",
        "storlets_default_project_member_password" : "member",
        "base_image_maintainer": "root",
        "base_os_image": "ubuntu_16.04",
        "storlets_image_name_suffix": "ubuntu_16.04_jre8_storlets",
        "swift_user_id": "1003",
        "swift_group_id": "1003",
        "storlet_middleware": "storlet_handler",
        "storlet_container": "storlet",
        "storlet_dependency": "dependency",
        "storlet_log": "storletlog",
        "storlet_images": "docker_images",
        "storlet_timeout": "40",
        "storlet_gateway_module": "docker",
        "storlet_execute_on_proxy_only": "false",
        "restart_linux_container_timeout": "3"
    }
}
```

从上面可以看出，这个配置文件主要是设置docker中的storlet。

然后便是编译好的storlet
jar包。这个需要手动复制到`functional`目录下，否则测试会报错。

接下来便是`StorletBaseFunctionalTest`，它先在读取docker配置文件，再通过get\_admin\_auth方法获取swift认证，事实上这个方法只是对swiftclient.get\_auth方法进行了进一步的封装。然后这个类就结束了。它的功能很简单，读取配置和向swift发起认证请求。在这里值得一提的是get\_admin\_auth方法被定义在`storlets/tools/utils.py`中，这个模块还定义了如何上传一个storlet的过程。作者先是定义了一个将本地文件上传到swift的方法`put_local_file`，然后在定义一个方法`put_storlet_object`来调用前者上传storlet。在第二个方法中，作者指定了storlet上传的容器是storlet，并且在请求元数据中加入了和storlet相关的部分，比如：

``` html
'X-Object-Meta-Storlet-Language': language,
'X-Object-Meta-Storlet-Interface-Version': '1.0',
'X-Object-Meta-Storlet-Object-Metadata': 'no',
'X-Object-Meta-Storlet-Main': storlet_main_class
```

此外，还有一个方法`put_storlet_executable_dependencies`，它的作用和`put_storlet_object`类似，但是它上传的是依赖，上传的容器是dependency。

当一个storlet和它的依赖都被上传时，这个storlet就算部署成功了。

``` python
def deploy_storlet(url, token, storlet, storlet_main_class, dependencies,
                   language='Java'):
    """
    Deploy storlet file and required dependencies as swift objects

    :param url: swift endpoint url
    :param token: token string to access swift
    :param storlet: storlet file to be registerd
    :param dependencies: a list of dependency files to be registered
    :param language: storlet language. default value is Java
    """
    # No need to create containers every time
    # put_storlet_containers(url, token)
    put_storlet_object(url, token, storlet,
                       ','.join(os.path.basename(x) for x in dependencies),
                       storlet_main_class, language)

    put_storlet_executable_dependencies(url, token, dependencies)
```

然后是`StorletFunctionalTest(StorletBaseFunctionalTest)`，这个类其实是对刚才介绍的util模块的一次应用。在这个模块对这些方法又进行了封装。

至此，整个storlet的溯源流程就结束了。总结一下：

-   strolet的类结构从上到下为：
-   unittest.TestCase
-   StorletFunctionalTest
-   StorletJavaFunctionalTest(StorletPythonFunctionalTest)
-   TestCompressStorlet(实际的storlet)
-   每一层的类都是对storlet/util.py中操作的进一步封装
-   storlet/util中定义部署storlet的方法

