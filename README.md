## 手把手实现一个简单的前端脚手架

### 一、什么是前端脚手架？

大家在实际开发项目中，有没有这样的困惑，就是每次起一个新的项目，都要从旧有的项目copy，然后改名、删除内容，（而且还会担心删错内容），留下基本框架，最后还有npm install。当我们每次新起时，我们就得重复一遍上述的工作，甚至更繁琐的流程。作为一名合格的程序猿，最基本的品质就是应该让重复的事情自动化，所以我们便有了前端脚手架。脚手架工具就是帮我们自动化完成某项任务，简化我们开发成本的工具。至于这个任务是什么，完全可以自定义，但是对于前端而言，脚手架工具的主要任务是用于快速生成并初始化项目文件。有了脚手架，我们只用在控制台简单的输入一行命令，就可以快速的生成一个项目目录。

所以我们可以理解为，前端脚手架可以分为两部分：脚手架模板 + 控制代码。
控制代码核心部分就是：
1）文件的增删改查
2）控制台命令的读取与脚本执行状态的log输出
而模板文件一般是要存储在云端的，所以应该是和控制代码分开打包的，这样模板改变后能实时更新。

### 二、常用的脚手架工具

实际开发过程中，我们会接触到许多脚手架，比如vue-cli,create-react-app等等，许多框架都有自己的脚手架工具，除此之外还有一个出名的脚手架工具就是yoeman。
yeoman 发布于 2012 年，现在看来依然不过时，没有因为框架、工具的迭代而被人弃用，并且它的应用场景不局限于前端项目。 yeoman 工具本身不提供脚手架模版，它是一个脚手架模版的管家和执行者。脚手架模版对应于 yeoman 定义的 generator，yeoman 鼓励大家开发 generator 发布到它的平台并且通过 npm 可以单独安装。


### 三、实现自己的脚手架
那么如何自己动手实现一个脚手架呢，这里我们以最基本的项目初始化脚手架为例来分析。让我们理一理思路。
首先这个脚手架应具备哪些基本功能：
- 读取控制台命令，并执行相应功能代码
- 功能实现：读取模板，创建目录，写入模板
- 打印日志

实现这些功能我们需要依赖哪些基础库呢：
- Commander.js,是一个 Node.js 命令行参数解析的库，是原生 process.argv 增强工具。
- Chalk,是命令行输出的“彩色粉笔”库，可以利用它在终端输出的提示字符串上添加不同的颜色，来区分提示信息。
- ora,实现Node.js 命令行环境的 loading效果， 和显示各种状态的图标等
- Inquirer.js 相当于浏览器提供的交互窗口(alert、confirm、prompt)，可以用它来实现命令行与用户输入的交互。
- fs-extra,封装了原生 fs 方法,提供 promise 支持，并且扩展了 fs 不支持的方法，例如删除文件夹。
- Metalsmith, 从源目录获取源文件，并将处理后的文件写入目标目录.可以通过添加一些插件对文件进行写入前的处理，如重命名、合并等。

理清这些后就让我们开始开发我们自己的脚手架工具吧


#### 第一步：创建项目、模板上传
脚手架文件夹项目结构如下：
```
  |__ bin
    |__ my
    |__ my-init
  |__ lib
    |__ init.js
  |__ node_modules
  |__ package.json
```
其中bin中为输入命令对应的执行入口，lib里则存放具体的功能实现代码或者一些工具函数
模板文件，我们则是发布成npm包，需要时再下载，目录结构如下：
```
  |__ template
  |__ node_modules
  |__ package.json
```
其中template里为项目的模板文件。模板文件创建好后需要发布个npm包，如何发布npm包这里就不多说了。

