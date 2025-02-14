---
title: k8s学习
abbrlink: 9d94e816
date: 2024-12-28 20:42:57
updated: 2024-12-28 20:42:57
tags:
  - kubernates
  - docker
categories: 云原生
---

## 知识点总结

<!-- more -->


---

### **一、基础概念**
1. **什么是 Kubernetes？它的核心功能是什么？**  
   - **Kubernetes**：容器编排平台，用于自动化部署、扩展和管理容器化应用。  
   - **核心功能**：服务发现、负载均衡、存储编排、自动扩缩容、自愈（重启容器、替换节点）等。

2. **Pod、Deployment、Service 的区别？**  
   - **Pod**：最小调度单元，包含一个或多个共享网络/存储的容器。  
   - **Deployment**：管理 Pod 的副本数量和滚动更新。  
   - **Service**：定义一组 Pod 的访问策略（如负载均衡、DNS 名称）。

3. **StatefulSet 和 Deployment 的区别？**  
   - **StatefulSet**：用于有状态应用（如数据库），保证 Pod 的唯一性和持久化存储的顺序性。  
   - **Deployment**：用于无状态应用，支持滚动更新和副本管理。

---

### **二、核心组件**
4. **Kubernetes Master 节点包含哪些组件？**  
   - **API Server**：集群操作的入口。  
   - **Scheduler**：将 Pod 调度到合适的 Node。  
   - **Controller Manager**：运行控制器（如 Deployment、Node 控制器）。  
   - **etcd**：分布式键值存储，保存集群状态。

5. **kubelet 和 kube-proxy 的作用？**  
   - **kubelet**：运行在 Node 上的代理，负责管理 Pod 生命周期（创建/删除容器）。  
   - **kube-proxy**：维护 Node 上的网络规则（如 Service 的 IP 和端口转发）。

6. **etcd 在 Kubernetes 中的作用？**  
   - 存储集群所有配置数据和状态（如 Pod、Service、Node 信息）。

---

### **三、网络与存储**
7. **Kubernetes 的网络模型？**  
   - 每个 Pod 拥有唯一 IP，Pod 间可直接通信（跨 Node 通过 CNI 插件如 Calico、Flannel 实现）。

8. **Service 的 ClusterIP、NodePort、LoadBalancer 区别？**  
   - **ClusterIP**：集群内部访问的虚拟 IP。  
   - **NodePort**：通过 Node 的静态端口暴露 Service（范围 30000-32767）。  
   - **LoadBalancer**：通过云提供商的负载均衡器暴露 Service。

9. **如何实现持久化存储？**  
   - 使用 **PersistentVolume（PV）** 和 **PersistentVolumeClaim（PVC）**，支持静态/动态存储分配（如 NFS、AWS EBS）。

10. **ConfigMap 和 Secret 的作用及区别？**  
    - **ConfigMap**：存储非敏感配置（如环境变量、配置文件）。  
    - **Secret**：存储敏感数据（如密码、令牌），以 Base64 编码存储。

---

### **四、运维与实践**
11. **如何滚动更新一个 Deployment？**  
    ```bash
    kubectl set image deployment/my-app my-container=my-image:v2
    # 或通过修改 YAML 中的镜像版本后 apply
    ```
    - Kubernetes 会逐步替换旧 Pod，确保服务不中断。

12. **如何查看 Pod 的日志？**  
    ```bash
    kubectl logs <pod-name> -c <container-name>  # 查看指定容器日志
    kubectl logs -f <pod-name>                   # 实时跟踪日志
    ```

13. **Pod 一直处于 Pending 状态的可能原因？**  
    - 资源不足（CPU/内存）。  
    - 未匹配 NodeSelector/Affinity。  
    - 持久化存储声明（PVC）未绑定。

---

### **五、监控与调试**
14. **如何监控 Kubernetes 集群？**  
    - **Prometheus + Grafana**：采集集群指标（如 Node 资源使用、Pod 状态）。  
    - **kubectl top**：查看资源使用情况（需安装 Metrics Server）。

15. **如何排查 Pod 无法启动的问题？**  
    1. `kubectl describe pod <pod-name>`：查看事件详情（如镜像拉取失败、资源限制）。  
    2. `kubectl get events --sort-by=.metadata.creationTimestamp`：按时间排序事件。

---

### **六、进阶问题**
16. **什么是 Operator？**  
    - 通过自定义资源（CRD）和控制器扩展 Kubernetes API，用于管理有状态应用（如数据库、中间件）。

17. **Helm 的作用是什么？**  
    - Kubernetes 的包管理工具，通过 Charts 定义、安装和升级复杂应用（如 MySQL、Redis）。

18. **如何实现蓝绿发布或金丝雀发布？**  
    - **蓝绿发布**：通过两个相同规模的 Deployment 切换流量。  
    - **金丝雀发布**：使用 Service 的流量权重（如 Istio）或 Deployment 的副本比例逐步替换。

19. **Kubernetes 的安全机制有哪些？**  
    - **RBAC**：基于角色的访问控制。  
    - **NetworkPolicy**：限制 Pod 的网络流量。  
    - **Pod Security Policies（PSP）**：定义 Pod 的安全约束（如禁止特权容器）。

---

### **七、常见命令速查**
20. **常用 kubectl 命令：**  
    ```bash
    kubectl get pods -n <namespace>          # 查看命名空间下的 Pod
    kubectl exec -it <pod-name> -- /bin/sh   # 进入 Pod 容器
    kubectl port-forward <pod-name> 8080:80  # 端口转发到本地
    kubectl rollout undo deployment/my-app   # 回滚 Deployment
    ```

---

### **八、扩展知识**
21. **什么是 Service Mesh（如 Istio）？**  
    - 在应用层实现服务间通信的治理（如流量管理、熔断、链路追踪），与 Kubernetes 集成增强微服务能力。

22. **如何优化 Kubernetes 资源利用率？**  
    - 设置 Pod 的 `requests` 和 `limits`。  
    - 使用 HPA（Horizontal Pod Autoscaler）基于 CPU/内存自动扩缩容。

23. **Kubernetes 集群升级流程？**  
    - 先升级 Master 组件（API Server、Controller Manager、Scheduler），再逐步升级 Node 的 kubelet 和 kube-proxy。

---
