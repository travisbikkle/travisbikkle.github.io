# Webpack 5 Mock 10分钟快速配置


不说废话，直接开始。我使用 webpack 5.90.3，以下是我的配置。

1. 文件结构
   ```text
   project_root/
     |-- webpack.config.js
     |-- mockserver.js
     |-- mock/
       |-- user.js
   ```
2. webpack.config.js

   {{&lt;notice tip&gt;}}
   因为 express 现在不再附带 body-parser，现在需要执行 `npm i -D body-parser`。如果你希望读取 request 里面的内容，你会需要这个插件。
   {{&lt;/notice&gt;}}

   ```javascript
   const webpack = require(&#34;webpack&#34;);
   var bodyParser = require(&#39;body-parser&#39;)
   const mockServer = require(&#34;./mockserver.js&#34;)

   module.exports = (env, argv) =&gt; {
       // ... some code
       return {
           devServer: {
               setupMiddlewares: (middlewares, devServer) =&gt; {
                   if (!devServer) {
                       throw new Error(&#39;webpack-dev-server is not defined&#39;);
                   }

                   devServer.app.use(bodyParser.json())
                   devServer.app.use(mockServer());
                   // don&#39;t forget this line
                   return middlewares;
               }
           },
           //... and so on
   ```

3. mockserver.js

   mockserver.js 返回给 express dev server 需要的一个中间件（其实在java开发看来，这个东西习惯称为切面或过滤器）。

   ```javascript
   const fs = require(&#34;fs&#34;);
   const path = require(&#34;path&#34;);

   module.exports = function () {
     let mockDataPath = path.resolve(__dirname, &#34;./mock/&#34;);
     let existsMockDir = fs.existsSync(mockDataPath);
     let getMockData = () =&gt; {
       if (existsMockDir) {
         let modules = fs.readdirSync(mockDataPath);
         return modules.reduce((pre, module) =&gt; {
           return {
             ...pre,
             ...require(path.join(mockDataPath, &#34;./&#34; &#43; module)),
           };
         }, {});
       } else {
         console.log(&#34;please create a mock directory under your project root!&#34;);
         return {};
       }
     };

     let splitApiPath = (mockData) =&gt; {
       let data = {};
       for (let path in mockData) {
         let [method, apiPath, sleep] = path.split(&#34; &#34;);
         let newApiPath = method.toLocaleUpperCase() &#43; apiPath;
         data[newApiPath] = {
           path: newApiPath,
           method,
           sleep,
           callback: mockData[path],
         };
       }
       return data;
     };

     let delayFn = (sleep) =&gt; {
       return new Promise((resolve) =&gt; {
         setTimeout(() =&gt; {
           resolve();
         }, sleep);
       });
     };

     async function ret(req, res, next) {
       let { path, method } = req;
       console.log(&#34;mock server received request: %s %s&#34;, method, path);

       if (path.indexOf(&#34;api&#34;) === -1 || !existsMockDir) {
         return next();
       }
       let mockData = splitApiPath(getMockData());
       let pathKey = method.toLocaleUpperCase() &#43; path;
       let { sleep, callback } = mockData[pathKey];
       let isFuntion = callback.__proto__ === Function.prototype;
       if (sleep &amp;&amp; sleep &gt; 0) {
         await delayFn(sleep);
       }
       if (isFuntion) {
         callback(req, res);
       } else {
         res.json({
           ...callback,
         });
       }
       next();
     }

     return async (req, res, next) =&gt; {
       // next();
       return ret(req, res, next);
     };
   };
   ```

4. mock/user.js

   user.js 里面定义 mock 的数据以及如何返回。
   可以按模块多写几个js, 都会自动加载。

   ```javascript
   module.exports = {
     &#34;GET /api/v0/user/list&#34;: {
       success: true,
       code: 200,
       data: {
         list: [
           {
             name: &#34;anyone&#34;,
             age: 18,
           },
         ],
       },
     },
     // method path delay(ms)
     &#34;POST /api/v0/reset 1000&#34;: (req, res) =&gt; {
       res.json({
         success: true,
         code: 200,
         data: { username: req.body.username },
       });
     },
   };
   ```

&gt; 本文参考了[掘金文章](https://juejin.cn/post/7196931784327741498)，我做了些改动以便让它能够在 webpack5 上运行。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/03/2024-03-15-webpack-react-mock/  

