---
title: vue-cli-service
date: 2019-09-05 09:46:34
cover: 'topbg.jpg'
---


### vue-cli-service 机制

`npm run dev` 之后发生了些什么？？？



#### 入口

从`package.json`里面可以看到`npm run dev`其实就是`vue-cli-service serve`

vue-cli3.0 安装的时候把`vue-cli-service`一并安装了，即执行了`npm install vue-cli-service --save-dev`

这样就可以在`./node_modules/.bin`目录下查看到`vue-cli-service`

```javascript
@IF EXIST "%~dp0\node.exe" (
  "%~dp0\node.exe"  "%~dp0\..\@vue\cli-service\bin\vue-cli-service.js" %*
) ELSE (
  @SETLOCAL
  @SET PATHEXT=%PATHEXT:;.JS;=;%
  node  "%~dp0\..\@vue\cli-service\bin\vue-cli-service.js" %*
)
```

> 入口：./node_modules/@vue/cli-service/bin/vue-cli-service.js



瞅一眼vue-cli-service.js的核心代码

```javascript
const Service = require('../lib/Service')
// 实例化Service
// VUE_CLI_CONTEXT为undefined，所以传入的值为process.cwd()及项目所在目录
const service = new Service(process.env.VUE_CLI_CONTEXT || process.cwd())

const rawArgv = process.argv.slice(2)
// minimist解析工具来对命令行参数进行解析
const args = require('minimist')(rawArgv, {
  boolean: [
    // build
    'modern',
    'report',
    'report-json',
    'watch',
    // serve
    'open',
    'copy',
    'https',
    // inspect
    'verbose'
  ]
})
// { 
// 	_: [ 'serve' ],
// 	modern: false,
// 	report: false,
// 	'report-json': false,
// 	watch: false,
// 	open: false,
// 	copy: false,
// 	https: false,
// 	verbose: false ,
// } 
const command = args._[0]

// 执行service方法传入:'serve'、agrs、['serve','--open',...]
service.run(command, args, rawArgv).catch(err => {
  error(err)
  process.exit(1)
})
```



#### 初始化配置

```javascript
module.exports = class Service {
  constructor (context, { plugins, pkg, inlineOptions, useBuiltIn } = {}) {
    process.VUE_CLI_SERVICE = this
    this.initialized = false
    this.context = context
    this.inlineOptions = inlineOptions
    this.webpackChainFns = []
    this.webpackRawConfigFns = []
    this.devServerConfigFns = []
    this.commands = {}
    this.pkgContext = context
    // 获取package.json中的依赖
    this.pkg = this.resolvePkg(pkg)
	
	// 如果有内联插件，不使用package.json中找到的插件
	// 最终得到的plugins为内置插件+@vue/cli-plugin-*
	// {id: 'xxx',apply: require('xxx')}
    this.plugins = this.resolvePlugins(plugins, useBuiltIn)

	// 解析每个命令使用的默认模式
	//{ serve: 'development',
	// build: 'production',
	// inspect: 'development' }
    this.modes = this.plugins.reduce((modes, { apply: { defaultModes }}) => {
      return Object.assign(modes, defaultModes)
    }, {})
  }
  
  // ......
}
```

1. 获取package.json中的依赖。通过```read-pkg```这个包读取package.json文件，并以JSON格式返回赋值给this.pkg。

2. 初始化相关插件。相关插件包括：内联插件、package.json中的cli-plugin-*插件。内联插件包含serve、build、inspect等。

   ```javascript
   resolvePlugins (inlinePlugins, useBuiltIn) {
       const idToPlugin = id => ({
         id: id.replace(/^.\//, 'built-in:'),
         apply: require(id)
       })
       
       let plugins

       const builtInPlugins = [
         './commands/serve',
         './commands/build',
         './commands/inspect',
         './commands/help',
         // config plugins are order sensitive
         './config/base',
         './config/css',
         './config/dev',
         './config/prod',
         './config/app'
       ].map(idToPlugin)

       if (inlinePlugins) {
         // ...
       } else {
   		// 开发环境依赖+生产环境的依赖中，获取cli-plugin-*插件
         const projectPlugins = Object.keys(this.pkg.devDependencies || {})
           .concat(Object.keys(this.pkg.dependencies || {}))
           .filter(isPlugin)	// 验证是否为vue-cli插件 cli-plugin-*
           .map(idToPlugin)
         plugins = builtInPlugins.concat(projectPlugins)
       }
     
     // ...
   }
   ```

3. 初始化模式。解析每个命令使用的默认模式，例如serve对应的development，build对应production。

   ```javascript
   // 解析每个命令使用的默认模式
   //{ serve: 'development',
   // build: 'production',
   // inspect: 'development' }
   this.modes = this.plugins.reduce((modes, { apply: { defaultModes }}) => {
         return Object.assign(modes, defaultModes)
   }, {})
   ```

   内联插件默认导出了一个defaultModes

   ```javascript
   // serve.js
   module.exports.defaultModes = {
     serve: 'development'
   }
   ```

   为什么要初始化模式？

   ​

#### 解析命令行参数

minimist解析工具来对命令行参数进行解析。shell命令携带的参数解析出来。

