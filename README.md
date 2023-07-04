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


2. 下发负载均衡配置 Configmap
注意： 根据实际应用信息替换变量值
- `<SERVICE_NAME>`:  应用服务Service名称
- `<SERVICE_PORT>`:  应用服务Service端口
- `<NAMESPACE_NAME>`: 应用服务所在的命名空间Namespace名称
- `<APP_VIP>`: 分配给应用的入口VIP

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
