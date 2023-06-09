初始化data里面的属性为每一个属性添加get，set的方法监听
为数组重写方法但是继承与Array，最终调用还是Array里面的方法，只是于自己重写的ArrayMethods去触发调用Array里的方法

1: npm init -y 初始化项目
2: 安装rollup环境
    npm install @babel/preset-env @babel/core rollup rollup-plugin-babel rollup-plugin-serve cross-env -D



03-soundCode
//2 ast 语法树变成 render 函数  （1） ast 语法树变成 字符串  （2）字符串变成函数 
//3 将render 字符串变成 函数



# 1.初始化new Vue({})里面的属性

1. 创建 **this._init()** 方法进行初始化，通过**initMixin**传入Vue对象进行初始化，在里面定义 **_init() ** 方法挂载到Vue原型上**Vue.prototype._init = function() {}**

   ```js
   function Vue(options) {
     // 初始化
     this._init(options);
   }
   initMixin(Vue);
   ```

   

2. 将通过**new VUE({})** 传入的对象绑定到原型**$options**上，然后通过**initState**方法进行对**options**对象初始化

   ```tex
   (1) initData() 传入data函数或对象，循环遍历data里面的属性通过proxy代理对象对每一个属性添加get和set的方法 vm => data里面的所有传入的new Vue({})，_data自己定义的属性，key是_data里面某一个值
   Object.defineProperty(对象，某一个属性，{
       get() {}
   	set() {}                                                                           
   })
   ```

   ```js
   //vue2 对data初始化
   function initData(vm) {
     //  console.log( vm) // （1） 对象  （2） 函数
     let data = vm.$options.data;
     data = vm._data = typeof data === "function" ? data.call(vm) : data; //注意  this
     //data数据进行劫持
     //将data 上的所有属性代理到 实例上 vm  {a:1,b:2}
     for (let key in data) {
       proxy(vm, "_data", key);
     }
     observer(data);
   }
   function proxy(vm, source, key) {
     Object.defineProperty(vm, key, {
       get() {
         return vm[source][key];
       },
       set(newValue) {
         vm[source][key] = newValue;
       },
     });
   }
   ```

   

3. 对初始化的时候注意data里面的属性值是否是对象还是基本类型，基本类型就直接返回，如果是对象还要继续添加**get**和**set**的方法进行*劫持*，通过**Observer**类来创建

   ```js
   observer(data) {
     // 1.判断 data里面的数据类型
     if (typeof data != "object" || data == null) {
       return data;
     }
     // 2.对象通过一个类
     return new Observer(data);
   }
   ```

4. **Observer**类使用构造器通过**Array.isArray**方法来判断是否是数组

   - 对象  =》通过**defineReactive***进行劫持*添加**get**和**set**的方法。

     注意值里面也有可能是对象要进行 ***递归调用***

     ```js
     class Observer {
       constructor(value) {
         // 判断是否是数组
         if (Array.isArray(value)) {
           value.__proto__ = ArrayMethods;
         } else {
           this.walk(value);
         }
       }
     
       walk(data) {
         let keys = Object.keys(data);
         for (let i = 0; i < keys.length; i++) {
           // 对象的每个属性进行劫持
           let key = keys[i];
           let value = data[key];
           defineReactive(data, key, value);
         }
       }
     }
     //对对象中的属性进行劫持
     function defineReactive(data, key, value) {
       //{a:{b:1}}
     
       observer(value); // 深度代理
       // 为对象中的属性添加get,set
       Object.defineProperty(data, key, {
         get() {
           return value;
         },
         set(newValue) {
           if (newValue == value) return;
           observer(newValue); // 如果用户设置的值是对象？递归调用
           value = newValue;
         },
       });
     }
     ```

   - 数组 =》重写数组**oldArrayProtoMethods**通过继承**Array.prototype**显示原型，定义一个**ArrayMethods**继承于这个重写的数组**oldArrayProtoMethods**，然后定义数组方法，通过循环遍历*劫持*重写这些方法，但是这些方法最终调用的还是原生数组的方法，只是对这些方法进行了一个*劫持*。

     ```js
     // 重写数组
     // 1. 获取原来的数组方法
     let oldArrayProtoMethods = Array.prototype;
     
     // 2. 继承
     export let ArrayMethods = Object.create(oldArrayProtoMethods);
     console.log("ArrayMethods", ArrayMethods);
     // 方法
     let methods =[
       "push",
       "pop",
       "unshift",
       "shift",
       "splice"
     ]
     
     // 3. 遍历劫持
     methods.forEach(item => {
        ArrayMethods[item] = function(...args) {
         let result = oldArrayProtoMethods[item].apply(this, args);
         return result;
        }
     })
     ```




