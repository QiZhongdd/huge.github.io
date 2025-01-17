**react事件机制**

react并不是将click事件绑定到div的真实dom上，而是在document处监听所有支持的事件，当事件发生冒泡至document时，react将事件内容封装并交由真正的处理函数运行，这样不仅减少了内存消耗，还能在组件挂载销毁时统一删除。冒泡至document是react的合成事件。具体流程：

react的事件机制主要有两个阶段：
事件注册阶段：收集相关的事件绑定到document中，但并不是所有的事件都会被收集比如form表单的submit、reset还有video和audio的媒体事件。
第二个阶段是执行阶段：它主要有以下几个步骤，第一个步骤是创建合成事件，并将原生事件包装到合成事件中，以原生事件的target为节点向上查找，如果fiber节点的tag=hostComponent则加入到path数组中。第二个步骤是捕获阶段，倒序遍历path数组，如果fiber节点还有onClickCapture属性，则添加到合成事件的dispatch_listner中。第三个步骤是收集冒泡的回调。顺序遍历path数组，如果找到了onClick回调，则添加到dispatch_listner，第四个步骤就是按顺序执行合成事件的dispatch_Listner。

React v17 中，React 不会再将事件处理添加到 document 上，而是将事件处理添加到渲染 React 树的根 DOM 容器中：这是因为在多版本并存的系统时，不同版本的事件系统是独立的，所以不同版本的 React 组件嵌套使用时，e.stopPropagation()无法正常工作，因为到document已经太晚，此时原声的原生 DOM 事件早已冒出document了。为了解决这个问题，React 17 不再往document上挂事件委托，而是挂到 root节点上。

**对react fiber的理解**

Stack Reconciler 是一个同步的递归过程，使用的是 JavaScript 引擎自身的函数调用栈，它会一直执行到栈空为止，所以当 React 在渲染组件时，从开始到渲染完成整个过程是一气呵成的。如果渲染的组件比较庞大，js 执行会占据主线程较长时间，会导致页面响应度变差。
而且所有的任务都是按照先后顺序，没有区分优先级，这样就会导致优先级比较高的任务无法被优先执行。

react fiber对任务的执行采用时间分片式的策略处理，主要利用的是requestIdleCallback的原理,该api会在线程空闲的时候执行相关的任务。如果设置了timeout，那么到时的任务就会被强制执行，在react中利用messagechannel模拟回调到最后一针执行，用了setTimeOut作为降级方法。时间切片的作用是在执行任务的过程中，不阻塞用户的交互和需要执行的代码，目前单位切片的时间为5ms，一个真中会有多个时间切片，切片时间不会与帧对齐。fiber的本质是个链表，包括chile、sibiling、return、stateNode，多个fIber会组成fiberTree，由于链表的特性，可以快速的找到打断的任务，然后重新执行。fiber更新组件分发reconcile阶段，和commit阶段。reconcile阶段是在render之前，这个阶段的任务是可以被打断的如果有优先级和到时的任务会优先执行，当执行完优先级别较高的任务后会回到原来的任务重新执行，所以这个阶段谁先执行由调度器决定,而commit阶段是在render之后对的，是不可以打断的，会依次执行。

react的调度过程

- 初始化调度之前先判读有没有同步任务，有则立即执行，如果没有则调用ensureRootIsScheduled进行初始化调度,调度的初始化首先会根据相关的规则进行调度
- - 判断有没有过期任务，有过期任务立即执行。
- - 没有新任务则立即退出调度
- - 有旧任务，则与旧任务比较优先级和过期时间。过期时间相同，旧任务优先级较高，则推出调度。过期时间不同，新任务的优先级较高会取消旧任务
- - 根据expirationTime执行调度
react的调度会调用unstable_scheduleCallback
- 维护了timerQueue和TaskQueue,TaskQueu用来存储等待被调度的任务、timerqueue用来存储调度中的任务。
- scheduleCallback提供了两个参数，一个是delay表示任务的 超时 时长；、一个是timeout任务的过期时长,如果没有指定，根据优先程度任务会被分配默认的 timeout 时长。
- 如果没有提供 delay，则任务被直接放到 taskQueue 中等待处理；
- 如果提供了 delay，则任务被放置在 timerQueue 中
- 如果 taskQueue 为空，且当前任务在 timerQueue 的顶部（当前任务的超时时间最近），则使用 requestHostTimeout 启动定时器（setTimeout），在到达当前任务的超时时间时将这个任务转移到 taskQueue 中。
- 时间分片每 5 毫秒检查一下，运行一次异步的 MessageChannel 的 port.postMessage(…)方法，检查是否存在事件响应、更高优先级任务或其他代码需要执行；如果有则执行，如果没有则重新创建工作循环，执行剩下的工作中 Fiber。



