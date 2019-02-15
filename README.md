::iis快速建站，部署php环境 IIS rapid station building and PHP environment deployment
:: $Name:         winsite-php.bat
:: $Version:      v1.5
:: $Function:     iis快速建站，部署php环境
:: $Author:       akbkcklxj
:: $Create Date:  2019-01-14
:: $Description:  1、支持普通用户iis下创建运行网站，提升网站安全性
::                2、仅支持windows系统64位环境
::                3、暂时支持php5.4,5.6,7.0,7.2
::
:: v1.1 bitsadmin下载速度和稳定性较差，引入wget下载，提升稳定性
:: v1.2 脚本基于iis，未安装无法运行，增加对于iis环境判断
:: v1.3 网站id设置脚本内自增长，解决站点id冲突问题
:: v1.4 增加对iis站点是否存在的判断，便于创建和部署php环境
:: v1.5 完善网站管理功能,增加切换php版本功能,增加查看，删除站点功能，彻底解决站点id冲突问题
:: v1.6 计划增加对mysql数据库的支持
::
@echo off&setlocal enabledelayedexpansion
@color 3f
title IIS自主建站程序v1.5
@set temp=D:\php\temp.txt
@set winrarfile="C:\Program Files\WinRAR\WinRAR.exe"
@set phppath=D:\php
@set appcmd="C:\Windows\System32\inetsrv\appcmd.exe"
@set down="C:\Windows\System32\bitsadmin.exe"
@set downdir=%phppath%\Downloads
@set wgetfile=%phppath%\Downloads\wget.exe
@set downurl=http://xixibaobei.gotoip11.com/shell
@set sedfile=%phppath%\Downloads\sed.rar
@set sedexe=%phppath%\Downloads\sed\sed.exe
@if not exist %downdir% (
	md %phppath%\Downloads
)
@if not exist %appcmd% (
    echo.
	echo 检测到缺少appcmd,脚本无法运行，请检查iis是否正确安装
)
@if not exist %down% (
    echo.
	echo 检测到缺少bitsadmin,脚本无法运行，请先安装
)
@if not exist %wgetfile% (
    echo.
	echo 检测到缺少wget工具正在下载,安装中...
	%down% /transfer wget /Download /Priority FOREGROUND  %downurl%/wget.exe  %downdir%\wget.exe  >nul 2>nul
	title IIS自主建站程序v1.5
)
@if not exist %wgetfile% (
	echo.
	echo 自动下载失败请访问 https://eternallybored.org/misc/wget 手动下载
	echo 并保存到默认下载目录[%downdir%]
	pause>nul
	exit
)
@if not exist %winrarfile% (
    echo.
	echo 检测到系统缺少winrar,请安装后再运行脚本
)
@if not exist %sedexe% (
	%wgetfile% -c %downurl%/sed.rar -O %downdir%/sed.rar >nul 2>nul
	%winrarfile% x -inul -o+  %downdir%/sed.rar  %downdir%
)
@sc query W3SVC >nul 2>nul
if errorlevel 1 (
    echo iis服务不存在，请安装后运行脚本
    pause>nul
    exit
) else (
    echo 服务器运行环境检测OK
)
@echo.
:start
title IIS自主建站程序v1.5
@echo   ________________________________________
@echo  ^|                                       ^|
@echo  ^|自主建站v1.5,支持php5.4,5.6,7.0,7.2    ^|
@echo  ^|站点运行账号支持数字字母               ^|
@echo  ^|                                       ^|
@echo  ^|1 查看站点              2 新建站点     ^|
@echo  ^|                                       ^|
@echo  ^|3 部署php环境           4 切换php版本  ^|
@echo  ^|                                       ^|
@echo  ^|5 删除站点                             ^|
@echo  ^|_______________________________________^|
@echo.
@set /p choice="请选择:"
@if %choice% ==1 call :show
@if %choice% ==2 call :new
@if %choice% ==3 call :install
@if %choice% ==4 call :change
@if %choice% ==5 call :delete
echo 不能输入除1、2、3、4、5之外其他字符! & goto start
:show
@echo off
%appcmd% list site >%temp% 
for %%a in (%temp%) do (
     if "%%~za" equ "0" (
	    echo 无站点
        ) else (
        goto query
)
)
:query
@for /f tokens^=2^,3^ delims^=^" %%1 in ('%appcmd% list site') do echo 站点: %%1
@pause>nul
@goto start
:new
%appcmd% list site >%temp% 
for %%a in (%temp%) do (
     if "%%~za" equ "0" (
	    set id=0
        ) else (
        goto siteid
)
)
:siteid
@for /f "tokens=2 delims=:," %%a in ('%sedexe%  -n ^"$p^" %temp%') do (set "id=%%a")
::echo  %id%
@set /a id+=1
@set /p sitename="输入站点名称:"
@if not  defined sitename (
	echo "站点名称不能为空，重新输入"
	goto news)
