---
title: 巧用jQuery选择器写表单办法总结（提高效率）
date: 2016-12-31
---

> 相信很多小伙伴都会遇到需要做表单的需求，就像我现在做的医院项目，茫茫多的表单无穷无尽。一开始做表单使用最笨的方法：一个个span去定义ID，每个数据都用ajax获取后赋值显示，然后在表单提交前一个个用jQuery根据ID获取元素的value，组成一个model来提交给服务器。
后来发现使用jQuery可以大大简化这个过程。于是我修改了工作方法，总结了我的一些提高写表单效率的方法。

## 需求
表单中存在最多的无非就是文本、文本框、单选框、多选框。而一些表单中会有几十个甚至几百个选项。我们的目标就是以最高的效率来完成这些表单的数据获取和数据提交。
**注意：如果元素ID和服务器上获取的json字段的名字是一样的，那么这篇文章或许能对你提高工作效率有所帮助。**

## 文本和文本框
对于文本和文本框，我们通常需要为元素添加ID来绑定和获取数据。

### 将数据显示到界面中
* 用for循环遍历解析成功的JSON数据
* 通过if判断过滤数据是span的还是input的。
* 将数据传给和数据名同名的ID元素。
```
for (var key in json) {
   //过滤type为text的文本框
   if ($('#' + key).attr('type') == 'text') {
       $('#' + key).val(json[key]);
   }
   if($('#' + key).prop('tagName') == 'SPAN'){
       $('#' + key).text(json[key]);
   }
}
```
### 快速获取数据对象用于提交服务器
* 定义空model对象。
* 通过jQuery选择器获取目标元素的value。
* 将数据传入model中，对象元素的字段就是HTML元素的ID。
```
var model = {};
$('input[type="text"]').each(function () {
   model[$(this).attr('id')]=$(this).val();
});
$('span').each(function () {
   model[$(this).attr('id')]=$(this).text();
});
console.log(model);
```
按照如下方法，我们只需要照着后端提供的数据字段给HTML定义id，而JS就不需要写太多代码就可以完成表单了。再也不怕表单字段太多了。

### 全部代码
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="jquery-2.2.3.js"></script>
</head>
<body>
    <div>
        <div>
            <label>姓名：<input type="text" id="name"></label>
            <label>性别：<input type="text" id="sex"></label>
            <label>年龄：<input type="text" id="age"></label>
            <label>时间：<input type="text" id="time"></label>
        </div>
        <div>
            <label>a：<span id="param01">1</span></label>
            <label>b：<span id="param02">2</span></label>
            <label>c：<span id="param03">3</span></label>
            <label>d：<span id="param04">4</span></label>
        </div>
    </div>
    <button onclick="showResult()">显示结果</button>
    <script>
        //多条input或者span的快速赋值
        //data是模拟服务器返回的JSON数据
        var data = '{"name":"张三","sex":"女","age":22,"time":"2016-5-10","param01":111,"param02":222,"param03":333,"param04":444}';
        //将数据显示到页面中
        var json = eval('(' + data + ')');
        for (var key in json) {
            if ($('#' + key).attr('type') == 'text') {
                $('#' + key).val(json[key]);
            }
            if($('#' + key).prop('tagName') == 'SPAN'){
                $('#' + key).text(json[key]);
            }
        }

		//获取文本和文本框的内容转为JSON对象
        function showResult() {
            var model = {};
            $('input[type="text"]').each(function () {
                model[$(this).attr('id')]=$(this).val();
            });
            $('span').each(function () {
                model[$(this).attr('id')]=$(this).text();
            });
            console.log(model);
        }
    </script>
</body>
</html>
```

## Radio和Checkbox

另外一种经常出现于表单中的就是radio、checkbox这些单选多选的元素了。这些元素通常用name命名一组选项。下面我同样用jQuery简化数据的显示和提交。

### 显示数据
和之前的文本一样，用for循环遍历json数据，然后通过if过滤后显示到界面。不同之处是这边是通过name来显示和绑定数据的。
**注：对radio和checkbox处理的方法写在了后面，所以复制粘贴的同学们请注意别漏了~**
```
for(var key in json){
   if ($('input[name=' + key +  ']').attr('type') == 'radio') {
      showRadioValue(key, json[key]);
   }
   if ($('input[name=' + key +  ']').attr('type') == 'checkbox') {
     showCheckBoxValue(key, json[key]);
   }
}
```
### 获取数据model的方法
* 定义空model对象。
* 定义name避免重复添加。
* 遍历所有radio获取结果传给model
* 遍历所有checkbox获取结果传给model

```
        function showResult() {
            var model = {};
            var radioName = '';
            var checkboxName = '';
            $("input[type='radio']").each(function () {
                if($(this).attr('name') != radioName){
                    radioName = $(this).attr('name');
                    model[radioName] = getRadioValue(radioName);
                }
            });
            $("input[type='checkbox']").each(function () {
                if($(this).attr('name') != checkboxName){
                    checkboxName = $(this).attr('name');
                    model[checkboxName] = getCheckboxValue(checkboxName);
                }
            });
            console.log(model);
        }
