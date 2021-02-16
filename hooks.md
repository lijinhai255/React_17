
### hooks 源码分析
1. 为什么会有hooks？解决了什么样的问题
class Component为主，function Component为辅， 只能通过props把state传递过来。
function Component 做UI组件。没有定义state的能力。
function Component + hooks  可以做到更简洁
可以做到更方便的抽取逻辑
2. hooks 的基本用法
   useContext + useReducer + createContext = redux
   useEffect: 模拟生命周期 componentDidMount/didUpdate/willUnMount
   useMemo： 缓存一个值
   useCallback： 缓存一个function
   useRef ： ref 在function里使用
   useState: 定义state和改变state的方式
   hooks常见的几大问题：
      0. 使用useState只能在顶层使用，不能出现在条件语句里.
      1. 死循环
      2. capture value   通过ref来解决。
   ```js
      function Test(){
         const [ count, setCount ] = useState(() => {
            return 0
         });
         const [ age, setAge ] = useState(0);
         const [ name, setName ] = useState(0);
         const [ age1, setAge1 ] = useState(0);
         const nameRef = useRef(null)
         const ref1 = useRef(0)
         // useMemo 是缓存一个值
         // 1. 每次都会被重新创建, 消耗浏览器资源
         // 2. 父组件任何的变化都会导致子组件的更新
         const addCount = useCallback(() => {
            // 既可以传值，也可以传函数
            setCount((count)=>{
              return count++;
            })
            console.log(age1);
            ref1.current = count++;
            // setTimeout(() => {
               // 还是保存的之前的变量
               // console.log(ref1.current)
            // }, 3000)
         }, [age1])
         // 每运行一次，都是创建新的函数
         const addCount1 = () => {
            // 既可以传值，也可以传函数
            setCount((count)=>{
              return count++;
            })
            ref1.current = count++;
            // setTimeout(() => {
               // 还是保存的之前的变量
               // console.log(ref1.current)
            // }, 3000)
         }

         // componentDidUpdate
         useEffect(() =>{
            const onScroll = () => {}
            window.addEventListener('scroll', onScroll);
            // distory
            return () => {
                window.removeEventListener('scroll', onScroll);
            }
         }, [age])

         useEffect(() =>{
            1. 有判断逻辑，用ref来做
            if(nameRef.current){
                // 去除死循环的目的
            }
            request().then(res => {
                setName(name + '_dd');
                // nameRef.current = name + '_dd';
                setName(name => name + '_dd')
                // 1. 会导致name变化
                // 2. name变化，导致useEffect的函数体会重新执行一次
                // 3. 又会导致name变化
            })
         }, [])

         // 1. 不写依赖数组， useEffect的回调每次渲染都会执行
         // 2. age变化的时候， useEffect的回调才会执行
         // 3. 可以返回一个函数，在函数销毁的时候会调用它
         // componentDidMount
         useEffect(() =>{
            unstable_batchedUpdates(() => {
                setAge();
                setAge1();
                setName();
            });
         }, [])

         // componentDidUpdate
         useEffect(() =>{
            if(age === 0) return;
            const onScroll = () => {}
            window.addEventListener('scroll', onScroll)
            // 组件销毁的时候，会调用这个useEffect的回调
            return () => {
               window.removeEventListener('scroll', onScroll)
            }
         }, [age])
         return <div>
         <div className='wrapper' ref={ref}>{count}</div>
         <div className='btn' onClick={addCount1}>点击</div>
            <Child addCount={addCount} />
            <Child addCount={addCount1} />
         </div>
      }

      
      const Child = ({addCount}) => {
         return <div onClick={addCount}></div>
      }
      // 对Child组件的props做了一次浅比较 React.memo + useCallback
      export React.memo(Child);

   ```

