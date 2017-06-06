## 15.4        AJAX中的前置过滤器和请求分发器

自jQuery1.5以后，AJAX模块提供了三个新的方法用于管理、扩展AJAX请求，分别是：

l  前置过滤器 [jQuery. ajaxPrefilter](undefined)

l  请求分发器 jQuery. ajaxTransport，

l  类型转换器 ajaxConvert

这里先分析前置过滤器和请求分发器，类型转换器下一节再讲。

### 15.4.1  前置过滤器和请求分发器的初始化

前置过滤器和请求分发器在执行时，分别遍历内部变量prefilters和[transports](undefined)，这两个变量在jQuery加载完毕后立即初始化，初始化的过程很有意思。

**首先**，prefilters和transports被置为空对象：

**然后**，创建[jQuery.ajaxPrefilter和jQuery.ajaxTransport](undefined)，这两个方法都调用了内部函数addToPrefiltersOrTransports，addToPrefiltersOrTransports返回一个匿名闭包函数，这个匿名闭包函数负责将单一前置过滤和单一请求分发器分别放入[prefilters](undefined)和transports。我们知道闭包会保持对它所在环境变量的引用，而jQuery.ajaxPrefilter和jQuery.ajaxTransport的实现又完全一样,都是对Map结构的对象进行赋值操作,因此这里利用闭包的特性巧妙的将两个方法的实现合二为一。函数addToPrefiltersOrTransports可视为模板模式的一种实现。

```js
ajaxPrefilter: addToPrefiltersOrTransports( prefilters ), // 通过闭包保持对prefilters的引用，将前置过滤器添加到prefilters
ajaxTransport: addToPrefiltersOrTransports( transports ), // 通过闭包保持对transports的引用，将请求分发器添加到transports
 
// 添加全局前置过滤器或请求分发器，过滤器的在发送之前调用，分发器用来区分ajax请求和script标签请求
function addToPrefiltersOrTransports( structure ) {
    // 通过闭包访问structure
    // 之所以能同时支持Prefilters和Transports，关键在于structure引用的时哪个对象
    // dataTypeExpression is optional and defaults to "*"
    // dataTypeExpression是可选参数，默认为*
    return function( dataTypeExpression, func ) {
       // 修正参数
       if ( typeof dataTypeExpression !== "string" ) {
           func = dataTypeExpression;
           dataTypeExpression = "*";
       }
 
       if ( jQuery.isFunction( func ) ) {
           var dataTypes = dataTypeExpression.toLowerCase().split( rspacesAjax ), // 用空格分割数据类型表达式dataTypeExpression
              i = 0,
              length = dataTypes.length,
              dataType,
              list,
              placeBefore;
 
           // For each dataType in the dataTypeExpression
           for(; i < length; i++ ) {
              dataType = dataTypes[ i ];
              // We control if we're asked to add before
              // any existing element
              // 如果以+开头，过滤+
              placeBefore = /^\+/.test( dataType );
              if ( placeBefore ) {
                  dataType = dataType.substr( 1 ) || "*";
              }
              list = structure[ dataType ] = structure[ dataType ] || [];
              // then we add to the structure accordingly
              // 如果以+开头，则插入开始位置，否则添加到末尾
              // 实际上操作的是structure
              list[ placeBefore ? "unshift" : "push" ]( func );
           }
       }
    };
}
```

**最后**，分别调用jQuery.ajaxPrefilter和jQuery.ajaxTransport填充prefilters和transports.

**填充****prefilters:**

