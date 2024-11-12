# 如何写一个 K8s Operator


使用过 k8s 的同学可能执行过以下命令：
```bash
kubectl edit sts myapp # 编辑一个名称为 myapp 的 StatefulSet
kubectl describe sts myapp # 查看一个名称为 myapp 的 StatefulSet
```
StatefulSet 是 k8s 定义的一种资源，类似的还有 Deployment、Job、ConfigMap 等。当你执行 edit 命令编辑这些资源后，k8s 会通过不停轮询的方式（核心概念：`control loop`），将目标资源调整（核心概念：`reconcile`）到你期望的状态。

例如，你按了空调的遥控器，希望将房间的温度下调到 20℃。空调的压缩机开始工作，并且同时不停的检测当前实际的温度与你期望的温度之间的差异，直到温度达到20℃，这就是一个 control loop 的例子。

&gt; 很简单，对吧？

设想一下如果不是这样，你将一手拿着温度计，然后不停的告诉空调温度仍然很高，或者已经变得过低了。

这就是声明式 API 的好处，用户只需要告诉程序你的期望，剩下的交给程序来做（对于程序开发者来说是雷锋行为），而程序实现目标最省力的方式，就是采用 control loop 的方式，不停的对比期望与现实的差距。

## Operator 是什么
试想我们不再满足于 k8s 提供的默认的资源，我们想利用这种省心省力的方式，来管理我们自己的资源，如：数据库的一个用户。

你可能想说，数据库的用户存在于数据库内，我知道数据库的集群可以定义为 StatefulSet 然后由 k8s 管理，用户又怎么使用 k8s 管理呢？为什么要用 k8s 来管理呢？

&gt; 为什么要用 k8s 管理用户资源？

以 MySql 为例，通常我们创建用户，是使用 root 用户登录到数据库，执行 sql 语句创建用户。但是设想以下几种场景：
1. 你不知道 root 用户的密码，或者因为安全要求，不能提供给你
2. 你不知道 MySql 的 IP
3. 你知道以上信息，但是因为没有开启相应的节点权限，你无法登录数据库
4. 你完成了以上所有步骤，结果其中某些登录或者创建步骤失败了，你和数据库运维人员开始扯皮

看到了吧？这些都是生产环境中，真实会遇到的事情。而使用以下步骤，我们就可以一举解决这些问题。

&gt; 怎么做到？

把大象关进冰箱需要三步，而我们要使用 Operator 完成在数据库中创建用户只需要两步：
1. 告诉数据库，我需要创建的用户信息

   ```yaml
   apiVersion: handsomeguy.cn/v1alpha1
   kind: DatabaseUser                        
   metadata:
     name: cnhandsomeguy                                 
   spec:
     user: cnhandsomeguy                
     password: changeit                     
   ```
   这就是一个最简单的自定义资源（核心概念：`custom resource`，简称 cr），包含用户名、密码，还有 k8s 资源的一些唯一性信息，如 apiVersion（假如你对自己的定义不满意，新加了一些字段，就需要更改版本号，但是这种做法要坚决避免，后文会提到原因和对策），Kind（就像 StatefulSet 和 Deployment 也是一种 Kind 一样，我们给自己起了一个名字叫 DatabaseUser）
   
   聪明的同学肯定能看到，这个资源还缺少了一些信息，如需要在哪个数据库创建？密码怎么明文写在这里了呢？我们将在后面的章节完善这些部分。

2. 数据库来创建用户

   实际上此时并不是数据库来执行创建用户的动作，而是我们的 Operator。Operator 一直在待命（持续的监控 Kind 为 DatabaseUser 的资源），在我们提交上面的请求后，它就可以连接到数据库，执行创建用户的动作，当然，这部分逻辑需要由我们自己来编写。

&gt; 简单吧！

Operator 是管理 k8s 自定义资源的一种扩展。它也遵循 `control loop` 的设计理念，通常我们需要在一个 Operator 中，编写一个控制器（核心概念：`controller`），它是一段代码（废话），这个控制器接收到资源的创建、更新、删除事件，由我们编码来决定：
1. 如何实现这些创建、更新、删除的逻辑
2. 检测是否达到了期望，如果返回了错误，则认为需要进入下一次循环，再来一遍！