```
### 处理radio和checkbox的一些方法
这里我对radio和checkbox的显示和获取结果写了几个方法。
```
        function showRadioValue(name, value) {
            $('input[name=' + name +  ']').each(function () {
                if(value == $(this).val()){
                    $(this).attr('checked', 'true');
                }
            });
        }

        function getRadioValue(name) {
            var value = 0;
            var i = 0;
            $('input[name=' + name + ']' ).each(function () {
                if ($('input[name=' + name + ']').eq(i).is( ':checked')) {
                    value = $('input[name=' + name + ']').eq(i).val();
                    return;
                }
                i++;
            });
            return value;
        }

        function showCheckBoxValue (name, value) {
            var values = value.split(',' );
            var row = 1;
            $('input[name="' + name + '"]').each( function () {
                if (values[row] == 1) {
                    $(this).attr("checked" , 'true');
                }
                row++;
            });
        }

        function getCheckboxValue (name) {
            var text = "" ;
            $('input[name="' + name + '"]').each( function () {
                var t = '' ;
                if ($(this ).is(':checked')) {
                    t = "1";
                } else {
                    t = "0";
                }
                text += "," + t;
            });
            return text;
        }
```

### 代码
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="jquery-2.2.3.js"></script>
</head>
<body>
    <div>
        <div>
            <label><input type="radio" name="param01" value="1">1</label>
            <label><input type="radio" name="param01" value="2">2</label>
            <label><input type="radio" name="param01" value="3">3</label>
        </div>
        <div>
            <label><input type="radio" name="param02" value="1">1</label>
            <label><input type="radio" name="param02" value="2">2</label>
            <label><input type="radio" name="param02" value="3">3</label>
        </div>
        <div>
            <label><input type="radio" name="param03" value="1">1</label>
            <label><input type="radio" name="param03" value="2">2</label>
            <label><input type="radio" name="param03" value="3">3</label>
        </div>
        <div>
            <label><input type="checkbox" name="param04">1</label>
            <label><input type="checkbox" name="param04">2</label>
            <label><input type="checkbox" name="param04">3</label>
            <label><input type="checkbox" name="param04">3</label>
        </div>
        <div>
            <label><input type="checkbox" name="param05">1</label>
            <label><input type="checkbox" name="param05">2</label>
            <label><input type="checkbox" name="param05">3</label>
            <label><input type="checkbox" name="param05">3</label>
        </div>
        <button onclick="showResult()">显示结果</button>
        <label id="result">result</label>
    </div>
    <script>
        //多条radio或者checkbox的快速赋值
        var data = '{"param01":"1","param02":"3","param03":"2","param04":",1,0,0,0","param05":",0,0,1,1"}';
        var json =eval( '(' + data + ')');
        for(var key in json){
            if ($('input[name=' + key +  ']').attr('type') == 'radio') {
                showRadioValue(key, json[key]);
            }
            if ($('input[name=' + key +  ']').attr('type') == 'checkbox') {
                showCheckBoxValue(key, json[key]);
            }
        }

        function showRadioValue(name, value) {
            $('input[name=' + name +  ']').each(function () {
                if(value == $(this).val()){
                    $(this).attr('checked', 'true');
                }
            });
        }

        function getRadioValue(name) {
            var value = 0;
            var i = 0;
            $('input[name=' + name + ']' ).each(function () {
                if ($('input[name=' + name + ']').eq(i).is( ':checked')) {
                    value = $('input[name=' + name + ']').eq(i).val();
                    return;
                }
                i++;
            });
            return value;
        }

        function showCheckBoxValue (name, value) {
            var values = value.split(',' );
            var row = 1;
            $('input[name="' + name + '"]').each( function () {
                if (values[row] == 1) {
                    $(this).attr("checked" , 'true');
                }
                row++;
            });
        }

        function getCheckboxValue (name) {
            var text = "" ;
            $('input[name="' + name + '"]').each( function () {
                var t = '' ;
                if ($(this ).is(':checked')) {
                    t = "1";
                } else {
                    t = "0";
                }
                text += "," + t;
            });
            return text;
        }

        function showResult() {
            var model = {};
            var radioName = '';
            var checkboxName = '';
            $("input[type='radio']").each(function () {
                if($(this).attr('name') != radioName){
                    radioName = $(this).attr('name');
                    model[radioName] = getRadioValue(radioName);
                }
            });
            $("input[type='checkbox']").each(function () {
                if($(this).attr('name') != checkboxName){
                    checkboxName = $(this).attr('name');
                    model[checkboxName] = getCheckboxValue(checkboxName);
                }
            });
            console.log(model);
        }
    </script>
</body>
</html>
```
## Tips
* 如果有一些特殊处理的内容（如时间格式文本），可以先遍历后在遍历之后单独进行赋值。
* 以上方法可以用于所有以ID定义的用于显示和获取数据的HTML元素。
* 用好jQuery的选择器可以获取到任何所需的元素、元素集合。
* 在each()方法中`$(this)`表示当前元素。
* 获取所选ID的元素标签：`$('#' + key).prop('tagName') == 'SPAN'`，注意：标签结果`'SPAN'`都是大写的,可以打log验证。
* 不断找规律、总结提炼，找出最好的偷懒方法，尽量避免体力劳动。