```javascript
const args = require('minimist')(rawArgv, {
  boolean: [
    // build
    // serve
    'open',
    'copy',
    'https',
  ]
})

// 'serve'、agrs、['serve','--open',...]
```



#### 加载插件

运行service.run方法，加载环境变量，加载用户配置，应用插件。

```javascript
async run (name, args = {}, rawArgv = []) {
    const mode = args.mode || (name === 'build' && args.watch ? 'development' : 		  this.modes[name])

    // 加载环境变量，加载用户配置，应用插件
    this.init(mode)
  
    args._ = args._ || []
  	// 注册完的命令集里获取对应的命令
    let command = this.commands[name]
    // ....
    
    const { fn } = command
    return fn(args, rawArgv)
}
```

1. 加载环境变量，从项目根目录下的.env.(mode)文件读取环境变量，并写入到process.env（声明环境变量必须是VUE_APP_*，不然会被过滤掉）

```javascript
init (mode = process.env.VUE_CLI_MODE) {
	// 加载development.env环境变量
    if (mode) {
      this.loadEnv(mode)
    }
    this.loadEnv()
	// ...
}
// 加载环境变量方法（部分）
loadEnv (mode) {
	// 项目路径/.env.development
    const basePath = path.resolve(this.context, `.env${mode ? `.${mode}` : ``}`)
	// 项目路径/.env.development.local
    const localPath = `${basePath}.local`

    const load = path => {
        const env = dotenv.config({ path, debug: process.env.DEBUG })
        // 这里会先调用 dotenv（位于 node_modules/dotenv ）的 config 函数，最终会返回这样的格式 { parsed: { YOUR_ENV_KEY: '你设定的环境变量值' } }，并且写入到process.env里面
        dotenvExpand(env)
        logger(path, env)
    }
    load(localPath)
    load(basePath)
  
    if (mode) {
      const defaultNodeEnv = (mode === 'production' || mode === 'test')
        ? mode
        : 'development'
      if (shouldForceDefaultEnv || process.env.NODE_ENV == null) {
        process.env.NODE_ENV = defaultNodeEnv
      }
    }
  }
```



2. 加载用户配置。读取vue.config.js文件内部的配置

```javascript
init (mode = process.env.VUE_CLI_MODE) {
    // 加载用户配置
    const userOptions = this.loadUserOptions()
    this.projectOptions = defaultsDeep(userOptions, defaults())
}
```



3. 应用插件。前面初始化插件的时候(resolvePlugins方法)有一个apply属性，调用这个方法就可以导入对应的插件，这里就是调用了serve.js的方法。

```javascript
//const idToPlugin = id => ({
//    id: id.replace(/^.\//, 'built-in:'),
//    apply: require(id)
//})
init (mode = process.env.VUE_CLI_MODE) {
    // 应用插件
    this.plugins.forEach(({ id, apply }) => {
      apply(new PluginAPI(id, this), this.projectOptions)
    })

    // 从项目配置文件中应用webpack配置
    if (this.projectOptions.chainWebpack) {
      this.webpackChainFns.push(this.projectOptions.chainWebpack)
    }
    if (this.projectOptions.configureWebpack) {
      this.webpackRawConfigFns.push(this.projectOptions.configureWebpack)
    }
  }
}
```

4. 注册命令。在service注册serve命令（前面run方法最后调用了fn函数，也就是registerCommand的第三个参数）

```javascript
// serve.js
api.registerCommand('serve', {
    description: 'start development server',
    usage: 'vue-cli-service serve [options] [entry]',
    options: {}
  }, async function serve (args) {
   // ...
});
```



#### 配置、启动服务

最终执行的serve.js 内注册serve时传递的方法。webpack获取到配置之后，实例化`Compiler` 传递给```webpackDevServer```，通过```webpackDevServer```实现自动编译和热更新。

```javascript
// serve.js serve函数
async function serve (args) {
  //创建webpack编译器
  const compiler = webpack(webpackConfig)
  // compiler.run()即可完成一次编译打包

  // 创建本地服务
  const server = new WebpackDevServer(compiler, Object.assign({
      clientLogLevel: 'none',
      historyApiFallback: {
        disableDotRule: true,
        rewrites: [
          { from: /./, to: path.posix.join(options.baseUrl, 'index.html') }
        ]
      },
      contentBase: api.resolve('public'),
      watchContentBase: !isProduction,
      hot: !isProduction,
      quiet: true,
      compress: isProduction,
      publicPath: options.baseUrl,
      overlay: isProduction // TODO disable this
        ? false
        : { warnings: false, errors: true }
    }, projectDevServerOptions, {
      https: useHttps,
      proxy: proxySettings,
  }))
  
  return new Promise((resolve, reject) => {
    compiler.hooks.done.tap('vue-cli-service serve', stats => {
    	// ...
    })
    // ...
    server.listen(port, host, err => {
        if (err) {
          reject(err)
        }
    })
  })
}
```

- 获取 ```webpack``` 配置：```api.resolveWebpackConfig()```
- 获取 ```devServer ```配置
- 注入 ```webpack-dev-server``` 和 ```hot-reload（HRM）```中间件入口
- 创建 ```webpack-dev-server``` 实例

启动webpack-dev-server后，在目标文件夹中是看不到编译后的文件的,实时编译后的文件都保存到了内存当中。

