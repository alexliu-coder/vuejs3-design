## 响应式系统

### 简易响应式系统：

data => key => [fn, fn ...];

​		=> key2 => [fn, fn ...];

```js
let activeFn;

function effect(fn) {
  activeFn = fn;
  fn();
}

effect(() => {
  document.title = obj.title;
})

const bucket = new WeakMap();
const obj = new Proxy(data, {
  get(target, key) {
    let depsMap = bucket.get(target);
    if (!depsMap) {
      bucket.set(target, (depsMap = new Map()));
    }
    let deps = depsMap.get(key);
    if (!deps) {
      depsMap.set(key, (deps = new Set()));
    }
    deps.add(activeFn);
    return target[key];
  },
  set(target, key, newValue) {
    target[key] = newValue;
    const depsMap = bucket.get(target);
    if (!depsMap) return;
    const deps = depsMap.get(key);
    deps && deps.forEach(fn => fn())
  }
});

```



### 分支的切换和cleanup

```js
let fn = () => {
  document.title = obj.isValid ? obj.title : '';
}
```

当isValid为false时，title属性就不需要去出发副作用函数

如果每次执行副作用前，先把副作用函数从所有与之已关联的依赖集合中删掉

```js
let fn = () => {
  document.title = obj.isValid ? obj.title : '';
}

effect(fn);

const bucket = new WeakMap();
const obj = new Proxy(data, {
  get(target, key) {
		track(target, key)
    return target[key];
  },
  set(target, key, newValue) {
    target[key] = newValue;
		trigger(target, key)
  }
});

// 收集依赖的函数
function track(target, key) {
  let depsMap = bucket.get(target);
  if (!depsMap) {
    bucket.set(target, (depsMap = new Map()));
  }
  let deps = depsMap.get(key);
  if (!deps) {
    depsMap.set(key, (deps = new Set()));
  }
  deps.add(activeFn);
  activeFn.deps.push(deps); // 把当前的key对应的set(即依赖的集合)加入到函数的数组去，在cleanup中使用
}

// 刷新副作用的函数
function trigger(target, key) {
  const depsMap = bucket.get(target);
  if (!depsMap) return;
  const effects = depsMap.get(key);
  // 🌟调用fn的时候，fn内部有访问obj.key的代码，这时候又会触发track函数，将当前副作用收集
  // 但是每次调用deps内的fn会删除所有依赖，而触发track又会加入一个新的依赖，为了避免无限循环
  // 需要复制依赖
  const effectsToRun = new Set(effects);
  effectsToRun.forEach(effect => effect())
  // effects && effects.forEach(fn => fn())
}

function effect(fn) {
  const effectFn = () => {
    cleanup(effectFn);
    activeFn = effectFn;
    // 🌟此处调用fn会触发obj的get函数
    fn();
  }
  effect.deps = [];
  effectFn();
}

function cleanup(effectFn) {
  for (let i = 0; i < effectFn.deps.length; i++) {
    const deps = effectFn.deps[i];
    deps.delete(effectFn);
  }
  effectFn.deps.length = 0;
}
```



### effect嵌套

```js
let activeFn;
const effectsStack = [];
function effect(fn) {
  const effectFn = () => {
    activeFn = effectFn;
    effectsStack.push(effectFn);
    fn();
    effectsStack.pop();
    activieFn = effectsStack[effectsStack.length - 1];
  }
  effectFn.deps = [];
  effectFn();
}
```



### 避免无限递归循环

```js
effect(() => {
  obj.count = obj.count + 1
});

如果一个属性自增的时候，首选会读取obj.count，会触发get函数收集依赖，把当前传入的箭头函数收集，get读取完以后优惠触发set，会把刚刚收集到的箭头函数再执行，再执行的过程中又会触发get，也就是在set的过程中，执行的函数增多了，最终会栈溢出

function trigger(target, key) {
  const depsMap = bucket.get(target);
  if (!depsMap) return;
  const effects = depsMap.get(key);
  const effectsToRun = new Set();
  
  effects && effects.forEach(effect => {
    if (effect !== activeFn) {
      effectsToRun.add(effect)
    }
  });
  
  effectsToRun.forEach(effect => effect());
}
```



### effect的配置options

