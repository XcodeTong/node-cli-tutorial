注：react-native 0.39版本已经不再使用yeoman ，但是yeoman还是有它使用的场景在的。
# Cli.js

通过上文我们知道了 react-native init ProjectName 执行的操作里面

1. 创建 AwesomeProject 文件夹
2. 创建 package.json
3. 创建 iOS，Android 初始化工程
4. 执行 npm install 安装依赖

后两个步骤是cli.js 的init 方法执行的

##init 方法 生成了 Android 和iOS 的工程文件，工程的名称是 ProjectName

###怎么做到的呢？

local-cli 文件夹下存放着iOS 和Android 的模板文件

RN启动入口的模板文件在

generator/templates/index.ios.js 

generator/templates/index.android.js 

```
export default class <%= name %> extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.ios.js
        </Text>
        <Text style={styles.instructions}>
          Press Cmd+R to reload,{'\n'}
          Cmd+D or shake for dev menu
        </Text>
      </View>
    );
  }
}
```

看到 ```<%= name %>``` 是不是一目了然的感觉，说明我们输入的工程名称，作为参数传入到了这里。

再打开
/generator-ios/templates/app/AppDelegate.m 看一下

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURL *jsCodeLocation;

  jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];

  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"<%= name %>"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
  rootView.backgroundColor = [[UIColor alloc] initWithRed:1.0f green:1.0f blue:1.0f alpha:1];

  self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
  UIViewController *rootViewController = [UIViewController new];
  rootViewController.view = rootView;
  self.window.rootViewController = rootViewController;
  [self.window makeKeyAndVisible];
  return YES;
}
```

安卓的工程模板在 generator-android/templates 下

里面的文件同样有 ```<%= name %>``` 



###那么是谁来做的替换呢？

回到cli.js 入口文件看

```
'use strict';

// This file must be able to run in node 0.12 without babel so we can show that
// it is not supported. This is why the rest of the cli code is in `cliEntry.js`.
require('./server/checkNodeVersion')();

//命令行如果要支持ES6语法，需要babel 转换
require('../packager/babelRegisterOnly')([
  /private-cli\/src/,
  /local-cli/,
  /react-packager\/src/
]);

//真正的入口文件，react native init 就是执行的它的init方法，let's go
var cliEntry = require('./cliEntry');

if (require.main === module) {
  cliEntry.run();
}

module.exports = cliEntry;


```



cliEntry.js 

从下往上看源码

导出了 init

```
module.exports = {
  run: run,
  init: init,
};
```

搜索init 的定义位置，在头部

```
const init = require('./init/init');

```

看到这，这个文件可以先不用看了，直接去看 ./init/init 文件


```

'use strict';

const path = require('path');
const TerminalAdapter = require('yeoman-environment/lib/adapter.js');
const yeoman = require('yeoman-environment');

class CreateSuppressingTerminalAdapter extends TerminalAdapter {
  constructor() {
    super();
    // suppress 'create' output generated by yeoman
    this.log.create = function() {};
  }
}

/**
 * Creates the template for a React Native project given the provided
 * parameters:
 *   - projectDir: templates will be copied here.
 *   - argsOrName: project name or full list of custom arguments to pass to the
 *                 generator.
 */
function init(projectDir, argsOrName) {
  console.log('Setting up new React Native app in ' + projectDir);
  const env = yeoman.createEnv(
    undefined,
    undefined,
    new CreateSuppressingTerminalAdapter()
  );

  env.register(
    require.resolve(path.join(__dirname, '../generator')),
    'react:app'
  );

  // argv is for instance
  // ['node', 'react-native', 'init', 'AwesomeApp', '--verbose']
  // args should be ['AwesomeApp', '--verbose']
  const args = Array.isArray(argsOrName)
    ? argsOrName
    : [argsOrName].concat(process.argv.slice(4));

  const generator = env.create('react:app', {args: args});
  generator.destinationRoot(projectDir);
  generator.run();
}

module.exports = init;
```


从这个文件我们可以知道 generator 的使用的是 yeoman 库了

##使用 yeoman-generator 构建模板工程

官网 http://yeoman.io/

yeoman 是套脚手架构建工具，包含三个部分：yo,grunt,bower

yo: 项目工程依赖目录和文件生成

grunt： 构建工具 

bower： 类似npm的包管理器

这一套其实是构建web应用的工具，开发命令行，使用其中的模板功能即可。

文档
http://yeoman.io/generator/Base.html
###构建模板需要准备：

* 继承 yeoman-generator 
* 数据源
* 模板文件

下面运行一个实例：

这个例子里，有两个文件 AppDelegate.h ,AppDelegate.m
AppDelegate.h 用来直接拷贝
AppDelegate.m 用来拷贝模板，并把数据填充进去。

首先准备文件结构，参看 tutorial-2-yeoman

>模板的文件夹规范为 generator-xxx

主入口文件 index.js

定义命令  node -n <name> -g <path>
接收参数，调用genios

```

