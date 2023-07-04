# cis-c-doc
cis-c document for PAS

# 下发七层业务
## 配置方法
1. 从VIP网段中给应用分配一个VIP
2. 给VIP配置域名映射，使用应用容器域名（即ingress资源里的spec.rules.host），提供域名入口对应VIP
2. 给应用服务 Service 打 label
3. 下发负载均衡 Configmap 配置（关联VIP和应用服务）
4. 观察 BIGIP 上对应配置是否已经生效


## 应用具体操作
1. 给应用服务 Service 打上label
```
label：
  cis.f5.com/as3-tenant=<NAMESPACE_NAME>
  cis.f5.com/as3-app=<SERVICE_NAME>
  cis.f5.com/as3-app=<NAMESPACE_NAME>_<NAMESPACE_NAME>_pool
```
举例：
```
kubectl label svc app-2-svc cis.f5.com/as3-tenant=test002 -n test002
kubectl label svc app-2-svc cis.f5.com/as3-app=app-2-svc -n test002
kubectl label svc app-2-svc cis.f5.com/as3-pool=test002_app-2-svc_pool -n test002
```

2. 下发负载均衡配置 Configmap
注意： 根据实际应用信息替换变量值
`<SERVICE_NAME>`:  应用服务Service名称
`<SERVICE_PORT>`:  应用服务Service端口
`<NAMESPACE_NAME>`: 应用服务所在的命名空间Namespace名称
`<APP_VIP>`: 分配给应用的入口VIP

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: <SERVICE_NAME>_cfg
  namespace: <NAMESPACE_NAME>
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
      "class": "AS3",
      "action": "deploy",
      "persist": true, 
      "declaration": {
        "class": "ADC",
        "schemaVersion": "3.11.0",
        "remark": "HTTP application",
        "<NAMESPACE_NAME>": {
          "class": "Tenant",
          "<SERVICE_NAME>": {
            "class": "Application",
            "template": "generic",
            "customHTTPProfile": {
              "class": "HTTP_Profile",
              "xForwardedFor": true
              },
            "<NAMESPACE_NAME>_<NAMESPACE_NAME>_vs": {
              "class": "Service_HTTP",
              "iRules": ["IRules_XFF"],
              "persistenceMethods": [ "cookie" ],
              "virtualAddresses": [
                "<APP_VIP>"
              ],
              "snat": "auto",
              "virtualPort": <SERVICE_PORT>,
              "profileHTTP": {
                "use": "customHTTPProfile"
              },
              "pool": "<NAMESPACE_NAME>_<NAMESPACE_NAME>_pool"
            },
            "<NAMESPACE_NAME>_<NAMESPACE_NAME>_pool": {
              "class": "Pool",
              "monitors": [
                "http"
              ],
              "loadBalancingMode": "least-connections-member",
              "members": [
              {
                "servicePort": <SERVICE_PORT>,
                "serverAddresses": []
              }
              ]
            }
          }
        }
      }
    }
```

举例：
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: one-tenant-multiple-apps
  namespace: f5-test002
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
      "class": "AS3",
      "action": "deploy",
      "persist": true, 
      "declaration": {
        "class": "ADC",
        "schemaVersion": "3.11.0",
        "id": "one-tenant-multiple-apps",
        "label": "one-tenant-multiple-apps",
        "remark": "HTTP application",
        "f5_test002": {
          "class": "Tenant",
          "f5_test002_1": {
            "class": "Application",
            "template": "generic",
            "app_1_svc_vs": {
              "class": "Service_HTTP",
              "iRules": ["IRules_XFF"],
              "persistenceMethods": [ "cookie" ],
              "virtualAddresses": [
                "192.168.2.217"
              ],
              "snat": "auto",
              "virtualPort": 80,
              "pool": "app_1_svc_pool"
            },
            "app_1_svc_pool": {
              "class": "Pool",
              "monitors": [
                "http"
              ],
              "loadBalancingMode": "least-connections-member",
              "members": [
              {
                "servicePort": 80,
                "serverAddresses": []
              }
              ]
            },
             "IRules_XFF": {
              "class": "iRule",
              "iRule": "when HTTP_REQUEST {\n  HTTP::header insert X-Forwarded-For [IP::remote_addr] \n }"
            }
          },
           "f5_test002_2": {
            "class": "Application",
            "template": "generic",
            "app_2_svc_vs": {
              "class": "Service_HTTP",
               "iRules": ["IRules_XFF1"],
              "persistenceMethods": [ "cookie" ],
              "virtualAddresses": [
                "192.168.2.216"
              ],
              "snat": "auto",
              "virtualPort": 80,
              "pool": "app_2_svc_pool"
            },
            "app_2_svc_pool": {
              "class": "Pool",
              "monitors": [
                "http"
              ],
              "loadBalancingMode": "least-connections-member",
              "members": [
              {
                "servicePort": 80,
                "serverAddresses": []
              }
              ]
            },
             "IRules_XFF1": {
              "class": "iRule",
              "iRule": "when HTTP_REQUEST {\n  HTTP::header insert X-Forwarded-For [IP::remote_addr] \n }"
            }
          }
        }
      }
    }
```
