# mybatis

DataSource

SqlSessionFactory

Q1：spring内置的SqlSessionFactoryBean可以接受一个DataSource创建SqlSessionFactory，而SqlSessionFactoryBuilder不可以，为什么呢？

SqlSessionFactoryBean is a Spring-specific class that integrates MyBatis with Spring's dependency injection and configuration management. It is designed to work seamlessly with Spring's DataSource and other beans, allowing you to configure MyBatis using Spring's configuration mechanisms.  

On the other hand, SqlSessionFactoryBuilder is a MyBatis class that is designed to build SqlSessionFactory instances using MyBatis' own configuration mechanisms, typically through XML configuration files or programmatic configuration using the Configuration class. It does not directly accept a DataSource because it expects a more comprehensive configuration setup, including transaction management and environment settings, which are encapsulated in the Configuration and Environment classes.

This design allows SqlSessionFactoryBean to leverage Spring's powerful configuration and dependency injection capabilities, while SqlSessionFactoryBuilder remains a more flexible and standalone way to configure MyBatis.

简单点来说，spring的SqlSessionFactoryBean简化了SqlSessionFactory的配置，在Bean中实现了SqlSessionFactory的装配（也是使用SqlSessionFactoryBuilder）。而SqlSessionFactoryBuilder就是mybatis原生的方式来构建SqlSessionFactory的，更灵活当然依赖的管理配置和环境在Configuration 和Environment中。

[mybatis生成数据库ID](generate-db-id.md)