优先级 从高到底 ，0-5
（1）No Priority，初始化、重置 root、占位用；
（2）Immediate Priority(-1)，这个优先级的任务会立即同步执行, 且不能中断，用来执行过期任务；
（3）UserBlocking Priority(250ms)， 会阻塞渲染的优先级，用户交互的结果, 需要及时得到反馈
（4）Normal Priority(5s)， 默认和普通的优先级，应对哪些不需要立即感受到的任务，例如网络请求
（5）Low Priority (10s)， 低优先级，这些任务可以放后，但是最终应该得到执行. 例如分析通知
（6）Idle Priority(没有超时时间)，空闲优先级，用户不在意、一些没必要的任务 (e.g. 比如隐藏的内容), 可能会被饿死；

不采用window.requestIdleCallback的原因：window.requestIdleCallback兼容情况一般。RequestIdleCallback 不重要且不紧急的定位。因为React渲染内容，并非是不重要且不紧急。不仅该api兼容一般，帧渲染能力一般，也不太符合渲染诉求，故React 团队自行实现。

为什么不使用 rAF()
如果上次任务调度不是 rAF() 触发的，将导致在当前帧更新前进行两次任务调度。
页面更新的时间不确定，如果浏览器间隔了 10ms 才更新页面，那么这 10ms 就浪费了

**为什么使用宏任务：**

核心是将主进程让出，将浏览器去更新页面。利用事件循环机制，在下一帧宏任务的时候，执行未完成的任务，如果是微任务，对一个事件循环机制来说，在页面更新前，会将所有的微任务全部执行完，故无法达成将主线程让出给浏览器的目的。

**既然用了宏任务，那为什么不使用 setTimeout 宏任务执行呢?**

如果不支持MessageChannel的话，就会去用 setTimeout 来执行，只是退而求其次的办法。
现实情况是: 浏览器在执行 setTimeout() 和 setInterval() 时，会设定一个最小的时间阈值，一般是 4ms。

React17
- 规划去掉不安全的生命周期，比如componentWillMount、componentWillReceiveProp、componentWillUpdate。
- concurrent mode 并发模式，Concurrent Mode的并发是指多个Task跑在同一个主线程（JS主线程）中，只不过每个Task都可以不断在“运行”和“暂停”两种状态之间切换，从而给用户造成了一种Task并发执行的假象。并发模式主要由下面两部分组成基于 fiber 实现的可中断更新的架构，通过调度器的优先级调度，快速响应用户操作和输入，提升用户交互体验，让动画更加流畅，通过调度，可以让应用保持高帧率，利用好 I/O 操作空闲期或者 CPU 空闲期，进行一些预渲染。 比如离屏(offscreen)不可见的内容，优先级最低，可以让 React 等到 CPU 空闲时才去渲染这部分内容。
- - 用 Suspense和useTransition 降低加载状态(load state)的优先级，减少闪屏。 在加载异步数据时，Suspense所包裹的子组件不会立即挂起，而是尝试在当前状态继续停留一段时间，这个时间由timeoutMs指定。如果在timeoutMs之内，异步数据已经加载完成，那么子组件就会直接更新成最新状态；反之，如果超过了timeoutMs，异步数据还没有加载完成，那么才会去渲染Suspense的fallback组件。

**react18 的更新点**

- Root API 更新之前是ReactDOM.render(<App />, container);现在ReactDOM.createRoot(container).render(<App />);可以为一个 React App 创建多个根节点，甚至在未来可以用不同版本的 React 来创建。React18 保留了上述两种用法，老项目不想改仍然可以用 ReactDOM.render() ；新项目想提升性能，可以用 ReactDOM.createRoot() 借并发渲染的东风。

- .startTransition API（用于非紧急状态更新）
UI 更新分紧急和不紧急，给不紧急的加 startTransition(() => {})，剩下更多资源留给紧急更新，可以让渲染更顺畅；
- 渲染的自动批处理 Automatic batching 优化，主要解决异步回调中无法批处理的问题
- SSR的时候，不用等 HTML 全部返回来就可以渲染，支持 Suspense 组件。
- v18 的 Strict Mode，由于包含了 Strict Effect 规则，mount 时的 useEffect 逻辑会被重复执行。