@%appcmd% list site | findstr /c:"%sitename%">nul 2>nul
@if %errorlevel% ==0 (
    echo 已存在站点%sitename%,请重新输入
    pause>nul
    goto start
) 
@set /p sitepwd="输入站点密码:"
@if not  defined sitepwd (
	echo "密码不能为空，重新输入"
	goto news)
@set /p url="输入域名如1i.com:"
@if not  defined url (
	echo "域名不能为空，重新输入"
	goto news)
@set /p choice="创建同名ftp: 1创建 2不需要"
@if %choice% ==1 call :createftp
@if %choice% ==2 call :createsite 
echo 不能输入除1、2之外其他字符! & goto start
:createftp
@sc query ftpsvc >nul 2>nul
if errorlevel 1 (
    echo iis默认ftp服务不存在，若要创建ftp请安装后运行脚本
    pause>nul
    goto start
)
@%appcmd% list site | findstr /c:"myftpserver">nul 2>nul
@if %errorlevel% ==1 (
    %appcmd% add site /name:myftpserver /bindings:ftp://*:21 /physicalpath:"C:\inetpub\ftproot"
	%appcmd% set site "myftpserver" -ftpServer.security.ssl.controlChannelPolicy:SslAllow -ftpServer.security.ssl.dataChannelPolicy:SslAllow -ftpServer.security.authentication.basicAuthentication.enabled:true
	%appcmd% set site "myftpserver" /ftpServer.userIsolation.mode:StartInUsersDirectory
	for /f "tokens=2 delims=:," %%a in ('%sedexe%  -n ^"$p^" %temp%') do (set "id=%%a")
)
@%appcmd% add vdir /app.name:"myftpserver/"  /path:/"%sitename%" /physicalpath:"d:\%sitename%"
@%appcmd% set config "myftpserver" -section:system.ftpServer/security/authorization /+"[accessType='Allow',users='%sitename%',permissions='Read, Write']" /commit:apphost
@echo 站点%sitename%同名ftp创建完成
:createsite
@net user  %sitename% %sitepwd% /add
@net localgroup IIS_IUSRS %sitename% /add
@md d:\%sitename% 
@echo ^<^?php phpinfo(); ^?^> >d:\%sitename%\index.php
@cacls D:\%sitename% /e /g %sitename%:f /T
@%appcmd% add apppool /name:"%sitename%" /processModel.userName:%sitename% /processModel.identityType:SpecificUser /processModel.password:%sitepwd% /managedRuntimeVersion:"v4.0"
@%appcmd% add site /name:"%sitename%" /id:%id% /bindings:http://%url%:80  /virtualDirectoryDefaults.userName:"%sitename%"  /virtualDirectoryDefaults.password:"%sitepwd%" /applicationDefaults.applicationPool:"%sitename%" /physicalPath:"d:\%sitename%"
@%appcmd% set config "%sitename%" /section:defaultDocument /+files.[@start,value='index.php'] /commit:"%sitename%"
@echo "%sitename%站点创建成功"
@pause>nul
@cls
@goto start
:install
@set /p sitename="输入部署php的站点名称:"
@if not  defined sitename (
	echo "站点名称不能为空，重新输入"
	goto install)
@%appcmd% list site | findstr /c:"%sitename%">nul 2>nul
@if %errorlevel% ==1 (
    echo 不存在站点%sitename%,请确认站点名称
    pause>nul
    goto install
)
@set /p version="输入格式php5.4:"
@if not  defined version (
	echo "php版本不能为空，重新输入"
	goto install)
