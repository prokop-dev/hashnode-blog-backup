## Nailing coffin of Application Server - 2PC / XA transactions with Spring Boot

Note: that article refers to Spring Boot 1.1.x, the XA support was embedded in Spring Boot 1.2.x, what I will cover another time.

Recently I had to write an app that synchronizes queues with databases. Pretty dumb application that is far too simple to use any overkill technology. However, for full piece of mind I needed XA transactions between JMS queues and database. Here is how to enable XA for Spring Boot.

1. Transaction manager. You need transaction manager. I've chosen the Bitronix JTA Transaction Manager. Open source and free. The role of transaction manager is to track the state of distributed transaction on bulletproof way. Note that for it uses synchronized log files that are keep locally (here I've put them under 'run' directory). In your Gradle build script you need the following dependency:

```compile("org.codehaus.btm:btm:2.1.4")```

2. Enabling transactions support and configuring JTA ransaction manager. It can be easily one by adding @Configuration annotated class.

```
package example;

import bitronix.tm.BitronixTransactionManager;
import bitronix.tm.TransactionManagerServices;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.jta.JtaTransactionManager;

@Configuration
@EnableTransactionManagement
public class BitronixConfiguration {

    @Bean
    public bitronix.tm.Configuration btmConfig() {
        System.out.println("btmConfig()");
        bitronix.tm.Configuration configuration = TransactionManagerServices.getConfiguration();
        configuration.setServerId("spring-btm");
        configuration.setLogPart1Filename("./run/btm-log1");
        configuration.setLogPart2Filename("./run/btm-log2");
        return configuration;
    }

    @Bean(destroyMethod = "shutdown")
    @DependsOn(value = "btmConfig")
    public BitronixTransactionManager bitronixTransactionManager() {
        System.out.println("bitronixTransactionManager()");
        return TransactionManagerServices.getTransactionManager();
    }

    @Bean
    public JtaTransactionManager transactionManager() {
        System.out.println("jtaTransactionManager()");
        JtaTransactionManager retVal = new JtaTransactionManager();
        retVal.setTransactionManager(bitronixTransactionManager());
        retVal.setUserTransaction(bitronixTransactionManager());
        return retVal;
    }

}
```

Note the @EnableTransactionManagement annotation.

3. Configure XA compatible resource. Example for JMS queue with IBM MQ Server:

```
package example;

import bitronix.tm.resource.jms.PoolingConnectionFactory;
import com.ibm.msg.client.wmq.WMQConstants;
import javax.jms.ConnectionFactory;
import javax.jms.JMSException;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class WebSphereMQConfig {

    @Bean(initMethod = "init", destroyMethod = "close")
    public PoolingConnectionFactory mqConnectionfactory() throws JMSException {
        System.out.println("mqConnectionfactory()");
        PoolingConnectionFactory factory = new PoolingConnectionFactory();
        factory.setClassName("com.ibm.mq.jms.MQXAConnectionFactory");
        factory.setUniqueName("wmq");
        factory.setMaxPoolSize(3);
        factory.setUser("user");
        factory.setPassword("pass");
        factory.getDriverProperties().setProperty("hostName", "192.168.1.1");
        factory.getDriverProperties().setProperty("port", "1414");
        factory.getDriverProperties().setProperty("transportType", WMQConstants.WMQ_CM_CLIENT + "");
        factory.getDriverProperties().setProperty("queueManager", "QM1");        
        factory.getDriverProperties().setProperty("channel", "CN1");        
        factory.setAllowLocalTransactions(false);
        return factory;
    }

}
```

Note the use of pooling connection factory.

4. Configure XA compatible resource. Example for JDBC datasouce with Oracle:

```
package example;

import bitronix.tm.resource.jdbc.PoolingDataSource;
import javax.sql.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DatasourceConfigXA {

    @Bean(initMethod = "init", destroyMethod = "close")
    public DataSource dataSource() {
        System.out.println("dataSource()");
        PoolingDataSource poolingDataSource = new PoolingDataSource();
        poolingDataSource.setClassName("oracle.jdbc.xa.client.OracleXADataSource");
        poolingDataSource.setUniqueName("oracle");
        poolingDataSource.setMaxPoolSize(5);
        poolingDataSource.getDriverProperties().setProperty("URL", "jdbc:...");
        poolingDataSource.getDriverProperties().setProperty("user", "USR");
        poolingDataSource.getDriverProperties().setProperty("password", "PWD");
        poolingDataSource.setAllowLocalTransactions(false);
        System.out.println(poolingDataSource.getDriverProperties());
        return poolingDataSource;
    }

}
```

5. Configure MySQL in XA mode:

```
package example;

import bitronix.tm.resource.jdbc.PoolingDataSource;
import com.mysql.jdbc.jdbc2.optional.MysqlXADataSource;
import javax.sql.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DatasourceConfigXA {

    @Bean(initMethod = "init", destroyMethod = "close")
    public PoolingDataSource dataSource1() {
        PoolingDataSource poolingDataSource = new PoolingDataSource();
        poolingDataSource.setClassName("com.mysql.jdbc.jdbc2.optional.MysqlXADataSource");
        poolingDataSource.setUniqueName("ds1");
        poolingDataSource.setMaxPoolSize(5);
        poolingDataSource.getDriverProperties().setProperty("URL", "jdbc:mysql://10.176.7.39/db1");
        poolingDataSource.getDriverProperties().setProperty("user", "monty");
        poolingDataSource.getDriverProperties().setProperty("password", "some_pass");
        poolingDataSource.setAllowLocalTransactions(false);
        return poolingDataSource;
    }
}
```

That is all. So little code, so powerful Spring Boot, no Application Server necessary.