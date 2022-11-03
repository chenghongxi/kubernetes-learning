# Operator Lifecycle Manager(OLM)

官网：https://docs.openshift.com/container-platform/3.11/install_config/installing-operator-framework.html

## OLM
```text
OLM(Operator Lifecycle Manager) 作为 Operator Framework 的一部分，可以帮助用户进行 Operator 的自动安装，
升级及其生命周期的管理。同时 OLM 自身也是以 Operator 的形式进行安装部署，可以说它的工作方式是以 Operators 来管理 Operators，
而它面向 Operator 提供了声明式 (declarative) 的自动化管理能力也完全符合 Kubernetes 交互的设计理念。
```

### 组件:
- `1. 生命周期管理：管理 operator 自身以及监控资源模型的升级和生命周期;`
- `2. 服务发现：发现在集群中存在哪些 operator，这些 operators 管理了哪些资源模型以及又有哪些 operators 是可以被安装在集群中的;`
- `3. 打包能力：提供一种标准模式用于 operator 以及依赖组件的分发，安装和升级`
- `4. 交互能力：在完成了上述能力的标准化后，还需要提供一种规范化的方式（如 CLI）与集群中用户定义的其他云服务进行交互。`

