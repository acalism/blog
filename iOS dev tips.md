### 以下收集一些很有用小技巧（持续更新中）

* 打开一些旧项目时，运行后上下出现黑块。  
  原因：缺少大屏对应的LaunchImage，此时`[UIScreen mainScreen].bounds`的结果也是不正确的。  
  解决办法：（不需要添加image）只需添加对相应屏幕尺寸的支持即可。  

* 创建非全屏大小的窗口
```
CGRect rect = [[UIScreen mainScreen] bounds];
//rect = UIEdgeInsetsInsetRect(rect, UIEdgeInsetsMake(20., 0., 30., 0.));
self.window = [[UIWindow alloc] initWithFrame:rect];
self.window.clipsToBounds = YES;
self.viewController = [ViewController new];
self.window.rootViewController = self.viewController;
[self.window makeKeyAndVisible];
```

* Xcode升级后插件失效的问题
```
XCODEUUID=`defaults read /Applications/Xcode.app/Contents/Info DVTPlugInCompatibilityUUID`
for f in ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins/*; do defaults write "$f/Contents/Info" DVTPlugInCompatibilityUUIDs -array-add $XCODEUUID; done
```
以下是来源：
http://stackoverflow.com/questions/20732327/xcode-5-required-plug-in-not-present-in-dvtplugincompatibilityuuids


* 保存照片至系统相册  
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

* NULL/Nil/nil/NSNull [来源](http://nshipster.cn/nil/)  

| 标志 |	值 |	含义 |  
|---|---:|:---:|  
|NULL |	(void \*)0 |	C指针的字面零值|  
|nil |	(id)0 |	Objective-C对象的字面零值|  
|Nil |	(Class)0 |	Objective-C类的字面零值|  
|NSNull |	[NSNull null] |	用来表示零值的单独的对象|  



* UiWebView debug  
http://stackoverflow.com/questions/2767902/what-are-some-methods-to-debug-javascript-inside-of-a-uiwebview

* Xcode常用快捷键  
^ + ⌘ + Y,  ⌘ + Y, ^ + ⌘ + arrow, ^ + ⌘ + E, ⌘ + ⌥ + J, ⌘ + ⌥ + L,  
http://spin.atomicobject.com/2014/03/23/xcode-keyboard-shortcuts/
