## For introductions, tutorials and manuals, please visit [jscex.info](http://jscex.info/).

## 介绍、向导及手册等详细信息，请访问[jscex.info](http://jscex.info/zh-cn/)。


## 该项目对 [jscex.info](http://jscex.info/) 尝试做了些改建：
	1、去掉显式调用eval + Jscex.compile 代码编译，延迟到运行时刻再做编译
	2、简化$await 调用方式

	示例：
	原来的 sorting-animations.html 的bubble排序例子：

            var compareAsync = eval(Jscex.compile("async", function (x, y, comparing) {
                paint(array, null, comparing);
                $await(Jscex.Async.sleep(compareCost));
                return x - y;
            }));

            var swapAsync = eval(Jscex.compile("async", function (array, i, j) {
                var t = array[i];
                array[i] = array[j];
                array[j] = t;

                paint(array, [i, j]);
                $await(Jscex.Async.sleep(updateCost));
            }));
			
			var sortOperations ={
			
			......
			
			Bubble: eval(Jscex.compile("async", function (array) {
				for (var i = 0; i < array.length; i++) {
					for (var j = 0; j < array.length - i; j++) {
						var r = $await(compareAsync(array[j], array[j + 1], [j, j + 1]));
						if (r > 0) $await(swapAsync(array, j, j + 1));
					}
				}
			})),

			......
			
			}
			
			调用代码：
			
            this.sortAsync = eval(Jscex.compile("async", function (name, array) {
                paint(array);
                $await(sortOperations[name](array));
                paint(array);
            }));
			
	可以简化为：

            function compareAsync(x, y, comparing) {
                //paint(array, null, comparing);  // 该处有问题，array数组不在可见范围内，注释掉不影响
                $call(Jscex.Async.sleep(compareCost));
                return x - y;
            };

			function swapAsync(array, i, j) {
                var t = array[i];
                array[i] = array[j];
                array[j] = t;

                paint(array, [i, j]);
                $call(Jscex.Async.sleep(updateCost)); // 该处改为调用 $call 等价 于 $await
            };
		
			var sortOperations ={
			
			......
			
			Bubble: function (array) {
				for (var i = 0; i < array.length; i++) {
					for (var j = 0; j < array.length - i; j++) {
						var r = compareAsync.$call(array[j], array[j + 1], [j, j + 1]);  // 该处改为调用 $call 等价 于 $await
						if (r > 0) swapAsync.$call(array, j, j + 1); // 该处改为调用 $call 等价 于 $await
					}
				}
			},
			
			......
			
			}

			调用：

            this.sortAsync = function (name, array) {
                paint(array);
                sortOperations[name].$call(array); // 该处改为调用 $call 等价 于 $await
                paint(array);
            };
			
## 实现机制

	1、增加Function.protoype原型方法

	@see jscex-jit.js @line ~1723 
	
	// --------------------------------------------------------------------->>
	// add simple async function call
	// --------------------------------------------------------------------->>
	
	Function.prototype.$call=function(){
		if (!this.__asyncall) this.__asyncall = eval(Jscex.compile("async", this));
		return this.__asyncall.apply(this,arguments);
	}

	当你要对某个方法做异步等待调用时，如compareAsync(args)，改为这样调用compareAsync.$call(args)，表示对该方法进行异步调用。
	
	$call方法首先检查是否已经编译了异步代码，如果没有，就调用Jscex的编译命令，即时生成代码，并缓存到该函数对象内__asyncall, 以及后再次调用该方法的时候，就无需再次编译了
	这里就是把原来显示编译的处理，推迟到调用时刻。
	
	2、修改异步代码生成规则
	
	由于对Function.prototype 增加了一个特殊的异步调用方法，所以，在Jscex compile 处理时，需要转换为为跟原来 $await(function(....)) 等价的代码，只需要修改JscexTreeGenerator 的方法：
		
		===>> @see jscex-jit.js  ~@line 243 
		
        _getBindInfo: function (stmt) {

            var type = stmt[0];
            if (type == "stat") {
                var expr = stmt[1];
                if (expr[0] == "call") {
                    var callee = expr[1];
                    if (callee[0] == "name" && callee[1] == this._binder && expr[2].length == 1) {
                        return {
                            expression: expr[2][0],
                            argName: "",
                            assignee: null
                        };
                    }
					// ---------------------------------->> add begin
					else if (callee[0] == "dot" && callee[2] == this._binder) {
						return {
							expression: expr,
                            argName: "",
                            assignee: null
						};
					}
					// ---------------------------------->> add end
                } else if (expr[0] == "assign") {
                    var assignee = expr[2];
                    expr = expr[3];
                    if (expr[0] == "call") {
                        var callee = expr[1];
                        if (callee[0] == "name" && callee[1] == this._binder && expr[2].length == 1) {
                            return {
                                expression: expr[2][0],
                                argName: "_result_$",
                                assignee: assignee
                            };
                        }
					// ---------------------------------->> add begin
						else if (callee[0] == "dot" && callee[2] == this._binder) {
							return {
								expression: expr,
                                argName: "_result_$",
                                assignee: assignee
							};
						}
					// ---------------------------------->> add end
                    }
                }
            } else if (type == "var") {
                var defs = stmt[1];
                if (defs.length == 1) {
                    var item = defs[0];
                    var name = item[0];
                    var expr = item[1];
                    if (expr && expr[0] == "call") {
                        var callee = expr[1];
                        if (callee[0] == "name" && callee[1] == this._binder && expr[2].length == 1) {
                            return {
                                expression: expr[2][0],
                                argName: name,
                                assignee: null
                            };                            
                        }
					// ---------------------------------->> add begin
						else if (callee[0] == "dot" && callee[2] == this._binder) {
                            return {
                                expression: expr,
                                argName: name,
                                assignee: null
                            };
                        }
					// ---------------------------------->> add end
                    }
                }
            } else if (type == "return") {
                var expr = stmt[1];
                if (expr && expr[0] == "call") {
                    var callee = expr[1];
                    if (callee[0] == "name" && callee[1] == this._binder && expr[2].length == 1) {
                        return {
                            expression: expr[2][0],
                            argName: "_result_$",
                            assignee: "return"
                        };
                    }
					// ---------------------------------->> add begin
					else if (callee[0] == "dot" && callee[2] == this._binder) {
						return {
							expression: expr,
                            argName: "_result_$",
                            assignee: "return"
						};
					}
					// ---------------------------------->> add end
                }
            }

            return null;
        },

	3、修改$await 为$call
		搜索代码中的$await ,改为 $call即可

		
	经过这样修改，
	func.$call(....)的调用，就等价 于 $await(function(...))，
	$call方法也可以直接调用，跟原来的$await 一样效果，
	形式上就简洁一些。
	
	
## 潜在问题
	1、由于compile推迟的执行时刻，编译所处的 context 环境可能不是全局环境， 所以function中用到的变量、函数需要注意是否可见性。
	2、项目中的示例只修改了 samples/async/sorting-animations.html一个例子，其他都没有修改，噢哦，-:)
