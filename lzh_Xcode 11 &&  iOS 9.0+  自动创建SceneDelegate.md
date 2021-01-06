# Xcode11 与 iOS 9.0 以后, 自动创建SceneDelegate

## 原因:
用于ipad 的多进程开发,如果不需要多进程,可以删掉,操作如下:

+ AppDelegate 文件中删除 SceneDelegate相关的代理
+ 删除SceneDelegate 文件
+ info.plist 里面删除Application Scene Manifest 字典

