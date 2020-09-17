## 1. 如何把html转换成images

### 1.1. 需求

因为钉钉机器人不能发送图文混排的消息，所以只能把html页面转换成图片。在网上看了下用到的库基本都是imgkit  

### 1.2. 安装

#### 1.2.1. 安装imgkit
```
pip install imgkit
```

#### 1.2.2. 安装wkhtmltopdf
```
# yum install -y libpng
# yum install -y libjpeg
# yum install -y openssl
# yum install -y icu
# yum install -y libX11
# yum install -y libXext
# yum install -y libXrender
# yum install -y xorg-x11-fonts-Type1
# yum install -y xorg-x11-fonts-75dpi
# wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
# unxz wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
# tar -xvf wkhtmltox-0.12.4_linux-generic-amd64.tar
# mv wkhtmltox/bin/* /usr/local/bin/
```

#### 1.2.3. 安装中文字体

如果不做此步骤，转换后的图片会出现乱码

```
# yum -y install fontconfig
# mkdir /usr/share/fonts/chinese
```  

把本机C:\Windows\Fonts下的所有文件传到/usr/share/fonts/chinese目录  

#### 1.2.4. 测试
```
import imgkit
config = imgkit.config(wkhtmltoimage='/usr/local/bin/wkhtmltoimage')
imgkit.from_url('http://www.baidu.com', 'out.jpg',config=config)

```

### 1.3. 参考

https://gist.github.com/paulsturgess/cfe1a59c7c03f1504c879d45787699f5
https://pypi.org/project/imgkit/
https://blog.csdn.net/weiguang1017/article/details/80229133