```js
// Detect, normalize options and install callbacks for jsonp requests
// 向前置过滤器对象中添加特定类型的过滤器
// 添加的过滤器将格式化参数，并且为jsonp请求增加callbacks
// MARK：AJAX模块初始化
jQuery.ajaxPrefilter( "json jsonp", function( s, originalSettings, jqXHR ) {
 
    var inspectData = s.contentType === "application/x-www-form-urlencoded" &&
       ( typeof s.data === "string" ); // 如果是表单提交，则需要检查数据
 
    // 这个方法只处理jsonp，如果json的url或data有jsonp的特征，会被当成jsonp处理
    // 触发jsonp的3种方式：
    if ( s.dataTypes[ 0 ] === "jsonp" || // 如果是jsonp
       s.jsonp !== false && ( jsre.test( s.url ) || // 未禁止jsonp，s.url中包含=?& =?$ ??
              inspectData && jsre.test( s.data ) ) ) { // s.data中包含=?& =?$ ??
 
       var responseContainer,
           jsonpCallback = s.jsonpCallback =
              jQuery.isFunction( s.jsonpCallback ) ? s.jsonpCallback() : s.jsonpCallback, // s.jsonpCallback时函数，则执行函数用返回值做为回调函数名
           previous = window[ jsonpCallback ],
           url = s.url,
           data = s.data,
           // jsre = /(\=)\?(&|$)|\?\?/i; // =?& =?$ ??
           replace = "$1" + jsonpCallback + "$2"; // $1 =, $2 &|$
 
       if ( s.jsonp !== false ) {
           url = url.replace( jsre, replace ); // 将回调函数名插入url
           if ( s.url === url ) { // 如果url没有变化，则尝试修改data
              if ( inspectData ) {
                  data = data.replace( jsre, replace ); // 将回调函数名插入data
              }
              if ( s.data === data ) { // 如果data也没有变化
                  // Add callback manually
                  url += (/\?/.test( url ) ? "&" : "?") + s.jsonp + "=" + jsonpCallback; // 自动再url后附加回调函数名
              }
           }
       }
      
       // 存储可能改变过的url、data
       s.url = url;
       s.data = data;
 
       // Install callback
       window[ jsonpCallback ] = function( response ) { // 在window上注册回调函数
           responseContainer = [ response ];
       };
 
       // Clean-up function
       jqXHR.always(function() {
           // Set callback back to previous value
           // 将备份的previous函数恢复
           window[ jsonpCallback ] = previous;
           // Call if it was a function and we have a response
           // 响应完成时调用jsonp回调函数，问题是这个函数不是自动执行的么？
           if ( responseContainer && jQuery.isFunction( previous ) ) {
              window[ jsonpCallback ]( responseContainer[ 0 ] ); // 为什么要再次执行previous呢？
           }
       });
 
       // Use data converter to retrieve json after script execution
       s.converters["script json"] = function() {
           if ( !responseContainer ) { // 如果
              jQuery.error( jsonpCallback + " was not called" );
           }
           return responseContainer[ 0 ]; // 因为是作为方法的参数传入，本身就是一个json对象，不需要再做转换
       };
 
       // force json dataType
       s.dataTypes[ 0 ] = "json"; // 强制为json
 
       // Delegate to script
       return "script"; // jsonp > json
    }
});
// Handle cache's special case and global
// 设置script的前置过滤器，script并不一定意思着跨域
// MARK：AJAX模块初始化
jQuery.ajaxPrefilter( "script", function( s ) {
    if ( s.cache === undefined ) { // 如果缓存未设置，则设置false
       s.cache = false;
    }
    if ( s.crossDomain ) { // 跨域未被禁用，强制类型为GET，不触发全局时间
       s.type = "GET";
       s.global = false;
    }
});
```

**填充****transports****：******

