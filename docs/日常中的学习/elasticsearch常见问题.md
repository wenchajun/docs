备注

curl -H 'Content-Type: application/json' -s -XPUT "http://elasticsearch-logging-data:9200/_cluster/settings" -d '{"transient":{"logger._root":"DEBUG"}}'
