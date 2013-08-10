修改说明
===========

本版本是在原avalon的基础之上进行了一些修改

## 初始化处理

在使用avalon时，我遇到一个问题：如何在avalon第一次渲染页面之后执行一些js函数。目前avalon没有这样的回调，
作者告诉我的方案是使用 `setTimeout(callback, 100)` 。但是我感觉这种方式其实无法保证执行的时机是对的。如
果页面处理花费的时间比较长，这样的处理有可能会有问题。于是我查看了avalon的源码，根据我的理解，它是同步
方式来渲染，没有看到有异步的处理。所以，其实可以在 `avalon.scan` 处理的最后进行初始化的工作。因此我修改
了avalon.scan的源码，添加了对初始化的回调。

使用方法如下：

```
avalon.config({$init:function(){
    //初始化处理
});
```

通过向 `avalon.config` 添加 `$init` 的回调函数，由 `avalon.scan` 执行时进行回调。

要注意，这里并没有处理当数据变化时，页面重新渲染后需要进行某些js的回调的支持。只是在第一次初始化时进行了处理。

## ms-xxx 方法的函数调用改进

### 改造原因

现在的avalon的版本在处理 `ms-click` 回调时一般是这样:

```
<div ms-click="check" ms-controller="test">点我
<p>value={{value}}</p>
</div>
<script>
avalon.define('test', function(vm){
    vm.value = '';
    vm.check = function(e){
        vm.value = 'click';
    }
});
</script>
```

可以看到，在 `ms-click` 中并没有直接写 `check()` ，而只是一个标识符。在avalon后续处理时，它会将 `check()` 取出来，
并绑定到对应的元素和事件上。因此，在你定义的函数中，需要添加一个接收事件对象的参数。这种定义是因为后面在调用时，
会以这种方式来传入参数。所以在DOM中的定义和实际定义的参数差异很大。并且，我感觉有一个不方便的就是无法定义其它的
参数，特别是在循环中。比如以下代码：

```
<div ms-controller="test1">
<ul ms-each-item="items">
    <li ms-click="echo">{{item.title}}</li>
</ul>
<p>value={{value}}</p>
</div>
<script>
avalon.define('test1', function(vm){
    vm.items = [{id:1, title:'one'}, {id:2, title:'two'}];
    vm.value = ''
    vm.echo = function(e){
        vm.value = this.$vmodel.item.title;
    }
});
</script>
```

我希望在点击每个 `<li>` 时，调用 `echo` 函数，将对应的值显示在 `<p>` 标签中。从上面的代码我们可以看出，在DOM定义时，
我们无法将循环变量传入echo中，所以只能在 `echo` 中通过 `this.$vmodel.item` 来访问。其中 `this` 为点击时对应的 `<li>`
元素。avalon会在循环时绑定一个局部的$vmodel对象，其中会包含 `item` 这个循环变量。

但是上面的代码并不是太好，原因有二：

1. 在DOM中，函数与循环变量不是直接关联，只能在函数处理时根据逻辑来hard code，这样使得这个函数无法通用，即无法用在其
   它的循环变量中。
2. 函数本身因为使用了 `this.$vmodel` 所以无法独立于模板调用来使用，也就是不通用，与环境绑得过死。

### 改造后的使用

为了解决上面的问题，我修改了avalon源码，添加了对函数调用方式的支持，这样上面的代码可以写为:

```
<div ms-controller="test1">
<ul ms-each-item="items">
    <li ms-click="echo(item.title)">{{item.title}}</li>
</ul>
<p>value={{value}}</p>
</div>
<script>
avalon.define('test1', function(vm){
    vm.items = [{id:1, title:'one'}, {id:2, title:'two'}];
    vm.value = ''
    vm.echo = function(title){
        vm.value = title;
    }
});
</script>
```

这里有两点变化：

1. `echo` 变为 `echo(item.title)`
2. `vm.echo = function(e)` 变为 `vm.echo = function(title)`

因此，我们可以在DOM中绑定参数，同时这个参数可以是循环变量。并且，我们的函数变得更通用。函数的处理也更简单。

目前改造版本在函数无参数时，也可以不带括号。

### 如何处理事件及this

因为这个改造只是针对事件回调，所以事件回调中很重要的就是要处理 `this` 和事件。对于 `this` 可以直接使用，它和
一般的回调函数一样，this表示发出事件的DOM元素。

对于事件，我们需要在调用及定义的地方显示处理，如：

```
<div ms-click="check($event)" ms-controller="test">点我
<p>value={{value}}</p>
</div>
<script>
avalon.define('test', function(vm){
    vm.value = '';
    vm.check = function(e){
        vm.value = this + e;
    }
});
</script>
```

在调用时，我们需要使用 `$event` 作为参数传入 `check` 中。在运行时，会直接替換为真正的事件对象。

我们还可以同时定义事件和其它的参数，如：

```
<div ms-click="check('ok', $event)" ms-controller="test">点我
<p>value={{value}}</p>
</div>
<script>
avalon.define('test', function(vm){
    vm.value = '';
    vm.check = function(msg, e){
        vm.value = msg + e;
    }
});
</script>
```

