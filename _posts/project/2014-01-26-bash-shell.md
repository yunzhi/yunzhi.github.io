---
layout: post
title: bash shell脚本
category: project
description: 该篇介绍下自己为方便工作写的一些脚本
---

# send 和 get

出于某些需要，目前就职的公司将日常工作的代码放到了ubuntu虚拟机上，日常用户通过网络远程访问虚拟机，虚拟机和访问主机之间没有usb连接，剪切板共享，虚拟机只能访问特定的网络。IT提供特定的ftp以中转代码编译后的镜像，软件，但不允许中转代码。

于是乎，平时编程在虚拟机，代码编译好后生成软件包，通过访问ftp，然后将镜像文件上传到ftp上。每次都进行ftp的操作有点麻烦，于是写了下面两个脚本。send用于将文件上传到ftp，get用于从ftp下载文件到本地。如果上传文件夹，会将文件zip压缩，get到了解压。功能比较简单，但用起来很方便。

如下为这两个脚本。

## send

    #!/bin/bash
    
    #one who want use this script to upload your file need to change dirowner to
    #youself
    
    dirowner=yunzhi #your ftp dir name
    ftpip="10.10.100.99 21" #ftp address ,21 is the port number
    userid=write
    passwd=123456
    
    checkfilename=false #if ftp do not permit to transfer code,pls set this one ture
    
    pr_err()
    {
    	echo -e "\e[1;31m$@\e[0m"
    }
    
    
    TIME=`date "+%H-%M-%S"`
    #echo  ${TIME}
    echo $@
    nowpath=$(pwd)
    
    
    get_fileinfo()
    {
    filepath=$(dirname $@)
    echo filepath=$filepath
    
    if [ "$filepath" == "." ];then
    	filename=$(echo $@ | sed 's;/$;;')
    else
    	filename=$(echo $@ | sed 's;'$filepath'/;;'|sed 's;/$;;')
    fi
    
    echo filename=$filename
    cd "$filepath"
    #pwd
    }
    
    process_dir()
    {
    if [ -d $filename ];then
    	echo "filename is a dir"
    	if [ -e ${filename}.zip ];then
    		echo -e "\e[1;31m[WARNING:] ${filename} exists,are you sure to overwrite it[enter Y/n,回车默认不覆盖，将退出继续执行...] .\e[0m"
    		read answer
    		if [ "$answer" == "n" -o "$answer" == "N" -o -z "$answer" ];then
    			pr_err "Failed to upload $filename to $ftpip";
    			exit
    		fi
    	fi
    	zip -r ${filename}.zip ${filename}
    	filename=${filename}.zip
    
    	if [ "$checkfilename" == "true" ];then
    		unzip -l $filename | egrep "\.c|\.h|\.java|\.cpp"
    		if [ $? -ne 0 ];then
    			pr_err "Base on K-touch privacy policy:
    			The file with .c .h .java .cpp suffix should not be transmitt from this ftp!!!"
    			exit
    		fi
    	fi
    fi
    }
    
    upload_ftp()
    {
    echo "will put $filename to $ftpip!"
    ftp -i -n <<!
    open $ftpip
    user $userid $passwd
    binary
    cd $dirowner
    mkdir ${TIME}_${filename}
    cd ${TIME}_${filename}
    put ${filename}
    exit
    bye
    !
    }
    
    cd "$nowpath"
    for item in $@
    do
    	get_fileinfo $item
    	suffix_var=${item##*.}
    	#echo $suffix_var
    	if [ "$checkfilename" == "true" ];then
    		if [ "$suffix_var" = "c" -o "$suffix_var" = "h" ];then
    			echo -e "\e[1;31m[WARNING:]Base on K-touch privacy policy:\nthe file with .c .h suffix should not be transmitt from this ftp!!!\e[0m"
    			exit
    		fi
    	fi
    	#pwd
    	process_dir
    
    	echo ========upload process begin========
    	upload_ftp
    
    done
    


## get

    #!/bin/bash
    # Descritption: used for download file from ftp
    # one who want to use this should change dirowner to hiself
    # For other FTP address, pls change it fromftp
    
    dirowner=yunzhi
    fromftp="10.10.100.100 21"  #ftp address,you can change it to yours
    username=read
    passwd=123456
    
    TIME=`date "+%H-%M-%S"`
    echo Now time is `date`
    
    pr_err()
    {
    	echo -e "\e[1;31m$@\e[0m"
    }
    
    for_help()
    {
    cat <<Help
    
    You can use this script to download file from ftp $fromftp!
    Make sure you have areadly change the dirname to you owner so that
    you can download file from $dirowner!
    Pls fell free to contact yunzhi(tangyao.made@gmail.com) if you have
    any question or suggestion!
    Good luck!
    
    Usage:
    get [filename/filefolder]
    get [-h/-H/--Help] for help
    
    Help
    }
    
    check_file()
    {
    ftp -i -n <<!
    open $fromftp
    user $username $passwd
    binary
    cd $dirowner
    ls
    exit
    !
    }
    
    cd_folder_and_download()
    {
    ftp -i -n <<FF
    open $fromftp
    user $username $passwd
    binary
    cd $dirowner
    cd ${FILENAME}
    get ${file_get}
    exit
    FF
    }
    
    download_file()
    {
    ftp -i -n <<FF
    open $fromftp
    user $username $passwd
    binary
    cd $dirowner
    get ${FILENAME}
    exit
    FF
    }
    
    decidec_unzip_file()
    {
    	file_var=$@
    	file_get_suffix=${file_var##*.}
    	#echo $file_get_suffix
    	if [ $file_get_suffix = "zip" ];then
    		read -p "Do you want to unzip $file_var?(Y/y)是否解压$file_var?回车默认Y. " UNZIPCHOICE
    		if [ "$UNZIPCHOICE" == "" -o "$UNZIPCHOICE" == "Y" \
    			-o "$UNZIPCHOICE" == "y" -o "$UNZIPCHOICE" == "yes" ];then
    			#unzip $file_get -d ${file_get%.*}
    			unzip $file_var
    		fi
    	fi
    }
    
    
    if [ $# -gt 0 ];then
    
    	if [ $# -eq 1 ];then
    		if [ $@ = "-h" -o $@ = "-H" -o $@ = "-Help" -o $@ = "-usage"\
    			-o $@ = "--help" ];then
    			for_help
    			exit
    		else
    			FILENAME=$@
    		fi
    	fi
    
    fi
    check_file | tee ${TIME}_tmp
    
    if [ -z $FILENAME ];then
    	read -p "please input the file name which you want get: " FILENAME
    fi
    
    filetype=`cat ${TIME}_tmp | grep "\ $FILENAME" | cut -c 1`
    file_get=`echo $FILENAME|cut -c 10-`
    
    if [ $filetype = "d" ]; then
    	if [ -e $file_get ];then
    		read -p "$file_get exists,overwrite it? (Enter Y/y)" CHOICE
    	fi
    
    	if [ "$CHOICE" == "" -o "$CHOICE" == "Y" \
    		-o "$CHOICE" == "y" -o "$CHOICE" == "yes" ];then
    		cd_folder_and_download
    		ls -al $file_get
    		decidec_unzip_file $file_get
    	else
    		pr_err "Failed to download $file_get int $FILENAME"
    	fi
    
    fi
    
    if [ $filetype = "-" ]; then
    	if [ -e $FILENAME ];then
    		read -p "$FILENAME exists,overwrite it(Enter Y/y) ?" CHOICE
    	fi
    
    	if [ "$CHOICE" == "" -o "$CHOICE" == "Y" \
    		-o "$CHOICE" == "y" -o "$CHOICE" == "yes" ];then
    		download_file
    		ls -al $FILENAME
    		decidec_unzip_file $FILENAME
    	else
    		pr_err "Failed to download $FILENAME in $dirowner"
    	fi
    
    fi
    
    rm -rf ${TIME}_tmp

## 说明
先列下bash脚本编写中经常使用的几个点，慢慢来总结下
1. 老容易忘记的功能：去后缀和取后缀的方法（hungry）。
2. 字符串比较
3. 局部变量的使用
4. sed 和 awk


[yunzhi]:    http://yunzhi.com  "Yunzhi"
