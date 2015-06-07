### 以下收集一些很有用小技巧

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