@if not exist %phppath%\%version% (
	mkdir %phppath%\%version%
	echo 创建目录:%phppath%\%version%成功
)
@if %version%==php5.4  (
    %appcmd% set config /section:system.webServer/fastCgi  /+"[fullPath='D:\php\php5.4\php-cgi.exe']" >nul 2>nul
	::%down% /transfer "%version%" /Download  /Priority FOREGROUND   https://windows.php.net/downloads/releases/archives/php-5.4.9-nts-Win32-VC9-x86.zip  %phppath%\%version%.zip
	%wgetfile% -c %downurl%/php5.4.zip -O  %phppath%\%version%.zip
	title IIS自主建站程序v1.5
			goto tar
    )else (
    if %version%==php5.6  (
          %appcmd% set config /section:system.webServer/fastCgi  /+"[fullPath='D:\php\php5.6\php-cgi.exe']" >nul 2>nul
		  %wgetfile% -c %downurl%/php5.6.zip -O  %phppath%\%version%.zip
		  title IIS自主建站程序v1.5
		  goto tar
    )else (
        if %version%==php7.0 (
            %appcmd% set config /section:system.webServer/fastCgi  /+"[fullPath='D:\php\php7.0\php-cgi.exe']" >nul 2>nul
			%wgetfile% -c %downurl%/php7.0.zip -O  %phppath%\%version%.zip    
			title IIS自主建站程序v1.5
				goto tar
        )else (
            if %version%==php7.2 (
                %appcmd% set config /section:system.webServer/fastCgi  /+"[fullPath='D:\php\php7.2\php-cgi.exe']" >nul 2>nul
				%wgetfile% -c  %downurl%/php7.2.zip -O  %phppath%\%version%.zip
				title IIS自主建站程序v1.5
				goto tar
            )
        )
    )
	echo "不支持输入php版本，请退出重新运行脚本"
	pause>nul
	exit
)
:tar
@%winrarfile% x -inul -o+  %phppath%\%version%.zip  %phppath%\%version%  -y >nul 2>nul
@if not exist d:\%sitename%\web.config (
	@%appcmd% unlock config -section:system.webServer/handlers >nul 2>nul
	@%appcmd% set config "%sitename%" /section:system.webServer/handlers /+"[name='PHP_FastCGI', path='*.php',verb='*',modules='FastCgiModule',scriptProcessor='%phppath%\%version%\php-cgi.exe',resourceType='Either']"  >nul 2>nul
	@echo "站点%sitename%部署%version%完成"
	@pause>nul
	@cls
	@goto start
)
@findstr /i "FastCgiModule"   D:\%sitename%\web.config >nul
@if %errorlevel% ==0 (
		goto alter
) else (
		%sedexe%  -i "/<system.webServer>/a\<handlers>\n\<add name=\"PHP_FastCGI\" path=\"*.php\" verb=\"*\" modules=\"FastCgiModule\" scriptProcessor=\"D:\\php\\%version%\\php-cgi.exe\" resourceType=\"Either\" />\n</handlers>"   D:\%sitename%\web.config
		echo 站点%sitename%部署%version%完成"
		pause>nul 
		goto start
)
:alter
@for /f "tokens=3 delims=\" %%i in ('%sedexe%  -n "/FastCgiModule/p"    D:\%sitename%\web.config') do (set "ver=%%i")
		%sedexe% -i "s/%ver%/%version%/" D:\%sitename%\web.config
		echo 站点%sitename%部署%version%完成"
		pause>nul 
		goto start
:change
@set /p sitename="输入切换php站点名称:"
@if not  defined sitename (
	echo "站点名称不能为空，重新输入"
	goto change)
@%appcmd% list site | findstr /c:"%sitename%">nul 2>nul
@if %errorlevel% ==1 (
    echo 不存在站点%sitename%,请确认站点名称，或查看站点
    pause>nul
    goto show
)
@set /p newver="输入切换版本如php5.4:"
@if not  defined newver (
	echo "不能输入空字符"
	goto change)

@if not exist %phppath%\%newver% (
    echo.
	echo 未安装%newver%，请直接部署%newver%
	pause>nul
	goto start
)
@if not exist D:\%sitename%\web.config (
    echo.
	echo 检测到不存在web.config，请直接部署%newver%
	pause>nul
	goto start
)
@for /f "tokens=3 delims=\" %%i in ('%sedexe%  -n "/FastCgiModule/p"    D:\%sitename%\web.config') do (set "ver=%%i")
@if %newver%==%ver% (
	echo 当前已经是%newver% ，不用切换
	pause>nul 
	goto start
	)
@if %newver% neq %ver% (
	%sedexe% -i "s/%ver%/%newver%/" D:\%sitename%\web.config
	echo %newver%切换完成
	pause>nul 
	goto start
	)
:delete
@set /p sitename="输入要删除站点名称:"
@if not  defined sitename (
	echo "站点名称不能为空，重新输入"
	goto delete)
@%appcmd% list site | findstr /c:"%sitename%">nul 2>nul
@if %errorlevel% ==1 (
    echo 不存在站点%sitename%,请确认站点名称
    pause>nul
    goto delete
)
@%appcmd% delete site "%sitename%"
@%appcmd% delete apppool "%sitename%"
@net user %sitename% /delete
@echo "站点%sitename% 删除完成"
@echo 是否删除站点目录和文件:1 删除，2 不删
@set /p choice="请输入:"
@if %choice% ==1 call :deletedir
@if %choice% ==2 call :start
echo 不能输入除1、2之外其他字符! & goto start
:deletedir
@rmdir /s/q d:\%sitename%
@echo 站点%sitename%文件已全部删除
@pause>nul
@cls
@goto start
