## Spring Boot Sample – Tomcat JNDI

Spring Boot sample application that demonstrates the configuration and use of
JNDI with embedded Tomcat.

Spring Boot 2.x
https://stackoverflow.com/questions/24941829/how-to-create-jndi-context-in-spring-boot-with-embedded-tomcat-container

Spring Boot 1.5.x
https://github.com/wilkinsona/spring-boot-sample-tomcat-jndi

Credit : https://stackoverflow.com/questions/24941829/how-to-create-jndi-context-in-spring-boot-with-embedded-tomcat-container

Datasource configuration/declaration:
-------------------------------------
You have to customize the bean that creates the TomcatServletWebServerFactory instance.
Two things to do :

1) enabling the JNDI naming which is disabled by default

2) creating and add the JNDI resource(s) in the server context

For example with PostgreSQL and a DBCP 2 datasource, do that :

@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    return new TomcatServletWebServerFactory() {
        @Override
        protected TomcatWebServer getTomcatWebServer(org.apache.catalina.startup.Tomcat tomcat) {
            tomcat.enableNaming(); 
            return super.getTomcatWebServer(tomcat);
        }

        @Override 
        protected void postProcessContext(Context context) {

            // context
            ContextResource resource = new ContextResource();
            resource.setName("jdbc/myJndiResource");
            resource.setType(DataSource.class.getName());
            resource.setProperty("driverClassName", "org.postgresql.Driver");

            resource.setProperty("url", "jdbc:postgresql://hostname:port/dbname");
            resource.setProperty("username", "username");
            resource.setProperty("password", "password");
            context.getNamingResources()
                   .addResource(resource);          
        }
    };
}
Here the variants for Tomcat JDBC and HikariCP datasource.

In postProcessContext() set the factory property as explained early for Tomcat JDBC ds :

    @Override 
    protected void postProcessContext(Context context) {
        ContextResource resource = new ContextResource();       
        //...
        resource.setProperty("factory", "org.apache.tomcat.jdbc.pool.DataSourceFactory");
        //...
        context.getNamingResources()
               .addResource(resource);          
    }
};
and for HikariCP :

    @Override 
    protected void postProcessContext(Context context) {
        ContextResource resource = new ContextResource();       
        //...
        resource.setProperty("factory", "com.zaxxer.hikari.HikariDataSource");
        //...
        context.getNamingResources()
               .addResource(resource);          
    }
};
Using/Injecting the datasource

You should now be able to lookup the JNDI ressource anywhere by using a standard InitialContext instance :

InitialContext initialContext = new InitialContext();
DataSource datasource = (DataSource) initialContext.lookup("java:comp/env/jdbc/myJndiResource");
You can also use JndiObjectFactoryBean of Spring to lookup up the resource :

JndiObjectFactoryBean bean = new JndiObjectFactoryBean();
bean.setJndiName("java:comp/env/jdbc/myJndiResource");
bean.afterPropertiesSet();
DataSource object = (DataSource) bean.getObject();
To take advantage of the DI container you can also make the DataSource a Spring bean :

@Bean(destroyMethod = "")
public DataSource jndiDataSource() throws IllegalArgumentException, NamingException {
    JndiObjectFactoryBean bean = new JndiObjectFactoryBean();
    bean.setJndiName("java:comp/env/jdbc/myJndiResource");
    bean.afterPropertiesSet();
    return (DataSource) bean.getObject();
}
And so you can now inject the DataSource in any Spring beans such as :

@Autowired
private DataSource jndiDataSource;
Note that many examples on the internet seem to disable the lookup of the JNDI resource on startup :

bean.setJndiName("java:comp/env/jdbc/myJndiResource");
bean.setProxyInterface(DataSource.class);
bean.setLookupOnStartup(false);
bean.afterPropertiesSet(); 
But I think that it is helpless as it invokes just after afterPropertiesSet() that does the lookup !


Specifying a datasource available in the classpath at runtime
--------------------------------------------------------------
You have multiple choices :

using the DBCP 2 datasource (you don't want to use DBCP 1 that is outdated and less efficient).
using the Tomcat JDBC datasource.
using any other datasource : for example HikariCP.

You have multiple choices :

using the DBCP 2 datasource (you don't want to use DBCP 1 that is outdated and less efficient).
using the Tomcat JDBC datasource.
using any other datasource : for example HikariCP.
If you don't specify anyone of them, with the default configuration the instantiation of the datasource will throw an exception :

Caused by: javax.naming.NamingException: Could not create resource factory instance
        at org.apache.naming.factory.ResourceFactory.getDefaultFactory(ResourceFactory.java:50)
        at org.apache.naming.factory.FactoryBase.getObjectInstance(FactoryBase.java:90)
        at javax.naming.spi.NamingManager.getObjectInstance(NamingManager.java:321)
        at org.apache.naming.NamingContext.lookup(NamingContext.java:839)
        at org.apache.naming.NamingContext.lookup(NamingContext.java:159)
		
Caused by: java.lang.ClassNotFoundException: org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory
        at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
		
To use Apache JDBC datasource, you don't need to add any dependency but you have to change the default factory class to  org.apache.tomcat.jdbc.pool.DataSourceFactory.

You can do it in the resource declaration :  

resource.setProperty("factory", "org.apache.tomcat.jdbc.pool.DataSourceFactory");

To use DBCP 2 datasource a dependency is required:

<dependency>
  <groupId>org.apache.tomcat</groupId>
  <artifactId>tomcat-dbcp</artifactId>
  <version>8.5.4</version>
</dependency>

Of course, adapt the artifact version according to your Spring Boot Tomcat embedded version.

To use HikariCP, add the required dependency if not already present in your configuration (it may be if you rely on persistence starters of Spring Boot) such as :

<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>3.1.0</version>
</dependency>

resource.setProperty("factory", "com.zaxxer.hikari.HikariDataSource");
