# 如何寫一個 K8s Operator


使用過 k8s 的同學可能執行過以下命令：
```bash
kubectl edit sts myapp # 編輯一個名稱爲 myapp 的 StatefulSet
kubectl describe sts myapp # 查看一個名稱爲 myapp 的 StatefulSet
```
StatefulSet 是 k8s 定義的一種資源，類似的還有 Deployment、Job、ConfigMap 等。當你執行 edit 命令編輯這些資源後，k8s 會通過不停輪詢的方式（核心概念：`control loop`），將目標資源調整（核心概念：`reconcile`）到你期望的狀態。

例如，你按了空調的遙控器，希望將房間的溫度下調到 20℃。空調的壓縮機開始工作，並且同時不停的檢測當前實際的溫度與你期望的溫度之間的差異，直到溫度達到20℃，這就是一個 control loop 的例子。

&gt; 很簡單，對吧？

設想一下如果不是這樣，你將一手拿着溫度計，然後不停的告訴空調溫度仍然很高，或者已經變得過低了。

這就是聲明式 API 的好處，用戶只需要告訴程序你的期望，剩下的交給程序來做（對於程序開發者來說是雷鋒行爲），而程序實現目標最省力的方式，就是採用 control loop 的方式，不停的對比期望與現實的差距。

## Operator 是什麼
試想我們不再滿足於 k8s 提供的默認的資源，我們想利用這種省心省力的方式，來管理我們自己的資源，如：數據庫的一個用戶。

你可能想說，數據庫的用戶存在於數據庫內，我知道數據庫的集羣可以定義爲 StatefulSet 然後由 k8s 管理，用戶又怎麼使用 k8s 管理呢？爲什麼要用 k8s 來管理呢？

&gt; 爲什麼要用 k8s 管理用戶資源？

以 MySql 爲例，通常我們創建用戶，是使用 root 用戶登錄到數據庫，執行 sql 語句創建用戶。但是設想以下幾種場景：
1. 你不知道 root 用戶的密碼，或者因爲安全要求，不能提供給你
2. 你不知道 MySql 的 IP
3. 你知道以上信息，但是因爲沒有開啓相應的節點權限，你無法登錄數據庫
4. 你完成了以上所有步驟，結果其中某些登錄或者創建步驟失敗了，你和數據庫運維人員開始扯皮

看到了吧？這些都是生產環境中，真實會遇到的事情。而使用以下步驟，我們就可以一舉解決這些問題。

&gt; 怎麼做到？

把大象關進冰箱需要三步，而我們要使用 Operator 完成在數據庫中創建用戶只需要兩步：
1. 告訴數據庫，我需要創建的用戶信息

   ```yaml
   apiVersion: handsomeguy.cn/v1alpha1
   kind: DatabaseUser                        
   metadata:
     name: cnhandsomeguy                                 
   spec:
     user: cnhandsomeguy                
     password: changeit                     
   ```
   這就是一個最簡單的自定義資源（核心概念：`custom resource`，簡稱 cr），包含用戶名、密碼，還有 k8s 資源的一些唯一性信息，如 apiVersion（假如你對自己的定義不滿意，新加了一些字段，就需要更改版本號，但是這種做法要堅決避免，後文會提到原因和對策），Kind（就像 StatefulSet 和 Deployment 也是一種 Kind 一樣，我們給自己起了一個名字叫 DatabaseUser）
   
   聰明的同學肯定能看到，這個資源還缺少了一些信息，如需要在哪個數據庫創建？密碼怎麼明文寫在這裏了呢？我們將在後面的章節完善這些部分。

2. 數據庫來創建用戶

   實際上此時並不是數據庫來執行創建用戶的動作，而是我們的 Operator。Operator 一直在待命（持續的監控 Kind 爲 DatabaseUser 的資源），在我們提交上面的請求後，它就可以連接到數據庫，執行創建用戶的動作，當然，這部分邏輯需要由我們自己來編寫。

&gt; 簡單吧！

Operator 是管理 k8s 自定義資源的一種擴展。它也遵循 `control loop` 的設計理念，通常我們需要在一個 Operator 中，編寫一個控制器（核心概念：`controller`），它是一段代碼（廢話），這個控制器接收到資源的創建、更新、刪除事件，由我們編碼來決定：
1. 如何實現這些創建、更新、刪除的邏輯
2. 檢測是否達到了期望，如果返回了錯誤，則認爲需要進入下一次循環，再來一遍！

