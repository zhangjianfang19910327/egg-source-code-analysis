# egg-source-code-analysis
Egg Chinese version source code analysis
中文版egg源码分析分享
# 启动
Node 命令行启动egg 服务的时候首先开启的是
node_modules/egg-bin/bin/egg-bin.js dev

```js
//${yourProjectFullPath}\node_modules/egg-bin/bin/egg-bin.js
const Command = require('..');

new Command().start();
```
# 实例化Command

实例化Command的时候会去调用load函数去加载配置的模式文件

```js
//${yourProjectFullPath}\node_modules\egg-bin\index.js
class EggBin extends Command {
  constructor(rawArgv) {
    super(rawArgv);
    this.usage = 'Usage: egg-bin [command] [options]';

    // load directory
    this.load(path.join(__dirname, 'lib/cmd'));
  }
}
```
New command().start函数即是继承至node_modules\common-bin\lib\command.js

```js
//${yourProjectFullPath}\node_modules\common-bin\lib\command.js
//this.load函数
load(fullPath) {
    assert(fs.existsSync(fullPath) && fs.statSync(fullPath).isDirectory(),
      `${fullPath} should exist and be a directory`);

    // load entire directory
    const files = fs.readdirSync(fullPath);
    const names = [];
    for (const file of files) {
      if (path.extname(file) === '.js') {
        const name = path.basename(file).replace(/\.js$/, '');
        names.push(name);
        this.add(name, path.join(fullPath, file));
      }
    }
}
    //load函数完成后开始调用start函数
start() {
    co(function* () {
      // replace `--get-yargs-completions` to our KEY, so yargs will not block our DISPATCH
      const index = this.rawArgv.indexOf('--get-yargs-completions');
      if (index !== -1) {
        // bash will request as `--get-yargs-completions my-git remote add`, so need to remove 2
        this.rawArgv.splice(index, 2, `--AUTO_COMPLETIONS=${this.rawArgv.join(',')}`);
      }
      yield this[DISPATCH]();
    }.bind(this)).catch(this.errorHandler.bind(this));
  }
}
```
在start函数中调用了DISPATCH函数，DISPATCH是个Symbol变量定义的属性,目的是找到运行模式输入的参数进行不同的实例初始化

```js
//${yourProjectFullPath}\node_modules\common-bin\lib\command.js
/**
   * dispatch command, either `subCommand.exec` or `this.run`
   * @param {Object} context - context object
   * @param {String} context.cwd - process.cwd()
   * @param {Object} context.argv - argv parse result by yargs, `{ _: [ 'start' ], '$0': '/usr/local/bin/common-bin', baseDir: 'simple'}`
   * @param {Array} context.rawArgv - the raw argv, `[ "--baseDir=simple" ]`
   * @private
   */
  * [DISPATCH]() {
    // define --help and --version by default
    this.yargs
      // .reset()
      .completion()
      .help()
      .version()
      .wrap(120)
      .alias('h', 'help')
      .alias('v', 'version')
      .group([ 'help', 'version' ], 'Global Options:');

    // get parsed argument without handling helper and version
    const parsed = yield this[PARSE](this.rawArgv);
    const commandName = parsed._[0];

    if (parsed.version && this.version) {
      console.log(this.version);
      return;
    }
```
因为用的dev模式所以启动的是node_modules\egg-bin\lib\cmd\dev.js 
文件dev.js重写command.js中run函数  dev.js中run函数为*function进入helperjs中的,然后调用了forknode函数

```js
//${yourProjectFullPath}\node_modules\common-bin\lib\helper.js
exports.forkNode = (modulePath, args = [], options = {}) => {
  options.stdio = options.stdio || 'inherit';
  debug('Run fork `%s %s %s`', process.execPath, modulePath, args.join(' '));
  const proc = cp.fork(modulePath, args, options);
  gracefull(proc);

  return new Promise((resolve, reject) => {
    proc.once('exit', code => {
      childs.delete(proc);
      if (code !== 0) {
        const err = new Error(modulePath + ' ' + args + ' exit with code ' + code);
        err.code = code;
        reject(err);
      } else {
        resolve();
      }
    });
  });
};
```