```js
// Bind script tag hack transport
// 绑定script分发器,通过在header中创建script标签异步载入js,实现过程很简介
// MARK：AJAX模块初始化
jQuery.ajaxTransport( "script", function(s) {
 
    // This transport only deals with cross domain requests
    if ( s.crossDomain ) { // script可能时json或jsonp，jsonp需要跨域，ajax模块大约有1/3的代码时跨域的
       // 如果在本域中设置了跨域会怎么处理呢？
 
       var script,
           head = document.head || document.getElementsByTagName( "head" )[0] || document.documentElement; // 充分利用布尔表达式的计算顺序
 
       return {
 
           send: function( _, callback ) { // 提供与同域请求一致的接口
 
              script = document.createElement( "script" ); // 通过创script标签来实现
 
              script.async = "async";
 
              if ( s.scriptCharset ) {
                  script.charset = s.scriptCharset; // 字符集
              }
 
              script.src = s.url; // 动态载入
 
              // Attach handlers for all browsers
              script.onload = script.onreadystatechange = function( _, isAbort ) {
 
                  if ( isAbort || !script.readyState || /loaded|complete/.test( script.readyState ) ) {
 
                     // Handle memory leak in IE
                     script.onload = script.onreadystatechange = null; // onload事件触发后，销毁事件句柄，因为IE内存泄漏？
 
                     // Remove the script
                     if ( head && script.parentNode ) {
                         head.removeChild( script ); // onloda后，删除script节点
                     }
 
                     // Dereference the script
                     script = undefined; // 注销script变量
 
                     // Callback if not abort
                     if ( !isAbort ) {
                         callback( 200, "success" ); // 执行回调函数，200为HTTP状态码
                     }
                  }
              };
              // Use insertBefore instead of appendChild  to circumvent an IE6 bug.
              // This arises when a base node is used (#2709 and #4378).
              // 用insertBefore代替appendChild，如果IE6的bug
              head.insertBefore( script, head.firstChild );
           },
 
           abort: function() {
              if ( script ) {
                  script.onload( 0, 1 ); // 手动触发onload事件,jqXHR状态码为0,HTTP状态码为1xx
              }
           }
       };
    }
});
 
// Create transport if the browser can provide an xhr
if ( jQuery.support.ajax ) {
    // MARK：AJAX模块初始化
    // 普通AJAX请求分发器，dataType默认为*
    jQuery.ajaxTransport(function( s ) { // *
       // Cross domain only allowed if supported through XMLHttpRequest
       // 如果不是跨域请求，或支持身份验证
       if ( !s.crossDomain || jQuery.support.cors ) {
 
           var callback;
 
           return {
              send: function( headers, complete ) {
 
                  // Get a new xhr
                  // 创建一个XHR
                  var xhr = s.xhr(),
                     handle,
                     i;
 
                  // Open the socket
                  // Passing null username, generates a login popup on Opera (#2865)
                  // 调用XHR的open方法
                  if ( s.username ) {
                     xhr.open( s.type, s.url, s.async, s.username, s.password ); // 如果需要身份验证
                  } else {
                     xhr.open( s.type, s.url, s.async );
                  }
 
                  // Apply custom fields if provided
                  // 在XHR上绑定自定义属性
                  if ( s.xhrFields ) {
                     for ( i in s.xhrFields ) {
                         xhr[ i ] = s.xhrFields[ i ];
                     }
                  }
 
                  // Override mime type if needed
                  // 如果有必要的话覆盖mineType,overrideMimeType并不是一个标准接口,因此需要做特性检测
                  if ( s.mimeType && xhr.overrideMimeType ) {
                     xhr.overrideMimeType( s.mimeType );
                  }
 
                  // X-Requested-With header
                  // For cross-domain requests, seeing as conditions for a preflight are
                  // akin to a jigsaw puzzle, we simply never set it to be sure.
                  // (it can always be set on a per-request basis or even using ajaxSetup)
                  // For same-domain requests, won't change header if already provided.
                  // X-Requested-With同样不是一个标注HTTP头,主要用于标识Ajax请求.大部分JavaScript框架将这个头设置为XMLHttpRequest
                  if ( !s.crossDomain && !headers["X-Requested-With"] ) {
                     headers[ "X-Requested-With" ] = "XMLHttpRequest";
                  }
 
                  // Need an extra try/catch for cross domain requests in Firefox 3
                  // 设置请求头
                  try {
                     for ( i in headers ) {
                         xhr.setRequestHeader( i, headers[ i ] );
                     }
                  } catch( _ ) {}
 
                  // Do send the request
                  // This may raise an exception which is actually
                  // handled in jQuery.ajax (so no try/catch here)
                  // 调用XHR的send方法
                  xhr.send( ( s.hasContent && s.data ) || null );
 
                  // Listener
                  // 封装回调函数
                  callback = function( _, isAbort ) {
 
                     var status,
                         statusText,
                         responseHeaders,
                         responses, // 响应内容,格式为text:text, xml:xml
                         xml;
 
                     // Firefox throws exceptions when accessing properties
                     // of an xhr when a network error occured
                     // http://helpful.knobs-dials.com/index.php/Component_returned_failure_code:_0x80040111_(NS_ERROR_NOT_AVAILABLE)
                     // 在FF下当网络异常时,访问XHR的属性会抛出异常
                     try {
 
                         // Was never called and is aborted or complete
                         if ( callback && ( isAbort || xhr.readyState === 4 ) ) { // 4表示响应完成
 
                            // Only called once
                            callback = undefined; // callback只调用一次,注销callback
 
                            // Do not keep as active anymore
                            if ( handle ) {
                                xhr.onreadystatechange = jQuery.noop; // 将onreadystatechange句柄重置为空函数
                                if ( xhrOnUnloadAbort ) { // 如果是界面退出导致本次请求取消
                                   delete xhrCallbacks[ handle ]; // 注销句柄
                                }
                            }
 
                            // If it's an abort
                            if ( isAbort ) { // 如果是取消本次请求
                                // Abort it manually if needed
                                if ( xhr.readyState !== 4 ) {
                                   xhr.abort(); // 调用xhr原生的abort方法
                                }
                            } else {
                                status = xhr.status;
                                responseHeaders = xhr.getAllResponseHeaders();
                                responses = {};
                                xml = xhr.responseXML;
 
                                // Construct response list
                                if ( xml && xml.documentElement /* #4958 */ ) {
                                   responses.xml = xml; // 提取xml
                                }
                                responses.text = xhr.responseText; // 提取text
 
                                // Firefox throws an exception when accessing
                                // statusText for faulty cross-domain requests
                                // FF在跨域请求中访问statusText会抛出异常
                                try {
                                   statusText = xhr.statusText;
                                } catch( e ) {
                                   // We normalize with Webkit giving an empty statusText
                                   statusText = ""; // 像WebKit一样将statusText置为空字符串
                                }
 
                                // Filter status for non standard behaviors
 
                                // If the request is local and we have data: assume a success
                                // (success with no data won't get notified, that's the best we
                                // can do given current implementations)
                                // 过滤不标准的服务器状态码
                                if ( !status && s.isLocal && !s.crossDomain ) {
                                   status = responses.text ? 200 : 404; //
                                // IE - #1450: sometimes returns 1223 when it should be 204
                                // 204 No Content
                                } else if ( status === 1223 ) {
                                   status = 204;
                                }
                            }
                         }
                     } catch( firefoxAccessException ) {
                         if ( !isAbort ) {
                            complete( -1, firefoxAccessException ); // 手动调用回调函数
                         }
                     }
 
                     // Call complete if needed
                     // 在回调函数的最后,如果请求完成,立即调用回调函数
                     if ( responses ) {
                         complete( status, statusText, responses, responseHeaders );
                     }
                  };
 
                  // if we're in sync mode or it's in cache
                  // and has been retrieved directly (IE6 & IE7)
                  // we need to manually fire the callback
                  // 同步模式下:同步导致阻塞一致到服务器响应完成,所以这里可以立即调用callback
                  if ( !s.async || xhr.readyState === 4 ) {
                     callback();
                  } else {
                     handle = ++xhrId; // 请求计数
                     // 如果时页面退出导致本次请求取消,修正在IE下不断开连接的bug
                     if ( xhrOnUnloadAbort ) {
                         // Create the active xhrs callbacks list if needed
                         // and attach the unload handler
                         if ( !xhrCallbacks ) {
                            xhrCallbacks = {};
                            jQuery( window ).unload( xhrOnUnloadAbort ); // 手动触发页面销毁事件
                         }
                         // Add to list of active xhrs callbacks
                         // 将回调函数存储在全局变量中,以便在响应完成或页面退出时能注销回调函数
                         xhrCallbacks[ handle ] = callback;
                     }
                     xhr.onreadystatechange = callback; // 绑定句柄,这里和传统的ajax写法没什么区别
                  }
              },
 
              abort: function() {
                  if ( callback ) {
                     callback(0,1); // 1表示调用callback时,isAbort为true,在callback执行过程中能区分出是响应完成还是取消导致的调用
                  }
              }
           };
       }
    });
}
```