## 看看一個案例
還有一個重要的概念沒有介紹：自定義資源的定義（custom resource definition，簡稱 crd）。有點繞口，但是試着這麼理解：
1. Operator 是一個進程，一直運行在 k8s 集羣內部，監控着某種 Kind 的資源的事件
2. 它到底在監控什麼呢？我們需要一個名字！（Kind）
3. 如果它監控到了 DatabaseUser，該如何去 spec 中找到用戶名、密碼這些信息呢？
我們需要一個定義，一個描述文件，來事先告訴 Operator 數據庫用戶的類型、細節，以便於 Operator 來監控、按照流程執行。

使用以下命令可以查看當前 k8s 中已經有哪些 crd：
```bash
kubectl get crd
```

所以現在，我們需要以下幾種東西：
1. 資源定義（crd）
2. Operator 程序的編碼和部署
3. 資源（cr）

&gt; 嚇到我了，我需要從零開始編碼，寫一個 Operator 嗎？可以用 Java 嗎？部署在哪？怎麼監控？怎麼對接 k8s？

Relax! 有框架，有示例，只要你的 &lt;kbd&gt;Ctrl&lt;/kbd&gt; &#43; &lt;kbd&gt;C/V&lt;/kbd&gt; 能用就行，可以開始了嗎？

### 使用 [Kubebuilder](https://book.kubebuilder.io/) 開發 Operator

&gt; 有點快了，Kubebuilder 是什麼？爲什麼選它？

還有個選擇是 operator-sdk，大同小異，都是生成代碼的工具罷了。當然你想手擼也不是不行。

我建議通篇閱讀一下 &lt;https://book.kubebuilder.io/&gt;，但是時間有限的同學，看本文熟悉下脈絡就行。本文有個作用是，幫助你避免一些坑，否則你生成的代碼很可能是在某些 k8s 版本上跑不起來的。

#### 準備
1. 準備一臺 linux 虛擬機，並且安裝好 gcc 和 make 命令。

   按照 &lt;https://book.kubebuilder.io/quick-start.html#installation&gt; 執行命令，下載 kubebuilder。
2. 初始化倉庫

   按照 &lt;https://book.kubebuilder.io/quick-start.html#create-a-project&gt; 執行命令，執行初始化
   
   ```bash
   kubebuilder init --domain handsomeguy.cn --repo handsomeguy.cn/databaseuser
   ```
3. 創建一個 API

   一個 Operator 可以管理多個資源，這些資源可以理解爲就是一個 API。
   
   ```bash
   kubebuilder create api --crd-version v1 --group mygroup --version v1alpha1 --kind DatabaseUser
   ```
   這裏注意一下， --crd-version 從 k8s 1.16 的版本後就不支持 v1beta1 了。
   這一步驟會生成一些 go 文件。

4. 編輯這些 go 文件。
   假設我們要增加用戶名和密碼：

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
   編寫好了，假設先寫這麼多。有兩點要說明一下：
   1. 上面我們說密碼是明文存儲的，不安全，這裏我們使用了一個 corev1.SecretKeySelector。假設你的密碼存儲在了某一個名爲 my-secret 的卷中的 my-key 字段，我們就可以在後面使用如下方式來取到它的明文。
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
   2. 用戶的定義還可以增加 status 這樣的 sub-resource，這樣在用戶創建失敗的時候，將失敗的信息刷回到 status 中，就可以使用 kubectl describe 命令來查看失敗信息，後文會給案例。

5. 生成 crd 文件
   ```bash
   make manifests #在項目的根目錄執行 
   ```

   &gt; 重大提醒

   根目錄中有MakeFile文件，有必要仔細閱讀一下。因爲 make manifests 步驟會調用一個 controller-gen 的工具，來生成 crd 文件。但是 controller-gen 和 k8s 是有嚴格的配套關係的。如果你想生成老版本的（k8s 1.12)的 v1beta1 版本的 crd 文件，就必須使用 0.6.2 版本之前的 controller-gen。可以查看其 release 頁面 &lt;https://github.com/kubernetes-sigs/controller-tools/releases&gt; 來確定使用什麼版本。
   controller-gen 是由 MakeFile 中指定並在 make 過程中自動下載的（你也可以下載好後放到指定位置），如果想要修改 controller-gen 的版本，可以在 MakeFile 的以下位置修改：
   ```bash
   controller-gen: ## Download controller-gen locally if necessary.
	 $(call go-get-tool,$(CONTROLLER_GEN),sigs.k8s.io/controller-tools/cmd/controller-gen@v0.6.2) #將0.6.2改爲你需要的版本
   ```
