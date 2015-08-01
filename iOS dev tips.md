### 以下收集一些很有用小技巧（持续更新中）

* 打开一些旧项目时，运行后上下出现黑块  
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

* NULL/Nil/nil/NSNull

|标志 |	值 |	含义|
|----:|---:|:---:|
|NULL |	(void \*)0 |	C指针的字面零值 |
|nil |	(id)0 |	Objective-C对象的字面零值 |
|Nil |	(Class)0 |	Objective-C类的字面零值 |
|NSNull |	[NSNull null] |	用来表示零值的单独的对象 |

* UIWebView debug  
http://stackoverflow.com/questions/2767902/what-are-some-methods-to-debug-javascript-inside-of-a-uiwebview

* Xcode常用快捷键（拾遗）  
^ + ⌘ + Y,  ⌘ + Y, ^ + ⌘ + arrow, ^ + ⌘ + E, ⌘ + ⌥ + J, ⌘ + ⌥ + L,  
http://spin.atomicobject.com/2014/03/23/xcode-keyboard-shortcuts/

* unwind segue programmatically  
    0. Create a method like ```- (IBAction)unwindFrom:(UIStoryBoardSegue*)sender``` in the ViewController that you will unwind to.
    1. Create a manual segue (ctrl-drag from File’s Owner to Exit),
    2. Choose it in the Left Controller Menu below green EXIT button.
    3. Insert Name of Segue to unwind.
    4. Then,```- (void)performSegueWithIdentifier:(NSString *)identifier sender:(id)sender``` with your segue identify.  

* CFBundleVersion与CFBundleShortVersionString  
   前者对应工程设置里显示的Build，后者对应Version，AppStore用的是Build，而用户看到的是Version，因为后者可以本地化显示，比如罗马数字，希伯来数字
   [参考](http://beginor.github.io/2014/07/22/ios-cfbundleshortversionstring-vs-cfbundleversion.html)

* XCTest
https://developer.apple.com/library/ios/documentation/DeveloperTools/Conceptual/testing_with_xcode/testing_1_quick_start/testing_1_quick_start.html

* app Distribution  
https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/AppDistributionGuide/Introduction/Introduction.html

* UTI  
https://developer.apple.com/library/mac/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_intro/understand_utis_intro.html

* AssetsPickerController  
https://github.com/chiunam/CTAssetsPickerController

* UIActivityViewController  
http://blog.sina.com.cn/s/blog_923fdd9b0101frwr.html

* p12, cer, mobileprovision  
p12是带公钥和私钥的加密文件（需要密码才能加入keychain），其含义为PKCS \#12，Public Key Cryptography Standard
cer是用于存储公钥证书的文件格式。是数字形式的标识，与护照或驾驶员执照十分相似。数字证书是数字凭据，它提供有关实体标识的信息以及其他支持信息。数字证书是由成为证书颁发机构（CA）的权威机构颁发的，由于数字证书有证书权威机构颁发，因此由该权威机构担保证书信息的有效性。此外，数字证书只在特定的时间段内有效。  
数字证书包含证书中所标识的实体的公钥（就是说你的证书里有你的公钥），由于证书将公钥与特定的个人匹配，并且该证书的真实性由颁发机构保证（就是说可以让大家相信你的证书是真的），因此，数字证书为如何找到用户的公钥并知道它是否有效这一问题提供了解决方案。
mobileprovision是mobile device provisioning profile（移动设备供给描述文件），包含设备信息（UDID）、证书信息，还有App ID等信息。所以只有其中列出的设备可以安装由其打包的api安装包。

* 如何处理gif动画（UIWebView)
```
// 设定位置和大小
CGRect frame = CGRectMake(50,50,0,0);
frame.size = [UIImage imageNamed:@"anim.gif"].size;
// 读取gif图片数据
NSData *gif = [NSData dataWithContentsOfFile:
    [[NSBundle mainBundle] pathForResource:@"anim" ofType:@"gif"]];
// view生成
UIWebView *view = [[UIWebView alloc] initWithFrame:frame];
[view loadData:gif MIMEType:@"image/gif" textEncodingName:nil baseURL:nil];
```
