# 1.模块
+	随着程序代码的越来越多,我们的维护会变得越来越麻烦,所以为了编写可以维护的代码,我们把
很多函数分组,分别放到不同的文件中.

+	使用模块不仅可以将模块复用,而且还解决了一个很大的问题就是函数名和变量名冲突的问题.

```
hello.js

var s = 'hello';

function greet (name) {
	console.log(s + ',' + name + '!');
}

module.exports = greet;
```

+	比如这个文件叫做 `hello.js` 那么这个模块的名字就这叫做 `hello` . 然后嘴关键的一步就是
最后所写的 `module.exports = greet` , 这里实现的就是将`greet`函数暴露给外部,可以让外部去
`require` .直接得到这个函数.而且 在 `hello.js` 中定义的变量全部在外部成为局部变量.

+	在使用这个模块的时候,我们需要如下这样坐着

```
mian.js

var greet = require('./hello.js');

var s = 'Michael';

greet(s);
```

+	可以注意到上面我们也直接使用了`var s` 这个变量,但是输出完全没有问题. 
+	在我又发现如果使用`./hello.js` 和 `./hello`都可以引用的这个模块.但是必须使用相对路径去引用!
比如 `hello` 这样直接应用就会报错.

### ComonJS规范

+	这个规范的定义很简单,主要分为 模块引用, 模块定义 和模块标示 3个部分.
* **模块引用** `var math = require('math')`; 在Node中,存在`require()`方法 ,这个方法接受模块标示,
并此引入一个模块的API到当前的上下文中.
* **模块定义** 在模块中,上下文提供`require()`方法来引入外部模块.对应引入的功能,上下文又提供了一个
`exports`对象用于导出当前模块的方法或者变量.并且它是唯一的导出的出口.在模块中还有一个对象叫
`module`,它是只模块本身.而`exports`是`module`的属性.
+	在Node中,一个文件就是一个模块,通过将方法挂在在`exports`对象上作为属性即可定义导出的方法!

+	这种模块加载机制成为CommonJS规范,这个规范下,每个.js文件都是一个模块,他们内部各自使用的变量名
和函数名都互不冲突.
+	想暴露在外的变量(函数) , 可以使用`module.exports = variable` ,一个模块要引用其他模块的变量,
用 `var ref = require("module_name")` ;就拿到模块的变量;
+	`module.exports` 如果在模块中出现两次, 并不会报错,而是用最后出现的位置的变量.那么这个到底是怎么
实现的呢?

**总结** 每个文件都是一个模块,这个模块本身有三个对象, `module`,`exports`,`require`对象.

### 深入理解模块原理

+	首先,在浏览器中,大量使用全局变量可不好使很好地事情,比如,如果我们在a.js中定义了一个全局变量`s`,那么
b.js中的对`s`的复制就会改变`a.js`中的`s`的运行逻辑;
+	`node.js` 是如何解决这个问题的呢? 很简单,其实因为javascript支持闭包,那么如果我们把一段javascript代码
用闭包包装起来,那么javascript里面所定义的所有变量不就会变成局部变量了嘛?
+	比如我们编写`hello.js`的代码如下:
	
```
	var s = 'hello';
	var name ='world';

	console.log(s+''+name+'!');
```
Node.js 加载了`hello.js`后,它可以把代码包装一下,变成这样执行:
```
(function(){
	var s ='hello';
	var name = 'world';
	console.log(s+''+name+'!');
})();
```
就是这样,全局变量变成了局部变量!

+	那么我们就要想到 `module.exports`怎么实现的呢? 首先我想到的肯定是不是有一个 `module`对象,这个
对象是一个全局对象,之后在这个对象中有一个 `exports`属性,用来储存变量;然后我去查node.js的官方文档,果然
在全局变量中有一个Module对象! 具体实现如下:

```
var module = {
	id:'hello',
	exports :{}
};

var load = function(module){
	//读取hello.js的代码
	//即定义一个greet函数
	function greet(name){
		console.log('hello',+name+'!');
	}
	//将传进来的module中的exports属性引用greet
	module.exports = greet;
	//hello.js 代码结束
	return module.exports;
};
var exported = load(module);

save(module,exported);
```

+	通过把`module`传递给`load()`函数,就顺利的把一个变量传递给了Node环境,Node会把`module`变量保存在
某个地方.
+	由于node保存了所有导入的`module`,当我们使用`require()`获取module时候,Node就会找到对应的`module`,
把这个`module`的`exports`变量返回.

### 上面是简单版的理解,这里是正正的模块编译的步骤

+	在node中,每个文件模块都是一个对象,它的定义如下:

```
function Module(id,parent){
	this.id = id;
	this.exports = {};
	this.parent = parent;
	if(parent&&parent.children){
		parent.children.push(this);
	}

	thisl.filename=null;
	this.loaded = false;
	this.children = [];
}
```