6. 同時生成多個版本的 crd 文件
   修改 MakeFile 的以下幾行，以同時生成支持新老版本 k8s 的 crd 文件。
   ```bash
   CRD_OPTIONS ?= &#34;crd:crdVersions={v1beta1,v1},trivialVersions=true,preserveUnknownFields=false&#34;
   manifests: controller-gen ## Generate WebhookConfiguration, ClusterRole and CustomResourceDefinition objects.
	$(CONTROLLER_GEN) $(CRD_OPTIONS) rbac:roleName=my-crd-manager webhook paths=./... output:crd:artifacts:config=config/crd/bases
   ```
   實際執行的命令其實是，
   ```bash
   bin/controller-gen rbac:roleName=my-crd-manager crd:crdVersions={v1,v1beta1} webhook paths=./... output:crd:artifacts:config=config/crd/ bases
   ```
   因此你也可以手動執行。
7. 在 k8s cluster 中創建這個 crd
   在進行這一步之前，你可以微調你的 crd，比如調整它的縮寫爲 dbu，這樣就可以執行 `kubectl get dbu` 來查看你的資源。
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
   使用以下命令創建並查看 crd。
   ```bash
   kubectl apply -f my-crd.yaml
   kubectl get crd
   ```
到這裏，準備工作就做完了（不出意外，你會在 make manifests 的時候遇到很多報錯，請耐心查看報錯，一一思考解決，都是有跡可循的。）。
我們現在有了
1. crd
2. 代碼
此時可以觀察一下生成的代碼，如 DatabaseUserController 的 Reconcile 方法，這裏將是你編碼的主要陣地。
還差億點點小細節，就可以編寫 cr 文件並部署測試了。

#### 對接 k8s

首先我們應該對接 k8s，不然怎麼知道我們的 operator 是否能正常監聽到資源呢？
查看 sigs.k8s.io/controller-runtime/pkg/client/config/config.go 的源碼，應該是有很多中配置的方式，我們選擇最簡單的一種：指定 KUBECONFIG 變量。

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
這個配置不是開箱即用的，多想一想怎麼獲取到這些證書吧，我寫本文的時候手頭沒有 k8s 集羣，暫不能提供方法了。

配置好後，在你的 go 程序運行的時候指定或在開發過程中在 goland 配置都可以，具體方式不再贅述。

#### 開始編碼

1. 監控指定 namespace 的資源

   operator 可以監控一個或多個 k8s namespace 下的資源。在 main.go 中找到如下位置，修改即可：
   ```go
   mgrOptions := ctrl.Options{
      Scheme:                 scheme,
      MetricsBindAddress:     setup.MetricsAddr,
      HealthProbeBindAddress: setup.ProbeAddr,
      NewCache:               cache.MultiNamespacedCacheBuilder(your_name_spaces), // 在此處指定需要監控的 namespace，實際生產過程中這些都是要做成可配置的，或通過啓動參數指定
   }
   mgr, err := ctrl.NewManager(cfg, mgrOptions)
   ```
