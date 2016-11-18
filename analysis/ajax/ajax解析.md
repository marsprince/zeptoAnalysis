# zepto模块——ajax解析

<h2>1.整体结构</h2>

标准的自执行函数，把zepto作为参数加载进去，前面加分号是为了避免压缩错误，主要为zepto添加了以下几个函数：

$.ajaxJSONP, $.ajaxSettings, $.ajax, $.get, $.post, $.getJSON, $.fn.load, $.param

<h2>2.函数解析</h2>

$.ajaxSettings 默认设置，不谈了

$.ajax：最主要的函数

```javascript
$.ajax = function(options){
    var settings = $.extend({}, options || {}),
        deferred = $.Deferred && $.Deferred(),
        urlAnchor, hashIndex
    for (key in $.ajaxSettings) if (settings[key] === undefined) settings[key] = $.ajaxSettings[key] //设置options

    ajaxStart(settings) //触发ajaxStart事件，调用了$.event，需要event模块支持

    /*
    是否设置了跨域，否则需要判断一下
    */
    if (!settings.crossDomain) {
      urlAnchor = document.createElement('a')
      urlAnchor.href = settings.url
      // cleans up URL for .href (IE only), see https://github.com/madrobby/zepto/pull/1049
      urlAnchor.href = urlAnchor.href
       //通过ip  协议 端口来判断跨域  location.host = host:port
      settings.crossDomain = (originAnchor.protocol + '//' + originAnchor.host) !== (urlAnchor.protocol + '//' + urlAnchor.host)
    }

    if (!settings.url) settings.url = window.location.toString() //未设置url则取当前地址栏
    if ((hashIndex = settings.url.indexOf('#')) > -1) settings.url = settings.url.slice(0, hashIndex) //如果有hash，截取掉，因为hash，ajax不会传递到后台
    serializeData(settings) //将get请求附加到url里。返回新的url

    var dataType = settings.dataType, hasPlaceholder = /\?.+=\?/.test(settings.url)
    if (hasPlaceholder) dataType = 'jsonp' //检测并设置为jsonp

    //缓存
    if (settings.cache === false || (
         (!options || options.cache !== true) &&
         ('script' == dataType || 'jsonp' == dataType)
        ))
      settings.url = appendQuery(settings.url, '_=' + Date.now())

    //如果是jsonp，走 ajaxJSONP
    if ('jsonp' == dataType) {
      if (!hasPlaceholder)
        settings.url = appendQuery(settings.url,
          settings.jsonp ? (settings.jsonp + '=?') : settings.jsonp === false ? '' : 'callback=?')
      return $.ajaxJSONP(settings, deferred)
    }

    var mime = settings.accepts[dataType],
        headers = { },
        setHeader = function(name, value) { headers[name.toLowerCase()] = [name, value] },
        protocol = /^([\w-]+:)\/\//.test(settings.url) ? RegExp.$1 : window.location.protocol,
        xhr = settings.xhr(),//新建XMLHttpRequest
        nativeSetHeader = xhr.setRequestHeader,
        abortTimeout

    if (deferred) deferred.promise(xhr) //回调函数处理，采用promise

    if (!settings.crossDomain) setHeader('X-Requested-With', 'XMLHttpRequest') //如果非跨域，设置请求头为ajax异步请求
    setHeader('Accept', mime || '*/*') //默认接受任何类型
    if (mime = settings.mimeType || mime) {
      if (mime.indexOf(',') > -1) mime = mime.split(',', 2)[0]
      xhr.overrideMimeType && xhr.overrideMimeType(mime)
    }
    if (settings.contentType || (settings.contentType !== false && settings.data && settings.type.toUpperCase() != 'GET'))
      setHeader('Content-Type', settings.contentType || 'application/x-www-form-urlencoded') //设置contentType

    if (settings.headers) for (name in settings.headers) setHeader(name, settings.headers[name]) //设置请求头
    xhr.setRequestHeader = setHeader

    /**
         * 0：请求未初始化（还没有调用 open()）。
             1：请求已经建立，但是还没有发送（还没有调用 send()）。
             2：请求已发送，正在处理中（通常现在可以从响应中获取内容头）。
             3：请求在处理中；通常响应中已有部分数据可用了，但是服务器还没有完成响应的生成。
             4：响应已完成；您可以获取并使用服务器的响应了。
    */
    xhr.onreadystatechange = function(){
      if (xhr.readyState == 4) { //相应完成
        xhr.onreadystatechange = empty
        clearTimeout(abortTimeout)//清除延时
        var result, error = false

        //根据状态来判断请求是否成功
        if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304 || (xhr.status == 0 && protocol == 'file:')) {
          dataType = dataType || mimeToDataType(settings.mimeType || xhr.getResponseHeader('content-type'))

          if (xhr.responseType == 'arraybuffer' || xhr.responseType == 'blob')  
            result = xhr.response
          else {
            result = xhr.responseText

            try {
              // http://perfectionkills.com/global-eval-what-are-the-options/
              // sanitize response accordingly if data filter callback provided
              result = ajaxDataFilter(result, dataType, settings) //调用默认的filter
              if (dataType == 'script')    (1,eval)(result)
              else if (dataType == 'xml')  result = xhr.responseXML
              else if (dataType == 'json') result = blankRE.test(result) ? null : $.parseJSON(result) //解析json
            } catch (e) { error = e }

            if (error) return ajaxError(error, 'parsererror', xhr, settings, deferred) //报错，
          }

          ajaxSuccess(result, xhr, settings, deferred) //成功时候的回调函数，这里会先后调用设置的成功回调函数，然后判断是否异步，最后触发全局的ajaxSuccess事件
        } else {
          ajaxError(xhr.statusText || null, xhr.status ? 'error' : 'abort', xhr, settings, deferred)
        }
      }
    }

    if (ajaxBeforeSend(xhr, settings) === false) { //请求前置器
      xhr.abort()
      ajaxError(null, 'abort', xhr, settings, deferred)
      return xhr
    }

    var async = 'async' in settings ? settings.async : true
    xhr.open(settings.type, settings.url, async, settings.username, settings.password) //发送请求
    // xhrFields 设置  如设置跨域凭证 withCredentials
    if (settings.xhrFields) for (name in settings.xhrFields) xhr[name] = settings.xhrFields[name]

    for (name in headers) nativeSetHeader.apply(xhr, headers[name])

    if (settings.timeout > 0) abortTimeout = setTimeout(function(){
        xhr.onreadystatechange = empty
        xhr.abort()
        ajaxError(null, 'timeout', xhr, settings, deferred)
      }, settings.timeout)

    // avoid sending empty string (#319)
    xhr.send(settings.data ? settings.data : null) //POST为发送data参数，GET直接发网址就可以了
    return xhr
  }
```

$.get,$.post，$.getJSON:调用$.ajax进行请求

$.fn.load：

```javascript
 /**
     * 载入远程 HTML 文件代码并插入至 DOM 中
     * @param url    HTML 网页网址    可以指定选择符，来筛选载入的 HTML 文档，DOM 中将仅插入筛选出的 HTML 代码。语法形如 "url #some > selector"。
     * @param data    发送至服务器的 key/value 数据
     * @param success 载入成功时回调函数
     * @returns {*}
     */
$.fn.load = function(url, data, success){
    if (!this.length) return this
    var self = this, parts = url.split(/\s/), selector,
        options = parseArguments(url, data, success),
        callback = options.success
    if (parts.length > 1) options.url = parts[0], selector = parts[1]
    options.success = function(response){
      self.html(selector ?
        $('<div>').html(response.replace(rscript, "")).find(selector)
        : response)
      callback && callback.apply(self, arguments)
    }
    $.ajax(options)
    return this
  }
```