### 15.4.2  前置过滤器和请求分发器的执行过程

prefilters中的前置过滤器在请求发送之前、设置请求参数的过程中被调用，调用prefilters的是函数inspectPrefiltersOrTransports；巧妙的时，transports中的请求分发器在大部分参数设置完成后，也通过函数inspectPrefiltersOrTransports取到与请求类型匹配的请求分发器：

函数inspectPrefiltersOrTransports从prefilters或transports中取到与数据类型匹配的函数数组,然后遍历执行，看看它的实现：

```js
/ some code...
 
// Apply prefilters
// 应用前置过滤器，参数说明：
inspectPrefiltersOrTransports( prefilters, s, options, jqXHR );
 
// If request was aborted inside a prefiler, stop there
// 如果请求已经结束，直接返回
if ( state === 2 ) {
    return false;
}
 
// some code...
// 注意：从这里开始要发送了
 
// Get transport
// 请求分发器
transport = inspectPrefiltersOrTransports( transports, s, options, jqXHR );
 
// some code...
函数inspectPrefiltersOrTransports从prefilters或transports中取到与数据类型匹配的函数数组,然后遍历执行，看看它的实现：
// Base inspection function for prefilters and transports
// 执行前置过滤器或获取请求分发器
function inspectPrefiltersOrTransports( structure, options, originalOptions, jqXHR,
       dataType /* internal */, inspected /* internal */ ) {
 
    dataType = dataType || options.dataTypes[ 0 ];
    inspected = inspected || {};
 
    inspected[ dataType ] = true;
 
    var list = structure[ dataType ],
       i = 0,
       length = list ? list.length : 0,
       executeOnly = ( structure === prefilters ),
       selection;
 
    for(; i < length && ( executeOnly || !selection ); i++ ) {
       selection = list[ i ]( options, originalOptions, jqXHR ); // 遍历执行
       // If we got redirected to another dataType
       // we try there if executing only and not done already
       if ( typeof selection === "string" ) {
           if ( !executeOnly || inspected[ selection ] ) {
              selection = undefined;
           } else {
              options.dataTypes.unshift( selection );
              selection = inspectPrefiltersOrTransports(
                     structure, options, originalOptions, jqXHR, selection, inspected );
           }
       }
    }
    // If we're only executing or nothing was selected
    // we try the catchall dataType if not done already
    if ( ( executeOnly || !selection ) && !inspected[ "*" ] ) {
       selection = inspectPrefiltersOrTransports(
              structure, options, originalOptions, jqXHR, "*", inspected );
    }
    // unnecessary when only executing (prefilters)
    // but it'll be ignored by the caller in that case
    return selection;
}
```