# 2.添加生命周期

1. 在各个阶段添加对应的生命周期：**beforeCreated **，**created **，**beforeMounted**，**mounted**，都挂载到了Vue的原型$options上

```js
// 生命周期的调用
export function callHook(vm, hook) {
  console.log(hook, "生命周期：", vm.$options[hook]);
  const handlers = vm.$options[hook];
  if (handlers) {
    for (let i = 0; i < handlers.length; i++) {
      handlers[i].call(this); // 改变生命周期中this指向问题
    }
  }
}
```



# 3.全局方法  Vue.mixin

1. 定义全局 **Mixin方法** 进行调用

   ```js
   //响应式  vue2  mvvm
   Vue.Mixin({
       //全局
       created: function a() {
           console.log("a----2");
       },
   });
   Vue.Mixin({
       //全局
       created: function b() {
           console.log("b----2");
       },
   });
   ```

   

2. 在初始化调用 **initGlobApi(Vue)** ，在Vue原型上挂载 **Mixin** 方法，里面的参数对应：**this** 指向 **Vue的实例**，**this.options = Vue.options** ，**minix** 指向调用传入的参数 全局：**Vue.Mixin({入参})**。

   ```js
   export function initGlobApi(Vue) {
     Vue.options = {};
     Vue.Mixin = function (mixin) {
       // 对象的合并
       this.options = mergeOptions(this.options, mixin);
       console.log("Vue.options", Vue.options);
     };
   }
   ```

   

3. **策略模式** 将多个全局的 **Minix方法** 合并在一起

   ```js
   // 对象合并 生命周期
   export const HOOKS = [
     "beforeCreate",
     "created",
     "beforeMount",
     "mounted",
     "beforeUpdate",
     "updated",
     "beforeDestory",
     "destroyed",
   ];
   
   // 策略模式
   let starts = {};
   starts.data = function (parentVal, childVal) {
     return childVal;
   };
   starts.computed = function () {}; // 合并computed
   starts.watch = function () {}; // 合并watch
   starts.methods = function () {}; // 合并methods
   // 遍历生命周期
   HOOKS.forEach((hooks) => {
     starts[hooks] = mergeHook;
   });
   
   function mergeHook(parentVal, childVal) {
     if (childVal) {
       if (parentVal) {
         return parentVal.concat(childVal);
       } else {
         return [childVal];
       }
     } else {
       return parentVal;
     }
   }
   
   export function mergeOptions(parent, child) {
     const options = {};
     // 如果有父亲 ，没有儿子
     for (let key in parent) {
       mergeField(key);
     }
     // 儿子有-父亲没有
     for (let key in child) {
       mergeField(key);
     }
     function mergeField(key) {
       // 根据key   策略模式
       if (starts[key]) {
         options[key] = starts[key](parent[key], child[key]);
       } else {
         options[key] = child[key];
       }
     }
     return options;
   }
   
   ```

   

4. s

# 4.依赖收集、派发更新（watcher,dep）

