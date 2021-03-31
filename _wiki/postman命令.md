---
title: postman命令
categories:
- 常用命令
---

* 目录
{:toc}

#### 1、录制接口

接口调用方希望测试业务逻辑时，用不着Fiddler/Charles抓包再往里面一个个填这么麻烦，开启Postman的代理（默认5555端口），浏览器/手机设好对应的IP和端口就行。
支持正则表达式过滤URL，建议排除掉静态资源、流量统计站和别的后台进程时不时请求的网站
baidu|google|microsoft|github|qq\.com|.*\.(html?|css|js|png|jpe?g|webp|ico)$
可以设置保存的位置：
1. 如果所有操作步骤都要做接口测试，建议直接保存到目标集合
2. 如果一大堆操作里只取其中一个到几个接口，建议保存到历史记录，挑出想要的另存到目标集合，不用浪费时间删

#### 2、Collection Runner

* 适合一次运行/调试多个用例
* 适合开发自测接口性能（例如循环个10000次），至少响应时间和稳定性都过关了才交给别人调用
* 跟Newman一样的运行环境，跟Newman一样默认不保存环境变量，跟Newman一样可以载入数据文件，适合用界面调通了用例再放到命令行做持续集成

【特别注意】
各个集合之间、同一集合各个文件夹之间、同一集合放文件夹和不放文件夹的用例之间，不能有任何依赖关系
Postman/Newman是用JS写的，批量执行的最小粒度是文件夹
单线程异步的性质决定了文件夹的执行顺序跟看上去的排列顺序没有任何关系
只有同一文件夹内的用例，或直接放集合下的用例，执行的顺序跟看上去的顺序一样
每个集合测1个方面，集合内每个文件夹测1个完全独立的子场景，此外

* 目前集合下只支持1级文件夹，不推荐场景搞太复杂，用例管理起来有难度
* 目前集合和文件夹只能按字母顺序排序，用例可以拖动排序
* 目前搜索不能搜到URL地址，建议用例名写成 /path/to/api 接口描述，方便看和搜索
* 如果打算做自动化和持续集成，集合、文件夹的名字建议英文无空格，方便命令行调用

#### 3、流程控制

只有在collection runner或Newman里才生效（在普通界面你选哪个发送就是哪个）

1. 假设2个用例的顺序为：用例A用例B如果希望执行顺序为： 用例A -> 用例B -> 用例A，又不想复制一份用例A，那么，用例A的Tests里写if (xxx) postman.setNextRequest('null');  终止执行
2. 用例B的Tests里写postman.setNextRequest('用例A');【注意】如果不设终止条件，用例A执行完到用例B，用例B执行完又指向用例A，会构成死循环
3. PS：postman是Postman提供的全局对象

#### 4、断言

在主界面可以靠肉眼看返回结果，但在collection runner/Newman里如果不加断言，跑完没法知道是成功还是失败</br>
断言写在Tests标签页里，上手可以参考文档和界面右边的代码模板：</br>
```javascript
tests['Status code is 200'] = responseCode.code === 200;  
```
推荐用全等 ===，确保类型和值都一致 
```javascript
tests['Response time is less than 200ms'] = responseTime < 200;
tests['Body matches string'] = responseBody.has('xxx'); 
```
只要有指定关键字就行，在哪、有几个等都不管
```javascript
tests['Content-Type is present']=postman.getResponseHeader('Content-Type') || false;

``` 
Postman的断言实际上就是往全局对象 tests 添加键值对，它的key会显示在报告界面上，value是可以解析为boolean的表达式，如果得出的值是true，在报告里显示为成功，false失败
【变通】用总是为真的断言来显示信息
```javascript
tests[`[INFO] Request params: ${JSON.stringify(request.data)}`] = true;  
```
显示所有请求参数（在自动化测试里很有用）
```javascript
tests[`跑第${iteration + 1}次`] = true;
```
* 用在runner里循环很多次时，迭代次数iteration是Postman提供的全局变量，从0开始
* PS：request是Postman提供的全局对象
* responseCode（对象）、responseTime（数字）、responseBody（字符串）目前是Postman收到服务器返回内容才声明的变量
* 【注意】如果你在做自动化测试，目前在接口超时没返回时：responseCode、responseTime、responseBody都没定义，使用时会导致脚本出错，判断是否超时没返回的只能靠header
* request.data里的变量在超时时不解析，很容易让人误会请求参数传错了，建议此时不显示这行
* 关于'use strict';因为Postman提供了不少全局变量，写了'use strict';会在用到这些变量的地方出警告提示，如果平时有养成好习惯，不用容易出错的写法，不写它也没关系