### 简版的实现
```js
    const [name, setName] = useState('yideng');
    const [name, setName] = useState('yideng2');
    const [name, setName] = useState('yideng3');
    第一次执行函数体： name = 'yideng';
    setName: setName('yideng1');
    第二次执行函数体: name = 'yideng1';

    let state
    function useState(defaultState) {
        function setState(newState) {
            state = newState;
            schedule();
        }

        if (!state) {
            state = defaultState
        }
        return [state, setState]
    }
    1. 只能定义一个state
    2. 没有上下文，只能在一个函数里使用
    function functionA() {
        const [name, setName] = useState(0);
        console.log(name)
        setState(name + 1)
    }
    function functionB() {
        const [name, setName] = useState(0);
        console.log(name)
        setState(name + 1)
    }
```
状态堆与上下文栈存储多个状态多个函数
```js

let contextStack = []

function useState(defaultState=0) {
  const context = contextStack[contextStack.length - 1]
  const nu = context.nu++

  const { states } = context

  function setState(newState) {
    states[nu] = newState
    states[0]
    states[1]
    states[2]
  }

  if (!states[nu]) {
    states[nu] = defaultState
  }

  return [states[nu], setState]
}

function withState(func) {
  const states = {}
  return (...args) => {
    contextStack.push({ nu: 0, states })
    const result = func(...args)
    contextStack.pop()
    return result
  }
}

1. 压入一个 contextStack 1
2. useState
const function1 = withState(
  function render() {
    const [state, setState] = useState(0)
    const [name, setName] = useState("name")
    render1()
    console.log('render', state)
    setState(state + 1)
  }
)
3.  contextStack 2
2. 
const function2 = withState(
  function render1() {
    const [state, setState] = useState(0)
    console.log('render1', state)
    setState(state + 2)
  }
)

function1()
```

useEffect
```js
// 每一个state变化的时候，useEffect的函数体都会重新执行
useEffect(() => {
   
}, [age]);
function useEffect(callback, depArray=[]) {
  const hasNoDeps = !depArray;
  const deps = memoizedState[cursor];
  const hasChangedDeps = deps
    ? !depArray.every((el, i) => el === deps[i])
    : true;
  if (hasNoDeps || hasChangedDeps) {
    callback();
    memoizedState[cursor] = depArray;
  }
  cursor++;
}
```

### hooks源码分析
## useState
const [state, setState] = useState(0)
useEffect(() => {
    setState(3);
    setState(() => 4);
    setTimeOut(() => {
        setState(5);
    }, 0)
})

this.setState => scheduleUpdateOnFiber
ReactDom.render => scheduleUpdateOnFiber 
1. MountState
    1. 默认值是function，执行function，得到初始state
    2. state是存放在memoizedState
    3. 新建一个queue
    4. 把queue传递给dispatch
    5. 返回默认值和dispatch
2. dispatchAction
   1. 创建一个update
   2. update添加到quene里
   3. 空闲的时候：提前计算出最新的state，保存在eagerState
   4. 最后调用一次scheduleUpdateOnFiber，进入schedule，触发function重新执行一次
3. UpdateState
   1. 递归执行quene里的update
   2. 计算最新的state，赋值给memoizedState
## useEffect
初始化：
在创建Fiber的时候：
    1. 函数体 会执行
    2. 执行useEffect本身的逻辑
        a. 打上flags标记
        b. push一个Effect
在commit阶段的LayoutEffects阶段：
    3. 执行commitHookEffectListMount
    4. 执行了create方法，有返回值 返回给destory.
有state变化发生的时候：
    5. dispatchAction， 调用一次scheduleUpdateOnFiber
    6. 函数体 会执行
    UpdateEffect
    7. 给destroy赋值
    1. 对比依赖的值有没有变化，有变化的话，重新发起一个Effect. 依赖不变，不会发起Effect，执行不到callBack
    2. 打上flags标记
    3. push一个Effect
在commit阶段的LayoutEffects阶段：
    1. 调用commitHookEffectListUnMount的时候，会执行destory()
    2. 执行commitHookEffectListMount
    3. 执行了create方法，有返回值 返回给destory.


1. MountEffect
   1. 打上flags标记
   2. push一个Effect
// 执行useEffect本身的逻辑的时候，还在beginWork阶段，执行函数体
useEffect(() => {
    return () => {

    }
}, [])
2. UpdateEffect
  0. 给destroy赋值
  1. 对比依赖的值有没有变化，有变化的话，重新发起一个Effect
  2. 打上flags标记
  3. push一个Effect
useEffect回调的执行时机：
在LayoutEffects里调用commitHookEffectListMount方法，执行了create方法，返回给
destory.
在调用commitHookEffectListUnMount的时候，会执行destory();