1. **watcher.js** 观察者：作为一个中介，当数据发生变化时，通过watcher中转通知组件。watcher实例分为渲染watcher、计算watcher、侦听器watcher

   ```js
   import { pushTarget, popTarget } from "./dep";
   import { nextTick } from "../utils/nextTick";
   
   //为什么封装成一个类 ，方便我们的扩展
   let id = 0; //全局的
   class Watcher {
     //vm 实例
     //exprOrfn vm._updata(vm._render())
     constructor(vm, exprOrfn, cb, options) {
       // 1.创建类第一步将选项放在实例上
       this.vm = vm;
       this.exprOrfn = exprOrfn;
       this.cb = cb;
       this.options = options;
       // computed 计算属性
       this.lazy = options.lazy; // 如果这个watcher 上有lazy 说明是 computed
       this.dirty = this.lazy; // dirty  表示用户是否执行
       // 2. 每一组件只有一个watcher 他是为标识
       this.id = id++;
       this.user = !!options.user;
       // 3.判断表达式是不是一个函数
       this.deps = []; //watcher 记录有多少dep 依赖
       this.depsId = new Set();
       if (typeof exprOrfn === "function") {
         this.getter = exprOrfn;
       } else {
         //{a,b,c}  字符串 变成函数
         this.getter = function () {
           //属性 c.c.c
           let path = exprOrfn.split(".");
           let obj = vm;
           for (let i = 0; i < path.length; i++) {
             obj = obj[path[i]];
           }
           return obj; //
         };
       }
       // 4.执行渲染页面
       this.value = this.lazy ? void 0 : this.get(); //保存watch 初始值
     }
     addDep(dep) {
       //去重  判断一下 如果dep 相同我们是不用去处理的
       let id = dep.id;
       //  console.log(dep.id)
       if (!this.depsId.has(id)) {
         this.deps.push(dep);
         this.depsId.add(id);
         //同时将watcher 放到 dep中
         // console.log(666)
         dep.addSub(this);
       }
       // 现在只需要记住  一个watcher 有多个dep,一个dep 有多个watcher
       //为后面的 component
     }
     run() {
       //old new
       let value = this.get(); //new
       let oldValue = this.value; //old
       this.value = value;
       //执行 hendler (cb) 这个用户wathcer
       if (this.user) {
         this.value = this.cb.call(this.vm, value, oldValue);
       }
     }
     get() {
       // Dep.target = watcher
   
       pushTarget(this); //当前的实例添加
       const value = this.getter.call(this.vm); // 渲染页面  render()   with(wm){_v(msg,_s(name))} ，取值（执行get这个方法） 走劫持方法
       popTarget(); //删除当前的实例 这两个方法放在 dep 中
       return value;
     }
     //问题：要把属性和watcher 绑定在一起   去html页面
     // (1)是不是页面中调用的属性要和watcher 关联起来
     //方法
     //（1）创建一个dep 模块
     updata() {
       //注意：不要数据更新后每次都调用 get 方法 ，get 方法回重新渲染
       //缓存
       // this.get() //重新渲染
       // queueWatcher(this)
       if (this.lazy) {
         // 这是计算属性的watcher
         this.dirty = true;
       } else {
         queueWatcher(this); //重新渲染
       }
     }
     evaluate() {
       this.value = this.get();
       this.dirty = false;
     }
     // 相互依赖
     depend() {
       // 收集watcher, 存放到dep dep在会存放我的watcher
       // 通过这个 watcher 找到对应的所有的 dep, 在让所有的 dep 都记住 这渲染的watcher
       let i = this.deps.length;
       while (i--) {
         this.deps[i].depend();
       }
     }
   }
   
   let queue = []; // 将需要批量更新的watcher 存放到一个列队中
   let has = {};
   let pending = false;
   function flushWatcher() {
     queue.forEach((item) => {
       item.run();
       // item.cb()
     });
     queue = [];
     has = {};
     pending = false;
   }
   function queueWatcher(watcher) {
     let id = watcher.id; // 每个组件都是同一个 watcher
     //    console.log(id) //去重
     if (has[id] == null) {
       //去重
       //列队处理
       queue.push(watcher); //将wacher 添加到列队中
       has[id] = true;
       //防抖 ：用户触发多次，只触发一个
       if (!pending) {
         //异步：等待同步    代码执行完毕之后，再执行
         // setTimeout(()=>{
         //   queue.forEach(item=>item.run())
         //   queue = []
         //   has = {}
         //   pending = false
         // },0)
         nextTick(flushWatcher); //  nextTick相当于定时器
       }
       pending = true;
     }
   }
   
   export default Watcher;
   
   ```

   - **queueWatcher方法** 是处理防止多次渲染

     每个组件都有自己的watcher对应的id唯一性，将id保存到has{}对象中，确保去重，不存在就添加到队列中，设置防抖 ：用户触发多次，只触发一次 **nextTick** 来完成处理。

   - **nextTick方法** 

     将nextTick挂载到Vue原型上：

     第一种Vue自己的nextTick用来处理多次触发执行。
     第二种是用户的nextTick用来元素已经加载完之后再进行逻辑处理。

     ```js
      let callback = []
      let pending = false
      function flush(){
         callback.forEach(cb =>cb())
         pending =false
      }
      let timerFunc
      //处理兼容问题
      if(Promise){
         timerFunc = ()=>{
             Promise.resolve().then(flush) //异步处理
         }
      }else if(MutationObserver){ //h5 异步方法 他可以监听 DOM 变化 ，监控完毕之后在来异步更新
        let observe = new MutationObserver(flush)
        let textNode = document.createTextNode(1) //创建文本
        observe.observe(textNode,{characterData:true}) //观测文本的内容
        timerFunc = ()=>{
         textNode.textContent = 2
        }
      }else if(setImmediate){ //ie
         timerFunc = ()=>{
             setImmediate(flush) 
         }
      }
      export function nextTick(cb){
          // 1vue 2
         //  console.log(cb)
          //列队 [cb1,cb2]
          callback.push(cb)
          //Promise.then()  vue3
          if(!pending){
              timerFunc()   //这个方法就是异步方法 但是 处理兼容问题
              pending = true
          }
      }
     ```

     

