## 说说在React项目中状态管理使用Redux还是Mobx的心里话

​	大多数人在首次接触React的状态管理都是Redux，但是flow的概念对于刚开始接触React的来说视乎又有点抽象，而且他必须要按照约定流程来组织代码，有时候很简单的事情用它感觉有点杀鸡用牛刀。于是乎mobx来出现了，我第一次接触到他的时候也有一种如沐春风的赶脚，网上有大量介绍关于使用Mobx各种好处的文章，于是我信了，坑次吭哧拿到项目首先就想到了它，眼里容不下其他，就像遇见美食总想一次吃个够，等到吃到腻了才会发现原来山珍海味都比不上粗茶淡饭。因此我写这一片文章不是来吹Mobx有多么牛逼的，我是结合我在项目中的使用经验来对比分析一下使用Mobx和Redux优势和劣势，什么时候适合使用Redux，什么时候适合使用Mobx。好了，话不多说开始入正题。

首先，先来看一下Redux。

Redux的使用步骤：

1. 创建reducers
2. 创建actionCreators
3. 通过createStore，combinReduces，applymiddleware来创建一个全局的store
4. 通过react-redux提供的高阶组件Provider和connect来将store和action挂载到子组件的props中去。

经过上面这个四步，大概就可以在react中将redux使用起来了，当然,这里还不涉及到中间件，个人感觉中间件是Redux中个人感觉比较有特点的地方，熟悉express和koa的前端来说，对中间件的概念并不陌生，这里不赘述，有兴趣可以网上搜一下。

​	大致说了一下使用流程，下面就用一段代码来具体实现一下上面的流程，毕竟程序猿还是靠代码说话的，有时候我们为了方便会将reducers和actincreators写在一个文件中了，我建议还是将reduces和actioncreators分文件夹和和分模块管理。这样随着项目的迭代，代码的层级结构会更加清楚，维护起来也更加得心应手。

types.js

```js
export const USERINFO = 'USERINFO';
```

common-reducer.js

```js
let initState = {
    userInfo:{},
    counter:0
}
function reducer(state = initState,action){
    let {type,preload=null} = action;
    switch(type){
        case USERINFO:
           state.userInfo = preload 
           return Object.;
        default:
           return state;
    }
}
export default reducer;
```

common-action.js

```js
export getUserInfo = function(){
    return {type:USERINFO,preload:{username:'wangyy'}}
}
```

index.js

```js
import {createStore,combineReducers,applyMiddleware} from "redux"
import module1 from "./common-reducer.js";
import thunk from "redux-thunk";
//这里使用分墓快
const reducers = combineReducers{
    module1
}
//applyMiddleware（thunk）使用thunk中间件，来让我们的action具备异步功能
const store = createStore(reducers，applyMiddleware（thunk）)
```

上面创建好之后，我们就可以开始在组件中开始应用了全局状态了。

redux本身提供了subsrible功能的，用来定于action的触发，这里我们使用react-redux提供的高阶组件Provider来将store全局下发，以及connect组件来实现state和action到组件的映射

main.js（入口文件）

```js
import React from "react";
import ReactDom from "react-dom";
import App from "./App.js";
import {Provider} from "react-redux";
function render(Component){
    return ReactDom.render(
    (<AppContainer>
        <Provider store = {store}>
            <Component></Component>
        </Provider>
    </AppContainer>),
    document.getElementById("root")
    )
}
render(App);
```

app.js（组件）

```js
import React from "react";
import {connect} from "react-redux";
import {getUserInfo} from "./store/actions/common-action.js"
import {USERINFO} from "./store/types/types.js";
function mapDispatchsToProps(dispatch){
    return {
       onClick:()=>{dispatch(getUserInfo())},
    }
}
function mapStateToProps(state){
   return {
       userInfo :state.common.userInfo
   }
} 
class App extends React.Component{
    render(){
        return (
            <div className="container">
                userInfo:{this.props.userInfo}
                <button onClick={this.props.onClick}>getUser</button>
                <hr/>
            </div>
        )
        
    }
}
export default connect(mapStateToProps,mapDispatchsToProps)(App)
```
可以使用ES7的decorators来进行修饰，可以简化一些代码。

通过上面的代码，我们可以很明显的发现，实现的功能很简单但是代码感觉写了好多，如果其他组件中有用到store，我们就需要在每个组件中定义mapStateToProps和mapDispatchsToProps，然后使用connect装饰该组件，这又是中间件又是高阶组件的，头有点大，这对于刚入门React的来说这个流程好像也稍微有点复杂。

那下面使用mobx来是重新组织一下上面的代码，看看mobx有什么神奇的地方

Mobx

官方文档：https://cn.mobx.js.org

使用步骤

1. 使用observable创建需要观察的数据对象
   1. state,@computd,action
2. 我们可以使用ES7的decorators来装饰对象，这里还是使用传统的方式来编写，便于理解。

具体代码实现：
这里还是采用分模块管理的形式

common.mobx.js

```js
import {observable,action} from "mobx";
let store = observable({
    userInfo:{
        id:'',
        avater:'',
        username:'',
    }
})

store.getUserInfo=(userInfo)=>{
    return action(()=>{
            //模拟异步请求
            setTimeout(()=>{
                state.userInfo = {
                    ...userInfo
                }
            },1000)
    })
}
export default store
```

