### 以下收集一些很有用小技巧（持续更新中）

1. 打开一些旧项目时，运行后上下出现黑块。  
  原因：缺少大屏对应的LaunchImage，此时`[UIScreen mainScreen].bounds`的结果也是不正确的。  
  解决办法：（不需要添加image）只需添加对相应屏幕尺寸的支持即可。  

2. 创建非全屏大小的窗口
```
CGRect rect = [[UIScreen mainScreen] bounds];
//rect = UIEdgeInsetsInsetRect(rect, UIEdgeInsetsMake(20., 0., 30., 0.));
self.window = [[UIWindow alloc] initWithFrame:rect];
self.window.clipsToBounds = YES;
self.viewController = [ViewController new];
self.window.rootViewController = self.viewController;
[self.window makeKeyAndVisible];
```

3. Xcode升级后插件失效的问题
```
XCODEUUID=`defaults read /Applications/Xcode.app/Contents/Info DVTPlugInCompatibilityUUID`
for f in ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins/*; do defaults write "$f/Contents/Info" DVTPlugInCompatibilityUUIDs -array-add $XCODEUUID; done
```
以下是来源：
http://stackoverflow.com/questions/20732327/xcode-5-required-plug-in-not-present-in-dvtplugincompatibilityuuids

4. 保存照片至系统相册  
```
UIImageWriteToSavedPhotosAlbum(UIImage *image,  id completionTarget, SEL completionSelector,  void *contextInfo);
```
或者
```
ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];
[library writeImageToSavedPhotosAlbum:[image CGImage] orientation:(ALAssetOrientation)[image imageOrientation] completionBlock:^(NSURL *assetURL, NSError *error){
    if (error) {
      // TODO: error handling
    } else {
      // TODO: success handling
    }
}];
```

5. NULL/Nil/nil/NSNull \(http://nshipster.cn/nil/)

标志 |	值 |	含义
---|---:|:---:
NULL |	```(void *)0``` |	C指针的字面零值
nil |	```(id)0``` |	Objective-C对象的字面零值
Nil |	```(Class)0``` |	Objective-C类的字面零值
NSNull |	```[NSNull null]``` |	用来表示零值的单独的对象  

6. UiWebView debug  
http://stackoverflow.com/questions/2767902/what-are-some-methods-to-debug-javascript-inside-of-a-uiwebview

7. Xcode常用快捷键（拾遗）  
^ + ⌘ + Y,  ⌘ + Y, ^ + ⌘ + arrow, ^ + ⌘ + E, ⌘ + ⌥ + J, ⌘ + ⌥ + L,  
http://spin.atomicobject.com/2014/03/23/xcode-keyboard-shortcuts/

8. unwind segue programmatically  
    0. Create a method like ```- (IBAction)unwindFrom:(UIStoryBoardSegue*)sender``` in the ViewController that you will unwind to.
    1. Create a manual segue (ctrl-drag from File’s Owner to Exit),
    2. Choose it in the Left Controller Menu below green EXIT button.
    3. Insert Name of Segue to unwind.
    4. Then,```- (void)performSegueWithIdentifier:(NSString *)identifier sender:(id)sender``` with your segue identify.  

9. CFBundleVersion与CFBundleShortVersionString  
   前者对应工程设置里显示的Build，后者对应Version，AppStore用的是Build，而用户看到的是Version，因为后者可以本地化显示，比如罗马数字，希伯来数字
   [参考](http://beginor.github.io/2014/07/22/ios-cfbundleshortversionstring-vs-cfbundleversion.html)

10. XCTest
https://developer.apple.com/library/ios/documentation/DeveloperTools/Conceptual/testing_with_xcode/testing_1_quick_start/testing_1_quick_start.html

11. app Distribution  
https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/AppDistributionGuide/Introduction/Introduction.html

12. UTI  
https://developer.apple.com/library/mac/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_intro/understand_utis_intro.html

13. AssetsPickerController  
https://github.com/chiunam/CTAssetsPickerController
