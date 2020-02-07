# Android手机配置burpsuite证书操作

### 说明

从Android Q版本开始，已经不能通过用户导入burpsuite证书拦截App的请求，应用App的请求默认不再信任用户安装的证书，除非另有说明，否则默认只信任系统证书。

### 安装系统级证书

+ 将手机root
+ 导出burpsuite证书，以DER格式导出证书
+ 转换证书格式，将der格式转换为pem格式
```
openssl x509 -inform DER -in cacert.der -out cacert.pem
```
+ Android 的受信任 CA 以特殊格式存储在 / system/etc/security/cacerts，需要将pem格式的证书保存为hash命名方式，以0结尾
```
openssl x509 -inform PEM -subject_hash_old -in cacert.pem |head -1  
 9a5ba575
 mv cacert.pem 9a5ba575.0
```
+ 将证书复制到设备，并修改权限为644
```
 adb remount 
 adb push 9a5ba575.0 /system/etc/security/cacerts/  
 adb reboot
```
+ 重启设备，WIFI设置代理即可访问