## 看看一个案例
还有一个重要的概念没有介绍：自定义资源的定义（custom resource definition，简称 crd）。有点绕口，但是试着这么理解：
1. Operator 是一个进程，一直运行在 k8s 集群内部，监控着某种 Kind 的资源的事件
2. 它到底在监控什么呢？我们需要一个名字！（Kind）
3. 如果它监控到了 DatabaseUser，该如何去 spec 中找到用户名、密码这些信息呢？
我们需要一个定义，一个描述文件，来事先告诉 Operator 数据库用户的类型、细节，以便于 Operator 来监控、按照流程执行。

使用以下命令可以查看当前 k8s 中已经有哪些 crd：
```bash
kubectl get crd
```

所以现在，我们需要以下几种东西：
1. 资源定义（crd）
2. Operator 程序的编码和部署
3. 资源（cr）

&gt; 吓到我了，我需要从零开始编码，写一个 Operator 吗？可以用 Java 吗？部署在哪？怎么监控？怎么对接 k8s？

Relax! 有框架，有示例，只要你的 &lt;kbd&gt;Ctrl&lt;/kbd&gt; &#43; &lt;kbd&gt;C/V&lt;/kbd&gt; 能用就行，可以开始了吗？

### 使用 [Kubebuilder](https://book.kubebuilder.io/) 开发 Operator

&gt; 有点快了，Kubebuilder 是什么？为什么选它？

还有个选择是 operator-sdk，大同小异，都是生成代码的工具罢了。当然你想手撸也不是不行。

我建议通篇阅读一下 &lt;https://book.kubebuilder.io/&gt;，但是时间有限的同学，看本文熟悉下脉络就行。本文有个作用是，帮助你避免一些坑，否则你生成的代码很可能是在某些 k8s 版本上跑不起来的。

#### 准备
1. 准备一台 linux 虚拟机，并且安装好 gcc 和 make 命令。

   按照 &lt;https://book.kubebuilder.io/quick-start.html#installation&gt; 执行命令，下载 kubebuilder。
2. 初始化仓库

   按照 &lt;https://book.kubebuilder.io/quick-start.html#create-a-project&gt; 执行命令，执行初始化
   
   ```bash
   kubebuilder init --domain handsomeguy.cn --repo handsomeguy.cn/databaseuser
   ```
3. 创建一个 API

   一个 Operator 可以管理多个资源，这些资源可以理解为就是一个 API。
   
   ```bash
   kubebuilder create api --crd-version v1 --group mygroup --version v1alpha1 --kind DatabaseUser
   ```
   这里注意一下， --crd-version 从 k8s 1.16 的版本后就不支持 v1beta1 了。
   这一步骤会生成一些 go 文件。