**setState 是同步还是异步**

setState 只在合成事件和钩子函数中是“异步”的，在原生事件和 setTimeout 中都是同步的。
setState的“异步”并不是说内部由异步代码实现，其实本身执行的过程和代码都是同步的，只是合成事件和钩子函数的调用顺序在更新之前，导致在合成事件和钩子函数中没法立马拿到更新后的值，形式了所谓的“异步”，当然可以通过第二个参数 setState(partialState, callback) 中的callback拿到更新后的结果。
setState 的批量更新优化也是建立在“异步”（合成事件、钩子函数）之上的，在原生事件和setTimeout 中不会批量更新，在“异步”中如果对同一个值进行多次 setState ， setState 的批量更新策略会对其进行覆盖，取最后一次的执行，如果是同时 setState 多个不同的值，在更新时会对其进行合并批量更新。

**setState的过程**

1.将setState传入的partialState参数存储在当前组件实例的_pendingStateQueue中。
2.判断当前React是否处于批量更新状态，如果是，将当前组件标记为dirtyCompontent,并加入待更新的组件队列中。
3.如果未处于批量更新状态，将isBatchingUpdates设置为true，用事务再次调用前一步方法，保证当前组件加入到了待更新组件队列中。
4.调用事务的waper方法，遍历待更新组件队列依次执行更新。
5.执行生命周期componentWillReceiveProps。
6.将组件的state暂存队列中的state进行合并，获得最终要更新的state对象，并将_pendingStateQueue置为空。
7.执行生命周期shouldComponentUpdate，根据返回值判断是否要继续更新。
8.执行生命周期componentWillUpdate。
9.执行真正的更新，render。
10.执行生命周期componentDidUpdate。




react的useState和Class中的state有什么区别
首先class中的state是immutable的，得通过setState去修改，会产生一个新的引用，可以通过this.state获取新的数据。
useState产生的数据也是immutable的，通过数组的第二个参数修改值。在下次渲染时，原来的值会产生一个新的引用。它的本质是闭包。
两者的状态值都会挂载到FiberNode的memorizeState中。但两者的数据结构是不相同的，类组件直接把state挂载到到memorizeState中，而hook是以链表的形式保存的，memorizeState是链表的头部。

useEffect会在初始化的时候以链表的形式挂载到FiberNode的updateQueue中，链表的节点属性有tag(用来标识依赖项有没有改变)，create(用户用useEffect传入的函数体)，destroy(上树函数执行后生成的用来清除副作用的函数)，deps(依赖选项表)，next(下一个节点)。组件渲染完成后，依次调用链表的effect执行
在update阶段，同样会依次调用useEffect语句，此时会判断传入的依赖列表，与链表节点的deps中保存的是否一致，如果一致那么就会在tag上标记NoHookEffect。组件渲染完成后就会进入useEffect的执行阶段。首先会遍历链表，如果遇到tag为NoHookEffect的节点就会跳过，如果destroy为函数类型，那么就执行清除副作用的函数，执行create，并将执行结果保存到destroy中。所以整体的流程是先清除上一轮的 effect，然后再执行本轮的 effect

react的优化：
- 使用React.memo缓存组件，这样只有当传入组件的状态只发生变化时才会重新渲染，如果传入的只和上一次没有发生变化，则返回缓存的组件：
- 使用useMemo和useCallback缓存相关的计算结果。
- memo 仅针对函数组件，对于 class 组件，我们可以使用 PureComponent 或者是自己书写 shouldComponentUpdate 是否需要重新渲染当前组件。
- 避免使用内联对象，在 JSX 中创建一个内联对象的时候，每次重新渲染都会重新生成一个新的对象，如果这里还存在了引用关系的话，会大大增加性能损耗，所以尽量避免使用内联对象
- 避免使用匿名函数，匿名函数可以更加方便的对函数进行传参，但是同内联对象一样，每一次重新渲染都会生成一个新的函数，所以我们应该尽量避免使用内联函数。
- 运用react.lazy延迟加载不必要的组件

useState和useReducer都是关于状态值的提取和更新，从本质上来说没有什么区别，背后都是一套逻辑。可以看做useState是useReducer的简化版。

