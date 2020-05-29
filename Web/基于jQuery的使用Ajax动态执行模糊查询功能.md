# **使用Ajax动态执行模糊查询功能**

* 目前脚本已经被我实现了封装，最新的使用方式可以访问 https://carolcoral.github.io/Lcript/ 查看

## 说明：

1.搜索模块仅仅使用了boostrap的样式以及Jquery.js文件

2.因为我使用的layui的弹出层里面做的搜索ifram，所以确定和取消按钮的关闭当前页面的功能都是layui的方式，如果不是ifram的窗口仅仅在当前窗口执行的情况下，可以使用下面的语句来进行关闭当前页面的操作：
        
        window.opener=null;
        window.open('','_self');
        window.close();

## 效果展示：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-rdLJU3oV-1590716385729)(https://raw.githubusercontent.com/carolcoral/SaveImg/master/fuzzysearch.gif)]

## 引用三方功能模块：
```js
<!--jquery-->
<script src="assets/scripts/jquery.js" type="application/javascript"></script>
<link rel="stylesheet" href="assets/vendor/bootstrap/css/bootstrap.min.css">
<!--layui-->
<link rel="stylesheet" href="assets/vendor/layui/css/layui.css">
<script type="application/javascript" src="assets/vendor/layui/layui.js"></script>
```

## 样式：
```css
<style>
/*html {
  -ms-overflow-style:none;
  overflow:-moz-scrollbars-none;
  }
  html::-webkit-scrollbar{width:0px}*/
  #select_template_group {
  position: fixed;
  float: left;
  display: none;
}

.selected_keywords {
	display: none;
	position: fixed;
	width: 590px;
	max-height: 300px;
	margin-top: 35px;
	z-index: 10002;
}

.selected_keywords > ul {
	max-height: 300px;
	min-width: 590px;
	display: inline-block;
	overflow-y: scroll;
	overflow-x: hidden;
}

.list-group-item:hover {
	color: #2E2D3C;
	background: #00AAFF;
}

#basic-addon1 > img {
	width: 80%;
	height: 80%;
}
</style>
```
        
## HTML代码：
```js
<html>
 <head></head>
 <body> 
  <!-- WRAPPER --> 
  <div id="wrapper"> 
   <!-- NAVBAR --> 
   <div id="nav"></div> 
   <!-- END NAVBAR --> 
   <!-- END LEFT SIDEBAR --> 
   <!-- MAIN --> 
   <div class="main"> 
    <!-- MAIN CONTENT --> 
    <div class="main-content"> 
     <div class="container-fluid" style="width: 700px;"> 
      <div class="form-horizontal"> 
       <!--查询位置--> 
       <div class="input-group" id=""> 
        <span class="input-group-addon" id="sizing-addon2">选择模版</span> 
        <input type="text" class="form-control" placeholder="选择模版，支持模糊查询" aria-describedby="sizing-addon2" id="fuzzy_search" oninput="template_choise(this)" /> 
        <!--搜索结果显示--> 
        <div class="input-group selected_keywords"></div> 
       </div> 
       <!--展示位置--> 
       <div class="panel panel-primary"> 
        <div class="panel-heading"> 
         <h4 class="panel-title">已选择模版</h4> 
        </div> 
        <div class="panel-body" id="results_2"></div> 
       </div> 
       <!--功能按钮--> 
       <div class="form-group" style="float: left;margin-left: 25%"> 
        <div class="col-sm-offset-2 col-sm-10"> 
         <button type="submit" class="btn btn-default" onclick="tsg_confirm()">确认</button> 
        </div> 
       </div> 
       <div class="form-group" style="float: left;margin-left: 30px"> 
        <div class="col-sm-offset-2 col-sm-10"> 
         <button type="submit" class="btn btn-default" onclick="tsg_cancle()">取消</button> 
        </div> 
       </div> 
      </div> 
     </div> 
    </div> 
    <!-- END MAIN CONTENT --> 
   </div> 
   <!-- END MAIN --> 
  </div> 
  <!-- END WRAPPER -->  
 </body>
</html>
```

