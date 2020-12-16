### 问题起源

- 项目中以前用`UIWebView`,会通过`NSURLProtocol`拦截所有的`http.https`, 给所有发起的请求，添加一些头部信息，后面由于苹果废弃`UIWebView`，需要替换为`WKWebView`，这时产生了一些问题
- `WKWebView`不支持`NSURLProtocol`，后通过`私有API`也可以进行注册
- `WKWebView`在独立于` app 进程`之外的进程中执行网络请求，请求数据不经过主进程，最终会通过IPC把请求`Message`发送给`App`线程，其中在线程间通信的时候会对请求`encode成Message`，出于性能的考虑，`encode`过程会把`HTTPBody和HTTPBodyStream`两个字段丢弃。

### 遇到问题注册拦截之后POST请求Body丢失(多方寻找仅App端修改的方式无果，放弃了)

不注册`NSURLProtocol`的话，`POST`请求我这边是没什么问题的，貌似有的人没注册`NSURLProtocol`，`POST`请求还是凉了，不清楚啥情况。

我这边注册`NSURLProtocol` 就是为了给加载`WebView`或所有`WebView`里的请求都增加一些头部信息

既然选择了不注册`NSURLProtocol`，那么就要想办法给加上头部，第一次加载`webview`的时候，我可以手动加上`header`，但是发现如果网页里还有交互或者`reload`，发起的请求是没有加上`header`的。

### 一种笨方法

在`WKWebView`的代理方法`decidePolicyForNavigationAction`，里拦截请求，如果是需要加头部的请求，就`decisionHandler(WKNavigationActionPolicyCancel); return;` 取消加载，然后加上头部之后重新`loadRequest`

例 ：

```
-(void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler{
    NSURL *url = navigationAction.request.URL;
    LYLog(@"\ndecidePolicy--  %@",url);
   
    NSMutableURLRequest *mutableRequest = [navigationAction.request mutableCopy];
    NSDictionary *dict = mutableRequest.allHTTPHeaderFields;
   
    NSString *urlString0 = url.absoluteString;
    
    if ([urlString0 containsString:@"pangong88.com"]&&![dict objectForKey:@"DEVICE"]) {
        decisionHandler(WKNavigationActionPolicyCancel);
        [LYURLProtocol canInitWithRequest:mutableRequest];
        [self.wkWebView loadRequest:mutableRequest];
        return ;
    }
    decisionHandler(WKNavigationActionPolicyAllow);
}
```

- 这里还需要你自己存一下加载过的链接，应该可能跳转二级页面， 需要支持返回。

### 另一种方式是iOS11之后的

- 通过`WKWebViewConfiguration` 设置

  ```
  if (@available(iOS 11.0, *)) {
              [config setURLSchemeHandler:self forURLScheme:@"https"];
              [config setURLSchemeHandler:self forURLScheme:@"http"];
          }
  ```

  拦截所有的`https, http`,自己处理。

  ```
  - (void)webView:(WKWebView *)webView startURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask API_AVAILABLE(ios(11.0)){
  
  
      NSURLRequest *request = urlSchemeTask.request;
      if ([request.URL.absoluteString containsString:@"pangong88.com"]) {
          [LYURLProtocol canInitWithRequest:request];
      }
      NSURLSession *session = [NSURLSession sharedSession];
  
      NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
  
          [urlSchemeTask didReceiveResponse:response];
  
          [urlSchemeTask didReceiveData:data];
  
          [urlSchemeTask didFinish];
      }];
  
      [task resume];
  
  }
  ```

  需要定义一个`WKWebView`的分类，实现方法

  ```
  +(BOOL)handlesURLScheme:(NSString *)urlScheme{
      return NO;
  }
  ```

  