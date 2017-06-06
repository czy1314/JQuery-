```js
10.4	.bind() .one()
10.4.1	如何使用
.bind( eventType, [eventData], handler(eventObject) )	在匹配的元素上绑定指定类型的事件处理函数
.bind( eventType, [eventData], preventBubble )	第三个参数为false，则阻止浏览器默认行为并停止事件冒泡，默认true
.bind( events )	绑定多个事件，events的键为事件类型，值为对应的事件处理函数
.one( eventType, [eventData], handler(eventObject) )	在匹配的元素上绑定指定类型的事件处理函数，这个函数最多执行一次
其实是改写了事件处理函数，函数执行时先解绑定，再执行
10.4.2	调用过程 
jQuery.fn.bind → jQuery.event.add
jQuery.fn.one → jQuery.event.add
10.4.3	源码分析
/**
 * bind:
 * .bind( eventType, [eventData], handler(eventObject) ) 在匹配的元素上绑定指定类型的事件处理函数
 * .bind( eventType, [eventData], preventBubble ) 第三个参数为false，则阻止浏览器默认行为并停止事件冒泡，默认true
 * .bind( events ) 绑定多个事件，events的键为事件类型，值为对应的事件处理函数
 * 
 * one:
 * .one( eventType, [eventData], handler(eventObject) ) 在匹配的元素上绑定指定类型的事件处理函数，这个函数最多执行一次
 * 其实是改写了事件处理函数，函数执行时先解绑定，再执行
 */
jQuery.each(["bind", "one"], function( i, name ) {
	// 为jQuery对象扩展bind、one方法
	jQuery.fn[ name ] = function( type, data, fn ) {
		var handler;

		// Handle object literals
		// 绑定多个事件，key为事件类型，type[key]是事件处理函数
		if ( typeof type === "object" ) { //
			for ( var key in type ) {
				this[ name ](key, data, type[key], fn);
			}
			return this;
		}
		// 修正参数：如果没有传入data，或data为false，（jQuery的这种写法很巧妙，但是可读性太差）
		// 如果有两个参数，则认为忽略了data：type, fn，结果是：type, undefined, fn
		// 如果data是false，则认为是type, false, fn，结果是：type, undefined, false
		if ( arguments.length === 2 || data === false ) {
			fn = data;
			data = undefined;
		}

		if ( name === "one" ) { // 只执行一次
			handler = function( event ) { // 创建一个新的事件处理函数句柄
				jQuery( this ).unbind( event, handler ); // 先删除事件
				return fn.apply( this, arguments ); // 再执行传入的事件处理函数fn，这里this是DOM元素
			};
			handler.guid = fn.guid || jQuery.guid++; // 同步guid，重新包装后的函数与原始函数的guid统一
		} else {
			handler = fn;
		}

		if ( type === "unload" && name !== "one" ) { // 如果是unload事件，则只执行一次
			this.one( type, data, fn ); // 迭代调用，没有清晰的结构很危险
		} else {
			for ( var i = 0, l = this.length; i < l; i++ ) {
				jQuery.event.add( this[i], type, handler, data ); // 遍历匹配的元素，并绑定事件处理函数
			}
		}
		// 返回jQuery对象，使得可以继续链式调用
		return this;
	};
});
10.4.4	jQuery.event.add
.bind()和.one()都调用了jQuery.event.add来实现，为元素elem添加类型types的句柄handler，事实上所有的事件绑定最后都通过jQuery.event.add来实现。其执行过程大致如下：
1.	先调用jQuery._data从$.cache中取出已有的事件缓存（私有数据，Cache的解析详见数据缓存）
2.	如果是第一次在DOM元素上绑定该类型事件句柄，在DOM元素上绑定jQuery.event.handle，作为统一的事件响应入口
3.	将封装后的事件句柄放入缓存中
传入的事件句柄，会被封装到对象handleObj的handle属性上，此外handleObj还会填充guid、type、namespace、data属性；DOM事件句柄elemData.handle指向jQuery.event.handle，即jQuery在DOM元素上绑定事件时总是绑定同样的DOM事件句柄jQuery.event.handle。
事件句柄在缓存$.cache中的数据结构如下，事件类型和事件句柄都存储在属性events中，属性handle存放的执行这些事件句柄的DOM事件句柄：
elemData = {                                                         
    events: {                                                        
        'click' : [                                                  
            { guid: 5, type: 'click', namespace: '', data: undefined,
                handle: { guid: 5, prototype: {} }                   
            },                                                       
            { ... }                                                  
        ],                                                           
        'keypress' : [ ... ]                                         
    },                                                               
    handle: { // DOM事件句柄                                             
        elem: elem,                                                  
        prototype: {}                                                
    }                                                                
}                                                                    
看看jQuery.event.add的源码解析：
jQuery.event = {
	// Bind an event to an element
	// Original by Dean Edwards
	// 为元素elem添加类型types的句柄handler
	add: function( elem, types, handler, data ) {
		// 忽略 Text Comment
		if ( elem.nodeType === 3 || elem.nodeType === 8 ) {
			return;
		}

		if ( handler === false ) { // 如果事件处理函数是false，则用returnFalse代替false
			handler = returnFalse; // returnFalse会取消事件的默认行为
		} else if ( !handler ) {
			// Fixes bug #7229. Fix recommended by jdalton
			return; // 如果handler是undefined null ''，则直接返回，不执行后边的代码
		}

		var handleObjIn, handleObj;

		if ( handler.handler ) { // 如果已经是封装过的jQuery事件对象（JS真是愁人啊，弱类型，只能通过特性、属性判断对象的类型）
			handleObjIn = handler; // handleObj是DOM事件句柄对象（丰富后的jQuery.event.handle）
			handler = handleObjIn.handler; // handler始终是个函数
		}

		// Make sure that the function being executed has a unique ID
		if ( !handler.guid ) { // 如果没有分配唯一id，则分配一个
			handler.guid = jQuery.guid++;
		}

		// Init the element's event structure
		// 内部数据存储在内部对象上，因此elemData不应该为空，除非elem不支持jQuery缓存（embed、object、applet）
		var elemData = jQuery._data( elem ); // 仅仅调用_data接口一次，后续直接在返回的对象上操作

		// If no elemData is found then we must be trying to bind to one of the
		// banned noData elements
		// 如果elemData不存在，说明是一个不支持缓存的元素上绑定事件，直接返回
		if ( !elemData ) {
			return;
		}

		var events = elemData.events, // 事件类型和事件句柄都存储在属性events中
			eventHandle = elemData.handle; // 属性handle存放的执行这些事件句柄的DOM事件句柄

		if ( !events ) {
			elemData.events = events = {}; // 初始化一个存放事件的对象，事件名为key，事件函数为value
		}

		if ( !eventHandle ) { // 初始化一个执行事件函数的函数
			elemData.handle = eventHandle = function( e ) {
				// Discard the second event of a jQuery.event.trigger() and
				// when an event is called after a page has unloaded
				// 调用jQuery.event.handler
				// jQuery.event.handler是什么意思？
				return typeof jQuery !== "undefined" && (!e || jQuery.event.triggered !== e.type) ?
					jQuery.event.handle.apply( eventHandle.elem, arguments ) :
					undefined;
			};
		}

		// Add elem as a property of the handle function
		// This is to prevent a memory leak with non-native events in IE.
		// 内存泄漏？
		// 将elem作为eventHandle的属性存储，用来避免IE中非本地事件的内存泄漏
		eventHandle.elem = elem;

		// Handle multiple events separated by a space
		// jQuery(...).bind("mouseover mouseout", fn);
		types = types.split(" "); // 同时绑定的多个事件可以用空格隔开
		// 为什么有的地方用字符串直接量，有的地方要用正则呢？因为正则可以进行全局匹配g

		var type, i = 0, namespaces;

		while ( (type = types[ i++ ]) ) { // 又一种遍历数组的方法，故意秀技巧么？
			handleObj = handleObjIn ?
				jQuery.extend({}, handleObjIn) :
				{ handler: handler, data: data }; // 创建事件句柄对象，后边还会添加属性：guid anmespace type

			// Namespaced event handlers
			// 如果事件字符串中有句号.，则说明有命名空间
			if ( type.indexOf(".") > -1 ) {
				namespaces = type.split(".");
				type = namespaces.shift(); // 取出第一个元素
				handleObj.namespace = namespaces.slice(0).sort().join("."); // 将于下部分作为命名空间，存储到属性namespace中

			} else {
				namespaces = [];
				handleObj.namespace = ""; // 没有命名空间
			}

			handleObj.type = type; // 事件类型
			if ( !handleObj.guid ) {
				handleObj.guid = handler.guid; // 事件对象唯一id
			}

			// Get the current list of functions bound to this event
			var handlers = events[ type ], // 事件句柄队列，取出已经存在函数数组
				special = jQuery.event.special[ type ] || {}; // 特殊处理的事件

			// Init the event handler queue
			if ( !handlers ) { // 如果没有绑定事件，则初始化为一个空数组
				handlers = events[ type ] = []; // 

				// Check for a special event handler
				// Only use addEventListener/attachEvent if the special
				// events handler returns false
				// 如果特殊事件没有setup属性 或 setup返回false，则用浏览器原生的绑定事件接口addEventListener/addEventListener，例如live事件
				if ( !special.setup || special.setup.call( elem, data, namespaces, eventHandle ) === false ) {
					// Bind the global event handler to the element
					// 绑定事件句柄
					if ( elem.addEventListener ) {
						elem.addEventListener( type, eventHandle, false ); // jQuery绑定的事件默认都是起泡阶段捕获

					} else if ( elem.attachEvent ) {
						elem.attachEvent( "on" + type, eventHandle ); // IE事件模型中没有2级DOM事件模型具有的事件捕捉的概念，只有起泡阶段
					}
				}
			}

			if ( special.add ) { // 如果有add方法，就调用
				special.add.call( elem, handleObj );

				if ( !handleObj.handler.guid ) { // 始终保证事件句柄有唯一的id
					handleObj.handler.guid = handler.guid;
				}
			}

			// Add the function to the element's handler list
			// 将句柄对象handleObj，加入到句柄数组handler中，
			// handleObj包含以下属性：data 唯一guid 命名空间namespace 事件类型type handler函数
			handlers.push( handleObj );

			// Keep track of which events have been used, for event optimization
			// 记录已经使用的事件，用于事件优化？怎么个优化？
			jQuery.event.global[ type ] = true;
		}

		// Nullify elem to prevent memory leaks in IE
		// 将elem置为null，避免IE中的内存泄漏（这行代码真是坑爹啊，没有无数次的测试确认，怎么可能想到这行代码呢）
		elem = null;
	},
	// ...
};
10.5	.unbind()
10.5.1	如何使用
.unbind( [eventType] [, handler(eventObject)] )	从匹配的元素上删除一个之前绑定的事件句柄
如果没有参数，删除所有事件句柄
.unbind( eventType, false )	删除通过.bind( eventType, false )绑定的事件句柄
.unbind( event )	看不懂。。。
10.5.2	调用过程
jQuery.fn.unbind → jQuery.event.remove
10.5.3	源码分析
jQuery.fn.extend({
	// 删除句柄：删除一个或多个之前附加的事件句柄，jQuery事件只在起泡阶段捕获，不需要再定义控制捕获阶段的参数
	unbind: function( type, fn ) {
		// Handle object literals
		// 一次删除多个事件句柄，key是事件类型，type[key]是事件句柄（这里是迭代复用）
		// !type.preventDefault 表明这不是一个就jQuery事件对象
		if ( typeof type === "object" && !type.preventDefault ) {
			for ( var key in type ) {
				this.unbind(key, type[key]);
			}
		// 到这里，type可能是字符串，也可能是jQuery事件对象
		} else {
			for ( var i = 0, l = this.length; i < l; i++ ) {
				jQuery.event.remove( this[i], type, fn ); // 调用remove删除句柄
			}
		}

		return this;
	},
	// ...
};
10.5.4	jQuery.event.remove
.unbind()通过调用jQuery.event.remove，删除之前绑定的一个或多个事件句柄，事实上所有的句柄删除最后都通过jQuery.event.remove实现，其执行过程大致如下：
1. 现调用jQuery._data从缓存$.cache中取出elem对应的所有数组（内部数据，与调用jQuery.data存储的数据稍有不同 
2. 如果未传入types则移除所有事件句柄，如果types是命名空间，则移除所有与命名空间匹配的事件句柄
3. 如果是多个事件，则分割后遍历
4. 如果未指定删除哪个事件句柄，则删除事件类型对应的全部句柄，或者与命名空间匹配的全部句柄 
5. 如果指定了删除某个事件句柄，则删除指定的事件句柄    
6. 所有的事件句柄删除，都直接在事件句柄数组jQuery._data( elem ).events[ type ]上调用splice操作 
7. 最后检查事件句柄数组的长度，如果为0，或为1但要删除，则移除绑定在elem上DOM事件 
8. 最后的最后，如果elem对应的所有事件句柄events都已删除，则从缓存中移走elem的内部数据
9. 在以上的各个过程，都要检查是否有特例需要处理 
看看它的源码分析：
jQuery.event = {
	// ...
	/**
	 * 移除元素elem上的一个或多个或一组事件
	 * pos 指要移除的是第几个types事件，可以减少遍历次数
	 * handler的属性guid唯一标识某个事件对象，移除时通过比较guid是否相等
	 */ 
	remove: function( elem, types, handler, pos ) { // add/remove都是作为jQuery内部接口使用
		// don't do events on text and comment nodes
		// 忽略文本元素Text和注释元素Comment
		if ( elem.nodeType === 3 || elem.nodeType === 8 ) {
			return;
		}

		if ( handler === false ) { // 替换布尔型handler为函数，保证handler始终是一个函数，使得remove接口的使用更加便捷
			handler = returnFalse;
		}

		var ret, type, fn, j, i = 0, all, namespaces, namespace, special, eventType, handleObj, origType,
			elemData = jQuery.hasData( elem ) && jQuery._data( elem ), // 如果在缓存cache中有数据，则取出，否则elemData为undefine
			// 因为jQuery._data总是返回一个对象，因此要先判断缓存cache中是否有数组
			events = elemData && elemData.events; // 取出事件句柄数组，用正则表达式可以将多行代码合并为一行，这里可以用if-else或三元表达式代替

		if ( !elemData || !events ) { // 如果未绑定，则忽略本次调用，直接返回
			return;
		}

		// types is actually an event object here
		// 如果types是一个jQuery Event对象，则。。。
		if ( types && types.type ) {
			handler = types.handler;
			types = types.type;
		}

		// Unbind all events for the element
		// 如果types为false（强制转换类型），则移除所有事件
		// 如果types是字符串，并且以句号开头，则移除types命名空间下的指定事件（type+types）
		if ( !types || typeof types === "string" && types.charAt(0) === "." ) {
			types = types || "";

			for ( type in events ) { // 遍历events，移除所有已绑定的事件，这里是迭代调用，jQuery的源码很精致，能复用的代码会尽量复用
				jQuery.event.remove( elem, type + types ); // 全部事件，或某一命名空间下的全部事件
			}

			return;
		}

		// Handle multiple events separated by a space
		// jQuery(...).unbind("mouseover mouseout", fn);
		// 同时移除多个事件，事件之间用空格隔开，使得remove接口很灵活
		types = types.split(" ");

		// 个人觉的变量i的定义与使用离得那么远，不是好习惯，像这种局部使用的变量，应该哪里用就在哪里定义
		// 可能作者觉得集中定义显得代码不凌乱吧
		while ( (type = types[ i++ ]) ) { // 有一种循环方式
			origType = type;
			handleObj = null;
			all = type.indexOf(".") < 0; // all在这里表示移除的是type命名空间下的全部事件对象，小于0表示不是以.开头
			// all为true表示是事件，false表示命名空间
			namespaces = [];

			if ( !all ) { // 如果以.开头，即type是命名空间，构建命名空间正则表达式
				// Namespaced event handlers
				namespaces = type.split("."); // 全部命名空间
				type = namespaces.shift(); // 取出第一个作为事件类型，其余的为命名空间

				namespace = new RegExp("(^|\\.)" +
					jQuery.map( namespaces.slice(0).sort(), fcleanup ).join("\\.(?:.*\\.)?") + "(\\.|$)");
				// 这里要动态创建正则，因此用了 new RegExp 的方式
			}

			eventType = events[ type ]; // 取出type对应的事件对象数组

			if ( !eventType ) { // 如果type对应的事件对象数组不存在，则跳过本次循环（既然不存在就不必移除了）
				continue;
			}

			if ( !handler ) { // 没有指定移除哪个事件对象，handler其实是一个带有guid属性的函数，因为通过guid可以找到对应的事件对象，因此这里依然称handler为事件对象
				for ( j = 0; j < eventType.length; j++ ) { // 遍历事件对象数组，这里因为是遍历一个动态的数组，所以没有定义eventType.length变量
					handleObj = eventType[ j ];

					if ( all || namespace.test( handleObj.namespace ) ) { // 如果是事件或命名空间匹配
						jQuery.event.remove( elem, origType, handleObj.handler, j ); // 取出事件对象，迭代调用remove接口
						// handleObj.handler有guid属性，因此迭代调用remove依然可以匹配到对应handleObj，事实上这里也可以改写为传入handleObj
						// 注意最后一个参数j，remove的pos不为null，就不会在其他地方被splice删除，所以下一行才会splice（这个问题的处理太乱了）
						eventType.splice( j--, 1 ); // 如果循环遍历的是一个变化的数组，则可以用这种方式：j++之前先执行j--，保证不会因为数组下标的错误导致某些数组元素遍历不到，秀！
					}
				}
				// 如果未指定删除哪个事件句柄，则删除事件类型对应的全部句柄，或者与命名空间匹配的全部句柄
				// 这里对$.cache的操作，是通过对 eventType = jQuery._data( elem ).events[ type ]的直接操作进行维护，直接操作事件句柄数组
				// 如果是我，可能会再开发一个接口供客户端进行这种微操，其实没必要。
				continue;
			}

			// 到这里，说明指定事件句柄handler
			
			special = jQuery.event.special[ type ] || {}; // 特例

			// 如果没有指定pos，则默认从0开始遍历，这里的pos可以减少遍历次数，提高性能，多么精致的代码
			// 本人喜欢这种极致的代码，更喜欢优雅的代码，更注重代码的可读性，稍微的性能下降可以接受
			for ( j = pos || 0; j < eventType.length; j++ ) {
				handleObj = eventType[ j ];

				if ( handler.guid === handleObj.guid ) { // 比较事件函数的guid和时间对象的guid，===可以避免类型转换，稍微提高性能
					// remove the given handler for the given type
					if ( all || namespace.test( handleObj.namespace ) ) {
						if ( pos == null ) { // 如果没有指定pos，匹配到后就删除
							eventType.splice( j--, 1 ); // j++之前先j--，避免数组变化导致遍历错误
						}

						if ( special.remove ) { // 如果有自定义的remove方法，则调用（似乎只有live有add remove方法）
							special.remove.call( elem, handleObj );
						}
					}
					// 因为guid是全局唯一的，所以匹配到guid对应的事件就可以退出了
					if ( pos != null ) { // 如果pos不为null，说明是要删除指定的事件对象，任务完成，退出
						break;
					}
				}
			}

			// remove generic event handler if no more handlers exist
			// 如果type对应的事件对象数组为空，或者发现只剩一个，则移除绑定在elem上浏览器原生事件
			if ( eventType.length === 0 || pos != null && eventType.length === 1 ) {
				// 检查special是否有自定义的teardown，优先调用
				// 这一行的巧妙支出在于：
				// 1. special不会为null或undefined，如果没有找到会被赋值为{}，是空设置模式的应用
				// 2. 首先巧妙的判断special.teardown是否存在，如果不存在则认为是普通事件，如果存在则可以调用teardown
				// 3. 在jQuery源码中你看到过没有返回值的函数码？似乎没有，teardown返回false表示失败，还得用普通方式删除
				if ( !special.teardown || special.teardown.call( elem, namespaces ) === false ) { 
					jQuery.removeEvent( elem, type, elemData.handle );
				}

				ret = null; // 这个变量搞笑了，自从定义之后就从来就没用过，明显多余，可以遗留代码里的变量吧
				delete events[ type ]; // 删除类型属性，还是直接对数组句柄数组操作，根本没有什么缓存微操接口
			}
		}

		// Remove the expando if it's no longer used
		// 从jQuery.cache中移除elem的数据，最后检查elemData.events
		if ( jQuery.isEmptyObject( events ) ) {
			var handle = elemData.handle;
			if ( handle ) {
				handle.elem = null; // 置空，删除对HTML元素的引用，避免因垃圾回收机制不起作用导致的浏览器内存泄漏
				// 这是一个非常好的技巧，我们在JavaScript中引用完HTML元素后一定要置空。
			}

			delete elemData.events; // 删除存储事件句柄的对象
			delete elemData.handle; // 删除DOM事件句柄

			if ( jQuery.isEmptyObject( elemData ) ) { // 都删了难道还不是isEmptyObject？有点多余，除非某些浏览器不支持delete，可问题是不支持也得移除数据，还是多余！
				jQuery.removeData( elem, undefined, true ); // 彻底的从缓存中移走elem的内部数据（存储在属性expando上）
			}
		}
	},

	// ...
};
10.6	.live() .die()
10.6.1	如何使用
live 在匹配当前选择器的元素上绑定一个事件处理函数，包括已经存在的和未来添加的，即任何添加的元素只要匹配当前选择器，就会被绑定事件处理函数
.live( eventType, handler )	在匹配当前选择器的元素上绑定一个指定类型的事件处理函数
.live( eventType, eventData, handler )	eventData可以传递数据给事件处理函数
.live( events )	绑定多个事件
.die()	删除之前通过.live()绑定的全部事件句柄
.die( eventType [, handler] )	eventType是字符串事件类型，删除一个或一类之前通过.live()绑定的事件句柄
.die( eventTypes )	eventTypes是Map对象，删除多类事件句柄
10.6.2	调用过程

.live()方法永远不会将事件绑定到匹配的元素上，而是将事件绑定到祖先元素（document或context），由该祖先元素代理子元素的事件。
以 $('.clickme').live('click', function() { … } ) 为例，当在新添加的元素上点击时，执行过程如下：
1. click事件被生成，并传递给<div>，待<div>处理            
2. 因为<div>没有直接绑定click事件，因此事件沿着DOM树进行冒泡传递          
3. 事件冒泡传递直到到达DOM树的根节点，在根节点上绑定了.live()指定的原始事件处理函数（在1.4以后的版本中，.live()绑定事件到上下文context，以提升性能） 
4. 执行.live()指定的事件处理函数 
5. 原始事件处理函数检查event的target属性以确定是否继续执行，检查的方式是 $(event.target).closes
6. 如果匹配，则原始事件处理函数执行，上下位被设置为找到的元素                                                                  
因为第5步的检查是在执行原始事件处理函数之前，因此元素可以在任何时候添加，并且事件可以生效
调用过程：
$('div').live( 'click', function(){ alert(1); } )
jQuery.fn.live → jQuery.event.add → jQuery._data → jQuery.event.special.live.add → jQuery.event.add
$('div').die( 'click', function(){ alert(1); } )
jQuery.fn.die → jQuery.fn.unbind → jQuery.event.remove jQuery._data
调试过程中发现个有趣的现象：在父节点上绑定live事件，而不是具体的时间类型？
 
10.6.3	源码分析
看看.live()/.die()的源码分析：
// 事件类型修正
var liveMap = {
	focus: "focusin",
	blur: "focusout",
	mouseenter: "mouseover",
	mouseleave: "mouseout"
};
// die 移除用live附加的一个或全部事件处理函数
// 对应关系：bind-unbind, live-die, delegate-undelegate
/**
 * live 在匹配当前选择器的元素上绑定一个事件处理函数，包括已经存在的，和未来添加的，即任何添加的元素只要匹配当前选择器就会被绑定事件处理函数
 * .live( eventType, handler ) 在匹配当前选择器的元素上绑定一个指定类型的事件处理函数
 * .live( eventType, eventData, handler ) eventData可以传递给事件处理函数
 * .live( events ) 绑定多个事件
 * 
 * 
 * 参考文档：
 * http://api.jquery.com/live
 * http://www.codesky.net/article/doc/201004/20100417042278.htm
 * http://hi.baidu.com/silveringsea/blog/item/55cd25016ecde30c1c958341.html
 * 
 * 如何使用
 * 示例
 * 局限
 * 修正
 * 
 * live被翻译为鲜活绑定、延迟绑定（更准确些）
 * live通过委派，而不是在匹配元素上直接绑定事件来实现
 * 
 * 官方文档翻译：事件代理
 * .live()方法能对尚未添加到DOM文档中的元素生效。
 * .live()方法永远不会将事件绑定到匹配的元素上，而是将事件绑定到祖先元素（document或context），由该祖先元素代理子元素的事件。	 * 
 * 以 $('.clickme').live('click', function() { … } ) 为例，当在新添加的元素上点击时，执行过程如下：                    
 * 1. click事件被生成，并传递给<div>，待<div>处理
 * 2. 因为<div>没有直接绑定click事件，因此事件沿着DOM树进行冒泡传递
 * 3. 事件冒泡传递直到到达DOM树的根节点，在根节点上绑定了.live()指定的原始事件处理函数（在1.4以后的版本中，.live()绑定事件到上下文context，以提升性能）
 * 4. 执行.live()指定的事件处理函数
 * 5. 原始事件处理函数检查event的target属性以确定是否继续执行，检查的方式是 $(event.target).closest(".clickme")
 * 6. 如果匹配，则原始事件处理函数执行，上下位被设置为找到的元素
 * 
 * 因为第5步的检查是在执行原始事件处理函数之前，因此元素可以在任何时候添加，并且事件可以生效
 * 
 */
jQuery.each(["live", "die"], function( i, name ) {
	jQuery.fn[ name ] = function( types, data, fn, origSelector /* Internal Use Only */ ) {
		var type, i = 0, match, namespaces, preType,
			selector = origSelector || this.selector, // 选择器表达式
			// 很巧妙，如果没有origSelector，那么采用当前jQuery对象的选择器，、
			// 可是，如果origSelector为true，后边的处理对象就变成了以origSelector为选择器的jQuery对象
			context = origSelector ? this : jQuery( this.context ); // 上下文,
			// 这里也是，如果没有origSelector，那么上下文就变成当前jQuery对象的上下文
			// 可是，如果origSelector为true（类型转换），当前jQuery对象就变成上下文了
			// 就是在这里，通过一个内部变量，改变了选择器表达式和上下分，区分开了live/die和delegate/undelegate，用同样的代码实现了两种接口

		// 一次绑定或删除多个事件
		if ( typeof types === "object" && !types.preventDefault ) { // types不是jQuery事件对象
			for ( var key in types ) { // 迭代复用
				context[ name ]( key, data, types[key], selector );
			}

			return this;
		}

		// 如果是die，移除origSelector匹配元素上的所有通过live绑定的事件
		// 没有指定事件类型 + origSelector + origSelector是CSS选择器 
		if ( name === "die" && !types &&
					origSelector && origSelector.charAt(0) === "." ) {

			context.unbind( origSelector ); // 事实上bind/unbind是基础，add/remove/tigger/handle是所有事件的基础

			return this;
		}
		
		// 修正参数（如果data为false或没有传入data参数）
		// 完整：types, data, fn, origSelector
		// types, false, fn, origSelector
		// 或 types, fn, origSelector
		if ( data === false || jQuery.isFunction( data ) ) {
			fn = data || returnFalse;
			data = undefined;
		}

		types = (types || "").split(" "); // 多个事件用空格隔开

		while ( (type = types[ i++ ]) != null ) { // 遍历事件类型数组
			match = rnamespaces.exec( type ); // 取到第一个.后的命名空间 
			namespaces = "";

			if ( match )  { // 如果有命名空间，即事件类型后还有.
				namespaces = match[0]; // .+命名空间
				type = type.replace( rnamespaces, "" ); // 过滤.+命名空间，初学者会尝试用indexOf查找.的位置，然后截取
			}

			if ( type === "hover" ) { // 将hover分解为mouseenter mouseleave，继续遍历
				types.push( "mouseenter" + namespaces, "mouseleave" + namespaces );
				continue;
			}

			preType = type; // 

			if ( liveMap[ type ] ) { // 修正事件名名，修正其实不准确，比如遇到focus，绑定了两个：focus focusin，不会造成两次事件响应么？
				types.push( liveMap[ type ] + namespaces ); // 将修正后的事件类型入队，因为用了while循环，所有不必担心遍历动态数组的问题
				type = type + namespaces; // 再恢复type，包含了命名空间，这不是蛋疼么，把命名空间去掉只为了检测hover和是否需要修正？

			} else {
				type = (liveMap[ type ] || type) + namespaces; // 修正事件名，这里可以直接写成：type + namespaces
				// 这行的逻辑有点问题，已经知道liveMap[ type ]是false了，这个地方的代码有些重复
			}
			/**
			 * 上边的if-else改写为：
			 * if ( liveMap[ type ] ) types.push( liveMap[ type ] + namespaces )
			 * type = type + namespaces
			 */

			// 这里开始live的特殊处理，虽说live/die的逻辑比较接近，再加上delegate/undelegate，复用的粒度小但是很精髓
			// 同时为了减少接口，并没有将公共部分提取出来，总的来说，live/die这段代码出来的还是刚刚好
			if ( name === "live" ) {
				// bind live handler
				for ( var j = 0, l = context.length; j < l; j++ ) {
					// 绑定到上下文，事件类型经过liveConvert后变为 live.type.selector
					// 前边做了那么多铺垫，这行才是关键！
					// add: function( elem, types, handler, data ) {
					// context[j] 如果有origSelector则是当前jQuery对象，如果没有则是当前jQuery对象的上下文
					// live事件的格式比较特殊，应该trigger里会有live的特殊处理，live的处理在特例special里
					jQuery.event.add( context[j], "live." + liveConvert( type, selector ),
						{ data: data, selector: selector, handler: fn, origType: type, origHandler: fn, preType: preType } );
				}

			} else {
				// unbind live handler
				// 删除live绑定的事件句柄
				context.unbind( "live." + liveConvert( type, selector ), fn );
			}
		}

		return this;
	};
});
/**
 * live执行句柄，live绑定的事件调用链：add.eventHandle → handler function liveHandler → _data → handler 
 * @param event
 * @return
 */
function liveHandler( event ) {
	var stop, maxLevel, related, match, handleObj, elem, j, i, l, data, close, namespace, ret,
		elems = [],
		selectors = [],
		events = jQuery._data( this, "events" );

	// Make sure we avoid non-left-click bubbling in Firefox (#3861) and disabled elements in IE (#6911)
	if ( event.liveFired === this || !events || !events.live || event.target.disabled || event.button && event.type === "click" ) {
		return;
	}

	if ( event.namespace ) {
		namespace = new RegExp("(^|\\.)" + event.namespace.split(".").join("\\.(?:.*\\.)?") + "(\\.|$)");
	}

	event.liveFired = this;

	var live = events.live.slice(0); // 返回一个新的数组，而不改变原来的数组

	for ( j = 0; j < live.length; j++ ) {
		handleObj = live[j];

		if ( handleObj.origType.replace( rnamespaces, "" ) === event.type ) {
			selectors.push( handleObj.selector );

		} else {
			live.splice( j--, 1 );
		}
	}

	match = jQuery( event.target ).closest( selectors, event.currentTarget );

	for ( i = 0, l = match.length; i < l; i++ ) {
		close = match[i];

		for ( j = 0; j < live.length; j++ ) {
			handleObj = live[j];

			if ( close.selector === handleObj.selector && (!namespace || namespace.test( handleObj.namespace )) && !close.elem.disabled ) {
				elem = close.elem;
				related = null;

				// Those two events require additional checking
				if ( handleObj.preType === "mouseenter" || handleObj.preType === "mouseleave" ) {
					event.type = handleObj.preType;
					related = jQuery( event.relatedTarget ).closest( handleObj.selector )[0];

					// Make sure not to accidentally match a child element with the same selector
					if ( related && jQuery.contains( elem, related ) ) {
						related = elem;
					}
				}

				if ( !related || related !== elem ) {
					elems.push({ elem: elem, handleObj: handleObj, level: close.level });
				}
			}
		}
	}

	for ( i = 0, l = elems.length; i < l; i++ ) {
		match = elems[i];

		if ( maxLevel && match.level > maxLevel ) {
			break;
		}

		event.currentTarget = match.elem;
		event.data = match.handleObj.data;
		event.handleObj = match.handleObj;

		ret = match.handleObj.origHandler.apply( match.elem, arguments );

		if ( ret === false || event.isPropagationStopped() ) {
			maxLevel = match.level;

			if ( ret === false ) {
				stop = false;
			}
			if ( event.isImmediatePropagationStopped() ) {
				break;
			}
		}
	}

	return stop;
}
// 在live事件类型type后增加上selector，作为命名空间
function liveConvert( type, selector ) {
	// . > `
	// 空格 > &
	// 在type后增加上selector，作为命名空间
	return (type && type !== "*" ? type + "." : "") + selector.replace(rperiod, "`").replace(rspaces, "&"); // 
}
10.7	.deletege() .undelegate()
10.7.1	如何使用
.delegate( selector, eventType, handler )	在匹配选择器表达式selector的元素上，将一个事件句handler柄绑定到一个或多个事件上
无论是现有的还是将来添加的，基于一组特定的根元素（即上下文）
.delegate( selector, eventType, eventData, handler )	Map型对象eventData的属性将被复制到事件句柄handler
.delegate( selector, events )	绑定一个或多个事件句柄
.undelegate()	在当前jQuery对象上，移除所有live事件句柄
.undelegate( selector, eventType, handler )	在当前jQuery对象上，移除live. eventType.selector事件句柄
为当前选择所匹配的所有元素移除一个事件处理程序，现在或将来，基于一组特定的根元素。
.undelegate( selector, events )	在当前jQuery对象上，移除多个live事件句柄，events是Map型对象，key
.undelegate( namespace )	在当前jQuery对象上，移除命名空间匹配的live事件句柄
从接口的含义看，.deletege()/.undelegate()与.live()/.die()的区别在于绑定事件时的上下文不同，上下文即事件绑定到的元素：
.deletege()/.undelegate()	当前jQuery对象
.live()/.die()	当前jQuery对象的上下文，经常是document，也可以是构造jQuery对象时指定的上下文

10.7.2	调用过程
jQuery.fn.delegate → jQuery.fn.live → jQuery.event.add
jQuery.fn.undelegate → jQuery.fn.unbind/die
10.7.3	源码分析
// 事件代理，调用live方法实现
delegate: function( selector, types, data, fn ) {
	// 因为传递了参数selector，live时的上下文变为this，而不是this.context
	return this.live( types, data, fn, selector );
},
// 删除事件代理，调用unbind或die实现
undelegate: function( selector, types, fn ) {
	if ( arguments.length === 0 ) {
		return this.unbind( "live" ); // 在当前jQuery对象上直接unbind，不需要在去查找上下文

	} else {
		// jQuery.fn[ die ] = function( types, data, fn, origSelector /* Internal Use Only */ ) {
		return this.die( types, null, fn, selector ); //　带有选择器字符串，改变了die里的上下文context变量
	}
},
本节小节：
1.	.bind()/.unbind()高级接口的基础，add/remove/tigger/handle是所有的基础，最后才是调用浏览器原生接口
2.	.deletege()/.undelegate()与.live()/.die()的区别在于绑定事件时的父节点不同
后文预告：封装事件对象 便捷接口解析 ready专题

```