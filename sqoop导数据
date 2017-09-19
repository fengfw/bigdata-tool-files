#列表显示
sqoop list-tables  --connect jdbc:oracle:thin:@192.168.109.222.131:1521:oraclesid  --username user  --password 123456

#创建hdfs目录
sudo -u hdfs hdfs dfs -mkdir /oracle

#导数据命令
sudo -u hdfs sqoop import --connect jdbc:oracle:thin:@192.168.109.222.131:1521:oraclesid  --username user  --password 123456  \
--table TEST_01 \
--target-dir /oracledb/TEST_01 \cd 
--fields-terminated-by "\\01" \
--hive-drop-import-delims     \
--null-string '\\N'           \
--null-non-string '\\N'       \
-m 1

#修改权限
sudo -u hdfs hdfs dfs -chmod -R 777 /oracle/*

