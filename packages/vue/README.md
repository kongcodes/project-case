# 创建项目

```
npm create vue@latest

npm i vant
```

# 一些问题

## vue-router使用history模式本地404的问题


```js
// vite.config.js
export default defineConfig({
  build: {
    assetsDir: 'user_h5_static', // think：这样做的目的是避免很多项目都放在根目录下 文件名重复的问题。 如果用history模式，是不是就不用这样了，因为用his模式后，可以单独放到一个目录下
    rollupOptions: {
      // input: 'user_h5.html', // 默认打包入口是根目录的 index.html 文件，这里重新配置后，需要把根目录的index.html文件改成和此处同名
      input: resolve(__dirname, 'user_h5.html'),
    },
  },
})
```
路由使用`createWebHistory`的前提下：
上述配置后，本地dev环境下，刷新页面会404。因为本地dev环境，vite做了默认配置会回退根目录下的`index.html`，但是上面修改了打包入口，实际应该回退到`user_h5.html`。

参考源码：https://github.com/vitejs/vite/blob/main/packages/vite/src/node/server/middlewares/htmlFallback.ts#L11
https://github.com/vitejs/vite/blob/main/packages/vite/src/node/server/index.ts#L921

### 生产环境

这种情况在生产环境下，可以配置nginx，即 将指定规则下的请求URL，当请求404时，指向一个默认的html入口文件。

```
# 当访问以 / 开头的URL时，如果没有找到想要的文件，会返回 root 配置下 /user_h5.html 文件。
location / {
   try_files $uri $uri/ /user_h5.html;
   # 报403错误时 删掉$uri/ 就不报错了
}

# 当设置了base路径时
    location /user_h5 {
      try_files $uri /user_h5.html /user_h5/user_h5.html;
    }
```

### 本地环境

> 上面说的直接将根目录下的index.html文件改user_h5.html文件，会有本地刷新的问题。又另辟蹊径试了一下，不更改文件名，还保留index.html文件，在根目录新增一个user_h5.html文件，这样不用下面任何解决方案也没问题了。

- 方法1：使用vite插件和钩子函数解决
https://cn.vitejs.dev/guide/api-plugin.html#configureserver

```js
// vite.config.js
import { dirname, resolve } from 'node:path'
import fs from 'fs'

const __dirname = dirname(fileURLToPath(import.meta.url))

// 判断是否为文件请求的辅助函数
const isFileRequest = (url) => {
  const cleanUrl = url.split('?')[0]
  const filePath = resolve(__dirname, cleanUrl.slice(1))
  try {
    return fs.statSync(filePath).isFile()
  } catch {
    return false
  }
}

const myPlugin = () => ({
  name: 'configure-server',
  configureServer(server) {
    server.middlewares.use((req, res, next) => {
      // 自定义请求处理...
      if (
        req.url.startsWith('/@') ||
        req.url.startsWith('/node_modules') ||
        isFileRequest(req.url)
      ) {
        return next()
      }
      // 重写请求到自定义入口文件
      req.url = '/user_h5.html'
      next()
    })
  },
})

export default defineConfig({
  plugins: [vue(), vueDevTools(), myPlugin()],
})
```

- 方法2：（未验证） 使用 `connect-history-api-fallback`。vite6使用的是 `connect`，不是用一个库。

- 方法3：（未验证，有点麻烦） 参考：https://cn.vite.dev/config/shared-options.html#apptype 
https://cn.vite.dev/config/server-options.html#server-middlewaremode

