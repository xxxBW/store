写在前面：平常工作的项目是用的是seajs来做的模块化，于是我就看了一下seajs的源码，看看seajs到底是怎么实现模块化的~

## 为什么要用模块化？
* Q : 模块化写起来那么麻烦，要多写那么多代码，为什么我们还要用呢？
* A : 使用模块化当然不是为了方便，而是为了更好的维护，当一个项目小的时候可能会觉得没必要，但是当一个项目大的时候，模块化可以让我们更好的解决模块问题，模块之间不互相影响，如果需要依赖模块也会比较清晰，后期维护方便。

## 首先我们来看看seajs中定义的模块状态
suatus      | description
------------|------------
FETCHING: 1 | 模块正在请求中
SAVED: 2    | 模块已经下载完毕
LOADING: 3  | 模块依赖准备中
LOADED: 4   | 模块依赖准备完毕
EXECUTING: 5| 模块正在执行
EXECUTED: 6 | 模块执行完毕

## 几个全局变量
suatus      | description
------------|------------
cachedMods（uri : mod）| 所有模块信息
fetchingList（uri : true）|正在请求的模块队列
fetchedList（uri : true）|请求完毕的模块队列
callbackList（uri : mod）|需要回调的模块队列

## 一个模块信息
suatus      | description
------------|------------
uri           |    模块的uri
dependencies   |   模块的依赖模块
exports      |     模块返回值
status      |      模块状态
_waitings     |    依赖该模块的模块数
_remain       |    没有准备完的依赖模块数

## 下面我们一步一步来看看seajs模块到底是怎么加载的
### 1.引入seajs文件
### 2.使用seajs.config定义base、alias、path
### 3.使用seajs.use(ids,callback)
seajs.use方法调用了Module.preload(callback)这个方法，在这里传入的callback是Module.use(ids, callback, uri)这个方法
### 4.调用Module.preload(callback)
这个函数的作用是用来预加载的，判断config的时候有没有需要预加载的模块，有的话进行预加载操作，没有则直接调用callback
### 5.调用Module.use(ids, callback, uri)
这里调用Module.get来获取模块实例，并且设置了当前模块的callback，然后对该模块调用load方法，如果有传入callback方法，则用apply使该方法在global下调用
### 6.调用Module.prototype.load()
把当前模块的状态设为LOADING: 3
在这里首先获取该模块的所有依赖，遍历依赖模块，如果依赖模块还没请求，则调用Module.prototype.fetch(requestCache)请求依赖模块，如果依赖模块数为0，则该模块执行onload操作
### 7.Module.prototype.fetch(requestCache)
把当前模块的状态设为FETCHING: 1
这里是请求操作，这里调用了request函数，发起HTTP请求请求当前模块，这里设置了onRequest函数，请求完毕之后调用onRequest函数，这里调用了Module.save(uri, meta)，保存模块信息之后看看callbackList的模块，再调用load方法，回到步骤6
### 8.Module.save(uri, meta)
把当前模块的状态设为SAVED: 2
这里把当前模块的信息保存到cachedMods对象中
### 9.Module.prototype.onload
把当前模块的状态设为LOADED: 4
执行到这里的时候，表示当前模块的依赖全部加载完毕或者该模块没有依赖模块，如果该模块有callback函数（Module.use设置的），则调用callback函数，获取依赖该模块的模块，然后遍历，设当前模块为mod，依赖该模块的模块为m，m的_remain等于m的_remain减去mod的_waitings，如果m的_remain为0，表示m模块的依赖全部加载完毕，执行onload操作，又再一次执行步骤9
### 10.mod.callback
这里是模块的回调函数，调用Module.use的时候就会给模块设置callback函数，这里遍历当前模块的依赖模块，然后调用Module.prototype.exec执行依赖模块
### 11.Module.prototype.exec
把当前模块的状态设为EXECUTING: 5
设置了require方法，判断mod.factory是不是一个函数，如果是则调用该方法，并且传入(require,exports,module),如果factory中需要require其他方法时会调用这里设置的require方法，获取该模块并执行exec方法，又再一次执行步骤11
把当前模块的状态设为EXECUTED: 6
到这里，所有模块加载并且执行结束
