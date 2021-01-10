
# SpringBoot笔记

# 自动配置原理

核心在于JavaConfig Spring bean Java配置类,ConfigurationProperties 绑定全局配置到Spring bean和@Condition注解

## 要点

1. 使用 @EnableConfigurationProperties 将全局配置文件中的配置和某个 被 @ConfigurationProperties 修饰的 xxxProperties 配置类绑定，并将这个配置类作为 bean 导入容器。
2. @SpringBootApplication 上的 @EnableAutoConfiguration 会使用 @Import 导入 AutoConfigurationImportSelector ，这个类中使用 SpringFactoryLoader 机制（类似JavaSPI）将ClassPath 下所有jar包中 META-INF/spring.factory 文件中配置的 EnableAutoConfiguration 的值中记录的 类全路径名对应的 xxxAutoConfiguration配置 类加载到容器
3. @EnableAutoConfiguration 加载的配置类依据@Condition条件注解要求的条件生效，可以生效的配置类会通过容器中被@EnableConfigurationProperties导入的配置文件映射bean中得到配置文件中的配置