4. 编辑这些 go 文件。
   假设我们要增加用户名和密码：

   ```go
   type DatabaseUser struct {
	  metav1.TypeMeta   `json:&#34;,inline&#34;`
	  metav1.ObjectMeta `json:&#34;metadata&#34;`
	  Spec   DatabaseUserSpec   `json:&#34;spec,omitempty&#34;`
   }
   type DatabaseUserSpec struct {
   	  User string `json:&#34;user,omitempty&#34;`
   	  Password corev1.SecretKeySelector `json:&#34;password,omitempty&#34;`
   }
   ```
   编写好了，假设先写这么多。有两点要说明一下：
   1. 上面我们说密码是明文存储的，不安全，这里我们使用了一个 corev1.SecretKeySelector。假设你的密码存储在了某一个名为 my-secret 的卷中的 my-key 字段，我们就可以在后面使用如下方式来取到它的明文。
   ```yaml
   apiVersion: handsomeguy.cn/v1alpha1
   kind: DatabaseUser                        
   metadata:
     name: cnhandsomeguy                                 
   spec:
     user: cnhandsomeguy                
     password: 
       name: my-secret
       key: my-key                     
   ```
   2. 用户的定义还可以增加 status 这样的 sub-resource，这样在用户创建失败的时候，将失败的信息刷回到 status 中，就可以使用 kubectl describe 命令来查看失败信息，后文会给案例。

5. 生成 crd 文件
   ```bash
   make manifests #在项目的根目录执行 
   ```

   &gt; 重大提醒

   根目录中有MakeFile文件，有必要仔细阅读一下。因为 make manifests 步骤会调用一个 controller-gen 的工具，来生成 crd 文件。但是 controller-gen 和 k8s 是有严格的配套关系的。如果你想生成老版本的（k8s 1.12)的 v1beta1 版本的 crd 文件，就必须使用 0.6.2 版本之前的 controller-gen。可以查看其 release 页面 &lt;https://github.com/kubernetes-sigs/controller-tools/releases&gt; 来确定使用什么版本。
   controller-gen 是由 MakeFile 中指定并在 make 过程中自动下载的（你也可以下载好后放到指定位置），如果想要修改 controller-gen 的版本，可以在 MakeFile 的以下位置修改：
   ```bash
   controller-gen: ## Download controller-gen locally if necessary.
	 $(call go-get-tool,$(CONTROLLER_GEN),sigs.k8s.io/controller-tools/cmd/controller-gen@v0.6.2) #将0.6.2改为你需要的版本
   ```
6. 同时生成多个版本的 crd 文件
   修改 MakeFile 的以下几行，以同时生成支持新老版本 k8s 的 crd 文件。
   ```bash
   CRD_OPTIONS ?= &#34;crd:crdVersions={v1beta1,v1},trivialVersions=true,preserveUnknownFields=false&#34;
   manifests: controller-gen ## Generate WebhookConfiguration, ClusterRole and CustomResourceDefinition objects.
	$(CONTROLLER_GEN) $(CRD_OPTIONS) rbac:roleName=my-crd-manager webhook paths=./... output:crd:artifacts:config=config/crd/bases
   ```
   实际执行的命令其实是，
   ```bash
   bin/controller-gen rbac:roleName=my-crd-manager crd:crdVersions={v1,v1beta1} webhook paths=./... output:crd:artifacts:config=config/crd/ bases
   ```
   因此你也可以手动执行。
7. 在 k8s cluster 中创建这个 crd
   在进行这一步之前，你可以微调你的 crd，比如调整它的缩写为 dbu，这样就可以执行 `kubectl get dbu` 来查看你的资源。
   ```yaml
   ---
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     annotations:
       controller-gen.kubebuilder.io/version: v0.6.2
       &#34;helm.sh/resource-policy&#34;: keep
     creationTimestamp: null
     name: databaseuserss.handsomeguy.cn
   spec:
     group: mygroup.handsomeguy.cn
     names:
       kind: DatabaseUser
       listKind: DatabaseUserList
       plural: databaseusers
       singular: databaseuser
       shortNames: [dbu]
     preserveUnknownFields: false
     scope: Namespaced
     versions:
     // ... 省略
   ```
   使用以下命令创建并查看 crd。
   ```bash
   kubectl apply -f my-crd.yaml
   kubectl get crd
   ```
到这里，准备工作就做完了（不出意外，你会在 make manifests 的时候遇到很多报错，请耐心查看报错，一一思考解决，都是有迹可循的。）。
我们现在有了
1. crd
2. 代码
此时可以观察一下生成的代码，如 DatabaseUserController 的 Reconcile 方法，这里将是你编码的主要阵地。
还差亿点点小细节，就可以编写 cr 文件并部署测试了。

#### 对接 k8s

首先我们应该对接 k8s，不然怎么知道我们的 operator 是否能正常监听到资源呢？
查看 sigs.k8s.io/controller-runtime/pkg/client/config/config.go 的源码，应该是有很多中配置的方式，我们选择最简单的一种：指定 KUBECONFIG 变量。

```yaml
kind: Config
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: false
    certificate-authority: {{CA_DIR}}/ca.crt
    server: https://{{KUBERNETES_MASTER}}
  name: cluster
users:
- user:
    client-certificate: {{CA_DIR}}/kubecfg.crt
    client-key-data: {{CLIENT_KEY}}
  name: user
contexts:
- context:
    cluster: cluster
    user: user
  name: defaultContext