调起守护进程gracefull函数
第一个参数为d:\egg\egg-example\node_modules\egg-bin\lib\start-cluster  （modelpath）
第二个参数为
[ '{"baseDir":"d:\\\\egg\\\\egg-example","framework":"d:\\\\egg\\\\egg-example\\\\node_modules\\\\egg","workers":1}' ]     (args)
第三个参数是一些系统的环境参数包括系统的环境变量 npm.config (options)
调用的cp.fork函数是nodejs自带的模块
运行的模块为${yourProjectFullPath}\node_modules\egg-bin\lib\start-cluster
options=[ '{"baseDir":"d:\\\\egg\\\\egg-example","framework":"d:\\\\egg\\\\egg-example\\\\node_modules\\\\egg","workers":1}' ]
所以options.framework=d:\\\\egg\\\\egg-example\\\\node_modules\\\\egg
进入egg.js文件
在D:\egg\egg-example\node_modules\egg\index.js中启动cluster

```js
//D:\myproject\egg-project\node_modules\egg-cluster\index.js
exports.startCluster = function(options, callback) {
  new Master(options).ready(callback);
};
```
# 实例化master
```js
//D:\myproject\egg-project\node_modules\egg-cluster\lib\master.js找到构造函数绑定了ready回调函数
constructor(options) {
    super();
    this.options = parseOptions(options);
    this.workerManager = new Manager();
    this.messenger = new Messenger(this);

    ready.mixin(this);

    this.isProduction = isProduction();
    this.agentWorkerIndex = 0;
    this.closed = false;
    this[REALPORT] = this.options.port;

    // app started or not
    this.isStarted = false;
    this.logger = new ConsoleLogger({ level: process.env.EGG_MASTER_LOGGER_LEVEL || 'INFO' });
    this.logMethod = 'info';
    if (process.env.EGG_SERVER_ENV === 'local' || process.env.NODE_ENV === 'development') {
      this.logMethod = 'debug';
    }
    ····
}
```
触发forkAgentWorker函数
在D:\egg\egg-example\node_modules\detect-port\lib\detect-port.js中找到detecport函数
在此函数中判断入参个数做处理，此例入参为functions（就这一个参数）
forkAgentWorker里面启动message监听,监听的是D:\egg\egg-example\node_modules\egg-cluster\lib\agent_worker.js
启动的代码中Agent初始化完成后会向父进程发送消息,父进程拿到消息后进行封装参数,将msg 属性填写完整交给自己的messenger去发送消息
D:\egg\egg-example\node_modules\egg-cluster\lib\utils\messenger.js