#### 5、提取接口返回值

1. 返回JSON时
```javascript
let json = JSON.parse(responseBody);  // responseBody是包含整个返回内容的字符串
```
1) 提取某字段的值：
```javascript
let foobar = json.foo.bar[0].foobar;  // 假设结构为 {"foo": {"bar": [{"foobar": 1}, {"baz": 2}]}}
```
2) 想用在自动化测试可以多写点：
```javascript
let json;
try {
    json = JSON.parse(responseBody);
} catch(err) {
    tests['Expect response body to be valid JSON'] = false;
    tests[`Response body: $ {
        responseBody
    }`] = true;
    console.error(err);
}
```
2. 返回HTML时
A. 用正则表达式匹配
```javascript
let foo = responseBody.match(/foo/g);  // g 全局 i 不分大小写 m 多行tests['blahblahblah'] = foo[0] === 'bar';
```
正则里包含变量时：
```javascript
let foo = 'xxx';let bar = responseBody.match(new RegExp(`^${foo}.*, 'g');
```
B. 用CheerioJS库：
```javascript
let html = cheerio(responseBody);
let titleText = html.find('title').text();  // 取
```
#### 6、设置环境变量

Postman的环境变量分为 environment 和 global 2种，实际上就是environment、globals这2个全局的对象（字典）里的内容。它们的key作为变量名，value作为变量的值。

environment满足99%的需要，平时只用它就够了，global留到后文讲

* 作用域为整个集合
* 能创建多个environment文件，方便切换不同测试环境
{% raw %}
在地址栏、header、请求参数、外部数据文件里，用 {{变量名}} 获取环境变量的值。
如：{{URL}}/path/to/api（更常见的${}在JS的ES6语法里被占用了，Postman只能选这么个奇怪写法）
{% endraw %}   
在Pre-request Script和Tests的代码里略有不同：官方提供的方法
* 设置 postman.setEnvironmentVariable('variableKey', value);//注意：通过菜单或以上方法设置的环境变量，值会转成字符串，取的时候要转换
* 获取let foo = postman.getEnvironmentVariable('variableKey');//字符串数字转数字：Number(foo)
* 字符串'true' 'false'转布尔值：Boolean(foo) 或 !!foo
* 更新就是再设置一次同名的环境变量，换别的值
* 清除环境变量postman.clearEnvironmentVariable('variableKey');//懒人版
* 既然知道实际上是操作 environment 对象，如果你有JS基础，可以直接：添加keyenvironment.variableKey = 12345;  少打字，取出时也不用转换类型
* 获取let foo = environment.variableKey;
* 清除delete environment.variableKey;
* obj.foo = undefined; 的写法无效，Postman把undefined转成了字符串……
* 如果你非要跟自己过不去，用了变量名不允许的字符做key（比如有空格），只能写成 environment['variableKey']
* 只要没跟自己过不去，可以用ES6的对象解构语法一次取多个：let {key1, key2, key3} = environment;
* 注意】在Postman主界面运行过后，通过代码设置的环境变量会存到IndexedDB，跟在菜单里设置一样，用例跑完不消失，但在collection runner或Newman跑则是默认不保存，跑完就消失，做自动化测试时要注意

#### 7、动态请求参数

在runner里循环发n次请求/做自动化测试时，有些接口不适合写死参数，Postman有以下内建变量，适合一次性使用：
{% raw %}
```javascript
guid  // 生成GUID，使用双花括号
timestamp  // 当前时间戳
randomInt  // 0-1000的随机整数
```
{% raw %}
* 参数依赖上一个请求的返回：上个请求的Tests里提取参数存环境变量，这个请求里用{{变量名}}取值
* 参数每次都不同，但之后的断言或别的请求里可能还要用：在Pre-request Script里写代码处理，存为环境变量，参数里用{{变量名}}取值
{% endraw %}
例如
```javascript
const randomInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;  // 随机整数
const getRandomValue = list => list[randomInt(0, list.length - 1)];  // 随机选项
environment.randomMobile = `18${randomInt(100000000, 999999999)}`;// 随机手机
environment.randomDevice = getRandomValue(['ios', 'android']);// 随机设备名
```
Postman目前没有很方便的重用代码的手段，编辑框也不是IDE，没智能提示，尽量别整那么复杂

#### 8、调试

cmd + alt + c（Windows ctrl + alt + c）打开Postman控制台，可以看请求和响应内容
用console.log()打印，到控制台看
或
```javascript
tests['这里拼出你想看的字符串'] = true //在界面报告看断言
```
mock http://blog.getpostman.com/2016/01/26/using-a-mocking-service-to-create-postman-collections/

#### 9、global

不做自动化测试可以跳过这段
跟environment几乎完全一样，在地址栏、header、请求参数、外部数据文件里也是{% raw %}{{}}{% endraw %} 调用，除了：
* 只有1个global文件
* 菜单藏得较深，在生成的API文档里也不解析，决定了它不适合做参数化
* environment和global同名时，优先用environment

global只建议用在1种场景：定义公共函数，先正常地写好函数，再用在线压缩工具压成一行，在菜单里选 Bulk Edit，以每行一对 key:value 的形式编辑，变量名做key，函数做value
```javascript
assertNotTimeout: var hasResponse = postman.getResponseHeader('Content-Type') ? true:false;
if (!hasResponse) tests['服务端在超时前没返回任何数据，请检查相关服务、网络或反向代理设置（以下跳过其他断言）'] = false;
logParams: if (hasResponse) tests[` [INFO]请求参数（超时没返回时不解析）：$ {
    JSON.stringify(request.data)
}`] = true;
getResponseJson: try {
    if (hasResponse) var json = JSON.parse(responseBody);
} catch(err) {
    tests['服务端没返回合法的JSON格式，请检查相关服务、网络或反向代理设置（以下跳过其他断言）'] = false;
    tests[` [INFO]返回：$ {
        responseBody
    }`] = true;
    console.error(err);
}
assertType: var assertType = (name, value, type) = >{
    let isType = (type === 'array') ? Array.isArray(value) : typeof value === type;
    tests[`$ {
        name
    }为$ {
        type
    }（实际值：$ {
        value
    }）`] = isType;
};
assertEqual: var assertEqual = (name, actual, expected) = >{
    tests[`$ {
        name
    }等于$ {
        expected
    }（实际值：$ {
        actual
    }）`] = actual === expected;
};
assertNotEqual: var assertNotEqual = (name, actual, expected) = >{
    tests[`$ {
        name
    }不等于$ {
        expected
    }（实际值：$ {
        actual
    }）`] = actual !== expected;
};
```
* 注意在这里定义变量只有 var 的作用域够大，用 let 或 const 的话eval后就销毁了
* 假设返回 {"name":"张三","userType":1,"settings":[]}，在Tests里一上来就写
```javascript
eval(globals.assertNotTimeout);  
```
* 判断是否超时（通过有没有Content-Type请求头），超时则断言失败
```javascript
eval(globals.logParams);  
```
* 如果不超时，显示发出的请求参数
```javascript
eval(globals.getResponseJson);  
```
* 如果不超时，解析返回的JSON对象，赋给json变量，返回值不合法则断言失败
* 下面定义了3个公共函数，免得每次断言都要写一大串
```javascript
eval(globals.assertType);
eval(globals.assertEqual);
eval(globals.assertNotEqual);
// 上面3个基本满足日常使用需要
```
```javascript
if (json) {
    assertType('用户名', json.name, 'string');// 在报告中显示为： '用户名为string，（实际值：张三）'
    assertType('设置', json.settings, 'array');// JS里其实没有array类型（数组是object），这里做了处理，让报告更好懂  
    assertEqual('用户类型', json.userType, 1);//显示为：'用户类型等于1，（实际值：1）'
    assertNotEqual('用户类型', json.userType, 2);//显示为：'用户类型不等于2，（实际值：1）'
}
```
* 在官方给出更方便的重用代码的方法前，这是除了复制粘贴外的重用方法，如果不做自动化测试，且断言写得很简单，不建议这么搞  
* 如果不幸跳了自动化的坑，通常一个项目会有100~200个接口要做自动化测试，要仔细比较哪种方法成本更高
* 定义函数前要仔细考虑好，万一中途要改参数和返回值，已经写好的n份也得改……
* 建议定义的公共函数不超过个位数，并保留好没压缩的版本，不然别人没法接手

http://blog.getpostman.com/2015/09/29/writing-a-behaviour-driven-api-testing-environment-within-postman/

如果确实要在代码里设global，官方的：
```javascript
 postman.setGlobalVariable('variableKey', value);  
