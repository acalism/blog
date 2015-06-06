### 以下收集一些很有用小技巧

* 打开一些旧项目时，运行后上下出现黑块。
  原因：缺少大屏对应的LaunchImage，此时`[UIScreen mainScreen].bounds`的结果也是不正确的。
  解决办法：（不需要添加image）只需添加对相应屏幕尺寸的支持即可。
