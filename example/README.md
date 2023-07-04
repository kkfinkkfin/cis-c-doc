# 样例目的：配置两个7层VS 和 两个POOL

需求：
1.	HTTP健康检查，根据请求路径及返回值预期结果的判断容器服务是否健康
2.	Cookie会话保持，并且Cookie需要启用加密
3.	Pool 所有Member健康检查失败后，对VS地址Telnet和ICMP请求都不通（手动干掉poolmember）
4.	关联iRule，插入XFF属 
5.	最小连接数的负载均衡算法
6.	VS对应所有Member Down 后Telnent 和 Ping验证  （3后进行验证）
7.	启用 SANT

# 操作步骤

1. 创建命名空间
kubectl create ns f5-test002

2. 创建两个应用Deployment
kubectl create deployment app-1 --image='nginxdemos/nginx-hello:plain-text' -n f5-test002
kubectl create deployment app-2 --image='nginxdemos/nginx-hello:plain-text' -n f5-test002
kubectl scale deployments app-1 --replicas=2 -n f5-test002
kubectl scale deployments app-2 --replicas=2 -n f5-test002

3. 创建应用服务Service
kubectl expose deployment app-1 --port=80 --target-port=8080 --type=ClusterIP --name=app-1-svc  -n f5-test002
kubectl label svc app-1-svc cis.f5.com/as3-tenant=f5_test002 -n f5-test002
kubectl label svc app-1-svc cis.f5.com/as3-app=f5_test002_1 -n f5-test002
kubectl label svc app-1-svc cis.f5.com/as3-pool=app_1_svc_pool -n f5-test002

kubectl expose deployment app-2 --port=80 --target-port=8080 --type=ClusterIP --name=app-2-svc  -n f5-test002
kubectl label svc app-2-svc cis.f5.com/as3-tenant=f5_test002 -n f5-test002
kubectl label svc app-2-svc cis.f5.com/as3-app=f5_test002_2 -n f5-test002
kubectl label svc app-2-svc cis.f5.com/as3-pool=app_2_svc_pool -n f5-test002

4. 创建负载均衡配置Configmap
kubectl create -f app-lb-configmap.yaml

