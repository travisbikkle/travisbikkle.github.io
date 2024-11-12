# Mocking Webpack 5 in 10 Minutes


No nonsense, let&#39;s get straight to the point. I&#39;m using webpack 5.90.3 and here is my configuration for mocking.

1. File structure
   ```text
   project_root/
     |-- webpack.config.js
     |-- mockserver.js
     |-- mock/
       |-- user.js
   ```
2. webpack.config.js

   {{&lt;notice tip&gt;}}
   express.js doesn&#39;t bundle with some plugins anymore so you need to run `npm i -D body-parser`. You will need body-parser to read params from request.
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

   mockserver.js returns an middleware(Java developers might rather call it aspect or filter) to express dev server.

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

   user.js defines what and how you want to give to the client.
   Feel free to add more, like data.js, product.js. `mockserver.js` will load them for you.

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

&gt; This article referenced [juejin.cn](https://juejin.cn/post/7196931784327741498), and I did some necessary changes to make it work with current webpack version.


---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/03/2024-03-15-webpack-react-mock/  