+	执行和编译是引入文件模块的最后一个阶段,定位到具体的文件后,Node会新建一个模块对象,然后根据路径
载入并编译.对于不同扩展名,其载入的方法也有所不同;

* .js文件,通过 fs 模块同步读取文件后编译执行.
* .node文件,这是用C/C++编写的扩展文件,通过dlopen()方法加载最后编译生成的文件.
* .json文件,通过fs模块同步读取后,用`JSON.parse()`方法加载最后编译生成的文件.
* 其余的文件都背当成js文件来执行.

+	根据不同的文件扩展名,调用的读取方式也不同,如果`.json`文件的调用如下:

```
Module._extensions['.json'] = function(module,filename){
	var content = nativeModule.require('fs').readFileSync(filename,'utf8');
	try{
	module.exports = JSON.parse(stripBOM(content));
	} catch (err) {
	err.message = fliename+''+err.message;
	throw err;
	}
}

```

+	因为Module._extensions 会被复制给require()的`extensions`属性,所以可以通过
`console.log(require.extensions)`来得到系统中已有的扩展加载方式;


### module.exports   和     exports

+	在Node环境中,这两种方法都可以在模块中输出变量:

``` 对module.exports 赋值
	function hello(){
	console.log('hello');
	}

	function greet(name){
		console.log('hello'+name+'!');
	}

	module.export = {
		hello:hello,
		greet:greet
	};

	直接使用exports

	exports.hello = hello;
	exports.greet = greet;

	但是不可以直接对exports 赋值;

```

+	从上理解可以解释为什么可以遮掩工作 

```
var module = {
	id:'',
	exports:{}
}

在默认情况下 module.exports 和 exports 实际上是同一个变量.所有实际上我们可以写成
exports.foo = function(){}
exports.bar = function(){}

也可以携程
module.exprots.foo = function(){}
module.exprots.bar = function(){}

```
### javascript模块的编译

+	我们可以发现每个模块文件中存在着`require`,`exports`,`module`这3个变量,甚至每个模块中还有
`__dirname`,`__filename`这两个变量的存在.用户也没有定义它们,那么他们是从何而来呢?
+	我们如果把定义变量放在浏览器中,那么就会出现污染全局变量的情况.
+	事实上node 对获取的javascript文件内容进行了头尾包装. 在头部==> 添加了 
`(function(exports,require,module,__filename, __dirname)){\n,在尾部添加了\n});`
+	所以一个正常的javascript文件被包装成:

```
(function(exports,require,module,__filename, __dirname){
	var math = require('math');
	exports.area = function(radius) {
		return Math.Pi * radius * radius;
	};
});
```


## 基本模块

### global , 在浏览器中 ,有一个全局对象叫做window对象, 而在node.js中,也唯一的全局对象,就是global
+	可以直接在Node交互模式中输入` global.console` .

### process,也是node.js提供的一个对象,它带便当前node.js进程,通过process对象可以拿到许多有用的信息:

```
	> process === global.process;
	true
	> process.version;
	'v5.2.0'
	> process.platform;
	'darwin'
	> process.arch;
	'x64'
	> process.cwd(); //返回当前工作目录
	'/Users/michael'
	> process.chdir('/private/tmp'); // 切换当前工作目录
	undefined
	> process.cwd();
	'/private/tmp'
```

+	javascript 程序师又时间驱动的单线程模型,node.js也不例外,它不断响应javascript函数,直到没有任何
响应时间的函数执行时,node.js就退出了;
+	如果我们想要在下一次时间响应中执行代码,可以调用process.nextTick();

```
process.nextTick(function(){
	console.log('nextTick callback');
});
console.log('nextTick is set');
```
输出 

```
nextTick was set!
nextTick callback!
```

+	Node.js的进程本事的时间就由`process`对象来处理.如果我们响应`exit`事件,就可以在进程即将退出
执行某一个回调函数:

```
process.on('exit',function(code){
	console.log('about to exit with code:'+code);
});
```

### 判断javascript执行环境
+	很多时候我们的javascript执行环境需要确定下来,所以我们要做的就是

```
if(typeof(window) ==='undefined'){
	console.log('node.js');
}else{
	console.log('browser');
}
```



## 3 异步I/O
### 3.1 为什么要使用异步?
#### 3.1.1 用户体验

+	如下,如果不采用异步操作的话

```
//消费时间为M
getData('from_db');
//消费时间为N
getData('from_remoteapi');

但是如果使用异步操作的话

getData('from_db',function(result){
	
});
getData('from_remoteapi',function(result){
	
})
```

+	还有一点就是目前的网站或者应用不断膨胀,数据将会分布到多台服务器上,分布式将是常态,
因此 上述例子中的 M 和  N 的值会线性增长. 因此 I/O是昂贵的.





















