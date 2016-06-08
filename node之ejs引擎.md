node ejs是一个express模版解析引擎，用于解析html模版生成对应的html的解析器。语法和php有些类似，语法比较直观。而express默认的模版引擎jade则语法比较怪异，给人感觉不太习惯，而且需要特别注意tab和空格对齐的使用，一个jade模版页面最好使用统一的一种，免得出现莫名的解析错误。
    ejs的使用可以说比较容易，可能起初接触的时候并不知道该如何下手，但其实在ejs中，'<% %>'中可以直接编写javascript代码，使用javascript的各种原生函数。同时由express中的response.render函数所传过来的变量也可以直接使用。

比如：  
```sh
[javascript] 
<% for(var i=0; i<4; ++i) { %>  
      <a href=#">this is a link!</a>  
<% } %>  
```
将生成为：
```sh
[html] 
<a href=#">this is a link!</a>  
<a href=#">this is a link!</a>  
<a href=#">this is a link!</a>  
<a href=#">this is a link!</a>  
```
再如：
```sh
[javascript] 
<%   
    //articles变量同样可以由后台提供，即res.render('article', {articles: allArticles});  
    var articles = [  { title : 'title name', content : 'this is a test content!'},    
tle : 'title name', content : 'this is a test content!'} ];  
%>  
<% articles.forEach(function(article) { %>  
        <div class="panel">  
            <h3><%= article.title %></h3>  
            <div class="panel-body">  
                <%= article.content %>  
            </div>  
        </div>  
<% }) %>  
```
或是
```sh
[javascript] 
<% for(var i=0; i<articles.length; ++i){ %>  
      <div class="panel">  
          <h3><%= articles[i].title %></h3>  
          <div class="panel-body">  
              <%= articles[i].content %>  
          </div>  
      </div>  
<% } %>  
```
将解析成：  
```sh
[html] 
<div class="panel">  
     <h3>title name</h3>  
     <div class="panel-body">  
             this is a test content!  
     </div>  
</div>  
<div class="panel">  
     <h3>title name</h3>  
     <div class="panel-body">  
         this is a test content!  
     </div>  
</div>  
```
### ejs的特性：
```
    1、缓存功能，能够缓存已经解析好的html模版；
    2、<% code %>用于执行其中javascript代码；
    3、<%= code %>会对code进行html转义；
    4、<%- code %>将不会进行转义；
    5、支持自定义标签，比如'<%'可以使用'{{'，'%>'用'}}'代替；
    6、提供一些辅助函数，用于模版中使用
    7、利用<%- include filename %>加载其他页面模版；
    
    使用示例
    1、ejs.compile(str, options); 将返回内部解析好的Function函数
    2、ejs.render(str, options); 返回经过解析的字符串

    其中options的一些参数为：
    1、cache：是否缓存解析后的模版，需要filename作为key；
    2、filename：模版文件名；
    3、scope：complile后的Function执行所在的上下文环境；
    4、debug：标识是否是debeg状态，debug为true则会输出生成的Function内容；
    5、compileDebug：标识是否是编译debug，为true则会生成解析过程中的跟踪信息，用于调试；
    6、client，标识是否用于浏览器客户端运行，为true则返回解析后的可以单独运行的Function函数；
    7、open，代码开头标记，默认为'<%'；
    8、close，代码结束标记，默认为'%>'；
    9、其他的一些用于解析模版时提供的变量。
    在express中使用时，options参数将由response.render进行传入，其中包含了一些express中的设置，以及用户提供的变量值。

    此外ejs还提供了一些辅助函数，用于代替使用javascript代码，使得更加方便的操纵数据。
    1、first，返回数组的第一个元素；
    2、last，返回数组的最后一个元素；
    3、capitalize，返回首字母大写的字符串；
    4、downcase，返回字符串的小写；
    5、upcase，返回字符串的大写；
    6、sort，排序（Object.create(obj).sort()？）；
    7、sort_by:'prop'，按照指定的prop属性进行升序排序；
    8、size，返回长度，即length属性，不一定非是数组才行；
    9、plus:n，加上n，将转化为Number进行运算；
    10、minus:n，减去n，将转化为Number进行运算；
    11、times:n，乘以n，将转化为Number进行运算；
    12、divided_by:n，除以n，将转化为Number进行运算；
    13、join:'val'，将数组用'val'最为分隔符，进行合并成一个字符串；
    14、truncate:n，截取前n个字符，超过长度时，将返回一个副本
    15、truncate_words:n，取得字符串中的前n个word，word以空格进行分割；
    16、replace:pattern,substitution，字符串替换，substitution不提供将删除匹配的子串；
    17、prepend:val，如果操作数为数组，则进行合并；为字符串则添加val在前面；
    18、append:val，如果操作数为数组，则进行合并；为字符串则添加val在后面；
    19、map:'prop'，返回对象数组中属性为prop的值组成的数组；
    20、reverse，翻转数组或字符串；
    21、get:'prop'，取得属性为'prop'的值；
    22、json，转化为json格式字符串
```
实例展示
```sh
var str = 'this is a test';  
var arr = [{name:'c'}, {name:'b'}, {name:'a'}];  
<%: str | truncate_words:2 %>  => 'this is'  
<%: arr | map:'name' | sort | join %> => 'a,b,c'  
<%: arr | prepend:1 %> => [1, {name:'c'}, {name:'b'}, {name:'a'}]; 
```