```js
//D:\myproject\egg-project\node_modules\egg-cluster\lib\utils\messenger.js
sendToMaster(data) {
    this.master.emit(data.action, data.data);
  }
  //触发agent-start   D:\myproject\egg-project\node_modules\egg-cluster\lib\master.js
  this.once('agent-start', this.forkAppWorkers.bind(this));
  //····略
   onAgentStart() {
    this.agentWorker.status = 'started';

    // Send egg-ready when agent is started after launched
    if (this.isAllAppWorkerStarted) {
      this.messenger.send({ action: 'egg-ready', to: 'agent', data: this.options });
    }

    this.messenger.send({ action: 'egg-pids', to: 'app', data: [ this.agentWorker.pid ] });
    // should send current worker pids when agent restart
    if (this.isStarted) {
      this.messenger.send({ action: 'egg-pids', to: 'agent', data: this.workerManager.getListeningWorkerIds() });
    }

    this.messenger.send({ action: 'agent-start', to: 'app' });
    this.logger.info('[master] agent_worker#%s:%s started (%sms)',
      this.agentWorker.id, this.agentWorker.pid, Date.now() - this.agentStartTime);
  }
  forkAppWorkers() {
    this.appStartTime = Date.now();
    this.isAllAppWorkerStarted = false;
    this.startSuccessCount = 0;

    const args = [ JSON.stringify(this.options) ];
    this.log('[master] start appWorker with args %j', args);
    cfork({
      exec: this.getAppWorkerFile(),//app_worker.js
      args,
      silent: false,
      count: this.options.workers,
      // don't refork in local env
      refork: this.isProduction,
    });

    let debugPort = process.debugPort;
    cluster.on('fork', worker => {
      worker.disableRefork = true;
      this.workerManager.setWorker(worker);
      worker.on('message', msg => {
        if (typeof msg === 'string') msg = { action: msg, data: msg };
        msg.from = 'app';
        this.messenger.send(msg);
      });
      this.log('[master] app_worker#%s:%s start, state: %s, current workers: %j',
        worker.id, worker.process.pid, worker.state, Object.keys(cluster.workers));

      // send debug message, due to `brk` scence, send here instead of app_worker.js
      if (this.options.isDebug) {
        debugPort++;
        this.messenger.send({ to: 'parent', from: 'app', action: 'debug', data: { debugPort, pid: worker.process.pid } });
      }
    });
    cluster.on('disconnect', worker => {
      this.logger.info('[master] app_worker#%s:%s disconnect, suicide: %s, state: %s, current workers: %j',
        worker.id, worker.process.pid, worker.exitedAfterDisconnect, worker.state, Object.keys(cluster.workers));
    });
    cluster.on('exit', (worker, code, signal) => {
      this.messenger.send({
        action: 'app-exit',
        data: { workerPid: worker.process.pid, code, signal },
        to: 'master',
        from: 'app',
      });
    });
    cluster.on('listening', (worker, address) => {
      this.messenger.send({
        action: 'app-start',
        data: { workerPid: worker.process.pid, address },
        to: 'master',
        from: 'app',
      });
    });
  }
  ```

  # fork appworker (生产app进程)

  实例化Application

  ```js
  //D:\myproject\egg-project\node_modules\egg-cluster\lib\app_worker.js
  const Application = require(options.framework).Application;
  const app = new Application(options);//node_modules\egg\lib\application.js
  ```
  node_modules\egg\lib\application.js

  ```js
  constructor(options = {}) {
    options.type = 'application';
    super(options);

    // will auto set after 'server' event emit
    this.server = null;

    try {
      this.loader.load();
    } catch (e) {
      // close gracefully
      this[CLUSTER_CLIENTS].forEach(cluster.close);
      throw e;
    }

    // dump config after loaded, ensure all the dynamic modifications will be recorded
    const dumpStartTime = Date.now();
    this.dumpConfig();
    this.coreLogger.info('[egg:core] dump config after load, %s', ms(Date.now() - dumpStartTime));

    this[WARN_CONFUSED_CONFIG]();
    this[BIND_EVENTS]();
  }
  ```
  构造函数调用this.loader.load();
  此方法继承在D:\egg\egg-example\node_modules\egg-core\lib\egg.js

  ```js
  const Loader = this[EGG_LOADER];
    assert(Loader, 'Symbol.for(\'egg#loader\') is required');
    this.loader = new Loader({
      baseDir: options.baseDir,
      app: this,
      plugins: options.plugins,
      logger: this.console,
      serverScope: options.serverScope,
    });
 get [EGG_LOADER]() {
    return require('./loader/egg_loader');
  }
  ```
找到D:\egg\egg-example\node_modules\egg-core\lib\loader\egg_loader.js

```js
const loaders = [
  require('./mixin/plugin'),
  require('./mixin/config'),
  require('./mixin/extend'),
  require('./mixin/custom'),
  require('./mixin/service'),
  require('./mixin/middleware'),
  require('./mixin/controller'),
  require('./mixin/router'),
];

for (const loader of loaders) {
  Object.assign(EggLoader.prototype, loader);
}

module.exports = EggLoader;
```
D:\egg\egg-example\node_modules\egg\lib\egg.js中的

```js
constructor(options) {
    super(options);

    // export context base classes, let framework can impl sub class and over context extend easily.
    this.ContextCookies = ContextCookies;
    this.ContextLogger = ContextLogger;
    this.ContextHttpClient = ContextHttpClient;
    this.HttpClient = HttpClient;

    this.loader.loadConfig();//集成过来的loader方法
    //略
}
```
loadConfig

```js
//D:\myproject\egg-project\node_modules\egg-core\lib\loader\mixin\config.js
loadConfig() {
    this.timing.start('Load Config');
    this.configMeta = {};

    const target = {};

    // Load Application config first
    const appConfig = this._preloadAppConfig();//下面介绍

    //   plugin config.default
    //     framework config.default
    //       app config.default
    //         plugin config.{env}
    //           framework config.{env}
    //             app config.{env}
    for (const filename of this.getTypeFiles('config')) {
      for (const unit of this.getLoadUnits()) {
        const isApp = unit.type === 'app';
        const config = this._loadConfig(unit.path, filename, isApp ? undefined : appConfig, unit.type);

        if (!config) {
          continue;
        }

        debug('Loaded config %s/%s, %j', unit.path, filename, config);
        extend(true, target, config);
      }
    }

    // You can manipulate the order of app.config.coreMiddleware and app.config.appMiddleware in app.js
    target.coreMiddleware = target.coreMiddlewares = target.coreMiddleware || [];
    target.appMiddleware = target.appMiddlewares = target.middleware || [];

    this.config = target;
    this.timing.end('Load Config');
  },
  ```
