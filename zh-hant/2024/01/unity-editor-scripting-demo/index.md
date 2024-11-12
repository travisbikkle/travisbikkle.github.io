# Unity Editor Scripting 樣例


Unity 的官方文檔非常詳細，但是不是所有人學習Unity都是從頭看到尾看文檔過來的。
我最近剛接觸Unity,最近想給編輯器擴展一些功能，比如點擊Inspector面板，在Project窗口中高亮選擇Prefab或者Script。
這麼小小的一個需求，查遍了中文互聯網，不得其果。大家要麼在CSDN上貼一些零碎的、沒有格式化的代碼，要麼互相複製粘貼。結果不言而喻。
看外網，似乎大家提到了一些代碼，但是Unity論壇上的人都是高手，作爲一個新手看到了代碼都不知道放在哪，怎麼用，非常苦惱。
今天我把自己折騰了幾個小時的一點小代碼貼上，作爲記錄。如果大家對於編輯器擴展有大量需求，建議還是系統地學習一下官方的教程，不要像我，只求快，結果花費的時間更多。

1. 擴展代碼需要放到Editor文件夾下
   這個Editor文件夾你可以建到任意的地方，我放到了Assets/Scripts/Editor下面，在其中創建一個C#文件，命名隨意。
   ![20240104_unity_editor_location.png](/images/other/20240104_unity_editor_location.png)
2. 將以下代碼粘貼進文件，保存，即可在Inspect面板點擊掛載腳本時高亮腳本。

   ```c#
   using UnityEditor;
   using UnityEditor.UIElements;
   using UnityEngine;
   using UnityEngine.UIElements;
   
   
   [CustomEditor(typeof(MonoBehaviour), true)]
   public class HighlightItem : Editor {
       public override VisualElement CreateInspectorGUI() {
           var root = new VisualElement();
           InspectorElement.FillDefaultInspector(root, serializedObject, this);
           root.Add(new Label(&#34;點擊以高亮&#34;));
           root.RegisterCallback&lt;ClickEvent&gt;(Highlight);
       
           return root;
       }
       
       private void Highlight(ClickEvent evt) {
           var script = MonoScript.FromMonoBehaviour((MonoBehaviour)target);
       
           UnityEditor.EditorGUIUtility.PingObject(script);
           //UnityEditor.Selection.activeObject = script;
       }
   }
   ```
當然這個功能Unity自帶了，這個例子只是用來給像我這樣完全沒有接觸過Unity的同學演示製作一個簡單的擴展腳本的步驟。
Youtube上有很多更詳細直觀的教程，建議有條件有需要的同學看一遍。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/01/unity-editor-scripting-demo/  

