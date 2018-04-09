title: RGW
author: qing
date: 2018-04-08
description: Ceph RWG
tags:
category:
acl: 00

# Ceph RGW 代码分析

## daemon 启动

通过dup2将STDERR重定向到STDOUT，分析命令行参数，获取配置文件列表，解析配置文件，根据配置文件内容设置`cct->_conf`数据结构, `cct`是`CephContext`的一个对象。选择API的Frontend。 接着调用`global/global_init.cc::global_init`，该函数会设置中断任务和完成用户切换，使用命令行中指定的用户运行程序。接着创建运行时目录，默认是`/var/run/ceph`。调用所有全局配置的观察者，通知他们配置发生了变化。这里的配置变化应该是由命令行参数的解析和配置文件的解析引起的。从代码注释看对全局配置的订阅应该是最近引入的，在`common/config.h::md_confg_t`中实现。设置运行目录，日志的所有者为当前程序的所有者和组。调用`rgw_tools_init`通过外部文件初始化MIME映射（文件后缀->MIME Type)。初始化DNS解析器。初始化Rados。启动性能计数。初始化REST，初始化rgw属性到http属性的映射，比如`content_language`->`Content-Language`。初始化http属性到rgw属性。TODO：为什么要加HTTP_?根据`rgw_extended_http_attrs`初始化HTTP扩展属性。初始化HTTP CODE到Text map(`http_status_names`)，比如`404`->`Not Found`。将zone group中的所有主机名称放入`hostnames_set`中。将zone group中的所有主机的S3名称放入`hostnames_s3website_set`中。初始化rgw用户(TODO: rgw chained cache和store的关系)。初始化rgw bucket(RGWMetadataManager)。根据设置决定要启用哪些API(比如，S3，Swift，swift_auth, admin)。将设置完的rest做为参数启动frontend framework。

## RGW Rest

## RGW REST

`class RGWREST`是REST的入口函数，其他使用rest功能的代码一般都在这里开始，比如注册REST Manager或者获取处理handler。

### RGW Framework

由于现在RGW支持几种不同的Framework，在代码中定义了`class RGWFrontend`类抽象了不同Framework的操作。

    :::c++
    class RGWFrontend {
    public:
      virtual ~RGWFrontend() {}
    
      virtual int init() = 0;
    
      virtual int run() = 0;
      virtual void stop() = 0;
      virtual void join() = 0;
    
      virtual void pause_for_new_config() = 0;
      virtual void unpause_with_new_config(RGWRados *store) = 0;
    };

J版中RGW默认使用`RGWMongooseFrontend`，这个类继承自`RGWFrontend`。下面是对这个类的分析。


#### RGWMongooseFrontend

在`rgw_civetweb_frontend.cc::RGWMongooseFrontend::run`中设置civeweb的启动参数，包括端口，keep alive,线程数等。设置回调函数,包括`begin_request`, `log_message`, `log_access`。`begin_request`最终会调用`rgw_process.cc::process_request`，`process_request`通过`rest->get_handler`获取资源的handler, `get_handler`其实就是URL分发的过程，具体步骤如下节。资源handler使用正确的op来最终处理这个请求。

### URL分发

RGWRESTMgr这个类主要负责资源的注册，url分发。如下代码是`rgw_main.cc::main`的节选，其中`RGWRESTMgr_Admin`, `RGWRESTMgr_Usage`, `RGWRESTMgr_User`...是`RGWRESTMgr`的子类,从`register_resource`可以看到他们的层级关系`RGWRESTMgr_Usage`, `RGWRESTMgr_User`...位于`RGWRESTMgr_Admin`的下层。

    :::c++
    RGWRESTMgr_Admin *admin_resource = new RGWRESTMgr_Admin;
    admin_resource->register_resource("usage", new RGWRESTMgr_Usage);
    admin_resource->register_resource("user", new RGWRESTMgr_User);
    admin_resource->register_resource("bucket", new RGWRESTMgr_Bucket);

URL分发是资源注册的反向过程，通过URL找到对应的handler。如下是获取handler的过程，通过URL找到对应的`RGWRESTMgr`，再根据manager来获得handler。

    :::c++
    RGWHandler_REST* RGWREST::get_handler(RGWRados *store, struct req_state *s,
    				      RGWStreamIO *sio, RGWRESTMgr **pmgr,
    				      int *init_error)
    {
      ...
    
      RGWRESTMgr *m = mgr.get_resource_mgr(s, s->decoded_uri, &s->relative_uri);
      if (!m) {
        *init_error = -ERR_METHOD_NOT_ALLOWED;
        return NULL;
      }
 
      ...   

      handler = m->get_handler(s);
      if (!handler) {
        *init_error = -ERR_METHOD_NOT_ALLOWED;
        return NULL;
      }

      ...

    } /* get stream handler */

获取RGWRESTMgr的过程比较简单，就是使用`/`拆分URL然后逐级查找对应的resource。

    :::c++
    RGWRESTMgr *RGWRESTMgr::get_resource_mgr(struct req_state *s, const string& uri, string *out_uri)
    {
      *out_uri = uri;
    
      multimap<size_t, string>::reverse_iterator iter;
    
      for (iter = resources_by_size.rbegin(); iter != resources_by_size.rend(); ++iter) {
        string& resource = iter->second;
        if (uri.compare(0, iter->first, resource) == 0 &&
    	(uri.size() == iter->first ||
    	 uri[iter->first] == '/')) {
          string suffix = uri.substr(iter->first);
          return resource_mgrs[resource]->get_resource_mgr(s, suffix, out_uri);
        }
      }
    
      if (default_mgr)
        return default_mgr;
    
      return this;
    }


REST基本操作GET/PUT/POST/DELETE...在RGWHandler_REST中定义，这个类主要做为基类给其他类使用。

## RGW store
