###                                         Init构造函数

从前文的Jquery架构分析可以看出，Init构造函数是框架内部入口函数，$()工厂方法返回的其实是Init构造函数的实例。

要理解init构造函数，得先从一个内部变量开始。

```js
quickExpr = /^(?:\s*(<[\w\W]+>)[^>]*|#([\w-]*))$/,
```
第一个匹配组用来匹配带括号的标签，第二个用来匹配ID。例如：

```js
quickExpr.exec('  <a>test<div></div></a>');
quickExpr.exec('#wraper');
```

结果：
​     ["  <a>test<div></div></a>", "<a>test<div></div></a>", undefined]

​     ["#wraper", undefined, "wraper"]

**接下来开始分析init构造函数**

### init函数12个分支：

![Markdown](http://i1.piimg.com/596412/a329f13c720af0da.png)

![Markdown](http://i1.piimg.com/596412/a64e9f103087452b.png)

```js
//rootjQuery = jQuery(document);
function( selector, context, rootjQuery ) {
		var match, elem, ret, doc;

		// Handle $(""), $(null), or $(undefined)
		if ( !selector ) {
			return this;
		}

		// Handle $(DOMElement)
		if ( selector.nodeType ) {
			this.context = this[0] = selector;
			this.length = 1;
			return this;
		}

		// The body element only exists once, optimize finding it
		if ( selector === "body" && !context && document.body ) {
			this.context = document;
			this[0] = document.body;
			this.selector = selector;
			this.length = 1;
			return this;
		}

		// Handle HTML strings
		if ( typeof selector === "string" ) {
			// Are we dealing with HTML string or an ID?
			if ( selector.charAt(0) === "<" && selector.charAt( selector.length - 1 ) === ">" && selector.length >= 3 ) {
				// Assume that strings that start and end with <> are HTML and skip the regex check
				match = [ null, selector, null ];

			} else {
				match = quickExpr.exec( selector );
			}

			// Verify a match, and that no context was specified for #id
			if ( match && (match[1] || !context) ) {

				// HANDLE: $(html) -> $(array)
				if ( match[1] ) {
					context = context instanceof jQuery ? context[0] : context;
					doc = ( context ? context.ownerDocument || context : document );

					// If a single string is passed in and it's a single tag
					// just do a createElement and skip the rest
					ret = rsingleTag.exec( selector );

					if ( ret ) {
						if ( jQuery.isPlainObject( context ) ) {
							selector = [ document.createElement( ret[1] ) ];
                             //调用属性操作方法，例如:$(selector).attr(context[0],context[1]);
							jQuery.fn.attr.call( selector, context, true );

						} else {
							selector = [ doc.createElement( ret[1] ) ];
						}
					//是复制html则创建文档片段
					} else {
						ret = jQuery.buildFragment( [ match[1] ], [ doc ] );
						selector = ( ret.cacheable ? jQuery.clone(ret.fragment) : ret.fragment ).childNodes;
					}

					return jQuery.merge( this, selector );

				// HANDLE: $("#id")
				} else {
					elem = document.getElementById( match[2] );

					// Check parentNode to catch when Blackberry 4.6 returns
					// nodes that are no longer in the document #6963
					if ( elem && elem.parentNode ) {
						// Handle the case where IE and Opera return items
						// by name instead of ID
						if ( elem.id !== match[2] ) {
							return rootjQuery.find( selector );
						}

						// Otherwise, we inject the element directly into the jQuery object
						this.length = 1;
						this[0] = elem;
					}

					this.context = document;
					this.selector = selector;
					return this;
				}

			// 不是字符串，contex为空，或者contex是一个jquery对象。HANDLE: $(expr, $(...))
			} else if ( !context || context.jquery ) {
              
				return ( context || rootjQuery ).find( selector );

			// HANDLE: $(expr, context)
			// (which is just equivalent to: $(context).find(expr)
			} else {
				return this.constructor( context ).find( selector );
			}

		// HANDLE: $(function)
		// Shortcut for document ready
		} else if ( jQuery.isFunction( selector ) ) {
			return rootjQuery.ready( selector );
		}

		if ( selector.selector !== undefined ) {
			this.selector = selector.selector;
			this.context = selector.context;
		}

		return jQuery.makeArray( selector, this );
	}
```

12个分支总结为：

![Markdown](http://i1.piimg.com/596412/d939411a188078f5.png)

## 2．4 jQuery.buildFragment( args, nodes, scripts）

### 2．4．1实现原理

​	方法 jQuery.buildFragment( args, nodes, scripts）先创建一个文档片段 DocumentFragment,然后调用方法 jQuery.clean( elems, context, fragment, scripts）将 HTML代码转换为 DOM元素，并存储在创建的文档片段中。

​	文档片段 DocumentFragment表示文档的一部分，但不属于文档树。当把 Document Fragment插人文档树时，插人的不是 DocumentFragment自身，而是它的所有子孙节点，即可以一次向文档树中插人多个节点。当需要插人大量节点时，相比于逐个插人节点，使用 ocumentFragment一次插人多个节点，性能的提升会非常明显 (ie除外)。

​	此外，如果 HTML代码符合缓存条件，方法 jQuery.buiIdFragment()还会把转换后的 DOM元素缓存起来，下次（实际上是第三次）转换相同的 HTML代码时直接从缓存中读取，不需要重复转换。

​	方法 jQuery.bmldFragment()同时为构造 jQue对象和 DOM操作提供底层支持， DOM操作将在第11章介绍和分析。

### 2．4．2源码分析

方法 jQuery.buildFragment( args， nodes， scripts）执行的5个关键步骤如下：

1）如果 HTML代码符合缓存条件，则尝试从缓存对象 jQuery.fragments中读取缓存的 DOM元素。

2）创建文档片段 DocumentFragment。

3）调用方法 jQuery.cIean( elems, context, fragment, scripts）将 HTML代码转换为 DOM元素，并存储在创建的文档片段中。

4）如果 HTML代码符合缓存条件，则把转换后的 DOM元素放人缓存对象 jQuery fragmentso

5）最后返回文档片段和缓存状态{ fragment: fragment, cacheable: cacheable}。

下面来看看该方法的源码实现。



```js
/*
口参数 args：数组，含有待转换为 DOM元素的 HTML代码。

口参数 nodes：数组，含有文档对象、 jQuery对象或 DOM元素，用于修正创建文档片段 DocumentFragment的文档对象

口参数scripts：数组，用于存放 HTML代码中的script元素。
*/
jQuery.buildFragment = function( args, nodes, scripts ) {
  
    //定义局部变量。变量fagment指向稍后可能创建的文档片段dcumentFragment；变量 cacheable表示 HTML代码是否符合缓存条件；变量 cacheresults指向从缓存对象 jQuery.fragments中取到的文档片段，其中包含了缓存的 DOM元素；变量 doc表示创建文档片段的文档对象。
	var fragment, cacheable, cacheresults, doc,
	first = args[ 0 ];

	// 修正文档对象 doc。数组 nodes可能包含一个明确的文档对象，也可能包含 jQ对象或 DOM元素，这里先尝试读取 nodes[0]的属性ownerDocument并赋值给doc， ownerDocument表示 DOM元素所在的文档对象。如果nodes[0].ownerDocument不存在，则假定nodes[0]为文档对象并赋值给doc；但doc依然可能不是文档对象，如果调用 jQuery构造函数时第二个参数是 JavaScript对象，此时 doc是传入的 JavaScript对象而不是文档对象，例如，执行"$('abc<div></div>'，{'class':'test'}）；”时， doc是{'class':'test'}，此时需要检查 doc.createDocumentFragment是否存在，如果不存在则修正 doc为当前文档对象 document

	if ( nodes && nodes[0] ) {
		doc = nodes[0].ownerDocument || nodes[0];
	}
	if ( !doc.createDocumentFragment ) {
		doc = document;
	}

	/*  HTML代码必须满足以下所有条件，才认为符合缓存条件：
口数组 args的长度为1，且第一个元素是字符串，即数组 args中只含有一段 HTML代码。
口 HTML代码的长度小于512（1/2KB),否则可能会导致缓存占用的内存过大。
口文档对象 doc是当前文档对象，即只缓存为当前文档创建的 DOM元素，不缓存其他框架(iframe)的。
口 HTML代码以左尖括号开头，即只缓存 DOM元素，不缓存文本节点。
口 HTML代码中不能含有以下标签：<script>、<0bject>、<embed>、<option>、<style>。
口当前浏览器可以正确地复制单选按钮和复选框的选中状态 checked，或者 HTML代码中的单选按钮和复选按钮没有被选中。
口当前浏览器可以正确地复制 HTML5元素，或者 HTML代码中不含有 HTML5标签。 HTML代码中不能含有的标签定义在正则 mocache中，该正则的定义代码如下所示：
 rnocache =/<(?:script|object|embed|option|style)／i，
 HTML代码中的单选按钮和复选框是否被选中通过正则 rchecked检测，该正则的定义代
码如下所示：
//checked—"checkedt" or checked
 rchecked =/checked\s*(？：[^=]|=\s*.checked.)/i,
HTML代码中是否含有 HTML5标签通过正则 moshimcache检测，该正则的定义代码如下所示：
var nodeNames ="abbr|article|aside|audio|canvas..."*/
	if ( args.length === 1 && typeof first === "string" && first.length < 512 && doc === document &&
		first.charAt(0) === "<" && !rnocache.test( first ) &&
		(jQuery.support.checkClone || !rchecked.test( first )) &&
		(jQuery.support.html5Clone || !rnoshimcache.test( first )) ) {

		cacheable = true;

		cacheresults = jQuery.fragments[ first ];
		if ( cacheresults && cacheresults !== 1 ) {
			fragment = cacheresults;
		}
	}
    //缓存没命中
	if ( !fragment ) {
		fragment = doc.createDocumentFragment();
		jQuery.clean( args, doc, fragment, scripts );
	}
	//符合缓存条件则缓存
	if ( cacheable ) {
		jQuery.fragments[ first ] = cacheresults ? fragment : 1;
	}

	return { fragment: fragment, cacheable: cacheable };
};

```

## 2．5 jQuery.clean( elems, context, fragment, scripts）

### 2，5，1实现原理

​	方法 jQuery.clean( elems, context, fragment, scnpts）负责把 HTML代码转换成 DOM元素，并提取其中的 script元素。该方法先创建一个临时的 div元素，并将其插人一个安全文档片段中，然后把 HTML代码赋值给 div元素的 innerHTML属性，浏览器会自动生成 DOM元素，最后解析 div元素的子元素得到转换后的 DOM元素。

​	安全文档片段指能正确渲染 HTML5元素的文档片段，通过在文档片段上创建 HTML5元素，可以教会浏览器正确地渲染 HTML5元素，稍后的源码分析会介绍其实现过程。

​	如果 HTML代码中含有需要包裹在父标签中的子标签，例如，子标签<option>需要包裹在父标签<select>中，方法 jQuery.clean()会先在 HTML代码的前后加上父标签和关闭标签，在设置临时div元素的 innerHTML属性生成 DOM元素后，再层层剥去包裹的父元素，取出 HTML代码对应的 DOM元素。

​	如果 HTML代码中含有<script>标签，为了能执行<script>标签所包含的 JavaScript代码或引用的 JavaScript文件，在设置临时 div元素的 innerHTML属性生成 DOM元素后，方法jQuery•clean()会提取其中的 script元素放入数组 script，将含有<script>标签的 HTML代码设置给某个元素的 innerHTML属性后，<script>标签所包含的 JavaScript代码不会自动执行，所引用的 JavaScript文件也不会加载和执行。在生成的 DOM元素插人文档树后，数组 scripts中的 script元素会被逐个手动执行。

```js
function( elems, context, fragment, scripts ) {
		var checkScriptType, script, j,
				ret = [];

		context = context || document;

		// !context.createElement fails in IE with an error but returns typeof 'object'
		if ( typeof context.createElement === "undefined" ) {
			context = context.ownerDocument || context[0] && context[0].ownerDocument || document;
		}

		for ( var i = 0, elem; (elem = elems[i]) != null; i++ ) {
			if ( typeof elem === "number" ) {
				elem += "";
			}

			if ( !elem ) {
				continue;
			}

			// Convert html string into DOM nodes
			if ( typeof elem === "string" ) {
				if ( !rhtml.test( elem ) ) {
					elem = context.createTextNode( elem );
				} else {
					// Fix "XHTML"-style tags in all browsers
					elem = elem.replace(rxhtmlTag, "<$1></$2>");

					// Trim whitespace, otherwise indexOf won't work as expected
					var tag = ( rtagName.exec( elem ) || ["", ""] )[1].toLowerCase(),
						wrap = wrapMap[ tag ] || wrapMap._default,
						depth = wrap[0],
						div = context.createElement("div"),
						safeChildNodes = safeFragment.childNodes,
						remove;

					// Append wrapper element to unknown element safe doc fragment
					if ( context === document ) {
						// Use the fragment we've already created for this document
						safeFragment.appendChild( div );
					} else {
						// Use a fragment created with the owner document
						createSafeFragment( context ).appendChild( div );
					}

					// Go to html and back, then peel off extra wrappers
					div.innerHTML = wrap[1] + elem + wrap[2];

					// Move to the right depth
					while ( depth-- ) {
						div = div.lastChild;
					}

					// Remove IE's autoinserted <tbody> from table fragments
					if ( !jQuery.support.tbody ) {

						// String was a <table>, *may* have spurious <tbody>
						var hasBody = rtbody.test(elem),
							tbody = tag === "table" && !hasBody ?
								div.firstChild && div.firstChild.childNodes :

								// String was a bare <thead> or <tfoot>
								wrap[1] === "<table>" && !hasBody ?
									div.childNodes :
									[];

						for ( j = tbody.length - 1; j >= 0 ; --j ) {
							if ( jQuery.nodeName( tbody[ j ], "tbody" ) && !tbody[ j ].childNodes.length ) {
								tbody[ j ].parentNode.removeChild( tbody[ j ] );
							}
						}
					}

					// IE completely kills leading whitespace when innerHTML is used
					if ( !jQuery.support.leadingWhitespace && rleadingWhitespace.test( elem ) ) {
						div.insertBefore( context.createTextNode( rleadingWhitespace.exec(elem)[0] ), div.firstChild );
					}

					elem = div.childNodes;

					// Clear elements from DocumentFragment (safeFragment or otherwise)
					// to avoid hoarding elements. Fixes #11356
					if ( div ) {
						div.parentNode.removeChild( div );

						// Guard against -1 index exceptions in FF3.6
						if ( safeChildNodes.length > 0 ) {
							remove = safeChildNodes[ safeChildNodes.length - 1 ];

							if ( remove && remove.parentNode ) {
								remove.parentNode.removeChild( remove );
							}
						}
					}
				}
			}

			// Resets defaultChecked for any radios and checkboxes
			// about to be appended to the DOM in IE 6/7 (#8060)
           	//遍历转换后的 DOM元素集合，在每个元素上调用函数findlnputs( elem）。函数 findlnputs( elem）会找出其中的复选框和单选按钮，并调用函数 fixDefaultChecked把属性 checked的值赋值给属性 defaultChecked
			var len;
			if ( !jQuery.support.appendChecked ) {
				if ( elem[0] && typeof (len = elem.length) === "number" ) {
					for ( j = 0; j < len; j++ ) {
						findInputs( elem[j] );
					}
				} else {
					findInputs( elem );
				}
			}

			if ( elem.nodeType ) {
				ret.push( elem );
			} else {
				ret = jQuery.merge( ret, elem );
			}
		}

		if ( fragment ) {
			checkScriptType = function( elem ) {
				return !elem.type || rscriptType.test( elem.type );
			};
			for ( i = 0; ret[i]; i++ ) {
				script = ret[i];
				if ( scripts && jQuery.nodeName( script, "script" ) && (!script.type || rscriptType.test( script.type )) ) {
					scripts.push( script.parentNode ? script.parentNode.removeChild( script ) : script );

				} else {
					if ( script.nodeType === 1 ) {
						var jsTags = jQuery.grep( script.getElementsByTagName( "script" ), checkScriptType );

						ret.splice.apply( ret, [i + 1, 0].concat( jsTags ) );
					}
					fragment.appendChild( script );
				}
			}
		}

		return ret;
	}
```

在 IE6/7中，复选框和单选按钮插人 DOM树后，其选中状态 checked会丢失，此时测试项 jQuery.support.appendChecked为 false。通过在插人之前把属性 checked的值赋值给属性 defaultChecked，可以解决这个问题。

```js
// Finds all inputs and passes them to fixDefaultChecked
function findInputs( elem ) {
	var nodeName = ( elem.nodeName || "" ).toLowerCase();
	if ( nodeName === "input" ) {
		fixDefaultChecked( elem );
	// Skip scripts, get other children
	} else if ( nodeName !== "script" && typeof elem.getElementsByTagName !== "undefined" ) {
		jQuery.grep( elem.getElementsByTagName("input"), fixDefaultChecked );
	}
}
```



![Markdown](http://i4.buimg.com/596412/57f9338ae454fe34.png)



### 新技能get：

```js
return ( context || rootjQuery ).find( selector );
```

拓展知识：if(test){}与if(!!test)

if(test){}与if(!!test)虽然效果是一样的，可是内部机制不一样，前者**可能**调用Bool(test)将test转换成布尔值，但是!!test通过位操作，在性能上后者是有优势的。