getTypeFiles

```js
//D:\myproject\egg-project\node_modules\egg-core\lib\loader\egg_loader.js
 getTypeFiles(filename) {
    const files = [ `${filename}.default` ];
    if (this.serverScope) files.push(`${filename}.${this.serverScope}`);
    if (this.serverEnv === 'default') return files;

    files.push(`${filename}.${this.serverEnv}`);
    if (this.serverScope) files.push(`${filename}.${this.serverScope}_${this.serverEnv}`);
    return files;
  }
  ```
_preloadAppConfig函数
已经约定好的文件名字
后续都是调用_loadConfig函数来完成加载

```js
//D:\myproject\egg-project\node_modules\egg-core\lib\loader\mixin\config.js
_preloadAppConfig() {
    const names = [
      'config.default',
      `config.${this.serverEnv}`,
    ];
    const target = {};
    for (const filename of names) {
      const config = this._loadConfig(this.options.baseDir, filename, undefined, 'app');
      extend(true, target, config);
    }
    return target;
  },
  ```
_loadConfig函数

```js
//D:\myproject\egg-project\node_modules\egg-core\lib\loader\mixin\config.js
_loadConfig(dirpath, filename, extraInject, type) {
    const isPlugin = type === 'plugin';
    const isApp = type === 'app';

    let filepath = this.resolveModule(path.join(dirpath, 'config', filename));
    // let config.js compatible
    if (filename === 'config.default' && !filepath) {
      filepath = this.resolveModule(path.join(dirpath, 'config/config'));
    }
    const config = this.loadFile(filepath, this.appInfo, extraInject);

    if (!config) return null;

    if (isPlugin || isApp) {
      assert(!config.coreMiddleware, 'Can not define coreMiddleware in app or plugin');
    }
    if (!isApp) {
      assert(!config.middleware, 'Can not define middleware in ' + filepath);
    }

    // store config meta, check where is the property of config come from.
    this[SET_CONFIG_META](config, filepath);

    return config;
  },

  [SET_CONFIG_META](config, filepath) {
    config = extend(true, {}, config);
    setConfig(config, filepath);
    extend(true, this.configMeta, config);
  },
};
```
_loadconfig调用loadfile函数
Load函数在D:\egg\egg-example\node_modules\egg-core\lib\loader\file_loader.js
主要解决了所有插件的配置对象的加载和继承到当前实例上的this.config属性（在此之前已经将所有插件的路径读取到数组中去）每个插件都有一个app.js文件

```js
//D:\myproject\egg-project\node_modules\egg\lib\loader\app_worker_loader.js
loadConfig() {
    this.loadPlugin();
    super.loadConfig();
  }
load() {
    // app > plugin > core
    this.loadApplicationExtend();
    this.loadRequestExtend();
    this.loadResponseExtend();
    this.loadContextExtend();
    this.loadHelperExtend();

    // app > plugin
    this.loadBootHook();
    this.loadCustomApp();
    // app > plugin
    this.loadService();
    // app > plugin > core
    this.loadMiddleware();
    // app
    this.loadController();
    // app
    this.loadRouter(); // Dependent on controllers
  }
```
其实调用的函数是上图这个函数先加载了插件  然后读取配置文件
loadPlugin

