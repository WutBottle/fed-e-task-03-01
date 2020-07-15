# fed-e-task-03-01
手写 Vue Router、手写响应式实现、虚拟 DOM 和 Diff 算法
##1、当我们点击按钮的时候动态给 data 增加的成员是否是响应式数据，如果不是的话，如果把新增成员设置成响应式数据，它的内部原理是什么。
答：不是响应式数据。响应式对象和响应式数组是指在vue初始化时期，利用Object.defineProperty()方法对其进行监听，这样在修改数据时会及时体现在页面上。  
设置为响应式数据有两种方法：  
1、给 dog 的属性 name 设置一个初始值，可以为空字符串或者 undefined 之类的：因为给 dog 对象添加了 name 的初始值后，dog 对象的 name 属性就有了 getter 和 setter 方法，故可以实现响应式。  
2、使用 Vue.set(target, key, value) 时，target 为需要添加属性的对象，key 是要添加的属性名，value 为属性 key 对应的值：  
* 如果target是一个数组，那么根据key值及数组长度更改数组的长度(取其中较大者)，然后直接使用splice函数修改数组，虽然vue没有监听数组变化，但是监听了数组的push,pop,shift,unshift,splice,sort,reverse函数，所以使用splice也可以达到更新dom的目的。  
* 如果target是一个对象，且key是对象已存在的私有属性，那么直接赋值就可以了，因为这个key必然是被监听过的。  
* 如果这个key目前没有存在于对象中，那么会进行赋值并监听。这里省略了ob的判断，那么ob是什么呢，vue中初始化的数据(比如data中的数据)在页面初始化的时候都会被监听，而被监听的属性都会被绑定ob属性，这里就是判断这个数据有没有被监听的。如果这个数据没有被监听，那么就默认你不想监听这个数据，所以直接赋值并返回。
##2、请简述 Diff 算法的执行过程
首先patch是整个diff的入口
```js
function patch (oldVnode, vnode) {
    // some code
    if (sameVnode(oldVnode, vnode)) {
        patchVnode(oldVnode, vnode)
    } else {
        const oEl = oldVnode.el // 当前oldVnode对应的真实元素节点
        let parentEle = api.parentNode(oEl)  // 父元素
        createEle(vnode)  // 根据Vnode生成新元素
        if (parentEle !== null) {
            api.insertBefore(parentEle, vnode.el, api.nextSibling(oEl)) // 将新元素添加进父元素
            api.removeChild(parentEle, oldVnode.el)  // 移除以前的旧元素节点
            oldVnode = null
        }
    }
    // some code 
    return vnode
}
```
判断新旧节点是否一致sameVnode
```js
function sameVnode (a, b) {
  return (
    a.key === b.key &&  // key值
    a.tag === b.tag &&  // 标签名
    a.isComment === b.isComment &&  // 是否为注释节点
    // 是否都定义了data，data包含一些具体信息，例如onclick , style
    isDef(a.data) === isDef(b.data) &&  
    sameInputType(a, b) // 当标签是<input>的时候，type必须相同
  )
}
```
当确定两个节点值得比较的时候，我们会进入到patchNode方法。
```js
function patchVnode (oldVnode, vnode) {
    const el = vnode.el = oldVnode.el
    let i, oldCh = oldVnode.children, ch = vnode.children
    if (oldVnode === vnode) return
    if (oldVnode.text !== null && vnode.text !== null && oldVnode.text !== vnode.text) {
        api.setTextContent(el, vnode.text)
    }else {
        updateEle(el, vnode, oldVnode)
        if (oldCh && ch && oldCh !== ch) {
            updateChildren(el, oldCh, ch)
        }else if (ch){
            createEle(vnode) //create el's children dom
        }else if (oldCh){
            api.removeChildren(el)
        }
    }
}
```
这个函数做了以下事情：  
* 判断Vnode和oldVnode是否相同，如果是，那么直接return；
* 如果他们都有文本节点并且不相等，那么将更新为Vnode的文本节点。
* 如果oldVnode有子节点而Vnode没有，则删除el的子节点
* 如果oldVnode没有子节点而Vnode有，则将Vnode的子节点真实化之后添加到el
* 如果两者都有子节点，则执行updateChildren函数比较子节点，而这个函数也是diff逻辑最多的一步 
 