2. 監控要創建到某個數據庫的資源

   可以利用 k8s 資源的 label。如：
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
   這樣我們就可以在 DatabaseUserController 中過濾出要創建到指定數據庫的資源，其它數據庫的資源都不管。
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
3. Reconcile 方法的編寫
   ```go
   func (r *DatabaseUserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
   	   // get user from cluster
   	   var DatabaseUser v1.DatabaseUser
   	   if err := r.Get(ctx, req.NamespacedName, &amp;DatabaseUser); err != nil {
   	   	   logger.Errorf(&#34;unable to fetch DatabaseUser: %v.&#34;, err)
   	   	   return ctrl.Result{}, client.IgnoreNotFound(err)
   	   }
      
   	   // 處理用戶的刪除事件
   	   if !DatabaseUser.ObjectMeta.DeletionTimestamp.IsZero() {
   	   	   return ctrl.Result{}, r.delete(ctx, &amp;DatabaseUser)
   	   }
      
   	   logger.Infof(&#34;%s before update %s.&#34;, DatabaseUser.Name, DatabaseUser.ResourceVersion)
   	   oldStatus := DatabaseUser.Status.DeepCopy()
   	   // 用戶創建、更新等邏輯編寫。可以調用 job 或者直接在 go 中連接本地數據庫進行用戶操作。
   	   reconcileError := r.do(ctx, &amp;DatabaseUser)
       // 將實際狀態刷新到資源中
   	   updateStatusError := r.updateStatus(ctx, &amp;DatabaseUser, oldStatus)
       // 在以下方法中判斷成功還是失敗，並決策是否進行下一輪循環
   	   return againOrDone(&amp;DatabaseUser, reconcileError, updateStatusError)
   }
   ```
   Reconcile 返回兩個結果： 
   ```go
   return reconcile.Result{
			Requeue:      true, // 重新開始下次循環
			RequeueAfter: requeueAfter,
		}, err // err 不爲 nil 的時候也重新開始下次循環
   ```
4. 如何從 secret 中獲取密碼

   示例代碼，可以參考如下，實際實現還需要考慮很多健壯性和擴展性。
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
5. finalizer 實現同步刪除資源

   當你執行 `kubectl delete dbu my-user` 的時候，我們希望 k8s 等待 operatoror 執行完成並返回刪除用戶成功後，才真的刪除這個 cr 文件。這樣就需要用到 finalizer的能力。見官方文檔：&lt;https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/finalizers/&gt;。
   
   代碼中，可以這麼實現：
   ```go
   	// finalizer is pre-delete hook
       myFinalizer = &#34;mygroup.handsomeguy.cn/databaseuser&#34;
       // 在新增用戶的時候，打上 finalizer 標記，使用 kubectl get dbu 可以看到 finalizer 的信息
   	if !controllerutil.ContainsFinalizer(user, myFinalizer) {
   		controllerutil.AddFinalizer(user, myFinalizer)
   		if err = r.Update(ctx, user); err != nil {
   			return
   		}
   	}
       // 在刪除用戶的時候，如果刪除成功就去除這個 finalizer
       if controllerutil.ContainsFinalizer(user, myFinalizer) &amp;&amp; !r.Opts.SkipUpdateStatus {
           // 先刪除用戶
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

#### 利用 status 提升可維護性

區分一個程序員水平的一個方法，是看他的代碼非功能性指標如何，如可維護性高不高，出現問題定位問題快不快。我們這個 operator 說實話還是蠻複雜的，出現問題沒有經驗的人還真的不好定位。我們需要一種便捷的手段，一條命令就可以定位大部分問題。對於用戶來說，熟悉的可能只有 `kubectl get dbu` 的命令，我們可不可以將創建用戶過程中的報錯放到這個結果裏面呢？答案是可以。

文檔在這裏。&lt;https://book-v1.book.kubebuilder.io/basics/status_subresource.html&gt;，但是不如直接看代碼：

1. 在 xxxxtypes.go 中增加 status
   當然你也可以按照官方的文檔去增加生成代碼的註解。
   
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
   加的代碼有點多，類爆炸了，但是是值得的。
2. 在成功或失敗的地方（DatabaseUserController中），調用 UpdateStatusCondition
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
   最後在 Reconcile 方法中刷回狀態即可。
   ```go
   func (r *DatabaseUserReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
       // ... 省略
       reconcileError := r.do(ctx, &amp;DatabaseUser)
       updateStatusError := r.updateStatus(ctx, &amp;DatabaseUser, oldStatus) // 刷回狀態到集羣
       return againOrDone(&amp;DatabaseUser, reconcileError, updateStatusError)
       // ... 省略
   ```
   如此，只有你在 controller 的全階段，將 err 信息寫入到 cr 中，用戶就可以使用 `kubectl describe dbu` 來看到這些報錯信息，而不用麻煩你了。

好了，以上就是一些實現一個 Operator 的步驟了。這些內容大部分靠回憶，代碼部分都是網上找的僞代碼，還有很多實戰中的坑需要注意，但是限於時間太久，已經想不起來了，以後想起來再補充吧。大家有什麼想法可以在評論區交流哦！


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2022/05/operator-dev/  

