# event模块解析

$.event:event核心模块，实际只暴露了两个函数：add,remove

$.fn.on:添加事件的入口,底层调用的是add，代码如下：

```javascript
$.fn.on = function(event, selector, data, callback, one){
    var autoRemove, delegator, $this = this
    if (event && !isString(event)) {
      $.each(event, function(type, fn){
        $this.on(type, selector, data, fn, one)
      })
      return $this
    }
    //处理各种输入
    //如果selector不是string并且callback不是function并且callback不为空
    if (!isString(selector) && !isFunction(callback) && callback !== false)
      callback = data, data = selector, selector = undefined
    if (callback === undefined || data === false)//如果callback为空或者data是false  on('click','.ss',function(){})
      callback = data, data = undefined

    if (callback === false) callback = returnFalse //如果callback是假，自动返回false

    return $this.each(function(_, element){ //循环数组，如果是一次性的，为autoRemove添加remove的function
      if (one) autoRemove = function(e){
        remove(element, e.type, callback)
        return callback.apply(this, arguments)
      }

      if (selector) delegator = function(e){ //添加delegator
        var evt, match = $(e.target).closest(selector, element).get(0)
        if (match && match !== element) {
          evt = $.extend(createProxy(e), {currentTarget: match, liveFired: element})
          return (autoRemove || callback).apply(match, [evt].concat(slice.call(arguments, 1)))
        }
      }

      add(element, event, callback, data, selector, delegator || autoRemove)
    })
  }
```

add：添加一个事件，是Bind,on等所有方法的底层实现

```javascript
function add(element, events, fn, data, selector, delegator, capture){
    var id = zid(element), set = (handlers[id] || (handlers[id] = []))//设置zid，读取缓存池
    events.split(/\s/).forEach(function(event){ //对多个Event事件进行分割
      if (event == 'ready') return $(document).ready(fn) //如果是ready事件，调用$.ready
      var handler   = parse(event) //解析event，返回形如{e: * 事件类型 , ns: string 命名空间}
      handler.fn    = fn
      handler.sel   = selector
      // emulate mouseenter, mouseleave
      if (handler.e in hover) fn = function(e){ //处理mouseenter, mouseleave事件，需要在移动中运行回调函数
        var related = e.relatedTarget
        if (!related || (related !== this && !$.contains(this, related)))
          return handler.fn.apply(this, arguments)
      }
      handler.del   = delegator  //设置代理
      var callback  = delegator || fn //如果委托存在，则设置委托为回调函数
      handler.proxy = function(e){
        e = compatible(e) //拓展event对象，代理preventDefault stopImmediatePropagation stopPropagation等方法，兼容浏览器
        if (e.isImmediatePropagationStopped()) return
        e.data = data
        var result = callback.apply(element, e._args == undefined ? [e] : [e].concat(e._args))  //执行回调函数，context：element，arguments：event,e._args(默认是undefind，trigger()时传递的参数）
        if (result === false) e.preventDefault(), e.stopPropagation()  //返回false时，阻止默认操作和冒泡
        return result
      }
      handler.i = set.length
      set.push(handler) //加入缓存
      if ('addEventListener' in element)
        element.addEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture)) //添加事件
    })
  }
```

$.fn.off:删除事件入口，底层调用remove方法，基本形式和On类似

remove:删除一个事件监听

```javascript
 function remove(element, events, fn, selector, capture){
    var id = zid(element) //获得Id
    ;(events || '').split(/\s/).forEach(function(event){
      findHandlers(element, event, fn, selector).forEach(function(handler){
        delete handlers[id][handler.i] //从缓存中删除
      if ('removeEventListener' in element) //调用removeEventListener
        element.removeEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
      })
    })
  }

```
