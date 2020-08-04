### **Maven**

```shell
#maven打jar包：
    1： mvn install:install-file -Dfile=sqljdbc4.jar -Dpackaging=jar -DgroupId=com.microsoft.sqlserver -DartifactId=sqljdbc4 -Dversion=4.0
    2： mvn archetype:generate
    

#查看完整的依赖树
mvn dependency:tree
```

