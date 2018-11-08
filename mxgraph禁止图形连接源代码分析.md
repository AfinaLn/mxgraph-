# 特殊情况下mxgraph图形间连接有效性判断源代码流程分析
## 一、分析图形连接处理的入口代码
 >  Class: mxConnectionHandler
 * Class: mxConnectionHandler
  Graph event handler that creates new connections. Uses <mxTerminalMarker>
  for finding and highlighting the source and target vertices and
  <factoryMethod> to create the edge instance. This handler is built-into
 <mxGraph.connectionHandler> and enabled using <mxGraph.setConnectable>.
 
 * #### mxConnectionHandler 负责绘制图形间的连接

> Function: mouseMove
  * Handles the event by updating the preview edge or by highlighting a possible source or target terminal.
  * mxConnectionHandler.prototype.mouseMove = function(sender, me)

* #### 在mouseMove方法中绘制预览连接线并高亮连接显示所要连接的图形

#### 分析代码，可以判断这个mxCoonnectionHandler的mouseMove方法就是图形连接的入口代码

## 二、分析mouseMove方法，寻找判断图形连接有效性代码

```
//Handles special case when handler is disabled during highlight
		if (!this.isEnabled() && this.currentState != null)
		{
			this.destroyIcons();
			this.currentState = null;
		}

		var view = this.graph.getView();
		var scale = view.scale;
		var tr = view.translate;
		var point = new mxPoint(me.getGraphX(), me.getGraphY());
		this.error = null;

		if (this.graph.isGridEnabledEvent(me.getEvent()))
		{
			point = new mxPoint((this.graph.snap(point.x / scale - tr.x) + tr.x) * scale,
				(this.graph.snap(point.y / scale - tr.y) + tr.y) * scale);
		}
		
		this.snapToPreview(me, point);
		this.currentPoint = point;
		
		if ((this.first != null || (this.isEnabled() && this.graph.isEnabled())) &&
			(this.shape != null || this.first == null ||
			Math.abs(me.getGraphX() - this.first.x) > this.graph.tolerance ||
			Math.abs(me.getGraphY() - this.first.y) > this.graph.tolerance))
		{
			this.updateCurrentState(me, point);
		}
```
#### mouseMove前部分代码作用是绘制出当前连线鼠标坐标点，并更新当前状态，接下来进去看看这个updateCurrentState方法做了些什么事。
> 
 * Function: updateCurrentState
 * Updates the current state for a given mouse move event by using the `<marker>`.
 * mxConnectionHandler.prototype.updateCurrentState = function(me, point)
 
```
	// Updates validation state
		if (this.previous != null)
		{
			this.error = this.validateConnection(this.previous.cell, this.constraintHandler.currentFocus.cell);
			
			if (this.error == null)
			{
				this.currentState = this.constraintHandler.currentFocus;
			}
			else
			{
				this.constraintHandler.reset();
			}
		}
```
#### updateCurrentState代码中有一段updates validation state的代码，其中调用了validateConnection方法，从字面意思来看就是验证连接的有效性，进去看看，嘿嘿嘿！

> 
 * Function: validateConnection
 * Returns the error message or an empty string if the connection for the
  given source target pair is not valid. Otherwise it returns null. This
 * implementation uses `<mxGraph.getEdgeValidationError>`.
 
```
mxConnectionHandler.prototype.validateConnection = function(source, target)
{
	if (!this.isValidTarget(target))
	{
		return '';
	}
	
	return this.graph.getEdgeValidationError(null, source, target);
};
```
#### 代码就简单的几行，就调用了两个方法，一个isValidTarget,一个graph的getEdgeValidationError方法，下面分别看下这两个方法。
```

/**
 * Function: isValidTarget
 * 
 * Returns true. The call to <mxGraph.isValidTarget> is implicit by calling
 * <mxGraph.getEdgeValidationError> in <validateConnection>. This is an
 * additional hook for disabling certain targets in this specific handler.
 * 
 * Parameters:
 * 
 * cell - <mxCell> that represents the target terminal.
 */
mxConnectionHandler.prototype.isValidTarget = function(cell)
{
	return true;
};
```
#### isValidTarget直接返回true,这是一个保留的钩子方法，因为只判断了目标图形，所以对判断是否是有效的连接意义不大
```
/**
 * Function: getEdgeValidationError
 * 
 * Returns the validation error message to be displayed when inserting or
 * changing an edges' connectivity. A return value of null means the edge
 * is valid, a return value of '' means it's not valid, but do not display
 * an error message. Any other (non-empty) string returned from this method
 * is displayed as an error message when trying to connect an edge to a
 * source and target. This implementation uses the <multiplicities>, and
 * checks <multigraph>, <allowDanglingEdges> and <allowLoops> to generate
 * validation errors.
 * 
 * For extending this method with specific checks for source/target cells,
 * the method can be extended as follows. Returning an empty string means
 * the edge is invalid with no error message, a non-null string specifies
 * the error message, and null means the edge is valid.
 * 
 * (code)
 * graph.getEdgeValidationError = function(edge, source, target)
 * {
 *   if (source != null && target != null &&
 *     this.model.getValue(source) != null &&
 *     this.model.getValue(target) != null)
 *   {
 *     if (target is not valid for source)
 *     {
 *       return 'Invalid Target';
 *     }
 *   }
 *   
 *   // "Supercall"
 *   return mxGraph.prototype.getEdgeValidationError.apply(this, arguments);
 * }
 * (end)
 *  
 * Parameters:
 * 
 * edge - <mxCell> that represents the edge to validate.
 * source - <mxCell> that represents the source terminal.
 * target - <mxCell> that represents the target terminal.
 */
```
#### getEdgeValidationError方法说明很长，很详细，简单就是说返回null表明连线是有效的，返回字符串表明两个图形之间的连线是无效的。并说明通过扩展getEdgeValidationError方法，检查图形间连线的有效性，并给出了示例代码。

