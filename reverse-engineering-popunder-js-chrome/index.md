### Where it all started?

It all started with a Youtube video by [@LiveOverflow](https://twitter.com/LiveOverflow), which is awesome by the way. This video talk is on a bit older version of [Popunder.js](https://popunderjs.com/).

Motivated & intrigued to checkout current version of popunder.js and reverse engineer to understand its engine.

To begin with, I cloned the demo site to my local machine, changed few parts to point my localhost (<i>a simple code change</i>). It consists of 5 files, out of which **script.js** is the heart of the engine payload.

First thing that caught my eye was the defensive code against **Debugger tools**. 

### Defensive Method – 1

```js
var element = document.createElement('div');
element.__defineGetter__('id', function() {
  window.devToolsEnabled = _true;
});

window.console.log(element);
window.console.clear();
```

Whenever a created element is printed onto the console, javascript engine reads the **id** of the element printed in-turn invoking the **getter** function defined by user. [StackOverflow](https://stackoverflow.com/a/36391435/1848109) post talks about this behavior used by coders to detect if the debugger is opened or not. **Chrome (74.0.3729.131)** and **Safari(12.1)** both execute **fake getter functions**, interestingly Mozilla Firefox was immune to this.

### Defensive Method – 2 (dead code in library)
Check if a function is proxied or not. If you have watched [@LiveOverflow](https://twitter.com/LiveOverflow) video’s on the Youtube , he wrote a proxy function for `window.open` to catch the execution of the popunder.js which uses this JS API to open popup in the browser.

<h3 align="center">Code extract – (Read inline comments)</h3>

```js
_isFunctionProxied: function(fnName) {
     var regex, fn, name;

     /* Extracts function name */
     fn = Utils._evaluate(fnName);
     name = fnName.replace(/^(?:.*[^\w])*([\w]+)\s*$/i, '$1');
     try {
       n7.u9(2);
       /* Creates a regex to check string "{ [native code] }" */
       regex = new RegExp(n7.c9('function ', name, '\\(\\)\\s* .{\\s*\\[\\s*native code\\s*\\]\\s*}'), 'im');

       /* Execute the regex and determine if proxied */
       return !regex.test(fn.toString());
     } catch (err) {
        if (err.toString().indexOf('is not a function') > -1) {
            return _true;
        }
        if (Browser._isFirefox) {
            return _true;
        }
        return _false;
     }
}
```

Next, i started reading the code line by line. There were bunch of helper API’s and stuff. Interesting section was the block of code that were written to target  specific browser’s & its version, thus exploiting the flaws to open pop-up which is the library’s goal.

Below are options available for popups to be opened **(Extract of SwitchCase)**

```js
switch (type) {
    case 'popup':
        popOpts = {
            newTab: false,
            under: false
        };
        break;

    case 'tabunder':
        popOpts = {
            newTab: true,
            under: true
        };
        break;

    case 'tabup':
        popOpts = {
            newTab: true,
            under: false
        };
        break;

    default:
        popOpts = {};
}
```

### Chrome
Thanks to **@LiveOverFlow**, filing bug report with Chromium which eliminated the **popunder** option, now this option defaults to tabunder only on Chrome. Watch [this](https://www.youtube.com/watch?v=PPzRcZLNCPY) video for more details by **@LiveOverFlow**. You will realize its an art to exploit a flaw for your own gain.

There was this interesting option called **tricksChrome** which is a boolean flag that ignites a `ChromeDance()` function on the `about:blank` window thats opened on main `window.onClick()` event trigger.

<h3 align="center">Refactored Code from Popunder library - <a href="https://jsfiddle.net/xvmp45we/" target="_blank">Jsfiddle link</a></h3>

```js
window.onclick = function () {
  var w = window.open('about:blank', 'sdffdfds', 'top=9999,left=0,width=100,height=100');
  var popUrl = "https://coder.rocks";
  var left = 0;
  w.document.write(`<script>
  var run = 0;
  var left = 0;
  function dance() {
    if (run > 0){
      return;
    }
    run = 1;

    setInterval(function () {
      if (screenY > 100) 
        window.moveTo(0, 0);
      else 
        window.moveTo(left, 9999)
    }, 20)
  }

  setTimeout(function () {
    var oldX = screenX, oldY = screenY,
      c = setInterval(function () {
        if (oldX != screenX || oldY != screenY) {
          clearInterval(c); 
          dance();
        } 
        oldX = screenX; 
        oldY = screenY;
      }, 50);
  }, 50);

  /*This annoys the user to put popup window in the background*/
  document.onmousemove = dance; 

  function finish() {
    window.moveTo(0, 0);
    window.resizeTo(${screen.width}, ${screen.height});
    window.location.href = "${popUrl}";
  }

  /* when put in background load the target site */
  window.onblur = finish;
  setTimeout(finish, 5 * 60e3);

  <\/script>`);
}
```

**Voilla!** Now you have a successful popup tricking the user.
Technically it’s not a bug, but its beautifully thought through to trick the user.

<h3 align="center">Code extract</h3>

```js
map.push([this._PostMessage, function() {
   return Browser._isChrome && Browser._version >= 64 && Browser._version < 67;
}]);
map.push([this._PDFViewer, function() {
   return Browser._isChrome && Utils._isPDFViewerEnabled() && Browser._isWin && Browser._version >= 43 && Browser._version < 64;
}]);
 map.push([this._PDFTabUnder, function() {
   return Browser._isChrome && Utils._isPDFViewerEnabled() && Browser._isMac && Browser._version >= 49 && Browser._version < 64;
}]);
```

Next section of code is a check for the vulnerable browser versions to exploit (<i>influenced by @LiveOverflow’s reverse engineering disclosure’s</i>), like the `window.postMessage` trick that is available in **Chrome ver 68** and below, PDF as an embedded object to gain focus etc.

For all these vulnerabilities & explanation on older version, I would highly recommend to watch [@LiveOverflow](https://www.youtube.com/c/LiveOverflow) videos on Youtube.

> Let me know what you think. I would be happy to listen to your feedback.