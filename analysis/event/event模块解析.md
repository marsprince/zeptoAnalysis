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

$.Event:创建一个Event对象，

```javascript
  $.Event = function(type, props) {
    if (!isString(type)) props = type, type = props.type //传入参数处理 可以只传一个对象，将type作为对象的一个属性
    var event = document.createEvent(specialEvents[type] || 'Events'), bubbles = true //创建一个event对象，如果是click,mouseover,mouseout时，创建的是MouseEvent,bubbles为是否冒泡
    if (props) for (var name in props) (name == 'bubbles') ? (bubbles = !!props[name]) : (event[name] = props[name]) //如果props存在，将props属性复制到events里
    event.initEvent(type, bubbles, true) //初始化事件
    return compatible(event)
  }
```

$.fn.trigger:触发一个事件

```javascript
$.fn.trigger = function(event, args){
    event = (isString(event) || $.isPlainObject(event)) ? $.Event(event) : compatible(event) //如果传入event是一个对象或字符串，初始化一个事件
    event._args = args
    return this.each(function(){
      // handle focus(), blur() by calling them directly
      if (event.type in focus && typeof this[event.type] == "function") this[event.type]() //如果事件是focus，blur，直接执行
      // items in the collection might not be DOM elements
      else if ('dispatchEvent' in this) this.dispatchEvent(event) //如果原生支持dispatchEvent，则执行
      else $(this).triggerHandler(event, args)//否则触发模拟事件
    })
  }
```

我们看一下这个模拟事件：

```javascript
// triggers event handlers on current element just as if an event occurred,
  // doesn't trigger an actual event, doesn't bubble
  $.fn.triggerHandler = function(event, args){
    var e, result
    this.each(function(i, element){
      e = createProxy(isString(event) ? $.Event(event) : event)
      e._args = args
      e.target = element
      $.each(findHandlers(element, event.type || event), function(i, handler){//查找元素上所有响应函数，调用findHandlers
        result = handler.proxy(e) //proxy是在注册事件的时候加载上去的，主要触发事件，不会触发浏览器默认的操作
        if (e.isImmediatePropagationStopped()) return false
      })
    })
    return result
  }
```

$.proxy:类似于apply，新函数this为旧函数内的this

$.one:用法类似on，不过当第一次执行事件以后，该事件将自动解除绑定，保证处理函数在每个元素上最多执行一次，主要是on函数最后一个One参数确定的，添加了一个autoRemove的回调函数

click,focus等方法：主要是通过以下的代码循环实现

```javascript
// shortcut methods for `.bind(event, fn)` for each event type
  ;('focusin focusout focus blur load resize scroll unload click dblclick '+
  'mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave '+
  'change select keydown keypress keyup error').split(' ').forEach(function(event) {
    $.fn[event] = function(callback) {
      return (0 in arguments) ?
        this.bind(event, callback) : //调用Bind为fn添加绑定事件
        this.trigger(event)
    }
  })
```

