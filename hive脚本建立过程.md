#第一步建立一个sql文件  
vi *.sql  
use default;  
create external table A2_REAL_MIX(ORG_NAME string,FILLING_DT string,YEAR_MONTH string,CUST_NM string,EMP_NMN string,EMP_NM_E string,C  
ERTI_TYPE string,CERTI_ID string,CANBAODI string,YANGLAO_COMP NUMBER) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TEXTFI  
LE;  
  
#第二部建立一个shell脚本  
vi t  
#!/bib/bash  
 beeline -u jdbc:hive2://169.254.109.130:10000/default -n hive -p 123456 -f /root/*.sql（注意root指的是路径，不一定你们的也是在/root下）  

#第三部执行shell脚本  
bash t  
