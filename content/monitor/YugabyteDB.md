+++
title = 'YugabyteDB'
date = 2024-04-03T15:30:58+08:00
draft = true
+++

参考：https://docs.yugabyte.com/preview/explore/observability/prometheus-integration/macos/

下载:

```sh
wget https://github.com/yugabyte/yb-sample-apps/releases/download/v1.4.1/yb-sample-apps.jar -O yb-sample-apps.jar
```

运行

```sh
java -jar ./yb-sample-apps.jar \
    --workload CassandraKeyValue \
    --nodes 127.0.0.1:9043 \
    --num_threads_read 1 \
    --num_threads_write 1
    
java -jar ./yb-sample-apps.jar \
    --workload CassandraKeyValue \
    --nodes 10.61.200.238:5433 \
    --num_threads_read 1 \
    --num_threads_write 1 \
    --username thingswise_admin \
    --password 5gwqLJqeIPHBue0cDSyJN05fo4deEf38QW2guas41zNw6qlrbqm8AaCG4Embg8AbAdoA1yq2DWbdw1CQHgm484NE7AQGHxc7c281dWpUWf9JZGs2e2PckrAd9D1bVNDf 


java -jar yb-sample-apps.jar --workload CassandraKeyValue --nodes 10.61.200.238:9043 --username thingswise_admin --password 5gwqLJqeIPHBue0cDSyJN05fo4deEf38QW2guas41zNw6qlrbqm8AaCG4Embg8AbAdoA1yq2DWbdw1CQHgm484NE7AQGHxc7c281dWpUWf9JZGs2e2PckrAd9D1bVNDf --ssl_key /etc/ssl/yugabyte/yugabytedb.crt


java -jar yb-sample-apps.jar \
--workload CassandraKeyValue \
--nodes 127.0.0.1:9042 \
--username thingswise_admin \
--password 5gwqLJqeIPHBue0cDSyJN05fo4deEf38QW2guas41zNw6qlrbqm8AaCG4Embg8AbAdoA1yq2DWbdw1CQHgm484NE7AQGHxc7c281dWpUWf9JZGs2e2PckrAd9D1bVNDf \
--ssl /etc/ssl/yugabyte/yugabytedb.crt

 java -jar ./yb-sample-apps.jar --workload CassandraKeyValue \
                                    --nodes localhost:9043 \
                                    --num_threads_write 1 \
                                    --num_threads_read 4 \
                                    --value_size 4096

```