updateChildren最关键的函数
```js
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
	let oldStartIdx = 0
	let newStartIdx = 0
	let oldEndIdx = oldCh.length - 1
	let oldStartVnode = oldCh[0]
	let oldEndVnode = oldCh[oldEndIdx]
	let newEndIdx = newCh.length - 1
	let newStartVnode = newCh[0]
	let newEndVnode = newCh[newEndIdx]
	let oldKeyToIdx, idxInOld, vnodeToMove, refElm
 
	// removeOnly is a special flag used only by <transition-group>
	// to ensure removed elements stay in correct relative positions
	// during leaving transitions
	const canMove = !removeOnly
 
	if (process.env.NODE_ENV !== 'production') {
		checkDuplicateKeys(newCh)
	}
 
	while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
	    //isUndef 是判断值是否等于undefined或者null
		if (isUndef(oldStartVnode)) {
			oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
		} else if (isUndef(oldEndVnode)) {
			oldEndVnode = oldCh[--oldEndIdx]
		} else if (sameVnode(oldStartVnode, newStartVnode)) {
			patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
			oldStartVnode = oldCh[++oldStartIdx]
			newStartVnode = newCh[++newStartIdx]
		} else if (sameVnode(oldEndVnode, newEndVnode)) {
			patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
			oldEndVnode = oldCh[--oldEndIdx]
			newEndVnode = newCh[--newEndIdx]
		} else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
			patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
			canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
			oldStartVnode = oldCh[++oldStartIdx]
			newEndVnode = newCh[--newEndIdx]
		} else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
			patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
			canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
			oldEndVnode = oldCh[--oldEndIdx]
			newStartVnode = newCh[++newStartIdx]
		} else {
		    //看节点是否有key 没有则新建一个 有的话可以复用
			if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
			idxInOld = isDef(newStartVnode.key)
				? oldKeyToIdx[newStartVnode.key]
				: findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
			if (isUndef(idxInOld)) { // New element
				createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
			} else {
				vnodeToMove = oldCh[idxInOld]
				if (sameVnode(vnodeToMove, newStartVnode)) {
					patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
					oldCh[idxInOld] = undefined
					canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
				} else {
					// same key but different element. treat as new element
					createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
				}
			}
			newStartVnode = newCh[++newStartIdx]
		}
	}
	if (oldStartIdx > oldEndIdx) {
		refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
		addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
	} else if (newStartIdx > newEndIdx) {
		removeVnodes(oldCh, oldStartIdx, oldEndIdx)
	}
}
```
从头得知，新旧节点分别都2个指针，分别指向各自的头部与尾部。那么接下来就开始操作了。  
1.当新旧节点的头部值得对比，进入patchNode方法，同时各自的头部指针+1；  
2.当新旧节点的尾部值得对比，进入patchNode方法，同时各自的尾部指针-1；  
3.当oldStartVnode，newEndVnode值得对比，说明oldStartVnode已经跑到了后面，那么就将oldStartVnode.el移到oldEndVnode.el的后边。oldStartIdx+1，newEndIdx-1；  
4.当oldEndVnode，newStartVnode值得对比，说明oldEndVnode已经跑到了前面，那么就将oldEndVnode.el移到oldStartVnode.el的前边。oldEndIdx-1，newStartIdx+1；  
5.当以上4种对比都不成立时，通过newStartVnode.key 看是否能在oldVnode中找到，如果没有则新建节点，如果有则对比新旧节点中相同key的Node，newStartIdx+1。  
6.当循环结束时，这时候会有两种情况。  

* oldStartIdx > oldEndIdx，可以认为oldVnode对比完毕，当然也有可能 newVnode也刚好对比完，一样归为此类。此时newStartIdx和newEndIdx之间的vnode是新增的，调用addVnodes，把他们全部插进before的后边。
* newStartIdx > newEndIdx，可以认为newVnode先遍历完，oldVnode还有节点。此时oldStartIdx和oldEndIdx之间的vnode在新的子节点里已经不存在了，调用removeVnodes将它们从dom里删除。  

总结：以上就是diff的解析过程，其实过程并不复杂，只需要理清楚思路，知道它是如何将新旧节点做对比即可。同时给节点加上key，能够有效复用。
