



```shell
[root@localhost ~]# docker pull apolloconfig/apollo-portal:latest
[root@localhost ~]# docker pull apolloconfig/apollo-adminservice:latest
[root@localhost ~]# docker pull apolloconfig/apollo-configservice:latest

appllo:
Apollo Config Service
docker run -d -p 8001:8001 \
    -e SPRING_DATASOURCE_URL="jdbc:mysql://192.168.56.10:3306/ApolloConfigDB?characterEncoding=utf8" \
    -e SPRING_DATASOURCE_USERNAME=root \
    -e SPRING_DATASOURCE_PASSWORD=root \
    -e "EUREKA_INSTANCE_PREFERIPADDRESS=true" \
    -e "EUREKA_INSTANCE_IPADDRESS=192.168.56.10" \
    -e "EUREKA_INSTANCE_NONSECUREPORT=8001" \
    -e "EUREKA_INSTANCE_INSTANCEID=192.168.56.10:8001" \
    -v /tmp/logs:/opt/logs  \
    --name apollo-configservice apolloconfig/apollo-configservice
Apollo Admin Service
    docker run -d -p 8002:8002 \
    -e SPRING_DATASOURCE_URL="jdbc:mysql://192.168.56.10:3306/ApolloConfigDB?characterEncoding=utf8" \
    -e SPRING_DATASOURCE_USERNAME=root \
    -e SPRING_DATASOURCE_PASSWORD=root \
    -e "EUREKA_INSTANCE_PREFERIPADDRESS=true" \
    -e "EUREKA_INSTANCE_IPADDRESS=root" \
    -e "EUREKA_INSTANCE_NONSECUREPORT=8002" \
    -e "EUREKA_INSTANCE_INSTANCEID=192.168.56.10:8002" \
    -v /tmp/logs:/opt/logs \
    --name apollo-adminservice apolloconfig/apollo-adminservice
Apollo Portal
    docker run -d -p 8003:8003 \
    -e SPRING_DATASOURCE_URL="jdbc:mysql://192.168.56.10:3306/ApolloPortalDB?characterEncoding=utf8" \
    -e SPRING_DATASOURCE_USERNAME=root \
    -e SPRING_DATASOURCE_PASSWORD=root \
    -e APOLLO_PORTAL_ENVS=pro \
    -e PRO_META=http://192.168.56.10:8001 \
    -v /tmp/logs:/opt/logs \
    --name apollo-portal apolloconfig/apollo-portal
    
    
    查询eureka地址：select * from ServerConfig
```

1. **SPRING_DATASOURCE_URL:** 对应环境ApolloConfigDB的地址
2. **SPRING_DATASOURCE_USERNAME:** 对应环境ApolloConfigDB的用户名
3. **SPRING_DATASOURCE_PASSWORD:** 对应环境ApolloConfigDB的密码
4. **APOLLO_PORTAL_ENVS(可选):** 对应ApolloPortalDB中的apollo.portal.envs配置项，如果没有在数据库中配置的话，可以通过此环境参数配置
5. **DEV_META/PRO_META(可选):** 配置对应环境的Meta Service地址，以${ENV}_META命名，需要注意的是如果配置了ApolloPortalDB中的apollo.portal.meta.servers配置，则以apollo.portal.meta.servers中的配置为准