```js
//D:\egg\egg-example\node_modules\egg-core\lib\loader\mixin\plugin.js
 loadPlugin() {
    this.timing.start('Load Plugin');

    // loader plugins from application
    const appPlugins = this.readPluginConfigs(path.join(this.options.baseDir, 'config/plugin.default'));
    debug('Loaded app plugins: %j', Object.keys(appPlugins));

    // loader plugins from framework
    const eggPluginConfigPaths = this.eggPaths.map(eggPath => path.join(eggPath, 'config/plugin.default'));
    const eggPlugins = this.readPluginConfigs(eggPluginConfigPaths);
    debug('Loaded egg plugins: %j', Object.keys(eggPlugins));

    // loader plugins from process.env.EGG_PLUGINS
    let customPlugins;
    if (process.env.EGG_PLUGINS) {
      try {
        customPlugins = JSON.parse(process.env.EGG_PLUGINS);
      } catch (e) {
        debug('parse EGG_PLUGINS failed, %s', e);
      }
    }

    // loader plugins from options.plugins
    if (this.options.plugins) {
      customPlugins = Object.assign({}, customPlugins, this.options.plugins);
    }

    if (customPlugins) {
      for (const name in customPlugins) {
        this.normalizePluginConfig(customPlugins, name);
      }
      debug('Loaded custom plugins: %j', Object.keys(customPlugins));
    }

    this.allPlugins = {};
    this.appPlugins = appPlugins;
    this.customPlugins = customPlugins;
    this.eggPlugins = eggPlugins;

    this._extendPlugins(this.allPlugins, eggPlugins);
    this._extendPlugins(this.allPlugins, appPlugins);
    this._extendPlugins(this.allPlugins, customPlugins);

    const enabledPluginNames = []; // enabled plugins that configured explicitly
    const plugins = {};
    const env = this.serverEnv;
    for (const name in this.allPlugins) {
      const plugin = this.allPlugins[name];

      // resolve the real plugin.path based on plugin or package
      plugin.path = this.getPluginPath(plugin, this.options.baseDir);

      // read plugin information from ${plugin.path}/package.json
      this.mergePluginConfig(plugin);

      // disable the plugin that not match the serverEnv
      if (env && plugin.env.length && !plugin.env.includes(env)) {
        this.options.logger.info('Plugin %s is disabled by env unmatched, require env(%s) but got env is %s', name, plugin.env, env);
        plugin.enable = false;
        continue;
      }

      plugins[name] = plugin;
      if (plugin.enable) {
        enabledPluginNames.push(name);
      }
    }

    // retrieve the ordered plugins
    this.orderPlugins = this.getOrderPlugins(plugins, enabledPluginNames, appPlugins);

    const enablePlugins = {};
    for (const plugin of this.orderPlugins) {
      enablePlugins[plugin.name] = plugin;
    }
    debug('Loaded plugins: %j', Object.keys(enablePlugins));

    /**
     * Retrieve enabled plugins
     * @member {Object} EggLoader#plugins
     * @since 1.0.0
     */
    this.plugins = enablePlugins;

    this.timing.end('Load Plugin');
  },
  ```
  主要介绍的函数有readPluginConfigs主要是读取app工程下的config/plugin.js中的配置项得到所有的配置对象然后遍历这些插件  将这些插件的路径都保存在一个数组中

loadConfig

```js
//
for (const filename of this.getTypeFiles('config')) {
      for (const unit of this.getLoadUnits()) {
        const isApp = unit.type === 'app';
        const config = this._loadConfig(unit.path, filename, isApp ? undefined : appConfig, unit.type);

        if (!config) {
          continue;
        }

        debug('Loaded config %s/%s, %j', unit.path, filename, config);
        extend(true, target, config);
      }
    }
```
getLoadUnits加载单元

```js
getLoadUnits() {
    if (this.dirs) {
      return this.dirs;
    }

    const dirs = this.dirs = [];

    if (this.orderPlugins) {
      for (const plugin of this.orderPlugins) {
        dirs.push({
          path: plugin.path,
          type: 'plugin',
        });
      }
    }

    // framework or egg path
    for (const eggPath of this.eggPaths) {
      dirs.push({
        path: eggPath,
        type: 'framework',
      });
    }

    // application
    dirs.push({
      path: this.options.baseDir,
      type: 'app',
    });

    debug('Loaded dirs %j', dirs);
    return dirs;
  }
loadFile(filepath, ...inject) {
if (!filepath || !fs.existsSync(filepath)) {
    return null;
}

// function(arg1, args, ...) {}
if (inject.length === 0) inject = [ this.app ];

let ret = this.requireFile(filepath);
if (is.function(ret) && !is.class(ret)) {
    ret = ret(...inject);
}
return ret;
}
```
上图中在这里要是加载的模块是函数的话就将实例APP传给函数 然后将插件挂载在实例上面如果只是一个对象模块就直接返回
然后调用D:\egg\egg-example\node_modules\egg\lib\loader\app_worker_loader.js中的load函数

