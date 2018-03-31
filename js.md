## AJAX

### 如何在ajax权限判断后跳转？

经常会遇到一种场景，直接访问某些权限被拒绝后跳转登陆页面，然而ajax不会跳转
这个时候使用全局的：

```
  $(function(){
//全局的ajax访问，处理ajax清求时sesion超时
$.ajaxSetup({ 
    complete:function(XMLHttpRequest,textStatus){ 
    var sessionstatus=XMLHttpRequest.getResponseHeader("sessionstatus"); //通过XMLHttpRequest取得响应头，sessionstatus，
    if(sessionstatus=="timeout"){ 
        //如果超时就处理 ，指定要跳转的页面
        window.location.replace(urlconfig.url.ctx+"/login.jsp"); 
    } 
} 
})
```

})  

在拦截器里面：

```js
        if (httpRequest.getHeader("x-requested-with") != null
            && httpRequest.getHeader("x-requested-with").equalsIgnoreCase("XMLHttpRequest"))// 如果是ajax请求响应头会有，x-requested-with；
        {
            httpResponse.setHeader("sessionstatus", "timeout");// 在响应头设置session状态
            httpResponse.setStatus(403);
            return false;
        } else {
            httpResponse.sendRedirect(httpResponse.encodeRedirectURL("/login.jsp"));
        }
```