2. **dep.js  **订阅者：用于收集当前响应式对象的依赖

   ```js
   let id = 0;
   class Dep {
     constructor() {
       this.subs = [];
       this.id = id++;
     }
     //收集watcher
     depend() {
       //我们希望water 可以存放 dep
       //实现双休记忆的，让watcher 记住
       //dep同时，让dep也记住了我们的watcher
       Dep.targer.addDep(this);
       // this.subs.push(Dep.targer) // id：1 记住他的dep
     }
     addSub(watcher) {
       this.subs.push(watcher);
     }
     //更新
     notify() {
       // console.log(Dep.targer)
       this.subs.forEach((watcher) => {
         watcher.updata();
       });
     }
   }
   
   //dep  和 watcher 关系
   Dep.targer = null;
   // 处理多个watcher
   let stack = []; // 栈
   export function pushTarget(watcher) {
     //添加 watcher
   
     Dep.targer = watcher; //保留watcher
     // console.log(Dep.targer)
     // 入栈
     stack.push(watcher);
   }
   export function popTarget() {
     // Dep.targer = null; //将变量删除
     // 解析完成一个watcher 就删除一个watcher [ watcher1, watcher2]
     stack.pop();
     Dep.targer = stack[stack.length -1] // 获取到你前面的一个 watcher
   }
   export default Dep;
   //多对多的关系
   //1. 一个属性有一个dep ,dep 作用：用来收集watcher的
   //2. dep和watcher 关系：(1)dep 可以存放多个watcher  (2):一个watcher可以对应多个dep
   
   ```

   

   - 对数据进行劫持(data)：

     - get()方法触发通过Dep收集这个属性对象，某一个属性对象都有自己唯一的watcher，id。
     - set()方法调用会触发Dep中的notify()方法去通知数据发生改变，从而组件发生跟新。内部其实就是调用get()的方法去跟新。

     ```js
     //对数据进行劫持
     function defineReactive(data, key, value) {
       // Object.defineProperty
       let chilidDep = Observer(value); //获取到数组对应的dep
       //1给我们的每个属性添加一个dep
       let dep = new Dep();
       //2将dep 存放起来，当页面取值时，说明这个值用来渲染，在将这个watcher和这个属性对应起来
       Object.defineProperty(data, key, {
         get() {
           //依赖收集
           // console.log('获取数据', data, key, value)
           if (Dep.targer) {
             //让这个属性记住这个watcher
             dep.depend();
             //3当我们对arr取值的时候 我们就让数组的dep记住这个watcher
             if (chilidDep) {
               chilidDep.dep.depend(); //数组收集watcher
             }
           }
           //检测一下 dep
           //获取arr的值，会调用get 方法 我希望让当前数组记住这个渲染watcher
     
           // console.log(dep.subs)
           return value;
         },
         set(newValue) {
           //依赖更新
           //注意设置的值和原来的值是一样的
           // console.log('设置值', data, key, value)
           if (newValue == value) return;
           Observer(newValue); //如果用户将值改为对象继续监控
           value = newValue;
           dep.notify();
         },
       });
     }
     ```

     

   - 数组：**当属性对象调用具有响应式方法的时候，通过调用dep.notify()通知跟新**

