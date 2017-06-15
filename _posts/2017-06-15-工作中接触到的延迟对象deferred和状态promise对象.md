## promise对象项目实际操作：
有这样一个需求，保存稿件时，验证标题是否重复，分别有两个接口，如果都不重复，那么调用保存函数saveDraftHandler(),如果至少有一个重复，调用dupHandleInPage()函数

### 第一步： 保存的handler里这样写：

`validateWaitTitle(data).then(function() {
        	resetForm();//重置表单
                saveDraftHandler(afterLoc);//resolve()操作，title都不重复
 }, dupHandleInPage) //title至少有一个接口重复`

### 第二步：

function validateWaitTitle(data){
	var dtd = $.Deferred();
	validateWithWaitTitle(data).then(function(status){
		dtd.resolve(status);
	},function(status){
		dtd.reject(status);
	});
	return dtd.promise();
}

### 第三步：

function validateWithWaitTitle(data){
	var dtd = $.Deferred(),
		waitTitleUrl = ctx + "/content/validateWaitTitle.htm",
		titleUrl = ctx + "/content/validateTitle.htm";
	$.when($.ajax({
		type : 'POST',
		url : waitTitleUrl,
		data: data,
	}),$.ajax({
		type : 'POST',
		url : titleUrl,
		data: data,
	})).then(function(res1,res2){		
		if(typeof res1[0]==='string'&&res1[0]=='false'&&typeof res2[0]==='string'&&res2[0]=='false'){  //全返回false，说明都不重复
			dtd.resolve();			
		}else if(res1[0]=='true'){//res1[0]返回true说明validateWaitTitle编发稿标题重复
			console.log("res1 error");
			dtd.reject("withTitle");		
		}else if(res2[0]=='true'){//res2[0]返回true说明validateTitle已发稿标题重复
			console.log("res2 error");
			dtd.reject("title");		
		}						    
	},errHandle);
	return dtd.promise()
}

代码说明：

$.when($.ajax("/page1.php"), $.ajax("/page2.php"))
.then(function(res1,res2){
   //第一和第二个ajax请求都是成功的
返回值res1和res2都是数组，数据结构是这样的，[false,"success",{}]
根据res1 和res2的返回值，指定延迟对象的结果，dtd.resolve() //表示调用done的回调
dtd.reject("title");//表示调用reject的回调并且传参数
},errHandle);//任一个ajax请求是错误的
return dtd.promise()  // 返回延迟对象的状态 resolve或者reject

### 第四步： 结果返回到了第二步

function validateWaitTitle(data){
	var dtd = $.Deferred();
	validateWithWaitTitle(data).then(function(status){
		dtd.resolve(status);  //如果延迟对象的promise状态是resolve的,于是，这一步的promise状态也是resolve的，表示不重复
	},function(status){  //如果延迟对象的promise状态是reject的，执行这一步，表示重复
		dtd.reject(status);  //状态带了参数
	});
	return dtd.promise();//返回的状态也是有参数的
}

### 第五步：结果返回到了第一步

validateWaitTitle(data).then(function() {  //延迟对象promise的状态是resolve的，代表不重复，那么。。。此处省略状态参数
        	resetForm();//重置表单
                saveDraftHandler(afterLoc);//resolve()操作，title都不重复
 }, dupHandleInPage) //title至少有一个接口重复 //延迟对象promise的状态是reject的，代表重复，执行dupHandleInPage函数，该函数有参数，参数就是返回的状态参数

### 第六步：

//该函数拿到了返回的状态参数，用以区分是哪一个请求返回的，以便进一步说明是哪个重复接口

//重复验证存在handle

function dupHandleInPage(ret) {

    dupHandle(function() {
        window.location.reload()
    },ret);
}

### 第七步：至此就完成了执行多个ajax请求，根据返回结果进一步操作的功能

function dupHandle(failCb,ret) {
	var tips ='';
	$('#releaseTitle,#transTitle,#videoForm').find(".dupTips").remove();
	if(ret&&ret=='withTitle'){
		tips = '已存在相同标题编发稿，请修改标题';
	}else{
		tips = '已经存在相同文章，请修改标题';
	}	   
    var tipsHtml = '<div class="dupTips" style="line-height:4;">'+tips+'</div>';  
    $('#releaseTitle,#transTitle,#videoForm').prepend(tipsHtml);      
        //bind event
    $('#releaseTitle .closePageBtn,#transTitle .closePageBtn,#videoForm .closePageBtn').on('click', function(e) {
        failCb && failCb()
    })
  }

### 总结：

上述就是Deferred对象返回deferred.promise()的例子，我新建一个deferred对象，把回调函数绑定在这个对象上面，而不是原来的deferred对象上面。这样的好处是，无法改变promise对象的执行状态，要想改变执行状态，只能操作原来的deferred对象。
