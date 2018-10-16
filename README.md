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
  }//触发agent-start
  ```
  # nodejs