```
同样存成字符串
```javascript
let bar = postman.getGlobalVariable('variableKey');
postman.clearGlobalVariable('variableKey');//清除该变量
```

#### 10、数据文件

在collection runner或命令行的Newman里可以加载数据文件  
##### JSON格式
1. 文件里有且只有1个数组，每个对象算1条用例（在Postman里的全局变量叫做data）
2. key作为变量名，value作为变量的值
3. 文件里依然可以用{% raw %}{{}}{% endraw %} 拿到环境变量，注意不要把自己绕进去：
4. 如果是Pre-request Script里生成的环境变量，直接写进请求参数，不用经这里
```json
[  {"mobile": "17000000001", "pwd": "123456"},  {"mobile": "17000000002", "pwd": "654321"},  {"mobile": "17000000003", "pwd": ""},  {"mobile": "{{ADMIN_MOBILE}}", "pwd": "{{ADMIN_PWD}}"}]
```
显然，这是json文件，并不能在里面写代码（除非你蛋疼在value里写字符串然后在用例里eval）
1. 用例的请求参数里依然用 {{key}} 拿到数据文件里的值，代码里则是 data.key
2. 如果key跟environment/globals里的key重名，这里的 > environment > globals 
##### CSV格式
1. 第1行变量名，下面是值，每行1条用例，没有空格
2. 没JSON格式的数据文件灵活 
```
mobile,pwd
17000000001,123456
17000000002,654321
17000000003,322323
```
    
#### 注意

谨慎使用。这东西增加了调试和定位问题的复杂性，也就大大增加了维护成本，而它带来的收益并不明显：
1. 针对单个接口的简单压力测试
Postman不是正经的压测工具，既然选择了它就是图简单方便，
像JMeter那样用CSV文件做数据源的意义不大，还得另外写程序/脚本生成这样的文件，时间上不划算
直接用代码生成数据就好，不差那一两毫秒
2. 数据驱动测试：一条用例测多种参数组合，包括合法和非法值，避免复制粘贴n条略有不同的
很诱人

但是：
* 产品设计时有考虑异常情况吗？
* 需求来源是否统一？
* 需求是否足够稳定？
* 整个项目有统一的异常处理规范还是看开发个人习惯？
如果不确定有些输入要不要/怎么处理，意味着改动可能会非常大
今天非法，明天变合法，后天又变非法  
如果冒烟用例用在持续集成，有测试不通过会阻止发布，会严重干扰正常发版，也影响大家对自动化测试的信心

此外：
* 内部的基础组件为了不同项目通用，可能会允许看起来相当没道理的值
* 不对外暴露的接口为了性能，可能会有意去掉所有校验
* 不要因为所谓“测试思维”，就在不了解的情况下为了测试而测试
* 这种力气留到探索性测试和安全测试，自动化测试还是要讲求稳定和省事
* 为了能匹配正常和异常情况（和具体哪一种异常情况），断言必须写得比平时复杂
* 然后你会希望把断言条件也写进数据文件里，一种格式，eval后到处通用
* 然后数据文件的格式会变得远比上面的示例复杂
* 然后你会准备一键生成模板的脚本，批量修改的脚本，封装Newman的脚本，一个框架的雏形……别问我怎么知道的，然后你回过头发现，一开始不用数据文件不就省事多了？！  

原文：http://f.dataguru.cn/thread-866724-1-1.html

仅整理格式