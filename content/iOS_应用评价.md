- [应用评价](#%E5%BA%94%E7%94%A8%E8%AF%84%E4%BB%B7)
  - [summary](#summary)
  - [关于](#%E5%85%B3%E4%BA%8E)
  - [实现](#%E5%AE%9E%E7%8E%B0)
    - [iOS 10.3之前实现应用内评价的方法](#ios-103%E4%B9%8B%E5%89%8D%E5%AE%9E%E7%8E%B0%E5%BA%94%E7%94%A8%E5%86%85%E8%AF%84%E4%BB%B7%E7%9A%84%E6%96%B9%E6%B3%95)
    - [iOS 10.3之后实现应用内评价的方法](#ios-103%E4%B9%8B%E5%90%8E%E5%AE%9E%E7%8E%B0%E5%BA%94%E7%94%A8%E5%86%85%E8%AF%84%E4%BB%B7%E7%9A%84%E6%96%B9%E6%B3%95)
  - [我的方案](#%E6%88%91%E7%9A%84%E6%96%B9%E6%A1%88)
  - [总结](#%E6%80%BB%E7%BB%93)
  - [参考文章](#%E5%8F%82%E8%80%83%E6%96%87%E7%AB%A0)
  - [Demo](#demo)

## 应用评价
### summary
```
    if (!产品经理遵守review GuideLines) {
        //Do whatever he want
    }else {
        if (iOSVersion >= 10.3) {
            //使用SKStoreReviewController的requestReview方法
        }else {
            //1.怎么展示:openUrl SKStoreProductViewController
            //2.什么时候展示: iRate,服务器控制
        }
    }
```
### 关于
1. 每个应用在App store上面都会有用户的评价和评分  
2. 评价和评分越高的应用将会被优先展示,排名提高.所以才有那么多人刷榜,尤其是游戏,但是从iOS11开始的App store将只会展示每个分类下前三个应用  
3. 在2017.6苹果更新了Review GuideLines,强制要求开发者使用iOS 10.3引入的api:`SKStoreReviewController的requestReview方法`, 也就是说iOS 10.3之后的版本弹出的框应用使用这个api,一旦发现会被拒绝,**可是很明显审核的时候是很难发现的**.

> 1.1.7 App Store Reviews:  
  App Store customer reviews can be an integral part of the app experience, so you should treat customers with respect when responding to their comments. Keep your responses targeted to the user’s comments and do not include personal information, spam, or marketing in your response.
Use the provided API to prompt users to review your app; this functionality allows customers to provide an App Store rating and review without the inconvenience of leaving your app, and we will disallow custom review prompts.

### 实现
#### iOS 10.3之前实现应用内评价的方法
怎么展示:  
1. 现在应用最多的做法:弹出一个UIAlertView,三个button,但是其中两个都是会跳到app store的  

```
	UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"程序猿奋力出新版" message:@"打滚儿求好评" delegate:self cancelButtonTitle:@"残忍拒绝" otherButtonTitles:@"好评(跳App store)",@"吐槽(跳App store)",nil];
	[alertView show];

//在UIAlertViewDelegate中
//支持直接跳转到AppStore的评论编辑页:通过在项目 URL 查询参数的最后加上 action=write-review 就可以跳转到 AppStore 并自动模态打开评论编辑页面。在此之前我们只能跳转到评论页。
	NSURL *url = [NSURL URLWithString:@"http://itunes.apple.com/WebObjects/MZStore.woa/wa/viewContentsUserReviews?id=1014844521&pageNumber=0&sortOrdering=2&type=Purple+Software&mt=8&action=write-review"];
	if([[UIApplication sharedApplication] canOpenURL:url]){
        [[UIApplication sharedApplication] openURL:url];
    }
```
2.在页面中Modal出一个SKStoreProductViewController,用户在应用即可完成评分和评论  

```
#import <StoreKit/StoreKit.h>

    SKStoreProductViewController *storeProductVC = [[SKStoreProductViewController alloc]init];
    storeProductVC.delegate = self;
    
    [storeProductVC loadProductWithParameters:@{SKStoreProductParameterITunesItemIdentifier:@"1014844521"} completionBlock:^(BOOL result, NSError * _Nullable error) {
        
        if (error) {
            NSLog(@"%@",[error localizedDescription]);
        } else {
            NSLog(@"加载完成");
            [self presentViewController:storeProductVC animated:YES completion:^{
                NSLog(@"界面弹出完成");
            }];
        }
    }];
    
    //modal出来的页面点击"取消"的回调
	- (void)productViewControllerDidFinish:(SKStoreProductViewController *)viewController{
    [self dismissViewControllerAnimated:YES completion:nil];
}

```

什么时候展示:  
1.[iRate](https://github.com/nicklockwood/iRate):提供了丰富的自定义弹出规则,之前我就是使用这个来做

2.服务器来控制弹出的时机

#### iOS 10.3之后实现应用内评价的方法
[官方文档看这里](https://developer.apple.com/app-store/ratings-and-reviews/)  

```
[SKStoreReviewController requestReview];
```
![SamuelChan/20170619174141.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20170619174141.png?imageView2/2/w/480/h/360/q/99|imageslim)

Feature:  
1. 只有评分,没有评论  
2. 弹出完全由苹果控制,所以不能在手势/按钮点击回调中使用这个api,你并不知道有没有弹出  
3. 每年只会弹出三次  
4. 只会在最新版本中弹出,低版本调用了这个api也不会弹出  
5. 在debug环境下,该框会一直弹出,TestFlight该方法无效
6. 关闭:如下图
![SamuelChan/20170619221921.png](https://samuel-image-hosting.oss-cn-shenzhen.aliyuncs.com/SamuelChan/20170619221921.png?imageView2/2/w/360/h/200/q/99|imageslim)

### 我的方案
```
   if ([UIDevice currentDevice].systemVersion.floatValue >= 10.3) {
       [self inAppSKStoreReviewController];   
    }else{
		//自定义弹出规则比如iRate,
		[[iRate shareInstance] logEvent:NO]
    }
```
### 总结
1. 我从来都没有被弹出框引导去评价,都是直接关掉的;可以有了10.3的api又有多少产品经理会遵守呢...
2. 这篇文章make no sense,只是用来练手的 哈哈

### 参考文章
[具透丨iOS 10.3 新 App Store 评价机制详解](https://sspai.com/post/38673)  
[App Store now requires developers to use official API to request app ratings, disallows custom prompts](https://9to5mac.com/2017/06/09/app-rating-custom-prompts-app-store-banned/)

### Demo:
欢迎star:[Demo](https://github.com/SenorSamuel/blog/tree/master/Blog相关Demo/应用评价)




