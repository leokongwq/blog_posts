---
layout: post
comments: true
title: jquery选择器详解
date: 2016-10-17 10:58:40
tags:
- jquery
categories:
- javascript
---

###  id选择器（指定id元素）                        

将id="one"的元素背景色设置为黑色。（id选择器返单个元素）

    $(document).ready(function () {
        $('#one').css('background', '#000');
    });

### class选择器（遍历css类元素）

将class="cube"的元素背景色设为黑色

    $(document).ready(function () {
        $('.cube').css('background', '#000');
    });

<!-- more -->

### element选择器（遍历html元素）
将p元素的文字大小设置为12px

    $(document).ready(function () {
        $('p').css('font-size', '12px');
    });

### * 选择器（遍历所有元素）

    $(document).ready(function () {
        // 遍历form下的所有元素，将字体颜色设置为红色
        $('form *').css('color', '#FF0000');
    });

### 并列选择器

    $(document).ready(function () {
        // 将p元素和div元素的margin设为0
        $('p, div').css('margin', '0');
    });

## 层次选择器

### parent > child（直系子元素）

    $(document).ready(function () {
        // 选取div下的第一代span元素，将字体颜色设为红色
        $('div > span').css('color', '#FF0000');
    });

### prev + next（下一个兄弟元素，等同于next()方法）

    $(document).ready(function () {
        // 选取class为item的下一个div兄弟元素
        $('.item + div').css('color', '#FF0000');
        // 等价代码
        //$('.item').next('div').css('color', '#FF0000');
    });

###  prev ~ siblings（prev元素的所有兄弟元素，等同于nextAll()方法）

    $(document).ready(function () {
        // 选取class为inside之后的所有div兄弟元素
        $('.inside ~ div').css('color', '#FF0000');
        // 等价代码
        //$('.inside').nextAll('div').css('color', '#FF0000');
    });

## 过滤选择器

### 基本过滤选择器

####  :first和:last（取第一个元素或最后一个元素）

    $(document).ready(function () {
        $('span:first').css('color', '#FF0000');
        $('span:last').css('color', '#FF0000');
    });

#### :not（取非元素）

    $(document).ready(function () {
        $('div:not(.wrap)').css('color', '#FF0000');
    });

####  :even和:odd（取偶数索引或奇数索引元素，索引从0开始，even表示偶数，odd表示奇数）

    $(document).ready(function () {
        $('tr:even').css('background', '#EEE'); // 偶数行颜色
        $('tr:odd').css('background', '#DADADA'); // 奇数行颜色
    });

#### :gt(x)和:lt(x)（取大于x索引或小于x索引的元素）

    $(document).ready(function () {
        $('ul li:gt(2)').css('color', '#FF0000');
        $('ul li:lt(2)').css('color', '#0000FF');
    });

####  :eq(x) （取指定索引的元素）

    $(document).ready(function () {
        $('tr:eq(2)').css('background', '#FF0000');
    });

#### :header（取H1~H6标题元素）

    $(document).ready(function () {
        $(':header').css('background', '#EFEFEF');
    });

### 内容过滤选择器

####  :contains(text)（取包含text文本的元素）

    $(document).ready(function () {
        // dd元素中包含"jQuery"文本的会变色
        $('dd:contains("jQuery")').css('color', '#FF0000');
    });

#### :empty（取不包含子元素或文本为空的元素）

    $(document).ready(function () {
        $('dd:empty').html('没有内容');
    });

#### :has(selector)（取选择器匹配的元素）

    $(document).ready(function () {
        // 为包含span元素的div添加边框
        $('div:has(span)').css('border', '1px solid #000');
    });

#### :parent（取包含子元素或文本的元素）

    $(document).ready(function () {
        $('ol li:parent').css('border', '1px solid #000');
    });

### 可见性过滤选择器