# 5.监听器Watch

1. 初始化获取**watch**对象有多种编写方式，通过**createrWatcher()**方法处理获取**vm实例对象，exprOrfn属性名称，handler函数方法，options配置项**。

   ```js
   export function stateMixin(vm) {
     //列队 :1就是vue自己的nextTick  2用户自己的
     (vm.prototype.$nextTick = function (cb) {
       //nextTick: 数据更新之后获取到最新的DOM
       nextTick(cb);
     }),
       (vm.prototype.$watch = function (Vue, exprOrfn, handler, options = {}) {
         //上面格式化处理
         //实现watch 方法 就是new  watcher //渲染走 渲染watcher $watch 走 watcher  user false
         //  watch 核心 watcher
         let watcher = new Watcher(Vue, exprOrfn, handler, {
           ...options,
           user: true,
         });
         if (options.immediate) {
           handler.call(Vue); //如果有这个immediate 立即执行
         }
       });
   }
   function initWatch(vm) {
     //1 获取watch
     let watch = vm.$options.watch;
     //2 遍历  { a,b,c}
     for (let key in watch) {
       //2.1获取 他的属性对应的值 （判断)
       let handler = watch[key]; //数组 ，对象 ，字符，函数
       if (Array.isArray(handler)) {
         //数组  []
         hendler.forEach((item) => {
           createrWatcher(vm, key, item);
         });
       } else {
         //对象 ，字符，函数
         //3创建一个方法来处理
         createrWatcher(vm, key, handler);
       }
     }
   }
   
   //vm.$watch(()=>{return 'a'}) // 返回的值就是  watcher 上的属性 user = false
   //格式化处理
   function createrWatcher(vm, exprOrfn, handler, options) {
     //3.1 处理handler
     if (typeof handler === "object") {
       options = handler; //用户的配置项目
       handler = handler.handler; //这个是一个函数
     }
     if (typeof handler === "string") {
       // 'aa'
       handler = vm[handler]; //将实例行的方法作为 handler 方法代理和data 一样
     }
     //其他是 函数
     //watch 最终处理 $watch 这个方法
     return vm.$watch(vm, exprOrfn, handler, options);
   }
   ```

   

2. 调用**$watch(Vue, exprOrfn, handler, options = {})**

3. 第一次会先触发一次会保留原始值，当数据发生改变时，(watcher,dep)会触发调用run()方法，保留最新的值

   ```js
   // 执行渲染页面
   this.value = this.lazy ? void 0 : this.get(); //保存watch 初始值
   run() {
       //old new
       let value = this.get(); //new
       let oldValue = this.value; //old
       this.value = value;
       //执行 hendler (cb) 这个用户wathcer
       if (this.user) {
           this.value = this.cb.call(this.vm, value, oldValue);
       }
   }
   ```

   

4. 无



# 6.计数属性computed

1. 初始化获取computed对象里面的值，遍历为某一个属性值通过 **defineComputed **配置。

   ```js
   function initComputed(vm, computed) {
     // 存放计算属性的watcher
     const watchers = (vm._computedWatchers = {});
     for (const key in computed) {
       const userDef = computed[key];
       // 获取get方法
       const getter = typeof userDef === "function" ? userDef : userDef.get;
       // 创建计算属性watcher
       watchers[key] = new Watcher(vm, getter, () => {}, { lazy: true });
       defineComputed(vm, key, getter);
     }
   }
   let sharedPropertyDefinition = {};
   function defineComputed(target, key, userDef) {
     sharedPropertyDefinition = {
       enumerable: true,
       configurable: true,
       get: () => {},
       set: () => {},
     };
     if (typeof userDef === "function") {
       sharedPropertyDefinition.get = createComputedGetter(key);
     } else {
       sharedPropertyDefinition.get = createComputedGetter(userDef.get);
       sharedPropertyDefinition.set = userDef.set;
     }
     // 使用defineProperty定义
     Object.defineProperty(target, key, sharedPropertyDefinition);
   }
   
   ```

   

