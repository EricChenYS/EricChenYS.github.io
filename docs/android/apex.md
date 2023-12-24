


# 解压apex
apex是一个zip，可以使用解压工具直接解压

apex_payload.img是一个ext4文件系统格式的img，可以mount来查看里面的内容
例如：mount  -o ro data/local/tmp/apex_payload.img data/local/tmp/system 


apex安装所在目录：
/apex/
例如：system/system_ext/apex/com.xx.framework.apex被安装到了apex/com.xx.framework@310000001