## Javascript代码：
```js
< script >
//点击空白区域隐藏搜索结果内容
$(document).click(function(e) {
    var _con = $('.selected_keywords'); // 设置目标区域
    if (!_con.is(e.target) && _con.has(e.target).length === 0) { // Mark 1
        // todo
        $(".selected_keywords").css("display", "none")
    }
});

// 搜索选择功能
function template_choise(obj) {
    // console.log($(obj).val().length)
    // 判断输入框内容为空的时候
    if ($(obj).val().length == 0) {
        $(".selected_keywords").css("display", "none") var selected_keywords = $(".selected_keywords") selected_keywords.empty()
    } else {
        //获取setinterval的索引
        var index = window.setInterval(function() {
            // 获取输入框中的搜索关键字
            var serach_keywords = $(obj).val() var list_template = "<ul class='list-group'></ul>"
            //执行模糊查询功能，延迟200ms
            $.ajax({
                type: "POST",
                url: "/admin/tsg_fuzzy_search",
                data: {
                    "keywords": serach_keywords
                },
                dataType: "json",
                success: function(data) {
                    $(".selected_keywords").css("display", "block") $(".selected_keywords").html(list_template) $(".list-group").empty() var code = data["code"]
                    var data = data["data"]

                    //判断是否存在搜索结果
                    if (code == "000000") {
                        $.each(data,
                        function(i, data) {
                            var data_complte = data["template_rule_name"] + "," + data["template_name"] + "," + data["template_desc"]
                            var html_style = "<a href='javascript:'><li class='list-group-item' onclick='choise_one_template(this)'>" + data_complte + "</li></a>"$(".list-group").append(html_style)
                        })
                    } else if (code == "200000") {
                        $(".list-group").append("<li class='list-group-item'>没有找到合适的结果</li>")
                    }

                    //清除当前interval
                    window.clearInterval(index)
                },
                fail: function() {
                    alert("查询失败")
                }

            })
            //    延时200ms
        },
        200)
    }
}

//添加选择的模版到展示栏
function choise_one_template(obj) {
    //获取选择的值
    var template_data = $(obj).text()
    //拆分获取的数据
    var selected_template = template_data.split(",")
    //获取指定的数据
    selected_template = selected_template[1]
    // console.log(selected_template)
    //制作html
    var selected_template_html = "<button type='button' class='btn btn-default template_data' aria-label='Left Align'><span class='glyphicon glyphicon-remove' aria-hidden='true' onclick='delete_template(this)'></span>  " + selected_template + "</button>"

    //在指定div内插入html
    $("#results_2").append(selected_template_html)
}

//删除当前已选择的template
function delete_template(obj) {
    //获取选择的值数据
    var template_data_selected = $(obj).text() console.log(template_data_selected)
    //获取当前删除的html内容
    var father_button_html = $(obj).closest("button")
    //从指定容器中删除该内容
    father_button_html.remove()
}

//关闭当前ifram
function tsg_cancle() {
    layui.use('layer',
    function() {
        var layer = layui.layer layer.confirm('您确定要关闭本页吗?', {
            icon: 3,
            title: '您确定要关闭本页吗？'
        },
        function(index) {
            //清除已选择的模版信息
            // localStorage.removeItem("selected_template_data")
            //清除已排序的模板信息
            localStorage.removeItem("template_data_sorted") var layer = layui.layer;
            var index = parent.layer.getFrameIndex(window.name); //先得到当前iframe层的索引
            parent.layer.close(index); //再执行关闭
            layer.close(index);
        })
    })
}

//确认按钮=提交
function tsg_confirm() {
    var selected_template_datas = [] $(".template_data").each(function(i, element) {
        selected_template_datas[i] = $(element).text()
    }) localStorage.setItem("template_data_sorted", selected_template_datas) layui.use('layer',
      function() {
          var layer = layui.layer
          var layer = layui.layer;
          var index = parent.layer.getFrameIndex(window.name); //先得到当前iframe层的索引
          parent.layer.close(index); //再执行关闭
          layer.close(index);
      }
    )
    //刷新父页面
    parent.location.reload()
}
< /script>
```
