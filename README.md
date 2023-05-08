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

     

5. sd

# 2.渲染模板 el --- #app

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

   

3. 









