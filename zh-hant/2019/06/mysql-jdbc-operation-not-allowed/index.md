# Mysql Jdbc 報錯 Operation Not Allowed After Statement Closed


背景
====

業務上來說，連你們的mysql連不上，連別人的（其它部門的mysql）都能連上。查看其日誌，報了一行錯誤&#34;No
operations allowed after statement closed&#34;。
這句話的意思，說的很清楚，不應該用關閉後的statement執行查詢。但是因爲我們是服務化的mysql，公司在遇到問題甩鍋給其它部門的習慣由來已久，所以還是要幫業務解決。

大致代碼
========

    class Main {
        private static connection;
        private static statement;

        static {
            connection = getConnection();
            statement = connection.createStatement();
        }
        public static query(){
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }
    }

定位
====

首先，業務反饋的是“連接不上”，但是報錯位置其實是在executeUpdate一行，在此鄙視這些人，日誌都不看清楚就開始推脫責任。在其代碼中加入debug日誌

        public static query(){
            log(statement.isClosed());
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }

發現日誌打了兩次，第一次爲false，第二次爲true並報錯。
查看該類引用位置，發現爲一個定義了init-method的bean，類似於

    &lt;bean id=&#34;xxxx&#34; class=&#34;xxxx&#34; init-method=&#34;query&#34; /&gt;

最終建議：

1.  將bean的scope更改爲singleton

2.  重構Main類類似如下

&lt;!-- --&gt;

    class Main {
        private static connection;
        private static statement;

        private static void getConnection  {
            DriverManager.xxxx
            statement = connection.createStatement();
        }
        public static query(){
            getConnection();
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }
    }

其實有更好的寫法，但是每個人的時間都很寶貴，沒有義務和必要爲你解決你自己的問題。而且從結果來看，其所謂的別人的數據庫都能連上其實是不可能的。在幫其定位過程中，看git發現這些有問題的代碼都是新增代碼，所以，做數據庫真的累。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/06/mysql-jdbc-operation-not-allowed/  