**对react-hook的理解**
react-hook是16.8后的特性，主要解决了以下几个问题：组件状态逻辑复用的问题，在class组件中如果要共同状态逻辑需要通过高阶组件，这种方式比较繁琐。第二就是class中的this指向问题。第三就是难以记忆的生命周期。react在维护hook是以链表的形式进行维护的，每一个节点都是hook，当使用hook事会创建hook对象，hook对象包含了函数式组建的状态、缓存、计算只等，而单单有hook对组件事没有用的，它还需要用Fiber于对应的组件关联起来，所以多个hook是以类似链表的形式进行串联的，而不是数组。这也是为什么react hook不能再循环、条件语句中使用，同时react-hook不能相互嵌套。react hook有以下几个:

- useState：返回创建的状态和修改状态的函数
- useEffect：用来处理副作用的函数。主要有以下几种用法，第一就是在没有依赖的情况下，就是第二个参数为空数组，这个时候类似于componentDidMount，调用ajax。第二就是有依赖的情况下，当依赖发送变化后会触发。第三就是函数中返回一个函数，返回的函数会在卸载阶段执行
- useContext:useContext使用createContext创建的上下文
- useRef：创建ref对象，current属性指向初始化值
- useImperativeHandle:子组件暴露给父组件能通过ref调用的属性
- useMemo:创建一个记忆的纯函数，返回一个值。接受两个参数，第一个是函数，函数返回的值即useMemo返回的值。第二个是依赖，当依赖发生变化会触发函数。
- useCallback：类似于useMemo。但是它返回的是函数，接受的参数，当依赖的值变化后就会更改。
- useReducer:useState的替代方案，接受类型为（state,action）=>newState的的reducer。
- useLayoutEffect:当dom更新的时候会触发。
- useMutationEffect:当兄弟发生更新，react在执行改变其dom的时候会触发。

**使用react-hook的缺点**

- 必须清楚的知道useEffect和useCallback的依赖数组的改变时机，有时候依赖数组可能依赖某个函数的不可变性，这个函数又依赖于其他函数。这样就会形成一个依赖链，一旦某个节点发生改变，useEffect就会意外触发，如果useEffect的操作是幂等的，可能带来的是性能问题，如果不是幂等的，那就糟糕了。所有相对于componentDidMount和componentDidUpdate，useEffect的维护成本可能更高。
- hooks不擅长异步代码，函数的运行时独立的作用域，当我们有异步操作的时候，引用的状态是之前的，不是最新的状态。

怎么解决hook的这些问题
- useEffect不要写太多的依赖项，遵循单一职责模式
- 如果遇到异步状态不同步，那么可以考虑传参的模式

如果组件比较复杂，或者使用了较多的异步。考虑到代码的维护性和可读性，我们可以使用class component

**useState和useReducer的区别**

useState和useReducer都是管理状态的hook,許多情況直接用 useState 會比要建一個 useReducer 更簡單。而useReducer跟希望我们像redux去管理状态，通过dispatch action去改变store的状态。当我们如果是简单的管理某个组件的state的时候可以用useState,如果state比较复杂，可以考虑使用useReducer，比如我们在做条件的联合查询的时候可以考虑使用useReducer。

实现一个useReducer

```

let lastState

function useReducer(reducer,initialState){
    lastState = lastState || initialState
    function dispatch(action){
      lastState = reducer(lastState,action)
      render()
    }
    return [lastState,dispatch]
}

function reducer(state,action){
  switch(action.type){
    case: 'increment':
    return {count: state.count + 1};
    default:
     throw new Error();
  }
}

```



**react-router**

在单页应用中，一个web项目只有一个html页面，一旦页面加载完成之后，就不用因为用户的操作而进行页面的重新加载或者跳转,本质是改变 url 且不让浏览器像服务器发送请求且刷新页面，主要分成了两种模式：hash模式和history 模式，React Router对应的hash模式和history模式对应的组件为：HashRouter，BrowserRouter。
主要原理是路由描述了 URL 与 UI之间的映射关系，这种映射是单向的，即 URL 变化引起 UI 更新（无需刷新页面），以hash模式为例子，改变hash值并不会导致浏览器向服务器发送请求，浏览器不发出请求，也就不会刷新页面，hash 值改变，触发全局 window 对象上的 hashchange 事件。所以 hash 模式路由就是利用 hashchange 事件监听 URL 的变化，从而进行 DOM 操作来模拟页面跳转。History 模式通过重写 a 标签的点击事件，阻止了默认的页面跳转行为，并通过 history API 无刷新地改变 url，最后渲染对应路由的内容。 