### 15.4.3  总结

通过前面的源码解析，可以将前置过滤器和请求分发器总结如下：

**前置过滤器 jQuery.ajaxPrefilter，prefilters**

| 属性     | 值            | 功能                                       |
| ------ | ------------ | ---------------------------------------- |
| *      | undefined    | 不做任何处理，事实上也没有*属性                         |
| json   | [ function ] | 被当作*处理                                   |
| jsonp  | [ function ] | 修正url或data，增加回调函数名在window上注册回调函数注册script>json数据转换器（被当作script处理） |
| script | [ function ] | 设置设置以下参数：是否缓存 cache、（如果跨域）请求类型 、（如果跨域）是否触发AJAX全局事件 |

**请求分发器 jQuery.ajaxTransport，transports**

| 属性     | 值            | 功能                                       |
| ------ | ------------ | ---------------------------------------- |
| *      | [ function ] | 返回xhr分发器，分发器带有send、abort方法send方法依次调用XMLHTTPRequest的open、send方法，向服务端发送请求，并绑定onreadystatechange事件句柄 |
| script | [ function ] | 返回script分发器，分发器带有send、abort方法send方法通过在header中创建script标签异步载入js，并在script元素上绑定onload、script.onreadystatechange事件句柄 |

 





## 15.5        AJAX中的类型转换器

前置过滤器、 请求分发器、类型转换器是读懂jQuery AJAX实现的关键，可能最难读的又是类型转换器。除此之外的源码虽然同样的让人纠结，但相较而言并不算复杂。

类型转换器将服务端响应的responseText或responseXML，转换为请求时指定的数据类型dataType，如果没有指定类型就依据响应头Content-Type自动猜测一个。在分析转换过程之前，很有必要先看看类型转换器的初始化过程，看看支持哪些类型之间的转换。

### 15.5.1  类型转换器的初始化

类型转换器ajaxConvert在服务端响应成功后，对定义在jQuery. ajaxSettings中的converters进行遍历，找到与数据类型相匹配的转换函数，并执行。我们先看看converters的初始化过程，对类型类型转换器的功能有个初步的认识。jQuery. ajaxSettings定义了所有AJAX请求的默认参数，我们暂时先忽略其他属性、方法的定义和实现：

然后在jQuery初始化过程中，对jQuery. ajaxSettings.converters做了扩展，增加了text>script的转换：



