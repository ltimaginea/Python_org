# Boost 官网指南

[Boost C++ Libraries](https://www.boost.org/)

[Boost Getting Started on Windows - 1.77.0](https://www.boost.org/doc/libs/1_77_0/more/getting_started/windows.html)



## ① 下载 Boost.7z包

- 下载 .7z包 `boost_1_77_0.7z` ，解压至目录： `C:\Program Files\boost\boost_1_77_0\`  。
               

![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830215350371-917732585.png)
             

## ② Prepare to Use a Boost Library Binary

- 以**管理员身份**运行PowerShell ，切换至 Boost 根目录 `C:\Program Files\boost\boost_1_77_0` 。因为 Boost 根目录的所处目录`C:\Program Files\` 的写入文件 需要Windows管理员权限，所以需要以管理员身份运行PowerShell 。

  在PowerShell的 Boost 根目录下，分别运行以下命令：

  ```bash
  .\bootstrap.bat
  .\b2.exe
  ```
  
  ​                  

![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830221726889-103616261.png)
                      
![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210829231637518-1951246550.png)


## ③ 配置 Visual Studio 属性

- 配置 Visual Studio 项目属性：

  ```bash
  # added to compiler include paths:
  C:\Program Files\boost\boost_1_77_0
  
  # added to linker library paths:
  C:\Program Files\boost\boost_1_77_0\stage\lib
  ```

  ​                 

![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830220931299-554339011.png)
                     
![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830221304833-1568064546.png)
                     
![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830221515834-703387954.png)
                                          
![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830222408798-497240156.png)
                                          
![](https://img2020.cnblogs.com/blog/2141093/202108/2141093-20210830221900016-120686030.png)
            