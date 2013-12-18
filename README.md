widget.js
=========
以一个简单的例子来介绍一下widget.js的使用方法。

我们现在需要实现一个下拉列表，要求鼠标进入操作区时下拉列表，鼠标移出时隐藏下拉。
首先我们先完成大概如下的HTML结构，注意这里的`data-widget`属性是一个url，标识着这个widget需要载入这个url所指的js模块进行初始化，如果这个url中含有`#`，那么会载入这个模块下指定的属性来初始化，这个属性名由`#`后面的字符串指定：
```html
<div data-widget="app/test/test_page.js#hoverWidget" class="hover-widget">
    <span>标题</span>
    <ul>
        <li><a href="#">标题1</a></li>
        <li><a href="#">标题2</a></li>
        <li><a href="#">标题3</a></li>
    </ul>
</div>
```

然后我们实现最简单的css样式, 我们通过一个名为`active`的class来控制下拉列表的显示：
```css
.hover-widget ul{
    display: none;
}
.hover-widget.active ul{
    display: block
}
```
最后，我们新建一个名为`app/test/test_page.js`的js文件，并且导出一个名为`hoverWidget`的接口(与`data-widget`属性相对应)：
```javascript
define('app/ms_v2/test/test_page.js', ['jquery'], function (require, exports) {
    var $ = require('jquery');
 
    exports.hoverWidget = function (config) {
        var $el = $(config.$el);
 
        $el
            .on('mouseenter', function () {
                $el.addClass('active');
            })
            .on('mouseleave', function () {
                $el.removeClass('active');
            });
    }
});
```
至此，开发工作基本结束了，最后一步我们需要在引入这段HTMl的页面里加上一句JS来初始化所有的widget：
```javascript
G.use('app/common/widget/widget.js', function (Widget) {
    Widget.initWidgets();
});
```
完成这一步之后，`app/test/test_page.js`会被自动下载，并且使用这个模块导出的`hoverWidget`函数来进行初始化。

# config参数

在刚才的示例中，我们看到`app/test/test_page.js`模块导出了一个函数，这个函数接受一个config参数，接下来我们还是以一个例子来罗列一下config中的属性分别有哪些，注意下面DOM中`data-` 开头的属性：
```html
<div data-widget="xxxWidget" data-foo="1" data-bar="2" data-pub-sub="3">
    <ul data-role="itemList">
        <li data-role="item">1</li>
        <li data-role="item">2</li>
        <li data-role="item">3</li>
    </ul>
</div>
```
在这样一个widget中，`config`会有以下字段
* **config.$el**
   
   `$el`指的是该widget的根元素，也就是标记着`data-widget`的那个元素。
* **config.$xxx**
   
   `$el`元素的子节点中如果出现`data-role`属性标记的子元素，那么这个子元素将会被自动收集起来作为一个`config`的一个以`$`开头的属性。
   例如上例中的HTMl结构，`config`中会出现一个`config.$itemList`的属性，指向`ul`元素。以及另一个`config.$item`，指向`li`元素集合。
* **config.xxx**
   
   $el元素的上如果出现`data-`开头的属性，那么这些属性会被自动收集作为`config`的属性出现。HTML中的`data-`属性采用连字符格式命名风格，收集到`config`中时会自动转变为小驼峰风格。
   例如上例中的HTML结构，`config`会出现有以下字段:

```javascript
config.foo === 1;
config.bar === 2;
config.pubSub === 3;
```

# Widget.initWidget($el)
该接口用于初始化一个widget，参数为一个元素，该元素用于生成之前介绍的`config`参数，同时从该元素的`data-widget`属性中加载指定的函数对widget进行初始化。

# Widget.initWidgets()
该接口用于初始化页面上所有的widget。实现源码超简单：

```javascript
Widget.initWidgets = function () {
    $('[data-widget]').each(function () {
        Widget.initWidget($(this));
    });
};
```

# Widget.define

在之前的例子中，我们的初始化函数只是一个简单的函数，这个函数接受一个`config`，然后进行初始化。面对简单的`widget`的时候，我们可以这么做，但是如果面对一个`data-role`众多，逻辑复杂的widget时，这个简单的函数就会变得巨大无比。而我们希望所有的函数不要超过80行。因此我们引入`Widget.define`。

同样以一个简单的例子来说明：
```javascript
var Widget = require('widget.js');
 
exports.fooWidget = Widget.define({
    events: {
        "click": "show",
        "click [data-role=close]": "hide",
        "click [data-role=xxx]": function () {
            this.xxx();
        }
    },
    init: function (config) {
        this.config = config;
    },
    show: function () {
        this.config.$el.show();
    },
    hide: function () {
        this.config.$el.hide();
    },
    xxx: function () {
        // whatever
    }
});
```
* events

   这个字段用于定义事件的绑定，通用的格式为:
```
   '事件名 [子元素选择符]': 回调
```
   其中：
   - __事件名__：如`click`, `mouseenter`, `mouseleave`等等。
   - __子元素选择符__：存在子元素选择符时则监听指定子元素的指定事件，如果不存在子元素选择符，则在$el节点上监听指定事件。
   - __回调__：可以是一个函数，也可以是一个字符串，如果是字符串则会被自动转换为同名的成员函数。***所有的回调内的`this`指针指向该widget实例，而不是事件触发的元素。***

* init

   这个函数用于初始化widget，接受一个config参数，该参数与之前简单版本的widget保持一致。函数内的this指向widget实例。

# Widget.ready(selector, callback)
该接口用于获取指定DOM的widget实例，示例：

```javascript
Widget.ready(['#filter', '#map'], function (filterWidget, mapWidget) {
    filterWidget.config.$area
        .on('mouseenter', function (e) {
            mapWidget.showArea($(e.target).data('name'));
        });
});
```
上例中，`Widget.ready`等待两个DOM对应的widget初始化完成后，执行后续的回调，回调中参数的顺序与依赖列表中一致。