```js
jQuery.extend({
    // some code ...
    // ajax请求的默认参数
    ajaxSettings: {
       // some code ...
 
       // List of data converters
       // 1) key format is "source_type destination_type" (a single space in-between)
       // 2) the catchall symbol "*" can be used for source_type
       // 类型转换映射,key格式为单个空格分割的字符串：源格式 目标格式
       converters: {
 
           // Convert anything to text、
           // 任意内容转换为字符串
           // window.String 将会在min文件中被压缩为 a.String
           "* text": window.String,
 
           // Text to html (true = no transformation)
           // 文本转换为HTML（true表示不需要转换，直接返回）
           "text html": true,
 
           // Evaluate text as a json expression
           // 文本转换为JSON
           "text json": jQuery.parseJSON,
 
           // Parse text as xml
           // 文本转换为XML
           "text xml": jQuery.parseXML
       }
    }
    // some code ...
});
然后在jQuery初始化过程中，对jQuery. ajaxSettings.converters做了扩展，增加了text>script的转换：
// Install script dataType
// 初始化script对应的数据类型
// MARK：AJAX模块初始化
jQuery.ajaxSetup({
    accepts: {
       script: "text/javascript, application/javascript, application/ecmascript, application/x-ecmascript"
    },
    contents: {
       script: /javascript|ecmascript/
    },
    // 初始化类型转换器,这个为什么不写在jQuery.ajaxSettings中而要用扩展的方式添加呢?
    // 这个转换器是用来出特殊处理JSONP请求的，显然,jQuery的作者John Resig,时时刻刻都认为JSONP和跨域要特殊处理!
    converters: {
       "text script": function( text ) {
           jQuery.globalEval( text );
           return text;
       }
    }
});
```

初始化过程到这里就结束了，很简单，就是填充jQuery. ajaxSettings.converters。

当一个AJAX请求完成后，会调用闭包函数done，在done中判断本次请求是否成功，如果成功就调用ajaxConvert对响应的数据进行类型转换（闭包函数done在讲到jQuery.ajax()时一并分析）：

```js
// 服务器响应完毕之后的回调函数，done将复杂的善后事宜封装了起来，执行的动作包括：
// 清除本次请求用到的变量、解析状态码&状态描述、执行异步回调函数队列、执行complete队列、触发全局Ajax事件
// status: -1 没有找到请求分发器
function done( status, statusText, responses, headers ) {
    // 省略代码...   
    // If successful, handle type chaining
    // 如果成功的话，处理类型
    if ( status >= 200 && status < 300 || status === 304 ) {      
       // 如果没有修改，修改状态数据，设置成功
       if ( status === 304 ) { 省略代码... }
       else {
 
           try {
              // 获取相应的数据
              // 在ajaxConvert这个函数中，将Server返回的的数据进行相应的转换（js、json等等）
              success = ajaxConvert( s, response ); // 注意:这里的succes变为转换后的数据对象!
              statusText = "success";
              isSuccess = true;
           } catch(e) {
              // We have a parsererror
              // 数据类型转换器解析时出现错误
              statusText = "parsererror";
              error = e;
           }
       }
    // 非200~300，非304
    } else {
       // We extract error from statusText
       // then normalize statusText and status for non-aborts
       // 其他的异常状态，格式化statusText、status，不采用HTTP标准状态码和状态描述
       error = statusText;
       if( !statusText || status ) {
           statusText = "error";
           if ( status < 0 ) {
              status = 0;
           }
       }
    }
    // 省略代码...
}
```

### 15.5.2  类型转换器的执行过程

类型转换器ajaxConvert根据请求时设置的数据类型，从jQuery. ajaxSettings.converters寻找对应的转换函数，寻找的过程非常绕。假设有类型A数据和类型B数据，A要转换为B（A > B），首先在converters中查找能 A > B 对应的转换函数，如果没有找到，则采用曲线救国的路线，寻找类型C，使得类型A数据可以转换为类型C数据，类型C数据再转换为类型B数据，最终实现 A > B。类型转换器的原理并不复杂，复杂的是它的实现：