#!/usr/bin/env node

var program = require('commander');
var genios = require('./genios/genios')

program
  .usage('-gen ')
  .version('0.0.1')
  .option('-n, --name <name> ', 'generate a ios project')
  .option('-g, --genios <path> ', 'generate a ios project')
  .parse(process.argv);


if (program.name && program.genios) {
   console.log('the path is :',program.genios)
   genios(program.genios,program.name)
}

```


genios.js

```
const path = require('path');
//引入yeoman的环境
const yeoman = require('yeoman-environment');



function init (projectDir,argsOrName){
//这是标准的 yeoman api，需要参数为 自定义的Generator 和nameSpace，nameSpace 随意取
    const env = yeoman.createEnv(
    );
    env.register(
        require.resolve(path.join(__dirname, '../generator-ios')),
        'ingage:app'
    );

    const args = Array.isArray(argsOrName)
        ? argsOrName
        : [argsOrName].concat(process.argv.slice(2));

//        console.log(process.argv,args)
     //把命令行收集到的参数传入到模板生成器中去
    const generator = env.create('ingage:app', {args: args});
    generator.destinationRoot(projectDir);
    //启动命令
    generator.run();
}


module.exports = init;
```

generator-ios


```
var yeoman = require('yeoman-generator');
var path = require('path');


module.exports = yeoman.Base.extend({

  constructor: function() {
    yeoman.Base.apply(this, arguments);
    this.argument('name',{type:String,required:true})
  },

  writing: function() { 
    this.fs.copy(
      this.templatePath(path.join('AppDelegate.h')),
      this.destinationPath(path.join('ios', 'AppDelegate.h'))
    );

    var templateVars = {name: this.name};
    this.fs.copyTpl(
      this.templatePath(path.join('AppDelegate.m')),
      this.destinationPath(path.join('ios', 'AppDelegate.m')),
      templateVars
    );
  },

  install: function() {

  },

  end:function(){
      var projectPath = path.resolve(this.destinationRoot(), 'ios', this.name);
      this.log('destinationPath：'+ projectPath)
  }

});
```

generator-ios 就是生成器的主函数了

1. 继承 yeoman.Base (新版本的yeoman的base 继承方式已经修改了，当前使用的yeoman-generator为0.24.1版本)

2.重写各个生命周期的函数


 * initializing - 初始化 (检查当前项目状态、获取配置文件内容等等)

*  prompting - 获取用户输入，实现与用户的交互 (通过this.prompt()调用)

*  configuring - 保存配置并配置整个项目 (比如创建 .editorconfig 文件和其他媒介文件)

*  default - 当定义的方法没有匹配任何基类方法的时候用到

*  writing - 根据自定义的规则写入具体的generator文件 (routes, controllers, etc)

*  conflicts - 内部冲突处理

*  install - 安装npm、bower等依赖的地方


所以我们重写了writing

并使用下面两个方法拷贝文件


 this.fs.copy(模板路径，目标路径)
 
 this.fs.copyTpl(模板路径，目标路径，替换文本)

输入的参数为
  this.templatePath  这个变量会自动去找 templates 文件夹
  
  
最后，执行命令，打开目标目录，查看生成的文件，<%=name> 已经被替换了，注意，替换过程会转义你的内容，如果你替换的内容是一段 html的话，不需要转义，模板里面定义改成 <%-name> 就可以了。
  


此时再去看RN的命令就简单多了，区别就在于它的模板是整个工程文件。

看一下RN的 local-cli/generator-ios/index.js

```
var templateVars = {name: this.name};
    // SomeApp/ios/SomeApp
    this.fs.copyTpl(
      this.templatePath(path.join('app', '**')),
      this.destinationPath(path.join('ios', this.name)),
      templateVars
    );

    // SomeApp/ios/SomeAppTests
    this.fs.copyTpl(
      this.templatePath(path.join('tests', 'Tests.m')),
      this.destinationPath(path.join('ios', this.name + 'Tests', this.name + 'Tests.m')),
      templateVars
    );
    this.fs.copy(
      this.templatePath(path.join('tests', 'Info.plist')),
      this.destinationPath(path.join('ios', this.name + 'Tests', 'Info.plist'))
    );

    // SomeApp/ios/SomeApp.xcodeproj
    this.fs.copyTpl(
      this.templatePath(path.join('xcodeproj', 'project.pbxproj')),
      this.destinationPath(path.join('ios', this.name + '.xcodeproj', 'project.pbxproj')),
      templateVars
    );
    this.fs.copyTpl(
      this.templatePath(path.join('xcodeproj', 'xcshareddata', 'xcschemes', '_xcscheme')),
      this.destinationPath(path.join('ios', this.name + '.xcodeproj', 'xcshareddata', 'xcschemes', this.name + '.xcscheme')),
      templateVars
    );