框架中将读取到的配置信息来注册插件

```js
//D:\egg\egg-example\node_modules\egg\lib\core\singleton.js
constructor(options = {}) {
    assert(options.name, '[egg:singleton] Singleton#constructor options.name is required');
    assert(options.app, '[egg:singleton] Singleton#constructor options.app is required');
    assert(options.create, '[egg:singleton] Singleton#constructor options.create is required');
    assert(!options.app[options.name], `${options.name} is already exists in app`);
    this.clients = new Map();
    this.app = options.app;
    this.name = options.name;
    this.create = options.create;
    /* istanbul ignore next */
    this.options = options.app.config[this.name] || {};//这里
  }
initSync() {
    const options = this.options;
    assert(!(options.client && options.clients),
      `egg:singleton ${this.name} can not set options.client and options.clients both`);

    // alias app[name] as client, but still support createInstance method
    if (options.client) {
      const client = this.createInstance(options.client);
      this.app[this.name] = client;
      this._extendDynamicMethods(client);
      return;
    }

    // multi clent, use app[name].getInstance(id)
    if (options.clients) {
      Object.keys(options.clients).forEach(id => {
        const client = this.createInstance(options.clients[id]);
        this.clients.set(id, client);
      });
      this.app[this.name] = this;//这里注册插件
      return;
    }
    ```
```js
//D:\myproject\egg-project\node_modules\egg\lib\egg.js
addSingleton(name, create) {
    const options = {};
    options.name = name;
    options.create = create;
    options.app = this;
    const singleton = new Singleton(options);
    const initPromise = singleton.init();
    if (initPromise) {
      this.beforeStart(async () => {
        await initPromise;
      });
    }
  }
  ```
init函数

```js
//D:\myproject\egg-project\node_modules\egg\lib\core\singleton.js
init() {
    return is.asyncFunction(this.create) ? this.initAsync() : this.initSync();
  }
```
initSync函数调用了createInstance
createInstance

```js
createInstance(config) {
    // async creator only support createInstanceAsync
    assert(!is.asyncFunction(this.create),
      `egg:singleton ${this.name} only support create asynchronous, please use createInstanceAsync`);
    // options.default will be merge in to options.clients[id]
    config = Object.assign({}, this.options.default, config);
    return this.create(config, this.app);
  }
```

在生命周期函数调用

```js
//D:\myproject\egg-project\node_modules\egg-core\lib\lifecycle.js
constructor(options) {
    super();
    this.options = options;
    this[BOOT_HOOKS] = [];
    this[BOOTS] = [];
    this[CLOSE_SET] = new Set();
    this[IS_CLOSED] = false;
    this[INIT] = false;
    getReady.mixin(this);

    this.timing.start('Application Start');
    // get app timeout from env or use default timeout 10 second
    const eggReadyTimeoutEnv = Number.parseInt(process.env.EGG_READY_TIMEOUT_ENV || 10000);
    assert(
      Number.isInteger(eggReadyTimeoutEnv),
      `process.env.EGG_READY_TIMEOUT_ENV ${process.env.EGG_READY_TIMEOUT_ENV} should be able to parseInt.`);
    this.readyTimeout = eggReadyTimeoutEnv;

    this[INIT_READY]();
    this
      .on('ready_stat', data => {
        this.logger.info('[egg:core:ready_stat] end ready task %s, remain %j', data.id, data.remain);
      })
      .on('ready_timeout', id => {
        this.logger.warn('[egg:core:ready_timeout] %s seconds later %s was still unable to finish.', this.readyTimeout / 1000, id);
      });

    this.ready(err => {
      this.triggerDidReady(err);
      this.timing.end('Application Start');
    });
  }
  ```
  构造函数中会调用这个方法
这个方法依赖了ready-callback这个npm包
ready-callback依赖get-ready这个包get-ready包就是给app
扩展属性的  扩展了一个ready（）的属性方法ready参数有两种
1）true----->直接触发所有的回调函数
2)函数---->给添加ready的回调函数（可以多个，需要每次添加一个）
只要触发ready（true）所有的回调就会执行
ready-callback是给app扩展了readyCallback（）、ready（）方法
const done =app.readyCallback('name');
done();
上面两行代码就是直接触发了app.ready(true)然后调用所有委托在app.ready()方法的回调函数

