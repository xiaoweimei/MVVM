# MVVM项目原理描述
### [单向绑定预览地址](https://xiaoweimei.github.io/MVVM/MVVM_onewaybind.html)
- 单向绑定：即从视图到模型，或从模型到视图，数据不能构成一个环路
### [双向绑定预览地址](https://xiaoweimei.github.io/MVVM/MVVM_twowaybind.html)
- 双向绑定：即数据可以从视图出发，传送到模型，再经由模型再次传递到视图
### 原理分类
- `Object.defineProperty的使用`
- 实现数据劫持
- 观察者模式和发布订阅模式
- 实现单向绑定和双向绑定
### 下面对源码进行细致的分析与解释
- 我们的html部分是这么写的
```
<div id="app" >
  <input v-model="name" v-on:click='sayHi' type="text">
  <h1>{{name}} 's age is {{age}}</h1>
</div>
```
#### 程序执行部分
- 一开始创建vm对象，代码为下述代码
```
let vm = new mvvm({
  el: '#app',
  data: {
    name: 'xiaoweimei',
    age: 25
  },
  methods:{
    sayHi(){
      console.log(`hi ${this.name}`)
    }
  }
})
```
- 在执行上述代码即创建vm对象时会调用mvvm构造函数实现函数，代码如下
```
class mvvm {
  constructor(opts) {
    this.init(opts)
    observe(this.$data)
    new Compile(this)
  }
  init(opts){
    this.$el = document.querySelector(opts.el)
    this.$data = opts.data || {}
    this.$methods=opts.methods || {}
    //把$data中的数据直接代理到当前vm对象
    for(let key in this.$data){
      Object.defineProperty(this,key,{
        enumerable:true,
        configurable:true,
        get:()=>{ //这里使用了箭头函数，所有里面的this就指代外面的this也就是vm
          return this.$data[key]
        },
        set:newVal=>{
          this.$data[key]=newVal
        }
      })
    }
    //让this.$methods里面的函数中的this，都指向当前的this，也就是vm
    for(let key in this.$methods){
      this.$methods[key]=this.$methods[key].bind(this)
    }
  }
}
```
- 在初始化vm对象过程中，会在页面中选取对应的DOM元素，并会用变量将DOM元素和数据及方法保存起来，同时使用Object对象的defineProperty属性及Function对象的bind方法对数据访问形式进行改造，更加便捷，初始化完成会进入observe函数，并将data传入当作参数，下面为observe代码
```
function observe(data) {
  if(!data || typeof data !== 'object') return
  for(var key in data) {
    let val = data[key]
    let subject=new Subject()
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function() {
        console.log(`get ${key}: ${val}`)
        if(currentObserver){
          console.log('has currentObserver')
          currentObserver.subscribeTo(subject)
        }
        return val
      },
      set: function(newVal) {
        val = newVal
        console.log('start notify...')
        subject.notify()
      }
    })
    if(typeof val === 'object'){
      observe(val)
    }
  }
}
```
- 在observe函数中，主要对data进行遍历，创建主题对象，每一个属性都会创建一个主题对象
```
class Subject{
  constructor(){
    this.id=id++
    this.observers=[]
  }
  addOberver(observer){
    this.observers.push(observer)
  }
  removeObserver(observer){
    var index=this.observers.indexof(observer)
    if(index>-1){
      this.observers.splice(index,1)
    }
  }
  notify(){
    console.log('notify被调用。。。。。')
    console.log(this.observers)
    this.observers.forEach(observer=>{
      console.log(observer)
      observer.update()
    })
  }
}
```
- 主题对象定制了添加观察者，删除观察者及更新等方法，回到observe函数，其中涉及到get、和set方法，主要用来获取值和更新值会调用的一些方法，后面会再介绍，observe函数执行完会进入Compile构造函数
```
class Compile{
  constructor(vm){
    this.vm=vm
    this.node=vm.$el
    this.compile()
  }
  compile(){
    this.traverse(this.node)
  }
  traverse(node){
    console.log(node)
    console.log(node.nodeType)
    console.log(789)
    if(node.nodeType===1){
      console.log(333)
      this.compileNode(node)
      node.childNodes.forEach(childNode=>{
        console.log('traverse...')
        this.traverse(childNode)
      })
    }else if(node.nodeType===3){//文本
      console.log(111)
        this.compileText(node)
    }
  }
  compileText(node){
    console.log(node)
    let reg=/{{(.+?)}}/g
    let match
    console.log('match')
    while(match=reg.exec(node.nodeValue)){
      console.log(1234567)
      console.log(match)
      let raw=match[0]
      let key=match[1].trim()
      node.nodeValue=node.nodeValue.replace(raw,this.vm.$data[key])
      console.log(7654321)
      new Observer(this.vm,key,function(val,oldVal){ //解析模板遇到变量创建观察者
        node.nodeValue=node.nodeValue.replace(oldVal,val)
      })
    }
  }
  compileNode(node){
    let attrs=[...node.attributes]//类数组对象转化为数组，也可以使用其他方法
    attrs.forEach(attr=>{
      //attr是个对象，attr.name是属性的名字如v-model,attr.value是对应的值,如name
      if(this.isModelDirective(attr.name)){
        this.bindModel(node,attr)
      }else if(this.isEventDirective(attr.name)){
        this.bindEventHander(node,attr)
      }
    })
  }
  bindModel(node,attr){
    let key=attr.value //attr.value==='name'
    node.value=this.vm[key]
    new Observer(this.vm,key,function(newVal){
      node.value=newVal
    })
    node.oninput=(e)=>{
      this.vm[key]=e.target.value//因为是箭头函数，所以这里的this是compile对象
    }
  }
  bindEventHander(node,attr){  //attr.name==='v-on:click',attr.value==='sayHi'
    let eventType=attr.name.substr(5)  //click
    let methodName=attr.value
    node.addEventListener(eventType,this.vm.$methods[methodName])
  }
  //判断属性名是否是指令
  isModelDirective(attrName){
    return attrName==='v-model'
  }
  isEventDirective(attrName){
    return attrName.indexOf('v-on')===0
  }
}
```
- compile函数主要完成了节点的选区与解析，具体包括data的选区与解析及渲染，以及事件的解析与绑定函数，并且为每一个属性赋予观察者，调用Observe构造函数
代码如下
```
class Observer{
  constructor(vm,key,cb){
    this.subjects={}
    this.vm=vm
    this.key=key
    this.cb=cb
    this.value=this.getValue()
  }
  update(){
    console.log('update 被调用。。。')
    let oldVal=this.value
    let value=this.getValue()
    console.log(value)
    console.log(oldVal)
    if(value!==oldVal){
      this.value=value
      this.cb.bind(this.vm)(value,oldVal)
    }
  }
  subscribeTo(subject){
    if(!this.subjects[subject.id]){
      console.log('subcribeTo..',subject)
      subject.addOberver(this)
      this.subjects[subject.id]=subject
    }
  }
  getValue(){
    console.log('调用观察者')
    currentObserver=this
    let value=this.vm.$data[this.key]
    currentObserver=null
    return value
  }
}
```
- Observe构造函数会在初始化过程中完成添加主题等内容，同时也提供了更新方法，至此代码运行结束
### 基本过程
1. 初始化vm对象
2. 观察vm对象
3. 对每一个vm对象的属性值创建主题对象
4. 解析模板，渲染页面
5. 在渲染页面过程中对每一个变量创建观察者对象，同时为观察者订阅主题
- 上述过程基本为单向传递的内容，双向传递即在Compile构造函数里面添加事件函数，使值传入模板，再经过模板渲染展示页面
