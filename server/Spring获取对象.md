
###简介
> 在Spring框架下，有的时候我们希望能够在代码里获取bean对象

###实现方式
####ApplicationContextAware
```
// org.springframework.context.ApplicationContextAware.java
public abstract interface org.springframework.context.ApplicationContextAware 
{
    public abstract void setApplicationContext
        (org.springframework.context.ApplicationContext applicationContext)
        throws org.springframework.beans.BeansException;
}
```
通过ApplicationContextAware可以拿到applicatioContext对象，然后调用applicationContext.getBean(name)就可以拿到对象

####SpringContextUtil
```
//com.test.SpringContextUtil.java
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class SpringContextUtil implements ApplicationContextAware
{
	private static ApplicationContext applicationContext; // Spring应用上下文环境
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException
	{
		SpringContextUtil.applicationContext = applicationContext;
	}
	public static ApplicationContext getApplicationContext()
	{
		return applicationContext;
	}
	@SuppressWarnings("unchecked")
	public static <T> T getBean(String name) throws BeansException
	{
		return (T) applicationContext.getBean(name);
	}
}
```
```
//spring配置文件
<bean id="springContextUtil" class="com.test.SpringContextUtil" scope="singleton" />
```
