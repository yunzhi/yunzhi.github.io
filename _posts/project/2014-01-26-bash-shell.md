---
layout: post
title: bash shell脚本
category: project
description: 该篇介绍下自己为方便工作写的一些脚本
---

## send 和 get

出于某些需要，目前就职的公司将日常工作的代码放到了ubuntu虚拟机上，日常用户通过网络远程访问虚拟机，虚拟机和访问主机之间没有usb连接，剪切板共享，虚拟机只能访问特定的网络。IT提供特定的ftp以中转代码编译后的镜像，软件，但不允许中转代码。

于是乎，平时编程在虚拟机，代码编译好后生成软件包，通过访问ftp，然后将镜像文件上传到ftp上。每次都进行ftp的操作有点麻烦，于是写了下面两个脚本。send用于将文件上传到ftp，get用于从ftp下载文件到本地。如果上传文件夹，会将文件zip压缩，get到了解压。功能比较简单，但用起来很方便。

如下为这两个脚本。

### send

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
    
    #获取脚本用户名 get the dirname from shell script user
    if [ -z $dirowner  ];then
	tyusername=`id | awk -F["()"] '{print $2}'`
	#pr_err "dirname was not defined! we set it to you computer id $tyusername"
	dirowner=$tyusername
    fi
    
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
    			pr_err "Base on company privacy policy:
    			The file with .c .h .java .cpp suffix should not be transmitt from this ftp!!!"
    			exit
    		fi
    	fi
    fi
    }
    
    upload_ftp()
    {
    echo "will put $filename to $dirowner in $ftpip!"
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
    


### get

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
    
    #获取脚本用户名 get the dirname from shell script user
    if [ -z $dirowner ];then
	tyusername=`id | awk -F["()"] '{print $2}'`
	#pr_err "dirname was not defined! we set it to you computer id $tyusername"
	dirowner=$tyusername
    fi
    for_help()
    {
    cat <<Help
    
    You can use this script to download file from ftp $fromftp!
    Make sure you have areadly change the dirname to you owner so that
    you can download file from $dirowner!
    Pls fell free to leave a message if you have
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
    
    decide_unzip_file()
    {
    	file_var=$@
    	file_get_suffix=${file_var##*.}
    	#echo $file_get_suffix
    	if [ $file_get_suffix = "zip" ];then
		read -p "Do you want to unzip $file_var?(Y/y/N/n)是否解压$file_var?回车默认Y. " UNZIPCHOICE
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
    
    cat ${TIME}_tmp | grep failed
    if [ $? -eq 0 ];then
	pr_err "$dirowner not exists in $fromftp,pls check!"
	exit 1
    fi

    if [ ! -s ${TIME}_tmp ];then
	pr_err "There is nothing in $dirowner of $fromftp,pls check!"
	rm ${TIME}_tmp
	exit 2
    fi
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
		decide_unzip_file $file_get
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
		decide_unzip_file $FILENAME
    	else
    		pr_err "Failed to download $FILENAME in $dirowner"
    	fi
    
    fi
    
    rm -rf ${TIME}_tmp

## 批量添加和替换的脚本

该脚本用于替换某个文件夹及其子文件夹下的一些文件中得某一行的文字，或者追加文字到某一行之后。主要使用了sed得相关技巧。

    #!/bin/bash
    ########################################################################
    #
    # Description: check if the macros has been changed by ourself
    #				If not changed, process carry on
    #				If yes, we should stop and take care of it
    #
    # Parameter:	
    #				$1:the original text
    #				$2:the target text
    #				
    #				if $1 is --append, $2 will be append to the end of file 
    #				
    #	
    ########################################################################
    
    top=$(pwd)
    
    source=$1
    target=$2
    
    insert_context_after_textname=$2
    insert_target=$3
    
    delete_context=$2
    
    echo "the source macros is-----   $source"
    echo "the target macros is-----   $target"
    
    if [ -z $source ];then
    	echo "Failed to find the source macros!"
    	exit
    fi
    
    somethingchanged=0
    
    ## listf is the file you want to modified by this script
    ## you can use ls , find .e.g command to get the files 
    listf=$(find $top/TBW9723* -name ProjectConfig.mk)
    
    if [ "$source" = "--append" ];then
    	for appendnum in $listf
    	do
    		echo "working on $appendnum ..."
    		echo "" >> $appendnum
    		echo $target >> $appendnum
    	done
    	echo "Finished append $target to every file!"
    	exit
    fi
    
    if [ "$source" = "--insert" ];then
    	echo "this time will insert --- $insert_target ---after----- $insert_context_after_textname"
    	for insertnum in $listf
    	do
    		echo "working on $insertnum ..."
    		sed "/${insert_context_after_textname}/a\\${insert_target}" $insertnum > tmp
    		cat tmp > $insertnum
    		rm tmp
    	done
    	echo "Finished insert $insert_target to every file !"
    	exit
    fi
    
    if [ "$source" = "--delete" ];then
    	for deletenum in $listf
    	do
    		echo "working on $deletenum ..."
    		delete_word_times=$(grep -c ${delete_context} ${deletenum})
    		echo "The $delete_context appear $delete_word_times line"
    		if [ $delete_word_times != 1 ];then
    			echo "The $delete_context appear not only in one line,need confirm!"
    			exit
    		fi
    		sed "s/${delete_context}//" $deletenum > tmp
    		cat tmp > $deletenum
    	done
    	echo "Finished delete $delete_context to every file!"
    	exit
    fi
    
    
    gtype=${1%=*}=
    echo $gtype-----
    
    for item in $listf
    do
    	result=$(cat $item | grep "$gtype")
    	#echo $result
    	if [ "$result" != "$source" ];then
    		echo $item
    		echo $result
    		somethingchanged=1
    	fi
    done
    
    if [ $somethingchanged -eq 1 ];then
    	echo "something changed this time,pls first check it"
    	exit
    fi
    
    #######################################################################
    #
    # Description: if the macros not modified by your company, can use this script to
    # 				replace the item in all the files
    #
    # Parameter:	
    #				$1:the original text
    #				$2:the target text
    #	
    #######################################################################
    
    for prodnum in $listf
    do
    	echo "working on $prodnum ..."
    	cat $prodnum | sed "s/${source}/${target}/g" > temp
    	cat temp > $prodnum
    done
    
    rm temp
    echo "the macros in all files are replaced! Congratulations!"


## LCD初始化参数转换脚本

在移植LCD 驱动时，最常见的操作就是将屏幕供应商提供的lcd driver IC的初始化参数转化成特定平台的格式。这些初始化参数有的时候特别多，如果人工去转化，不免会出现一些错误，因此在实际工作中，结合shell, awk, sed等一些linux 工具，编写了一些脚本来完成这个工作。当然，因为初始文件的不确定性，该脚本在使用中都会做一些改动。

### MTK平台的初始化参数转化脚本

该脚本用于将这样的语句

	//description
	SPI_WriteCmd(0x00); 
	SPI_WriteDat(0x00);

	SPI_WriteCmd(0xFF);  //command
	SPI_WriteDat(0x80);
	SPI_WriteDat(0x09); 
	SPI_WriteDat(0x01);
	
	SPI_WriteCmd(0x00); 
	SPI_WriteDat(0x80);

	SPI_WriteCmd(0xFF); 
	SPI_WriteDat(0x80);
	SPI_WriteDat(0x09);

转换成

	{0x00,1,{0x00}},
	{0xFF,3,{0x80,0x09,0x01}},
	{0x00,1,{0x80}},
	{0xFF,2,{0x80,0x09}}

因为源文件格式不确定，因此需要针对源文件进行处理。下面时脚本所有的操作说明

1.awk '/SPI/{print $1}' 将源文件过滤成

	SPI_WriteCmd(0x00); 
	SPI_WriteDat(0x00);
	SPI_WriteCmd(0xFF);
	SPI_WriteDat(0x80);
	SPI_WriteDat(0x09); 
	SPI_WriteDat(0x01)；

2.awk -F"[()]" '/SPI_WriteCmd/{T=$2;next}{print T" "$2;}' 将第一步处理后的文件转换成

	0x00 0x00
	0xFF 0x80
	0xFF 0x09
	0xFF 0x01

这一步的操作存在缺陷，如果最后连续两个

	SPI_WriteCmd(0x00);
	SPI_WriteCmd(0x00);
则会丢失

3.awk '{if($1==x){i=i","$2}else{if(NR>1){print i};i=$0};x=$1;y=$2}END{print i}' 转换成

	00 0x00
	FF 80，09，01

4.调用awk_shell生成最终的数组

	awk -f awk_shell  init_code > code

	BEGIN{FS="[ ,]"}
	{
		for(i=2;i<=NF;i++){
			if(i==2){
				line=$i
			}else{
				line=line","$i;
			}
		}
		printf("%s, %d, %s,\n",$1,NF-1,line);
	}

总结下，运用管道将上面的处理连在一起就是：

	awk '/SPI/{print $1}' | awk -F"[()]" '/SPI_WriteCmd/{T=$2;next}{print T" "$2;}' |awk '{if($1==x){i=i","$2}else{if(NR>1){print i};i=$0};x=$1;y=$2}END{print i}' | awk "BEGIN{FS="[ ,]"}{for(i=2;i<=NF;i++){if(i==2){line=$i}else{line=line","$i;}}printf("%s, %d, %s,\n",$1,NF-1,line);}"

### 高通平台的dtsi文件

当时写这个脚本是为了将mtk平台的参数转化到高通平台上，其实还可以继续优化的，后来功能实现了，就没有再继续处理了。当然如果是别的格式的参数，稍微处理下，应该就可以了。如下是当时的脚本

    #!/bin/bash
    
    #############################################################
    #
    # Description: generate lcd init code from given source code
    #
    # Author     : tangyao (tangyao@k-touch.cn)
    # Version    : 1.0 
    # Date	     : 2014-03-08
    #
    # Parameters :
    # 	1. the source file (temporarily only support for mtk source file)
    #	2. the num names. e.g. ty_4inch_trust_otm8018b_wvga_video_on_cmd
    #	(output header file will be named num.h. e.g. ty_4inch_trust_otm8018b_wvga_video_on_cmd.h)
    #	(for kernel,will be named num.dtsi. e.g ty_4inch_trust_otm8018b_wvga_video_on_cmd.dtsi)
    #
    ##############################################################
    
    nowtime=`date +%m-%d-%H-%m-%S`
    nowpath=`pwd`
    
    pr_err()
    {
    #set the forecolor red when error happened!
    	echo -e "\e[1;31m$@\e[0m"
    }
    
    
    if [ $# -ne 2 ];then
    	pr_err "Need two parameters:"
    	pr_err "\t First one shuld be source file name(full patch).\t"
    	pr_err "\t Second should be the output file name without suffix"
    	exit
    fi
    
    export input_file_name=$1
    export output_file_name=$2
    
    mkdir "${nowtime}_${output_file_name}"
    
    
    # c head file format
    generate_c_header_description()
    {
    upletter_for_para=`echo "$1" |tr 'a-z' 'A-Z'`
    echo "/****************************************************************************"
    echo " * This is auto generate file by scripts,you can modified it by yourself!"
    echo " * Author:tangyao(tangyao@k-touch.cn)"
    echo " * Date:$nowtime"
    echo "*****************************************************************************/"
    echo "#ifndef _${upletter_for_para}_H_"
    echo "#define _${upletter_for_para}_H_"
    #echo "#include \"mipi_dsi.h\""
    echo ""
    }
    
    generate_11_29_code()
    {
    echo "extern char ty_common_11code_video_on_cmd[];"
    echo ""
    echo "extern char ty_common_29code_video_on_cmd[];"
    echo ""
    }
    
    generate_11_29_code_num()
    {
    echo "{0x04,ty_common_11code_video_on_cmd, 150},"
    echo "{0x04,ty_common_29code_video_on_cmd, 150}"
    }
    
    # ok,let's first generate header file
    cat $input_file_name |awk -F"[{}]" '{print $2,$3}' | sed 's/,[0-9]*, /,/g' | awk -F, '{if(NF<16){printf("0x0%x,0x00,0x29,0xc0,",NF)}else{printf("0x%x,0x00,0x29,0xc0,",NF)};for(i=1;i<=NF;i++)printf("%s,",$i);left=(4-NF%4);if(left!=4){while(left--)printf("0xFF,")};printf("t\n");}' |sed 's/,t/tt/' | awk -F, '{ for(i=0;i<NF/4;i++){ for(j=1;j<5;j++){printf "%s,",$(i*4+j)};if(j=3)printf("\n");}}' | sed '/0x00,0x29,0xc0/i\\' |sed "/0x00,0x29,0xc0/i\static char ${output_file_name}[yunzhi]={" | sed '/tt/a\};' |sed 's/tt,//' |awk 'BEGIN{i=-1}{ if(match(sprintf("%s",$0),/yunzhi/))i++;sub(/yunzhi/,sprintf("%d",i));print}' | sed 's/\[//g' |sed 's/\]/\[\]/g' > $nowpath/${nowtime}_${output_file_name}/tmp1
    
    # generate the defination of the num
    cat $input_file_name |awk -F"[{}]" '{print $2,$3}' | sed 's/,[0-9]*, /,/g' |  awk -F, '{if(NF<16){printf("0x0%x,0x00,0x29,0xc0,",NF)}else{printf("0x%x,0x00,0x29,0xc0,",NF)};for(i=1;i<=NF;i++)printf("%s,",$i);left=(4-NF%4);if(left!=4){while(left--)printf("0xFF,")};printf("t\n");}' |sed 's/,t/tt/' | awk -F, '{if(NF<16)printf("{0x0%x,%s%d},\n",NF,"'$output_file_name'",NR-1);else{printf("{0x%x,%s%d},\n",NF,"'$output_file_name'",NR-1)}}' > $nowpath/${nowtime}_${output_file_name}/tmp2
    
    # finale generate the source file
    generate_c_header_description $output_file_name > $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    echo "" >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    generate_11_29_code >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    
    cat $nowpath/${nowtime}_${output_file_name}/tmp1 >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    echo "" >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    
    echo "static struct mipi_dsi_cmd $output_file_name[]={" >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    cat $nowpath/${nowtime}_${output_file_name}/tmp2 >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    generate_11_29_code_num >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    echo "};" >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    echo "" >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    
    num_of_command=`grep -c $output_file_name $nowpath/${nowtime}_${output_file_name}/tmp2`
    let num_of_command=num_of_command+2
    count=`echo ${output_file_name}_NUM | tr 'a-z' "A-Z"`
    echo "#define $count $num_of_command" >>$nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    echo "" >>$nowpath/${nowtime}_${output_file_name}/$output_file_name.h
    
    echo "#endif" >> ./${nowtime}_${output_file_name}/$output_file_name.h
    
    rm $nowpath/${nowtime}_${output_file_name}/tmp1
    rm $nowpath/${nowtime}_${output_file_name}/tmp2
    
    if [ $? -eq 0 ];then
    	echo "Successfully generated $nowpath/${nowtime}_${output_file_name}/$output_file_name.h!"
    else
    	pr_err "Error happen!pls check!"
    fi
    
    ###########################
    ## now generate dtsi file
    ###########################
    
    cat $input_file_name |awk -F"[{}]" '{print $2,$3}' | sed 's/,[0-9]*, /,/g' |  awk -F, '{if(NF<16){printf("0x29,0x01,0x00,0x00,0x00,0x00,0x0%x,",NF);}else{printf("0x29,0x01,0x00,0x00,0x00,0x00,0x%x,",NF);};for(i=1;i<=NF;i++)printf("%s,",$i);printf("t\n");}' |sed 's/,t//' |awk -F, '{printf("%s %s %s %s %s %s %s\n",$1,$2,$3,$4,$5,$6,$7);for(i=0;i<NF/4;i++){ for(j=1;j<5;j++){printf "%s,",$(i*4+j+7)};if(j=3)printf("\n");}}' | tr 'a-f' 'A-F' | sed 's/,0x/ /g' |sed 's/0x//g' |sed 's/,//g' | sed '/^$/d' |awk '{printf("\t\t\t%s\n",$0)}' > $nowpath/${nowtime}_${output_file_name}/tmp3
    
    echo -e "\t\tqcom,mdss-dsi-on-command = ["	 > $nowpath/${nowtime}_${output_file_name}/$output_file_name.dtsi
    cat $nowpath/${nowtime}_${output_file_name}/tmp3 >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.dtsi
    echo -e "\t\t\t05 01 00 00 C8 00 02 11 00" 	 >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.dtsi
    echo -e "\t\t\t05 01 00 00 14 00 02 29 00];"	 >> $nowpath/${nowtime}_${output_file_name}/$output_file_name.dtsi
    
    if [ $? -eq 0 ];then
    	echo "Successfully generated $nowpath/${nowtime}_${output_file_name}/$output_file_name.dtsi!"
    	rm -rf $nowpath/${nowtime}_${output_file_name}/tmp3 
    else
    	pr_err "Error happen!pls check!"
    fi

## 说明

先列下bash脚本编写中经常使用的几个点，慢慢来总结下
1. 老容易忘记的功能：去后缀和取后缀的方法（hungry）。
2. 字符串比较
3. 局部变量的使用
4. sed 和 awk

[yunzhi]:    http://yunzhi.com  "Yunzhi"