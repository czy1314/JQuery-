```js
10.8	DOM ready

初学jQuery时，很可能学到的第一个知识点就是.ready()，但是网络上流转的文章和API对此的解释或剖析，经常是模棱两可甚至是不准确或错误的，本节将详细的阐述.ready()的用法和原理，并纠正这些误区。

10.8.1	如何使用.ready()

.ready() 指定一个事件句柄，当DOM完全加载完成时执行

虽然JavaScript提供了load事件，当页面渲染完成之后会执行这个函数，在所以元素加载完成之前，这个函数不会被调用，例如图像。但是在大多数情况下，只要DOM结构加载完，脚本就可以尽快运行。传递给.ready()的事件句柄在DOM准备好后立即执行，因此通常情况下，最好把绑定事件句柄和其他jQuery代码都到这里来。但是当脚本依赖于CSS样式属性时，一定要在脚本之前引入外部样式或内嵌样式的元素。

如果代码依赖于需加载完的元素（例如，想获取一个图片的尺寸大小），应该用.load()事件代替，并把代码放到load事件句柄中。

.ready()方法不同于<body onload="">属性。如果必须用load事件，那么就不要使用.ready()，或用.load()代替，把load事件句柄绑定到window或其他的具体元素上，例如图片。

以下三个语法全部是等价的：
$(document).ready(handler)
$().ready(handler) (this is not recommended)
$(handler)

$(document).bind("ready", handler)的行为类似于ready方法，但有一个例外：如果ready事件已触发后，再尝试用.bind("ready")绑定的处理函数将不会被执行。而用以上三种方法绑定的句柄会被执行，无论是什么时候绑定的。
.ready()方法只能在包含了document的jQuery上调用，因此选择器可以省略。
.ready()方法典型的用法是使用一个匿名函数：
$(document).ready(function() {
  // Handler for .ready() called.
});
等价于：
$(function() {
 // Handler for .ready() called.
});

如果在DOM加载完毕之后调用.ready()，新的句柄将会立即执行。

特别强调两点：
1.	之所以花费篇幅来阐述.ready()如何使用（翻译的官网文档），是因为网络上的一些文章和中文API不准确甚至是错误的（应该是工具翻译的），所以需谨慎参考
2.	.ready()的触发要早于.load()，但是不要完全迷信依赖，如果要获取或操作样式，或依赖于像图片这样的必须加载完成才能获取和操作的元素，就用<body onload="">或.load()

10.8.2	源码分析

到底.ready()是如何实现更快的触发ready事件的呢，关键在jQuery.bindReady方法的实现里：

对标准浏览器绑定：document.addEventListener( "DOMContentLoaded", DOMContentLoaded, false );
对IE浏览器绑定：document.attachEvent( "onreadystatechange", DOMContentLoaded );

当然还有一些其他的关键技巧，现在完成的看下jQuery.bindReady的实现：

// 绑定DOM ready监听器，跨浏览器，兼容标准浏览器和IE浏览器
bindReady: function() { // jQuery.bindReady
if ( readyList ) {
	return;
}

readyList = jQuery._Deferred(); // 初始化ready异步事件句柄队列

// Catch cases where $(document).ready() is called after the
// browser event has already occurred.
// 如果DOM已经完毕，立即调用jQuery.ready
if ( document.readyState === "complete" ) {
	// Handle it asynchronously to allow scripts the opportunity to delay ready
	// 重要的是异步
	return setTimeout( jQuery.ready, 1 );
}

// Mozilla, Opera and webkit nightlies currently support this event
// DOM 2级事件模型，Mozilla, Opera, webkit等 
if ( document.addEventListener ) {
	// Use the handy event callback
	// 使用快速事件句柄
	document.addEventListener( "DOMContentLoaded", DOMContentLoaded, false );

	// A fallback to window.onload, that will always work
	// 再在window上绑定load事件句柄，这个句柄总是会执行
	// 为什么同时绑定document window呢？我想是为了安全起见，防御性编码！
	window.addEventListener( "load", jQuery.ready, false ); 

// If IE event model is used
} else if ( document.attachEvent ) {
	// ensure firing before onload,
	// maybe late but safe also for iframes
	// 同样onreadystatechange会在onload之前触发，但是对于iframe会有延迟但安全一定会触发
	// 看看DOMContentLoaded的实现，是检测document.readyState的状态是否为complete，这里有些像ajax中的检测 
	document.attachEvent( "onreadystatechange", DOMContentLoaded );

	// A fallback to window.onload, that will always work
	window.attachEvent( "onload", jQuery.ready ); // 同样的安全起见，防御性编码！及时前边的所以hack技巧失败了，onload是最后的保障

	// If IE and not a frame
	// continually check to see if the document is ready
	var toplevel = false;

	try {
		toplevel = window.frameElement == null; // 检查window的frameElement属性，看是否是顶层窗口
	} catch(e) {}
	
	// 做doScroll检测，如果在iframe中就不检测了，onreadystatechange对于ifame很可靠
	if ( document.documentElement.doScroll && toplevel ) {
		doScrollCheck(); 
		// 在doScrollCheck中，不断的（每隔1ms）执行document.documentElement.doScroll("left")，直到不抛出异常为止
		// 这是IE下检测DOM ready的技巧
		}
	}
},

然后是完整的ready相关方法的分析，我剔除了其他无关的代码，保留了整体结构和相关方法：

(function( window, undefined ) {

	var jQuery = (function() {
		
		// Define a local copy of jQuery
		var jQuery = function( selector, context ) {
				// The jQuery object is actually just the init constructor 'enhanced'
				var ret = new jQuery.fn.init( selector, context, rootjQuery );
				return ret;
			},
		
			// A central reference to the root jQuery(document)
			rootjQuery, // 包含了document的jQuery对象
		
			// The deferred used on DOM ready
			readyList, // ready事件处理函数队列
		
			// The ready event handler
			DOMContentLoaded, // DOM ready hack句柄，.ready()能早于.load()的关键所在
		
		jQuery.fn = jQuery.prototype = {
			constructor: jQuery,
			init: function( selector, context, rootjQuery ) {
				// HANDLE: $(function)
				// Shortcut for document ready
				// 如果函数，则认为是DOM ready句柄
				if ( jQuery.isFunction( selector ) ) {
					return rootjQuery.ready( selector );
				}
			},
		
			ready: function( fn ) {
				// Attach the listeners
				jQuery.bindReady(); // 绑定DOM ready监听器，跨浏览器，兼容标准浏览器和IE浏览器
		
				// Add the callback
				readyList.done( fn ); // 将ready句柄添加到ready异步句柄队列
		
				return this;
			}
		};
		
		// Give the init function the jQuery prototype for later instantiation
		jQuery.fn.init.prototype = jQuery.fn;
		
		jQuery.extend = jQuery.fn.extend = function() {};
		
		jQuery.extend({
		
			// Is the DOM ready to be used? Set to true once it occurs.
			// DOM是否加载完毕
			isReady: false,
		
			// A counter to track how many items to wait for before
			// the ready event fires. See #6781
			// DOM加载完毕之前的等待次数
			readyWait: 1,
		
			// Hold (or release) the ready event
			// 是否延迟触发DOM ready？在jQuery中没有任何地方有调用到holdReady，这是个遗留方法还是预留方法，有待继续研究。
			// 因此在当前版本中，readyWait总是1，直到自减
			holdReady: function( hold ) {
				if ( hold ) { // 继续等待
					jQuery.readyWait++;
				} else {
					jQuery.ready( true ); // 
				}
			},
		
			// Handle when the DOM is ready
			// 判断DOM是否加载完毕，如果已完毕，调用DOM ready事件异步队列readyList，如果未完，每个1ms检查一次
			ready: function( wait ) { // jQuery.ready
				// Either a released hold or an DOMready/load event and not yet ready
				// 条件1：wait === true && !--jQuery.readyWait 还在等待加载完成，但是等待计数器jQuery.readyWait已经是0，
				// 换句话说参数wait为true表示尝试一下看是否能开始调用readyList了，如果发现计数器jQuery.readyWait变成0，啥也不管了，开始调用吧
				// 再换句话说，计数器已经是0了，开干吧
				
				// 条件2：wait !== true && !jQuery.isReady 明确的说不用了，即使DOM ready标记还是false
				// 换句话说，及时DOM ready标记还是false，但是调用jQuery.ready的客户端认为不比再等了，可以开干了
				if ( (wait === true && !--jQuery.readyWait) || (wait !== true && !jQuery.isReady) ) {
					// Make sure body exists, at least, in case IE gets a little overzealous (ticket #5443).
					// 检查document.body是否存在，这是特定于IE的检测，保证在IE中DOM ready事件能正确判断
					if ( !document.body ) {
						return setTimeout( jQuery.ready, 1 );
					}

					// Remember that the DOM is ready
					// 到这里，jQuery.isReady被强制为true
					// 在条件2中，如果jQuery.isReady为true，那么说明readyList的状态已经确定，添加到readyList中的函数会被立即执行
					// 如果jQuery.isReady为是false，那么接下来readyList中函数将被执行
					jQuery.isReady = true;

					// If a normal DOM Ready event fired, decrement, and wait if need be
					// 虽然上边的条件2的意思是，别管其他情况，调用方认为ready了，但是这里仍然做了防御性检测，如果等待计数器仍然大于1，结束ready调用
					// 如果这个条件成立，那么一定是哪里出问题了！
					// 在当前版本1.6.1中，这个判断永远不会成立，因为没有地方调用会使readyWait加一的holdReady
					if ( wait !== true && --jQuery.readyWait > 0 ) {
						return;
					}

					// If there are functions bound, to execute
					// 调用DOM ready事件异步队列readyList，注意整理的参数亮了！
					// document指定了ready句柄的上下文，这样我们在执行ready事件句柄时this指向document
					// [ jQuery ]指定了ready事件句柄的第一个参数，这样即使调用$.noConflict()交出了$的控制权，我们依然可以将句柄的第一个参数命名为$，继续在句柄内部使用$符号
					// 如此的精致！赞叹之余，但不要在你的项目中也这么用，理解和维护将成为最大的难题。
					readyList.resolveWith( document, [ jQuery ] );

					// Trigger any bound ready events
					if ( jQuery.fn.trigger ) {
						// 触发ready事件，然后删除ready句柄，DOM ready事件只会生效一次，ready是个自定义事件
						jQuery( document ).trigger( "ready" ).unbind( "ready" );
					}
				}
			},
		
			// 绑定DOM ready监听器，跨浏览器，兼容标准浏览器和IE浏览器
			bindReady: function() { // jQuery.bindReady
				if ( readyList ) {
					return;
				}

				readyList = jQuery._Deferred(); // 初始化ready异步事件句柄队列

				// Catch cases where $(document).ready() is called after the
				// browser event has already occurred.
				// 如果DOM已经完毕，立即调用jQuery.ready
				if ( document.readyState === "complete" ) {
					// Handle it asynchronously to allow scripts the opportunity to delay ready
					// 重要的是异步
					return setTimeout( jQuery.ready, 1 );
				}

				// Mozilla, Opera and webkit nightlies currently support this event
				// DOM 2级事件模型，Mozilla, Opera, webkit等 
				if ( document.addEventListener ) {
					// Use the handy event callback
					// 使用快速事件句柄
					document.addEventListener( "DOMContentLoaded", DOMContentLoaded, false );

					// A fallback to window.onload, that will always work
					// 再在window上绑定load事件句柄，这个句柄总是会执行
					// 为什么同时绑定document window呢？我想是为了安全起见，防御性编码！
					window.addEventListener( "load", jQuery.ready, false ); 

				// If IE event model is used
				} else if ( document.attachEvent ) {
					// ensure firing before onload,
					// maybe late but safe also for iframes
					// 同样onreadystatechange会在onload之前触发，但是对于iframe会有延迟但安全一定会触发
					// 看看DOMContentLoaded的实现，是检测document.readyState的状态是否为complete，这里有些像ajax中的检测 
					document.attachEvent( "onreadystatechange", DOMContentLoaded );

					// A fallback to window.onload, that will always work
					window.attachEvent( "onload", jQuery.ready ); // 同样的安全起见，防御性编码！及时前边的所以hack技巧失败了，onload是最后的保障

					// If IE and not a frame
					// continually check to see if the document is ready
					var toplevel = false;

					try {
						toplevel = window.frameElement == null; // 检查window的frameElement属性，看是否是顶层窗口
					} catch(e) {}
					
					// 做doScroll检测，如果在iframe中就不检测了，onreadystatechange对于ifame很可靠
					if ( document.documentElement.doScroll && toplevel ) {
						doScrollCheck(); 
						// 在doScrollCheck中，不断的（每隔1ms）执行document.documentElement.doScroll("left")，直到不抛出异常为止
						// 这是IE下检测DOM ready的技巧
					}
				}
			},
		});
		
		// All jQuery objects should point back to these
		rootjQuery = jQuery(document);
		
		// Cleanup functions for the document ready method
		// 构造浏览器加载完毕事件处理函数DOMContentLoaded，需要检测浏览器添加事件的方法
		if ( document.addEventListener ) {
			DOMContentLoaded = function() {
				// document ready之后移除
				document.removeEventListener( "DOMContentLoaded", DOMContentLoaded, false );
				jQuery.ready();
			};
		
		} else if ( document.attachEvent ) {
			DOMContentLoaded = function() {
				// Make sure body exists, at least, in case IE gets a little overzealous (ticket #5443).
				// 确保body是存在的，IE会有一些延迟
				if ( document.readyState === "complete" ) {
					// 先移除，难道浏览器重新渲染会再次触发onreadystatechange吗？
					document.detachEvent( "onreadystatechange", DOMContentLoaded );
					jQuery.ready();
				}
			};
		}
		
		// The DOM ready check for Internet Explorer
		// 在IE里，每隔1ms检测IE浏览器的ready状态
		function doScrollCheck() {
			if ( jQuery.isReady ) {
				return;
			}
		
			try {
				// If IE is used, use the trick by Diego Perini
				// http://javascript.nwbox.com/IEContentLoaded/
				// 如果是IE，则使用该技巧来检测浏览器加载状态
				document.documentElement.doScroll("left");
			} catch(e) {
				setTimeout( doScrollCheck, 1 );
				return;
			}
		
			// and execute any waiting functions
			// 执行其他的等待函数
			jQuery.ready();
		}
		
		// Expose jQuery to the global object
		return jQuery;
	
	})();

	// 见系列文章中关于异步队列的解析
	jQuery.extend({
		// Create a simple deferred (one callbacks list)
		_Deferred: function() {},
	
		// Full fledged deferred (two callbacks list)
		Deferred: function( func ) {},
	
		// Deferred helper
		when: function( firstParam ) {}
	});
	
	// 见系列文章中关于事件的解析
	jQuery.event = {
		trigger: function( event, data, elem, onlyHandlers ) {},

		handle: function( event ) {},

		special: {
			ready: {
				// Make sure the ready event is setup
				setup: jQuery.bindReady,
				teardown: jQuery.noop
			}
		}
	};

	window.jQuery = window.$ = jQuery;
})(window);

```