```


再看一下RN的 local-cli/generator-android/index.js

```
var templateParams = {
      package: this.options.package,
      name: this.name
    };
    if (!this.options.upgrade) {
      this.fs.copyTpl(
        this.templatePath(path.join('src', '**')),
        this.destinationPath('android'),
        templateParams
      );
      this.fs.copy(
        this.templatePath(path.join('bin', '**')),
        this.destinationPath('android')
      );
    } else {
      this.fs.copyTpl(
        this.templatePath(path.join('src', '*')),
        this.destinationPath('android'),
        templateParams
      );
      this.fs.copyTpl(
        this.templatePath(path.join('src', 'app', '*')),
        this.destinationPath(path.join('android', 'app')),
        templateParams
      );
    }

    var javaPath = path.join.apply(
      null,
      ['android', 'app', 'src', 'main', 'java'].concat(this.options.package.split('.'))
    );
    this.fs.copyTpl(
      this.templatePath(path.join('package', '**')),
      this.destinationPath(javaPath),
      templateParams
    );
```


两份模板文件，初始化了RN的HelloWorld 工程，和我刚写的
区别在于RN 拷贝的是整个工程目录。

所以说，在templates 文件夹下面放任何的文件都可以，只要把想替换的内容换成 <%= param>  就可以了。

###Commands.js

```
const documentedCommands = [
  require('./android/android'),
  require('./server/server'),
  require('./runIOS/runIOS'),
  require('./runAndroid/runAndroid'),
  require('./library/library'),
  require('./bundle/bundle'),
  require('./bundle/unbundle'),
  require('./link/link'),
  require('./link/unlink'),
  require('./install/install'),
  require('./install/uninstall'),
  require('./upgrade/upgrade'),
  require('./logAndroid/logAndroid'),
  require('./logIOS/logIOS'),
  require('./dependencies/dependencies'),
];

const commands: Array<Command> = [
  ...documentedCommands,
  ...undocumentedCommands,
  ...getUserCommands(),
];
module.exports = commands;
```

RN的所有命令注册在了Commands.js 下

然后在 cliEntry.js 里面使用，循环注册给了Commander


```

function run() {
  const setupEnvScript = /^win/.test(process.platform)
    ? 'setup_env.bat'
    : 'setup_env.sh';

  childProcess.execFileSync(path.join(__dirname, setupEnvScript));

  const config = getCliConfig();
  commands.forEach(cmd => addCommand(cmd, config));

  commander.parse(process.argv);

  const isValidCommand = commands.find(cmd => cmd.name.split(' ')[0] === process.argv[2]);

  if (!isValidCommand) {
    printUnknownCommand(process.argv[2]);
    return;
  }

  if (!commander.args.length) {
    commander.help();
  }
}

```



这就属于对项目结构划分的范畴了，增加一个命令，增加一个文件夹，然后注册进来，降低主文件耦合性。


最新版本的0.39去掉了yeoman的方式，采用了纯查找字符串替换的方式(copyProjectTemplateAndReplace.js)，对于使用RN的人来讲没有任何影响，比较它的模板文件里面只有一个“HelloWorld” 字符串，但是yeoman的思想就再也不能通过它来传播了。

```
export default class HelloWorld extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.ios.js
        </Text>
        <Text style={styles.instructions}>
          Press Cmd+R to reload,{'\n'}
          Cmd+D or shake for dev menu
        </Text>
      </View>
    );
  }
}

```
```
 copyAndReplace(
      absoluteSrcFilePath,
      path.resolve(destPath, relativeRenamedPath),
      {
        'HelloWorld': newProjectName,
        'helloworld': newProjectName.toLowerCase(),
      },
      contentChangedCallback,
    );
```