current-context: defaultContext
```
这个配置不是开箱即用的，多想一想怎么获取到这些证书吧，我写本文的时候手头没有 k8s 集群，暂不能提供方法了。

配置好后，在你的 go 程序运行的时候指定或在开发过程中在 goland 配置都可以，具体方式不再赘述。

#### 开始编码

1. 监控指定 namespace 的资源

   operator 可以监控一个或多个 k8s namespace 下的资源。在 main.go 中找到如下位置，修改即可：
   ```go
   mgrOptions := ctrl.Options{
      Scheme:                 scheme,
      MetricsBindAddress:     setup.MetricsAddr,
      HealthProbeBindAddress: setup.ProbeAddr,
      NewCache:               cache.MultiNamespacedCacheBuilder(your_name_spaces), // 在此处指定需要监控的 namespace，实际生产过程中这些都是要做成可配置的，或通过启动参数指定
   }
   mgr, err := ctrl.NewManager(cfg, mgrOptions)
   ```
2. 监控要创建到某个数据库的资源

   可以利用 k8s 资源的 label。如：
   ```yaml
   apiVersion: handsomeguy.cn/v1alpha1
   kind: DatabaseUser                        
   metadata:
     label:
       target: that-database
     name: cnhandsomeguy                                 
   spec:
     user: cnhandsomeguy                
     password: 
       name: my-secret
       key: my-key   
   ```
   这样我们就可以在 DatabaseUserController 中过滤出要创建到指定数据库的资源，其它数据库的资源都不管。
   ```go
   func GetLabelEventFilter(label string, value string) predicate.Predicate {
   	  return predicate.Funcs{
   	  	UpdateFunc: func(event event.UpdateEvent) bool {
   	  		return strings.ToLower(event.ObjectOld.GetLabels()[&#39;target&#39;]) == strings.ToLower(value)
   	  	},
   	  	DeleteFunc: func(deleteEvent event.DeleteEvent) bool {
   	  		return strings.ToLower(deleteEvent.Object.GetLabels()[&#39;target&#39;]) == strings.ToLower(value)
   	  	},
   	  	CreateFunc: func(createEvent event.CreateEvent) bool {
   	  		return strings.ToLower(createEvent.Object.GetLabels()[&#39;target&#39;]) == strings.ToLower(value)
   	  	},
   	  	GenericFunc: func(genericEvent event.GenericEvent) bool {
   	  		return strings.ToLower(genericEvent.Object.GetLabels()[&#39;target&#39;]) == strings.ToLower(value)
   	  	},
   	  }
   }
   // SetupWithManager 方法由 kubebuilder 生成
   func (r *DatabaseUserReconciler) SetupWithManager(mgr ctrl.Manager) error {
       if err := mgr.GetFieldIndexer().IndexField(context.Background(),
          // ... 省略
       }
   
       logger.Infof(&#34;setup with manager, %s, %s&#34;, constants.CrLabelKey, r.Opts.Target)
       return ctrl.NewControllerManagedBy(mgr).
       	  For(&amp;v1.DatabaseUser{}).
       	  WithEventFilter(GetLabelEventFilter(constants.CrLabelKey, r.Opts.Target)).
       	  Owns(&amp;v1.DatabaseUser{}).
       	  Complete(r)
           // ... 省略
   ```
3. Reconcile 方法的编写
   ```go
   func (r *DatabaseUserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
   	   // get user from cluster
   	   var DatabaseUser v1.DatabaseUser
   	   if err := r.Get(ctx, req.NamespacedName, &amp;DatabaseUser); err != nil {
   	   	   logger.Errorf(&#34;unable to fetch DatabaseUser: %v.&#34;, err)
   	   	   return ctrl.Result{}, client.IgnoreNotFound(err)
   	   }
      
   	   // 处理用户的删除事件
   	   if !DatabaseUser.ObjectMeta.DeletionTimestamp.IsZero() {
   	   	   return ctrl.Result{}, r.delete(ctx, &amp;DatabaseUser)
   	   }
      
   	   logger.Infof(&#34;%s before update %s.&#34;, DatabaseUser.Name, DatabaseUser.ResourceVersion)
   	   oldStatus := DatabaseUser.Status.DeepCopy()
   	   // 用户创建、更新等逻辑编写。可以调用 job 或者直接在 go 中连接本地数据库进行用户操作。
   	   reconcileError := r.do(ctx, &amp;DatabaseUser)
       // 将实际状态刷新到资源中
   	   updateStatusError := r.updateStatus(ctx, &amp;DatabaseUser, oldStatus)
       // 在以下方法中判断成功还是失败，并决策是否进行下一轮循环
   	   return againOrDone(&amp;DatabaseUser, reconcileError, updateStatusError)
   }
   ```
   Reconcile 返回两个结果： 
   ```go
   return reconcile.Result{
			Requeue:      true, // 重新开始下次循环
			RequeueAfter: requeueAfter,
		}, err // err 不为 nil 的时候也重新开始下次循环
   ```
4. 如何从 secret 中获取密码

   示例代码，可以参考如下，实际实现还需要考虑很多健壮性和扩展性。
   ```go
   func (r *DatabaseUserReconciler) GetPasswordInCluster(ctx context.Context, user *v1alpha1.DatabaseUser) (string, error) {
	   secret := &amp;corev1.Secret{}
	   secretKey := client.ObjectKey{Name: user.Spec.Password.Name, Namespace: user.Namespace}
   
	   if err := r.Get(ctx, secretKey, secret); err != nil {
	   	   return &#34;&#34;, err
	   }
       var path = &amp;user.Spec.Password.Key
       return string(sec.Data[path])
    }
   ```
5. finalizer 实现同步删除资源

   当你执行 `kubectl delete dbu my-user` 的时候，我们希望 k8s 等待 operatoror 执行完成并返回删除用户成功后，才真的删除这个 cr 文件。这样就需要用到 finalizer的能力。见官方文档：&lt;https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/finalizers/&gt;。
   
   代码中，可以这么实现：
   ```go
   	// finalizer is pre-delete hook
       myFinalizer = &#34;mygroup.handsomeguy.cn/databaseuser&#34;
       // 在新增用户的时候，打上 finalizer 标记，使用 kubectl get dbu 可以看到 finalizer 的信息
   	if !controllerutil.ContainsFinalizer(user, myFinalizer) {
   		controllerutil.AddFinalizer(user, myFinalizer)
   		if err = r.Update(ctx, user); err != nil {
   			return
   		}
   	}
       // 在删除用户的时候，如果删除成功就去除这个 finalizer
       if controllerutil.ContainsFinalizer(user, myFinalizer) &amp;&amp; !r.Opts.SkipUpdateStatus {
           // 先删除用户
   		if err := DeleteUser(user); err != nil {
   			return err
   		}
           // 再去除阻塞器 finalizer
   		controllerutil.RemoveFinalizer(user, myFinalizer)
   		// update to delete finalizer
   		if err := r.Update(ctx, user); err != nil {
   			return nil
   		}
   		return nil
   	}
   ```

#### 利用 status 提升可维护性

区分一个程序员水平的一个方法，是看他的代码非功能性指标如何，如可维护性高不高，出现问题定位问题快不快。我们这个 operator 说实话还是蛮复杂的，出现问题没有经验的人还真的不好定位。我们需要一种便捷的手段，一条命令就可以定位大部分问题。对于用户来说，熟悉的可能只有 `kubectl get dbu` 的命令，我们可不可以将创建用户过程中的报错放到这个结果里面呢？答案是可以。

文档在这里。&lt;https://book-v1.book.kubebuilder.io/basics/status_subresource.html&gt;，但是不如直接看代码：

1. 在 xxxxtypes.go 中增加 status
   当然你也可以按照官方的文档去增加生成代码的注解。
   
   ```go
   //&#43;kubebuilder:object:root=true
   //&#43;kubebuilder:subresource:status
   
   // DatabaseUser is the Schema for the DatabaseUsers API
   type DatabaseUser struct {
      metav1.TypeMeta   `json:&#34;,inline&#34;`
      metav1.ObjectMeta `json:&#34;metadata&#34;`
      
      Spec   DatabaseUserSpec   `json:&#34;spec,omitempty&#34;`
      Status DatabaseUserStatus `json:&#34;status,omitempty&#34;` // 新增部分
   }
   // DatabaseUserStatus defines the observed state of DatabaseUser
   type DatabaseUserStatus struct {
      Conditions     []DatabaseUserCondition `json:&#34;conditions,omitempty&#34;`
      // 其它想放的字段，省略
   }
   // DatabaseUserConditionType custom type
   type DatabaseUserConditionType string
   // DatabaseUserCondition v3 condition
   type DatabaseUserCondition struct {
   	  // Type of user condition.
   	  Type DatabaseUserConditionType `json:&#34;type,omitempty&#34;`
   	  // Status of the condition, one of True, False, Unknown.
   	  Status corev1.ConditionStatus `json:&#34;status,omitempty&#34;`
   	  // The last time this condition was updated.
   	  LastUpdateTime metav1.Time `json:&#34;lastUpdateTime,omitempty&#34;`
   	  // Last time the condition transitioned from one status to another.
   	  LastTransitionTime metav1.Time `json:&#34;lastTransitionTime,omitempty&#34;`
   	  // The reason for the condition&#39;s last transition.
   	  Reason string `json:&#34;reason,omitempty&#34;`
   	  // A human readable message indicating details about the transition.
   	  Message string `json:&#34;message,omitempty&#34;`
   }

   // UpdateStatusCondition update status condition
   func (u *DatabaseUser) UpdateStatusCondition(condType DatabaseUserConditionType,
   	   status corev1.ConditionStatus, reason, message string) (cond *DatabaseUserCondition, changed bool) {
   	   t := metav1.NewTime(time.Now())
   	   existedCondition, exists := u.ConditionExists(condType)
   	   if !exists {
   	       newCondition := DatabaseUserCondition{
   	      	   Type: condType, Status: status, Reason: reason, Message: message,
   	      	   LastTransitionTime: t, LastUpdateTime: t,
   	       }
   	   	   u.Status.Conditions = append(u.Status.Conditions, newCondition)
      
   	   	   return &amp;newCondition, true
   	   }
      
   	   if status != existedCondition.Status {
   	   	   existedCondition.LastTransitionTime = t
   	   	   changed = true
   	   }
      
   	   if message != existedCondition.Message || reason != existedCondition.Reason {
   	   	   existedCondition.LastUpdateTime = t
   	   	   changed = true
   	   }
      
   	   existedCondition.Status = status
   	   existedCondition.Message = message
   	   existedCondition.Reason = reason
      
   	   return existedCondition, changed
   }
   ```
   加的代码有点多，类爆炸了，但是是值得的。
2. 在成功或失败的地方（DatabaseUserController中），调用 UpdateStatusCondition
   ```go
   user.UpdateStatusCondition(
   	   &#34;Ready&#34;, corev1.ConditionTrue,
   	   &#34;Provision Succeeded&#34;, &#34;The user provisioning has succeeded.&#34;,
   )      
   if *err != nil {
   	   user.UpdateStatusCondition(
   	   	&#34;NotReady&#34;, corev1.ConditionFalse,
   	   	&#34;Provision Failed&#34;, fmt.Sprintf(&#34;The user provisioning has failed: %s&#34;, *err),
   	   )
   }
   ```
   最后在 Reconcile 方法中刷回状态即可。
   ```go
   func (r *DatabaseUserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
       // ... 省略
       reconcileError := r.do(ctx, &amp;DatabaseUser)
       updateStatusError := r.updateStatus(ctx, &amp;DatabaseUser, oldStatus) // 刷回状态到集群
       return againOrDone(&amp;DatabaseUser, reconcileError, updateStatusError)
       // ... 省略
   ```
   如此，只有你在 controller 的全阶段，将 err 信息写入到 cr 中，用户就可以使用 `kubectl describe dbu` 来看到这些报错信息，而不用麻烦你了。

好了，以上就是一些实现一个 Operator 的步骤了。这些内容大部分靠回忆，代码部分都是网上找的伪代码，还有很多实战中的坑需要注意，但是限于时间太久，已经想不起来了，以后想起来再补充吧。大家有什么想法可以在评论区交流哦！


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2022/05/operator-dev/  

