# touch模块解析

查看代码最后一段，可知，touch主要添加了'swipe', 'swipeLeft', 'swipeRight', 'swipeUp', 'swipeDown','doubleTap', 'tap', 'singleTap', 'longTap'等方法。
他们主要是通过on方法监听事件添加回调函数。
在touch源码里，对touchmove等事件进行了监听，会判断是上面的哪种情况，触发上面注册的事件和回调函数。

先看主程序：

```javascript
$(document).ready(function(){
    var now, delta, deltaX = 0, deltaY = 0, firstTouch, _isPointerType

    if ('MSGesture' in window) { //这里主要是兼容IE
      gesture = new MSGesture()
      gesture.target = document.body
    }

    $(document)
      .bind('MSGestureEnd', function(e){
        var swipeDirectionFromVelocity =
          e.velocityX > 1 ? 'Right' : e.velocityX < -1 ? 'Left' : e.velocityY > 1 ? 'Down' : e.velocityY < -1 ? 'Up' : null
        if (swipeDirectionFromVelocity) {
          touch.el.trigger('swipe')
          touch.el.trigger('swipe'+ swipeDirectionFromVelocity)
        }
      })
      .on('touchstart MSPointerDown pointerdown', function(e){ //这里监听了三个事件，不用说MS就是IE的事件
        if((_isPointerType = isPointerEventType(e, 'down')) && //这里主要判断的是否是指针事件并且不是第一个事件的话，直接屏蔽掉所有事件
          !isPrimaryTouch(e)) return
        firstTouch = _isPointerType ? e : e.touches[0] //根据是否是指针事件取e
        if (e.touches && e.touches.length === 1 && touch.x2) { //清除x2,y2
          // Clear out touch movement data if we have it sticking around
          // This can occur if touchcancel doesn't fire due to preventDefault, etc.
          touch.x2 = undefined
          touch.y2 = undefined
        }
        now = Date.now()
        delta = now - (touch.last || now) //计算当前和上一次触发touchstart时间间隔，如果大约250ms，视为双击
        touch.el = $('tagName' in firstTouch.target ?
          firstTouch.target : firstTouch.target.parentNode) //取初始dom位置，如果target存在则取target，否则取父节点
        touchTimeout && clearTimeout(touchTimeout)
        touch.x1 = firstTouch.pageX //开始时位置记录
        touch.y1 = firstTouch.pageY
        if (delta > 0 && delta <= 250) touch.isDoubleTap = true //双击
        touch.last = now //touch.last记录最后一次的时间
        longTapTimeout = setTimeout(longTap, longTapDelay) //长按的延时
        // adds the current touch contact for IE gesture recognition
        if (gesture && _isPointerType) gesture.addPointer(e.pointerId)
      })
      .on('touchmove MSPointerMove pointermove', function(e){
        if((_isPointerType = isPointerEventType(e, 'move')) &&
          !isPrimaryTouch(e)) return
        firstTouch = _isPointerType ? e : e.touches[0]
        cancelLongTap()//取消长按=>clearTimeout(longTapTimeout)
        touch.x2 = firstTouch.pageX //运动时位置记录
        touch.y2 = firstTouch.pageY

        deltaX += Math.abs(touch.x1 - touch.x2) //x轴偏移
        deltaY += Math.abs(touch.y1 - touch.y2) //y轴偏移
      })
      .on('touchend MSPointerUp pointerup', function(e){
        if((_isPointerType = isPointerEventType(e, 'up')) &&
          !isPrimaryTouch(e)) return //同start
        cancelLongTap() //取消长按=>clearTimeout(longTapTimeout)

        // swipe
        if ((touch.x2 && Math.abs(touch.x1 - touch.x2) > 30) ||
            (touch.y2 && Math.abs(touch.y1 - touch.y2) > 30)) //如果在某一个方向上运动的绝对值大于30

          swipeTimeout = setTimeout(function() {
            if (touch.el){
              touch.el.trigger('swipe') ？//先触发swipe
              touch.el.trigger('swipe' + (swipeDirection(touch.x1, touch.x2, touch.y1, touch.y2))) //然后触发swipe方向上的事件
            }
            touch = {}
          }, 0)

        // normal tap
        else if ('last' in touch) //如果最后一次的时间存在
          // don't fire tap when delta position changed by more than 30 pixels,
          // for instance when moving to a point and back to origin
          if (deltaX < 30 && deltaY < 30) { //累计距离小于30（为了防止划回到原点的情况）
            // delay by one tick so we can cancel the 'tap' event if 'scroll' fires
            // ('tap' fires before 'scroll')
            tapTimeout = setTimeout(function() {

              // trigger universal 'tap' with the option to cancelTouch()
              // (cancelTouch cancels processing of single vs double taps for faster 'tap' response)
              var event = $.Event('tap')
              event.cancelTouch = cancelAll
              // [by paper] fix -> "TypeError: 'undefined' is not an object (evaluating 'touch.el.trigger'), when double tap
              if (touch.el) touch.el.trigger(event) 

              // trigger double tap immediately
              if (touch.isDoubleTap) {  //触发双击事件
                if (touch.el) touch.el.trigger('doubleTap')
                touch = {}
              }

              // trigger single tap after 250ms of inactivity
              else {
                touchTimeout = setTimeout(function(){ //定时25ms后触发单击事件
                  touchTimeout = null
                  if (touch.el) touch.el.trigger('singleTap')
                  touch = {}
                }, 250)
              }
            }, 0)
          } else {
            touch = {}
          }
          deltaX = deltaY = 0

      })
      // when the browser window loses focus,
      // for example when a modal dialog is shown,
      // cancel all ongoing events
      .on('touchcancel MSPointerCancel pointercancel', cancelAll)

    // scrolling the window indicates intention of the user
    // to scroll, not tap or swipe, so cancel all ongoing events
    $(window).on('scroll', cancelAll)
  })


```


    