H5 引入的 pushState()和 replaceState()、back方法等，能够让我们在不刷新页面的前提下，修改 URL，并监听到 URL 的变化，为 history 路由的实现提供了基础能力。

Link和a标签的区别，link如果有onClick事件,那么会优先执行click。同时它会阻止a标签的默认事件，使用history的api，无刷新的改变url,渲染对应路由组件的内容。





**实现useCallback**

```
let lastFn,lastDependencies;

function useCallback(fn,dependencies) {
  if(lastDependencies){
    const changed=!dependencies.every((item,index)=>{
      return item===lastDependencies[index];
    })
    if(changed){
      lastFn=fn
      lastDependencies=dependencies
    }
  }else{
    lastFn=fn,
    lastDependencies=dependencies
  }
}

```


**suspense的原理**

suspense依赖于componentDidCatch,如果某个组件定义了 componentDidCatch，那么这个组件中所有的子组件在渲染过程中抛出异常时，这个 componentDidCatch 函数就会被调用，捕获到错误时通过setState重新渲染，并移除失败的组件。componentDidCatch原理内部通过try{}catch(error){}来捕获渲染错误，然后处理渲染错误。当加载异步组件的时候，Suspense 就是用抛出异常的方式中止的渲染，Suspense 会封装异步createFetcher，当尝试从 createFetcher 返回的结果读取数据时，有两种可能：一种是数据已经就绪，那就直接返回结果；还有一种可能是异步操作还没有结束，数据没有就绪，这时候 createFetcher 会抛出一个“异常”，捕捉到异常后加载loading组件。

```
Class Suspense extend React.component{
  state={
    loading:false
  }
  componentDidCatch(e){
   if(e instancof Promise){
     this.setState({
       loading:true
     },()=>{
       e.then(()=>{
         this.setState({
           loading:false
         })
       })
     })
   }
  }

  render(){
    const {children,fallback}=this.props;
    const {loading}=this.state;
    return loading?fallback:children
  }
}

```


**react 的生命周期**

- 加载
- - Constructor
- - getDeriedStateFromProps static方法，拿到nextProps和preState方法，返回一个对象更新state，返回null不更新
- - render
- - componentDidMount
  
- 更新
- - getDeriedStateFromProps
- - shouldComponentUpdate
- - render
- - getsnatshotbeforeupdate 在真正渲染之前可以拿到dom信息，比如滚动。返回的值会传递给componentdidupdate
- - componentDidUpdate

- 卸载
- componentWillUnmount


- 父子组件初始化
父子组件第一次进行渲染加载时：
控制台的打印顺序为：

Parent 组件： constructor()
Parent 组件： getDerivedStateFromProps()
Parent 组件： render()
Child 组件： constructor()
Child 组件： getDerivedStateFromProps()
Child 组件： render()
Child 组件： componentDidMount()
Parent 组件： componentDidMount()

- 修改父组件中传入子组件的 props
点击父组件中的 [改变传给子组件的属性 count] 按钮，则界面上 [父组件传过来的属性 count] 的值会 + 1，控制台的打印顺序为：
Parent 组件： getDerivedStateFromProps()
Parent 组件： shouldComponentUpdate()
Parent 组件： render()
Child 组件： getDerivedStateFromProps()
Child 组件： shouldComponentUpdate()
Child 组件： render()
Child 组件： getSnapshotBeforeUpdate()
Parent 组件： getSnapshotBeforeUpdate()
Child 组件： componentDidUpdate()
Parent 组件： componentDidUpdate()

- 卸载子组件
点击父组件中的 [卸载 / 挂载子组件]  按钮，则界面上子组件会消失，控制台的打印顺序为：
Parent 组件： getDerivedStateFromProps()
Parent 组件： shouldComponentUpdate()
Parent 组件： render()
Parent 组件： getSnapshotBeforeUpdate()
Child 组件： componentWillUnmount()
Parent 组件： componentDidUpdate()

- 重新挂载子组件
Parent 组件： getDerivedStateFromProps()
Parent 组件： shouldComponentUpdate()
Parent 组件： render()
Child 组件： constructor()
Child 组件： getDerivedStateFromProps()
Child 组件： render()
Parent 组件： getSnapshotBeforeUpdate()
Child 组件： componentDidMount()
Parent 组件： componentDidUpdate()