### 分析到这里基本上就完成了整个禁止特殊情况下的图形连接的源代码分析，通过扩展getEdgeValidationError方法，实现自定义的约束连接规则。

#### one more thing，看看getEdgeValidationError方法，嘿嘿嘿
```
mxGraph.prototype.getEdgeValidationError = function(edge, source, target)
{

	if (edge != null && !this.isAllowDanglingEdges() && (source == null || target == null))
	{
		return '';
	}
	
	if (edge != null && this.model.getTerminal(edge, true) == null &&
		this.model.getTerminal(edge, false) == null)	
	{
		return null;
	}
	
	// Checks if we're dealing with a loop
	if (!this.allowLoops && source == target && source != null)
	{
		return '';
	}
	
	// Checks if the connection is generally allowed
	if (!this.isValidConnection(source, target))
	{
		return '';
	}


	if (source != null && target != null)
	{
		var error = '';

		// Checks if the cells are already connected
		// and adds an error message if required			
		if (!this.multigraph)
		{
			var tmp = this.model.getEdgesBetween(source, target, false);
			
			// Checks if the source and target are not connected by another edge
			if (tmp.length > 1 || (tmp.length == 1 && tmp[0] != edge))
			{
				error += (mxResources.get(this.alreadyConnectedResource) ||
					this.alreadyConnectedResource)+'\n';
			}

		}

		// Gets the number of outgoing edges from the source
		// and the number of incoming edges from the target
		// without counting the edge being currently changed.
		var sourceOut = this.model.getDirectedEdgeCount(source, true, edge);
		var targetIn = this.model.getDirectedEdgeCount(target, false, edge);

		// Checks the change against each multiplicity rule
		if (this.multiplicities != null)
		{
			for (var i = 0; i < this.multiplicities.length; i++)
			{
				var err = this.multiplicities[i].check(this, edge, source,
					target, sourceOut, targetIn);
				
				if (err != null)
				{
					error += err;
				}
			}
		}

		// Validates the source and target terminals independently
		var err = this.validateEdge(edge, source, target);
		
		if (err != null)
		{
			error += err;
		}
		
		return (error.length > 0) ? error : null;
	}
	
	return (this.allowDanglingEdges) ? null : '';
};
```
#### 整个方法就各种不同情况，例如是否允许悬空的连线，是否可以循环连接，是否重复连接等条件进行了判断，不过其中这段代码貌似有些意思：
```
// Validates the source and target terminals independently
        var err = this.validateEdge(edge, source, target);
```
#### 进去看看
```
/**
 * Function: validateEdge
 * 
 * Hook method for subclassers to return an error message for the given
 * edge and terminals. This implementation returns null.
 *
 * Parameters:
 * 
 * edge - <mxCell> that represents the edge to validate.
 * source - <mxCell> that represents the source terminal.
 * target - <mxCell> that represents the target terminal.
 */
mxGraph.prototype.validateEdge = function(edge, source, target)
{
	return null;
};
```
#### 又是一个保留方法，留给子类实现的，并且有edge,source,target,三个参数，非常适合单独做连接有效性判断，所以可以通过重新该方法实现连接有效性判断。

## 最后，通过分析阅读代码，可以通过扩展mxGraph.prototype.getEdgeValidationError方法和实现mxGraph.prototype.validateEdge方法自定义图形间连接的有效性判断。