#### 第二步：控制台命令处理——执行入口
首先,我们在package.json里添加：
```
  "bin": {
    "my": "bin/my"
  }
```
然后，我们在bin文件夹下添加对应的文件，这里的my我们可以理解为入口文件，其他所有命令的分发与处理都在这个文件里实现。文件my中代码如下：
```
#!/usr/bin/env node
// 指定脚本的执行环境
const program = require('commander')

program
  .version(require('../package').version)  // 定义当前版本
  .usage('<command> [options]')  // 定义使用方法
  .command('init <template-name> [project-name]', '根据指定模板生成项目')  // 定义命令
  .description('Generate a new project')
  .parse(process.argv)
```
使用node运行这个文件，看到输出如下，证明入口文件已经编写完成了。
```
$ node bin/my

Usage: my <command> [options]

Generate a new project

Options:
  -V, --version                       output the version number
  -h, --help                          output usage information

Commands:
  init <template-name> [project-name]  用一个模板生成项目
  help [cmd]                          display help for [cmd]

```
显然每次加上node执行命令很麻烦，在本地调试时，可以先执行
```
npm link
```
然后我们就可以直接使用我们定义的命令了，如下;
```
$ my
```
上面说到过my文件只是命令分发的入口文件，真正实现对命令的响应与处理我们继续实现。
这里我们定义了初始化项目的命令为my init,我们在控制台输入my init后，命中了init命令，此时会运行bin文件夹下的my-init文件，我们在里面对输入命令进行处理
```
const program = require('commander');
program
  .usage('<template-name> [project-name]')
  .parse(process.argv)

```
接下来，便可获得输入参数。处理没有参数输入的情况：
```
program.on('--help', () => {
  console.log(chalk.yellow('#使用方法'))
  console.log('# my init <template-name> [project-name]')
})

if (!program.args.length) {
  program.help()
  return;
}
```
#### 第三步，功能实现——下载模板、生成项目
响应命令后，便可以进入我们的核心代码了，也就是下载模板、生成项目的实现
首先是判断需要新建的项目当前是否已经存在，如果存在则需要询问用户是否需要覆盖以继续。
这里会用到`fse.existsSync()`来判断文件夹是否已经存在，利用`inquirer`来实现与用户的交互。
```
if(!template){
  console.log(chalk.red('请输入模板名称'))
  return;
}

if(!proj){
  console.log(chalk.red('请输入项目名称'))
  return;
}

const projPath = path.join(process.cwd(),'/',proj)
getProjName(proj);


function getProjName (proj){
  if(fse.existsSync(proj)){
    inquirer.prompt([{
      type: 'confirm',
      message: '该目录已存在，是否覆盖？',
      name: 'ok'
    }],(ans)=>{
      if(ans.ok){
        fse.removeSync(proj)
        run()
      }else{
        process.exit(1);
      }
    })
  }else{
    run();
  }
}
```
拿到目标项目路径后，就可以下载模板文件了，其实就是安装npm包，然后利用`Metalsmith`实现文件的拷贝。
```
function run(){
  //下载模板
  console.log(chalk.green(`使用模板：${template}创建项目`));
  const  spinner = ora('正在下载模板');
  spinner.start();
  cprocess.exec(`tnpm i ${template}`,(err, data)=>{
    spinner.stop();
    if(err){
      console.log(chalk.red('下载模板失败', err,message));
    }
    console.log(chalk.green('模板下载完毕！'));
    console.log(chalk.green('开始创建项目'));
    const tplPath = `${process.cwd()}/node_modules/${template}`;
    
    //复制模板
    Metalsmith(`${tplPath}/template`)
    .source('.')
    .destination(projPath)
    .build((err)=>{
      if(err){
        console.log(chalk.red('创建项目失败'),err)
      }else{
        console.log(chalk.green('创建项目成功！'));
      }
    })
  })  
}
```
至此最基本的新建项目功能已经实现，可以运行`my init template projPath`来试一试吧。

### 四、补充
1、这里只实现了最基础的功能，主要是帮助理清开发前端脚手架的思路与流程，更多定制功能完全可以依次类推，这里的代码也可以更加完善，兼容更多情况
2、这里使用的模板是纯静态模板，没有考虑用户输入的动态数据的写入。上面有说到过
Metalsmith支持添加插件在文件写入前对其进行处理，所以这里可以利用Handlebars.js(一个简单高效的语义化模板构建引擎)来解析模板。

### 参考文章
KM