**useCallback和useMemo的区别和应用场景以及useMemo**

useCallback主要用来缓存函数，减少创建事件具柄，主要的应用场景是父组件传递函数给子组件或者创建自定义hook返回一个函数。useMemo用来缓存密集运算的结果，useMemo也是可以缓存一个函数的，但这样就相当于是一个闭包了，通常返回函数都是用useCallback。memo主要是将一个组件变成一个缓存组件，只有当组件的props发生变化的时候才会重新渲染，这样可以减少不必要的渲染。如果要让它失效，传递第二个参数进行手动比较，比较时候可以使用immer.js。
Immer.js，一个 Immutable(不可变数据) 库，不可变数据概念来源于函数式编程，每次更改都要创建一个新的“变量”。新的数据进行有副作用的操作都不会影响之前的数据。这也就是Immutable的本质。react为了加速diff 算法中reconcile阶段， 只检查object的索引有没有变即可确定数据有没有变化，更新某个复杂类型数据时只要它的引用地址没变，哪怕是属性值发生了变化，也不会重新渲染组件，使用purecomponentd的时候也是如此。所以通常我们在修改react的state的值的时候都会使用到迭代运算符，就是为了修改索引的地址重新进行渲染。但遇到深层对象的时候使用迭代运算符就比较麻烦，如果使用了深拷贝的话那么会造成额外的渲染，并且深拷贝也是一个比较耗费性能的操作。而使用immer.js能达到精确的替换某一个值，并且使用immer生成的对象与原来对象是结构共享的。immer的核心实现是利用ES6的proxy实现JavaScript的不可变结构，其基本思想在于所有的更改都应用在临时的draftState。一旦完成所有的变更，将草稿状态的变更生成nextState。这就通过简单的修改数据同时保留不可变数据的优点。

proxy的操作
- getter主要用来懒初始化代理对象，当代理对象的属性被访问的时候才会生成其代理对象
举个例子：当访问draft.a时，通过自定义getter生成draft.a的代理对象darftA所用访问draft.a.x相当于darftA.x，同时如果draft.b没有访问，也不会浪费资源生成draftB
setter:当对draft对象发生修改，会对base进行浅拷贝保存到copy上，同时将modified属性设置为true，更新在copy对象上

**useEffect和useLayoutEffect的区别**

useEffect 是异步执行的，useEffect 的执行时机是浏览器完成渲染之后，不会阻塞渲染，
而useLayoutEffect是同步执行的，执行时机是浏览器把内容真正渲染到界面之前，和 componentDidMount 等价。如果需要操作 dom ，我们需要放到 useLayouteEffect 中去，避免导致闪烁。


useRef： useRef 是一个使用相同 ref 的钩子。它在功能组件中的重新渲染之间保存其值，每次重新渲染引用不会发生改变。

createRef： createRef 是一个每次都会创建一个新 ref 的函数。与 useRef 不同，它不会在重新渲染之间保存其值，而是为每次重新渲染创建一个新的 ref 实例，多用于dom的引用。



**react diff**

react 的diff主要基于三种策略，第一种是react的操作不涉及到跨层级的操作，或者涉及到跨层级的操作很少。第二种是同一类型的组件结构是相同的，不同类型的组件结构是不同的。第三种是通过key值做唯一的标识。基于这三种策略，react分别对tree diff、component diff、element diff做了优化。首先tree diff基于第一种策略，react不涉及到跨层级的操作，这样通过遍历一次比较新老节点就能知道哪些节点有没被引用，如果没有被引用那么就删掉。如果涉及到跨层级的操作，比如A和B原来是兄弟节点，经过操作A变成了B的子节点需要把原来的节点删掉，然后在添加到B节点下面，这是一个比较耗费性能的操作，react也不建议这么操作。第二个就是相同类型的组件的结构是一样的，不同类型的组件的结构是不同的。假如能够知道react组件结构是否发生改变，如果不改变那么就不进行diff,这样就能节省大量的性能。所以react提供了shouComponentUpdate给用户判断是否需要更新。第三个就是element diff,通过唯一的表示key值用来判断节点是否能够复用，如果key值相同那么就对节点进行复用，必要的时候对节点进行移动，最后比较一下新老节点，将没有是用的老节点进行删除。












