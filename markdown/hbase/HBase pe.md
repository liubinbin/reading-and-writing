hbase pe

nohup ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --table=petest01 --autoFlush=True --columns=10 --valueSize=100 --presplit=20 --bloomFilter=NONE --rows=1000000 sequentialWrite 100 > pe02.log 2>&1 &