#### :hidden（取不可见的元素）
jQuery至1.3.2之后的:hidden选择器仅匹配display:none或<input type="hidden" />的元素，而不匹配visibility: hidden或opacity:0的元素。这也意味着hidden只匹配那些“隐藏的”并且不占空间的元素，像visibility:hidden或opactity:0的元素占据了空间，会被排除在外。

    <html xmlns="http://www.w3.org/1999/xhtml" >
    <head runat="server">
    <title></title>
    <style type="text/css">
        div
        {
            margin: 10px;
            width: 200px;
            height: 40px;
            border: 1px solid #FF0000;
            display:block;
        }
        .hid-1
        {
            display: none;
        }
        .hid-2
        {
            visibility: hidden;
        }
    </style>
    <script type="text/javascript" src="js/jquery.min.js"></script>
    <script type="text/javascript">
        $(document).ready(function() {
            $('div:hidden').show(500);
            alert($('input:hidden').val());
        });
    </script>
    </head>
    <body>
        <div class="hid-1">display: none</div>
        <div class="hid-2">visibility: hidden</div>
        <input type="hidden" value="hello"/>
    </body>
    </html>

#### :visible（取可见的元素）
 

    <script type="text/javascript">
    $(document).ready(function() {
        $('div:visible').css('background', '#EEADBB');
    });
    </script>
    <div class="hid-1">display: none</div>
    <div class="hid-2">visibility: hidden</div>
    <input type="hidden" value="hello"/>
    <div>
        jQuery选择器大全
    </div>

### 属性过滤选择器

####  [attribute]（取拥有attribute属性的元素）

    <script type="text/javascript">
        $(document).ready(function() {
            $('a[title]').css('text-decoration', 'none');
       });
    </script>  

#### [attribute = value]和[attribute != value]（取attribute属性值等于value或不等于value的元素）

    <script type="text/javascript">
       $(document).ready(function() {
           $('a[class=item]').css('color', '#FF99CC');
           $('a[class!=item]').css('color', '#FF6600');
       });
    </script>  

#### [attribute ^= value], [attribute $= value]和[attribute *= value]（attribute属性值以value开始，以value结束，或包含value值）

    <script type="text/javascript">
        // 识别大小写，输入字符串时可以输入引号，[title^=jQuery]和[title^="jQuery"]是一样的
        $('a[title^=jQuery]').css('font-weight', 'bold');
        $('a[title$=jQuery]').css('font-size', '24px');
        $('a[title*=jQuery]').css('text-decoration', 'line-through');
    </script>

#### [selector1][selector2]（复合型属性过滤器，同时满足多个条件）

    <script type="text/javascript">
        $(document).ready(function() {
            $('a[title^=jQuery][class=item]').hide();
        });
    </script>  

### 子元素过滤选择器

#### :first-child和:last-child

:first-child表示第一个子元素，:last-child表示最后一个子元素。

需要大家注意的是，:fisrst和:last返回的都是单个元素，而:first-child和:last-child返回的都是集合元素。举个例子：div:first返回的是整个DOM文档中第一个div元素，而div:first-child是返回所有div元素下的第一个元素合并后的集合。
这里有个问题：如果一个元素没有子元素，:first-child和:last-child会返回null吗？请看下面的代码：

    <html xmlns="http://www.w3.org/1999/xhtml" >
    <head runat="server">
    <title></title>
    <script type="text/javascript" src="js/jquery.min.js"></script>
    <script type="text/javascript">
    $(document).ready(function() {
        var len1 = $('div:first-child').length;
        var len2 = $('div:last-child').length;
     });
    </script>
    </head>
    <body>
    <div>
        <div>
            <div></div>
        </div>
    </div>
    </body>
    </html>

也许你觉得这个答案，是不是太简单了？len1 = 2, len2 = 2。但实际确并不是，它们俩都等于3。 
把上面的代码稍微修改一下：                    

    <html xmlns="http://www.w3.org/1999/xhtml" >
    <head runat="server">
    <title></title>
    <script type="text/javascript" src="js/jquery.min.js"></script>
    <script type="text/javascript">
    $(document).ready(function() {
        var len1 = $('div:first-child').length;
        var len2 = $('div:last-child').length;
        $('div:first-child').each(function() {
            alert($(this).html());
        });
     });
    </script>
    </head>
    <body>
    <div>123
        <div>456
            <div></div>
        </div>
    </div>
    </body>
    </html>

####  :only-child 

    <html xmlns="http://www.w3.org/1999/xhtml" >
    <head runat="server">
    <title></title>
    <script type="text/javascript" src="js/jquery.min.js"></script>
    <script type="text/javascript">
        $(document).ready(function() {
            $('div:only-child').css('border', '1px solid #FF0000').css('width','200px');
        });
    </script>
    </head>
    <body>
    <div>123
        <div>456
            <div></div>
        </div>
    </div>
    </body>
    </html>

#### 
                    
                    