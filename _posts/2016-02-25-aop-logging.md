---
layout: post
title: AOP Logging
---

I was trying to use Aspect Oriented based logging method entry, exit logs. A lot of examples I found on web were mostly tightly coupled with Spring, which had its own drawback of configuration through application context and XML files.
The below is implementation using only AspectJ in which weaving happens at compile time.

The Annotation Class.


{% highlight java %}
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.TYPE })
public @interface Loggable {
 int TRACE = 0;
 int DEBUG = 1;
 int INFO = 2;
 int WARN = 3;
 int ERROR = 4;
 int value() default Loggable.TRACE;
 boolean skipResult() default false;
 boolean skipArgs() default false;
}
{% endhighlight %}

This is a simple annotation class which defines parameters like Log level at which logging should be enabled, controlling arguments and return values log messages for sensitive methods.

Maven Configuration

{% highlight xml %}
<dependencies>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.3</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.3</version>
    </dependency>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjrt</artifactId>
        <version>1.8.2</version>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>aspectj-maven-plugin</artifactId>
            <version>1.7</version>
            <configuration>
                <complianceLevel>1.8</complianceLevel>
                <showWeaveInfo>true</showWeaveInfo>
                <source>1.8</source>
                <target>1.8</target>
                <verbose>true</verbose>
            </configuration>
            <executions>
                <execution>
                    <id>weave-classes</id>
                    <phase>process-classes</phase>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>java</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <mainClass>ninjabricks.logger.App</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
{% endhighlight %}
Sample Class

{% highlight java %}
public class App {
 
    @Loggable(value = Loggable.INFO)
    String method1(String s1, String s2) {
        return "Coding Gibberish";
    }
 
    public static void main(String[] args) {
        App sample = new App();
        sample.method1("Ninja", "Bricks");
    }
}
{% endhighlight %}