```js
// Chain conversions given the request and the original response
// 转换器,内部函数,之所以不添加到jQuery中,可能是因为这个函数不需要客户端显示调用吧
function ajaxConvert( s, response ) {
 
    // Apply the dataFilter if provided
    // dataFilter也是一个过滤器,在调用时的参数options中设置,在类型类型转换器执行之前调用
    if ( s.dataFilter ) {
       response = s.dataFilter( response, s.dataType );
    }
 
    var dataTypes = s.dataTypes, // 取出来,减少作用域查找,缩短后边的拼写
       converters = {},
       i,
       key,
       length = dataTypes.length,
       tmp,
       // Current and previous dataTypes
       current = dataTypes[ 0 ], // 取出第一个作为起始转换类型
       prev, // 每次记录前一个类型,以便数组中相邻两个类型能形成链条
       // Conversion expression
       conversion, // 类型转换表达式 被转换类型>目标了类型
       // Conversion function
       conv, // 从jQuery.ajaxSetting.converts中取到特定类型之间转换的函数
       // Conversion functions (transitive conversion)
       conv1, // 两个临时表达式
       conv2;
 
    // For each dataType in the chain
    // 从第2个元素开始顺序向后遍历,挨着的两个元素组成一个转换表达式,形成一个转换链条
    for( i = 1; i < length; i++ ) {
 
       // Create converters map
       // with lowercased keys
       // 将s.converters复制到converters,为什么要遍历转换呢?直接拿过来用不可以么?
       if ( i === 1 ) {
           for( key in s.converters ) {
              if( typeof key === "string" ) {
                  converters[ key.toLowerCase() ] = s.converters[ key ];
              }
           }
       }
 
       // Get the dataTypes
       prev = current; // 取出前一个类型,每次遍历都会把上一次循环时的类型记录下来
       current = dataTypes[ i ]; // 取出当前的类型
 
       // If current is auto dataType, update it to prev
       if( current === "*" ) { // 如果碰到了*号,即一个任意类型,而转换为任意类型*没有意义
           current = prev; // 所以回到前一个,跳过任意类型,继续遍历s.dataTypes
       // If no auto and dataTypes are actually different
       // 这里开始才是函数ajaxConvert的重点
       // 前一个不是任意类型*,并且找到了一个不同的类型(注意这里隐藏的逻辑:如果第1个元素是*,跳过,再加上中间遇到的*都被跳过了,所以结论就是s.dataTypes中的*都会被忽略!)
       } else if ( prev !== "*" && prev !== current ) {
 
           // Get the converter 找到类型转换表达式对应的转化函数
           conversion = prev + " " + current; // 合体,组成类型转换表达式
           conv = converters[ conversion ] || converters[ "* " + current ]; // 如果没有对应的,就默认被转换类型为*
 
           // If there is no direct converter, search transitively
           // 如果没有找到转变表达式,则向后查找
           // 因为:jsonp是有浏览器执行的呢,还是要调用globalEval呢?
           if ( !conv ) { // 如果没有对应的转换函数,则寻找中间路线
              conv2 = undefined; //
              for( conv1 in converters ) { // 其实是遍历s.converts
                  tmp = conv1.split( " " ); // 将s.convsert中的类型转换表达式拆分,tmp[0] 源类型 tmp[1] 目标类型
                  // 如果tmp[0]与前一个类型相等,或者tmp[0]是*(*完全是死马当作活马医,没办法的办法)
                  if ( tmp[ 0 ] === prev || tmp[ 0 ] === "*" ) { // 这里的*号与conv=这条语句对应
                     // 形成新的类型转换表达式, 看到这里,我有个设想,简单点说就是:
                     // A>B行不通,如果A>C行得通,C>B也行得通,那么A>B也行的通:A > C > B
                     // 将这个过程与代码结合起来,转函数用fun(?>?)表示:
                     // A == tmp[0] == prev
                     // C == tmp[1]
                     // B == current
                     // conv1 == A>C
                     conv2 = converters[ tmp[1] + " " + current ]; // 看看C>B转换函数有木有,conv==fun(C>B)
                     if ( conv2 ) { // 如果fun(C>B)有,Great!看来有戏,因为发现了新的类型转换表达式A>C>B
                         conv1 = converters[ conv1 ]; // conv1是A>C,将A>C转换函数取出来,conv1由A>C变成fun(A>C)
                         if ( conv1 === true ) { // true是什么东西呢?参看jQuery.ajaxSettings知道 "text html": true,意思是不需要转换,直接那来用
                            conv = conv2; // conv2==fun(C>B),赋给conv,conv由func(A>B)变成fun(C>B)
                            // 详细分析一下:
                            // 这里A==text,C==html,B是未知,发现A>B行不通,A>C和C>B都行得通,
                            // 但是因为A>C即text>html不需要额外的转换可以直接使用,可以理解为A==C,所以可以忽略A>C,将A>C>B链条简化为C>B
                            // 结果就变成这样: A>C>B链条中的A>C被省略,表达式简化为C>B
                         // 如果conv1不是text>html,即A!=C,那么就麻烦了,但是,又发现conv2即fun(C>B)是text>html,C==B,那么A>C>B链条简化为A>C
                         } else if ( conv2 === true ) { // conv2==func(C>B)
                            conv = conv1; // A>C>B链条简化为A>C
                         }
                         /**
                          * 将上边与代码紧密结合的注释再精炼:
                          * 目标是A>B但是行不通,但是A>C可以,C>B可以,表达式变成A>C>B
                          * 如果A=C,表达式变成C>B,上边的conv2
                          * 如果C=B,表达式变成A>C,上边的conv1
                          */
                         /**
                          * 但是要注意,到这里还没完,如果既不是A==C,也不是C==B,表达式A>C>B就不能简化了!
                          * 怎么办?虽然到这里conv依然是undefined,但是知道了一条A通往B的路,剩下的工作在函数ajaxConvert的最后完成!
                          */
                         break; // 找到一条路就赶紧跑,别再转悠了
                        
                     }
                  }
              }
           }
           // If we found no converter, dispatch an error
           // 如果A>B行不通,A>?>B也行不通,?表示中间路线,看来此路是真心不通啊,抛出异常
           if ( !( conv || conv2 ) ) { // 如果conv,conv2都为空
              jQuery.error( "No conversion from " + conversion.replace(" "," to ") );
           }
           // If found converter is not an equivalence
           // 如果找到的conv是一个函数或者是undefined
           if ( conv !== true ) {
              // Convert with 1 or 2 converters accordingly
              // 分析下边的代码之前,我们先描述一下这行代码的运行环境,看看这些变量分别代表什么含义:
              // 1. conv可能是函数表示A>B可以
              // 2. conv可能是undefined表示A>B不可以,但是A>C>B可以
              // 3. conv1是fun(A>C),表示A>C可以
              // 4. conv2是fun(C>B),表示C>B可以
             
              // 那么这行代码的含义就是:
              // 如果conv是函数,执行conv( response )
              // 如果conv是undefined,那么先执行conv1(response),即A>C,再执行conv2( C ),即C>B,最终完成A>C>B的转换
              response = conv ? conv( response ) : conv2( conv1(response) );
           }
       }
    }
    return response;
}
```