2. 缓存机制通过dirty属性控制，获取值通过get()方法调用将值存返回出去存储起来。

   ```js
   function createComputedGetter(key) {
     return function computedGetter() {
       const watcher = this._computedWatchers[key];
       if (watcher) {
         if (watcher.dirty) {
           // 如果dirty为true
           watcher.evaluate(); // 计算出新值，并将dirty 更新为false
         }
         // 判断一下有没有渲染的watcher 有执行： 相互存放 watcher
         if (Dep.targer) {
           // 说明还有渲染的watcher，收集起来
           watcher.depend(); // watcher 收集
         }
         // 如果依赖的值不发生变化，则返回上次计算的结果
         return watcher.value;
       }
     };
   }
   get() {
       // Dep.target = watcher
       pushTarget(this); //当前的实例添加
       const value = this.getter.call(this.vm); // 渲染页面  render()   with(wm){_v(msg,_s(name))} ，取值（执行		get这个方法） 走劫持方法
       popTarget(); //删除当前的实例 这两个方法放在 dep 中
       return value;
   }
   ```

   

3. 无

# 7.渲染模板 el --- #app

1. 将 **$mount** 方法挂载到VUE原型上，调用 **$mount()** 方法传入 **vm.$options.el** =》 **#app**

   ```js
   // 渲染模板 el --- #app
   if (vm.$options.el) {
       // 调用mount方法
       vm.$mount(vm.$options.el);
   }
   ```

   

2. 将 **$mout** 挂载到VUE的原型上，传入**el =》 #app**，通过**document.querySelector(el)**获取 **html** 的元素，挂载到VUE的原型 **$el** 上，判断 **options** 上是否有 **render** 模板

   - 无：通过 **el = el.outerHTML** 获取纯HTML字符串，传入HTML字符串调用 **compileToFunction(el)** 方法生成 **render函数**

   ```js
   Vue.prototype.$mount = function (el) {
       let vm = this;
       // 获取的dom
       console.log("el", el);
       el = document.querySelector(el);
       vm.$el = el;
       let options = vm.$options;
       if (!options.render) {
         // 没有进入
         let template = options.template;
         if (!template && el) {
           // 没有进入
           // 获取 html
           el = el.outerHTML;
           console.log("HTML", el);
           // 为这个Html 改变成 ast抽象语法树
           let render = compileToFunction(el);
           //(1) 将render  函数变成vnode  (2) vnode 变成 真实DOM 放到页面上去
           options.render = render;
           console.log("this-vm", vm);
         }
       }
       //挂载组件
       mounetComponent(vm, el); // vm._updata vm._render
     };
   ```

   #### 步骤：

   1. **将html 变成ast 语法树**

   2. **ast 语法树变成 render 函数  （1） ast 语法树变成 字符串**

   3. **将render 字符串变成 函数**

      ```js
      export function compileToFunction(el) {
        //1 将html 变成ast 语法树
        let ast = parseHTML(el);
        console.log("ast", ast);
        //2 ast 语法树变成 render 函数  （1） ast 语法树变成 字符串  （2）字符串变成函数
        let code = generate(ast); // _c _v _s
        console.log("code", code);
        // 3将render 字符串变成 函数
        let render = new Function(`with(this){return ${code}}`);
        console.log("render", render);
        return render;
      }
      
      ```

      

   4. **内部是通过：**这些都是挂载到原型Vue上

      ```tex
      '_c()'的方法是处理标签，'_v()'方法是处理文本，'_s()'方法是处理变量。
      ```

      

3. 