index.js

```js
import common from "./common.mobx.js"
export default{
    common
}
```

在组件中使用：

main.js（入口文件），和redux没什么区别，就是react-redux变成了mobx-react

```js
import React from "react";
import ReactDom from "react-dom";
import App from "./App.js";
import {Provider} from "mobx-react";
function render(Component){
    return ReactDom.render(
    (<AppContainer>
        <Provider store = {store}>
            <Component></Component>
        </Provider>
    </AppContainer>),
    document.getElementById("root")
    )
}
render(App);
```

app.js（组件）

```js
import React from "react";
import {inject,observer} from "react-redux";
import {common} from "./store/index.js"
@inject('store')
@observer
class App extends React.Component{
    componentWillMounted(){
        this.store = this.props.store;
    },
    render(){
        return (
            <div className="container">
                userInfo:{this.store.userInfo}
                <button onClick={this.store.setUserInfo}>getUser</button>
                <hr/>
            </div>
        ) 
    }
}
export default connect(mapStateToProps,mapDispatchsToProps)(App)
```

这样，大致就是可以通过mobx实现状态的全局管理了，有没有感觉比Redux好写多了。当然mobx不仅可以实现全局状态管理，我们也完全可以将组件内使用，将组件内的状态做成响应式的，这样可以实现干掉setState的摸底。

	看完了上面两种全局状态管理不知道各位看官自己有怎样的想法呢？下面我来说说我的想法，我前面说了，我这边文章不是来吹mobx的有多牛逼的，我是分析一下在使用过程中使用这两种全局状态管理方式的利弊。

首先来看看他们的共同点：

1. 两者都可以用来作为全局状态管理，mobx在这点上做的更加极致，他不仅可以用来管理全局状态，还可以用来管理组件内部状态，从而达到干掉setState的目的
2. 上手难易程度，两者相差无几，花半个小时，基本上都能写一个demo出来，但是要详细去理解也还是需要花费一点时间的，mobx需要理解ES7的decorators，redux主要是flow的那一套概念有点抽象

下面来看看两者的区别：

首先来看看redux的设计理念

Redux是flow单项数据流设计理念的一种具体实现，就像下面这张图一样：

通过这张图，我们可以很清楚的看出数据的流向，Redux只能dispatch去触发action然后通过reducers去处理state，这非常便于我们追踪数据和用户的操作以及问题的排查。

那下面在来看看mobx，mobx是一种响应式的设计，通过依赖收集和通知来实现自动的setState的目的。在mobx中也定义action来修改状态，但是在2.0之前不通过action，也可以直接修改状态，在2.0后面的版本中加入了严格模式，在严格模式下必须通过action才才能改变状态，但是还是保留了非严格模式，在多人共同的开发的项目中，谁知道有没有用严格模式呢。如果没有，这样就有可能导致状态的修改比较随意，没法有效跟踪数据的流向，从而快速的分析和定位。还有有一个问题就是在使用严格模式的情况下也可能会遇见下面整中情况

```js
import {useStrict,observable,action} from "mobx";
useStrict(true);
let store = observable({
    userInfo:{
        id:'',
        avater:'',
        username:'',
    }
})

store.getUserInfo=()=>{
    return action(()=>{
            getTimeout(()=>{
                let userInfo = {name:"wangyy"};
                state.userInfo = {
                    ...userInfo
                }
            },1000)
    })
}
export default store
```

这里会报错，因为在严格模式下，不允许在action以外修改state,在action中的异步函数中修改state也不行，这个时候我们需要单独定义一个action来修改异步action中的状态

```js
store.changedata = (data)=>{
    return action(()=>{
           state.userinfo = data;
    })
}
store.getUserInfo=()=>{
    return action(()=>{
            setTimeout(()=>{
                let userInfo = {name:"wangyy"};
               store.changedata(userInfo)
            },1000)
    })
}
```

这样下来，好像也挺多余的,所以这样看来mobx也不是尽善尽美。
	
	那么反过头来在来看Redux，Redux他有着严格的约定，这在项目中感觉有点杀鸡用牛刀。另外Redux被人诟病很多的就是Reducer处理action的时候每次都需要用到扩展语句或者Object.assign来对state对象进行浅克隆，从而返回一个新的状态，这在一定程度会增加内存开销，降低性能。这确实是一个问题，但是我们可以结合使用不可突变数据immutable.js来实现优化，在React官网中也提到了使用不可突变数据结构来实现性能优化，这里不介绍immutable.js,给个地址自己去看看：immutable.js文档：https://www/zcfy.cc/article/immutable-js-241.html


### 总结：

说了这么多其实就是想高速大家在选择Redux和Mobx做全局状态管理方案的时候需要权衡一下利弊。

redux有一套严格的规范，适合大型项目特别和团队开发的项目，虽然代码编写稍显繁复，但是整体数据流向清楚，便于问题跟踪和后期维护。

mobx编写过于自由奔放，适合小型项目特别是那种一锤子买卖一次编写后续不维护的项目，新手还是建议从redux开始用，先理解单项数据流的理念，再出去浪。
