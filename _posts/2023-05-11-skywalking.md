TODO

OpenTelemetry
docker run --name oap -d -e SW_STORAGE=elasticsearch -e SW_STORAGE_ES_CLUSTER_NODES=ecstatic_jang:9200 --network docker_default -p 11800:11800 apache/skywalking-oap-server
docker run --name oap-ui -e SW_OAP_ADDRESS=http://oap:12800 --network docker_default -d -p 8080:8080 apache/skywalking-ui

多行日志
https://www.qikqiak.com/post/collect-multiline-logs/