ajaxConvert的源码分析让我破费一番脑筋，因为它的很多逻辑没有显示的体现在代码上，是隐式的。我不敢对这样的编程方式妄下定论，也许这样的代码不易维护，也许仅仅是我的水平还不够。

### 15.5.3  总结

通过前面的源码解析，可以将类型转换器总结如下：

| 属性          | 值                                        | 功能                          |
| ----------- | ---------------------------------------- | --------------------------- |
| * text      | window.String                            | 任意内容转换为字符串                  |
| text html   | true                                     | 文本转换为HTML（true表示不需要转换，直接返回） |
| text json   | jQuery.parseJSON                         | 文本转换为JSON                   |
| text script | **function**( text ) {    jQuery.globalEval( text );    **return** text;} | 用eval执行text                 |
| text xml    | jQuery.parseXML                          | 文本转换为XML                    |

如果在上表示中没有找到对应的转换函数（类型A > 类型B），就再次遍历上表，寻找新的类型C，使得可以 A > C > B。

 

后记：

到此，最关键的前置过滤器、请求分发器、类型转换器已经分析完毕，但是AJAX实现的复杂度还是远远超过了我最初的认识，还有很多地方值得深入学习和分析，比如：jQuery.ajax的实现、数据的解析、异步队列在AJAX中的应用、多个内部回调函数调用链、jqXHR的状态码与HTTP状态码的关系、数据类型的修正、跨域、对缓存的修正、对HTTP状态码的修正，等等等等，每一部分都可以单独成文，因此对AJAX的分析还会继续。提高自己的同时，希望对读者有所启发和帮助。