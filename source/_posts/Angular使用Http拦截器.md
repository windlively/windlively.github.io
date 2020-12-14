---
title: Angular使用Http拦截器
date: 2020-12-15 00:56:01
tags: 前端开发
---
&emsp;&emsp;在前端开发的过程中，一般通过HTTP接口与后端进行数据交互，而后端一般会固定一个返回格式，例如：
```json
{
    "code": 0,
    "success": true,
    "message": "",
    "result":{
        "id":9,
        "title":"",
        "content":"",
        "createTime":"2020-11-25 19:22:31",
        "updateTime":"2020-11-25 19:47:22",
        "available":true
    }
}
```
&emsp;&emsp;前端在拿到数据后，首先判断是否成功，然后取出数据体显示到页面上，但是每一次都这么去做，几乎都是重复的操作，在后端开发的过程中，有诸如过滤器、Spring提供的拦截器等工具实现对HTTP请求的统一处理，在前端其实也有类似的东西，Angular框架就提供了拦截器接口:
```typescript
interface HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>>
}
```
&emsp;&emsp;可以实现统一的HTTP请求处理，帮助我们简化代码，并且方便代码的维护，直接放上一个我实际使用过的示例代码：
```typescript
import {
  HttpErrorResponse,
  HttpEvent,
  HttpHandler,
  HttpInterceptor,
  HttpRequest,
  HttpResponse,
  HttpResponseBase
} from '@angular/common/http';
import {Observable, of, throwError} from 'rxjs';
import {Injectable} from '@angular/core';
import {catchError, debounceTime, finalize, mergeMap, retry} from 'rxjs/operators';
import {AppService} from '../service/app.service';
/**
 * 全局HTTP请求拦截器
 */
@Injectable()
export class AppHttpInterceptor implements HttpInterceptor {
  // 当前正在通信中的HTTP请求数量，此值是为了在http请求的过程中添加加载动画
  public processingHttpCount = 0;
  // 依赖注入
  constructor(public appService: AppService) {
  }
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // /monitor前缀的不做处理
    if (req.url.includes('/monitor')) {
      return next.handle(req);
    }
    //; setTimeout(() => this.appService.showLoadingBar = true);
    // 
    this.processingHttpCount ++;
    return next.handle(req.clone({
      // 为所有拦截的请求添加一个'/starry'前缀
      url: '/starry' + (req.url.startsWith('/') ? req.url : '/' + req.url)
    }))
      .pipe(
        debounceTime(1000),
        // 失败时重试2次
        retry(2),
        mergeMap((event: any) => {
          // 处理后端HTTP接口返回结果
          if (event instanceof HttpResponseBase) {
            // HTTP返回代码正常
            if (event.status >= 200 && event.status < 400) {
              // 处理HTTP Response
              if (event instanceof HttpResponse) {
                const body = event.body;
                // 判断后端的成功标志字段是否为true
                if (body && body.success) {
                  // 取出响应体数据的data部分并继续后续操作(将原有的body替换为了body['result'])
                  return of(new HttpResponse(Object.assign(event, {body: body.result})));
                } else {
                  // 抛出异常
                  throw Error(body.message);
                }
              }
            }
          }
          // 其余事件类型不作拦截处理
          return of(event);
        }), catchError((err: HttpErrorResponse) => {
          // 如果发生5xx异常，显示一个错误信息提示
          this.appService.showSnackBar(err.message, 4000);
          console.error(err.message)
          return throwError(err);
        }), finalize(() => {
          // 最终processingHttpCount减一，并且在减至0的时候移除掉加载动画
          setTimeout(() => --this.processingHttpCount === 0 ?
            this.appService.showLoadingBar = false : this.appService.showLoadingBar = true, 500);
        }));
  }
}
```
&emsp;&emsp;并将此拦截器注册在Angular模块的provider中去，在providers中添加：
```typescript
{
    provide: HTTP_INTERCEPTORS, useClass: AppHttpInterceptor, multi: true
}
```
&emsp;&emsp;这样在调用后端接口的时候我们不用每次再处理后端的返回数据，可直接在返回值中拿到数据体部分去做页面展示，其余的事情就交给拦截器去处理，例如：
```typescript
this.httpClient.get('/config/template/' + template).subscribe(data => this.configTemplates.push({template, data}));
```
&emsp;&emsp;至此，就配置好了一个HTTP拦截器，还可以统一处理请求的URL以及异常处理等等，十分方便。官方文档地址：[https://angular.io/api/common/http/HttpInterceptor](https://angular.io/api/common/http/HttpInterceptor) (中文文档将 **.io** 改为 **.cn** 即可)  
&emsp;&emsp;除此之外，对于页面路由，Angular其实也提供了类似的工具，可以对路由以及页面组件进行拦截控制，有机会再作介绍。