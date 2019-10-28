### 1. BeanFactory

`BeanFactory` 是一个接口，其定义了一个 `容器` 所具备的接口

+ getBean
+ containsBean
+ isSingleton
+ isPrototype
+ isTypeMatch
+ getType

意味着 `BeanFactory` 实现了 `Dependent Injection`，比较常见的用法

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
		new XmlBeanDefinitionReader(factory).loadBeanDefinitions(WITH_AUTOWIRING_CONTEXT);
final Object bean = factory.getBean("TestFactoryBean");
```

这个工厂的源码解析在上一层目录 `beans`中，在这不多讨论。

### 2. FactoryBean

其实`BeanFactory` 没什么好讨论的，这个接口基本贯穿了整个`beans` 组件，唯一比较容易搞混的就是`FactoryBean`

先概览一下其接口:

```java
public interface FactoryBean<T> {
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

	@Nullable
	T getObject() throws Exception;

	@Nullable
	Class<?> getObjectType();

	default boolean isSingleton() {
		return true;
	}
}

```

这玩意和我们用的什么 `sqlSessionFactory`、`LoggerFactory` 都有着相同的思想，就是一个用于生产对象的工厂类。

`BeanFactory` 是一个容器，而 `FactoryBean` 则是一个实实在在的 `Bean` ，但其也有不普通的地方，也就是其的作用只是用来生产 `Bean`。下面写一个测试类 :

首先是实现这个接口

```java
public class TestFactoryBean implements FactoryBean<Object> {

	@Override
	public Object getObject() throws Exception {
		return new Integer(24234);
	}

	@Override
	public Class<?> getObjectType() {
		return Integer.class;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
}
```

然后定义一下

```xml
<bean id="TestFactoryBean" class="org.springframework.tests.beans.TestFactoryBean">

	</bean>
```

最后调用

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
		new XmlBeanDefinitionReader(factory).loadBeanDefinitions(WITH_AUTOWIRING_CONTEXT);
		final Object bean = factory.getBean("TestFactoryBean");
```

输出：

```bash
bean = 24234
```

这很容易能看出 `FactoryBean` 的作用，当我们调用 `getBean`的时候，其会实例化 `TestFactoryBean` 对象，然后调用 `getObject` 方法拿到`Bean` 最后返回给我们。

如果`TestFactoryBean` 是一般的 `Bean` 也就是没有实现 `FactoryBean` 接口，那么我们调用上述的 `getBean` 则直接会返回 `TestFactoryBean` 的实例对象 

由此我们可以得出一个结论：

```
FactoryBean是一个工厂 Bean，spring会自动调用其 getObject方法拿到最终的 Bean
```

---

所以下面我们从源码层面剖析以及扩展涉及到的知识.

```java
if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// [6] 创建Bean
							/** {@link AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])}*/
							return createBean(beanName, mbd, args);
						}
					});

					// 在这里判断是不是FactoryBean 如果是则调用其 getObject 方法
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
```

`createBean` 不是这一小节关键的地方，其作用是负责创建 `Instance`并且进行 `DI`，下面的 `getObjectForbeanInstance` 才是解密`FactoryBean` 的关键

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		/**---------------------------------------------------------------------------------------------------
		 * [DESC] 处理 FactoryBean
		 * [1] 首先，判断是根据 type 还是 name来 getBean
		 * [2] 其次其是不是 FactoryBean
		 * [3] 如果是根据 type 则直接返回其本身，无论是普通bean还是FactoryBean
		 * [4] 如果是根据名字，当是FactoryBean则调用其getObject方法
		 *----------------------------------------------------------------------------------------------------*/

		// [1] 如果是根据类型，则 name 则带有一个 & 字符
		// 如果是根据名字则没有
		// Note 当然你可以在名字上自己一个 & 符号，但这并不规范，容易让人混淆
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// 执行到这里则说明是根据名字来获取

		// 如果不是 FactoryBean 则直接返回，不需要处理
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		// 老规矩从缓存中获取
		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());

			// 缓存没有，则继续获取 NOTE step in
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

如果说源码看多了就会发现这样一个规律就是:

```
实际上很简单的东西，习惯性分为两个或者多个函数，前者负责参数检查，或者预处理，后者才是真正实现的方法
```

总结上面的功能：

+ 如果是 byType 则 `name` 上则会带有 `&` 字符，这个时候它就不处理了，直接返回 `Instance `
+ 如果是 byName 则 `name ` 上没有 `&` 字符，如果为 `FactoryBean` 则进行处理，否则直接返回 `Instance `

这就很有意思了，如果你通过 `byType` 你只能拿到 `FactoryBean` 的实例，你也可以通过 `byName` 并且在 `name` 前面加一个 `&` 也可以拿到 `FactoryBean` 对象，但这貌似不符合规范，哈哈，不过在这里不推荐这样搞 . 否则就失去 `FactoryBean` 的意义了

那么 继续跟。

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 确保在工厂中已经生成了该 bean
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				// 尝试从缓存中拿到工厂bean生成的对象
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					// 缓存中拿不到, 则调用  getObject 方法
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
								// 调用 BeanProcessor#postProcessAfterInitialization
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						// 丢入缓存
						if (containsSingleton(beanName)) {
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			// 这里是处理 prototype 类型的 bean，所以涉及不到缓存，也就是直接调用 getObject方法
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```

总结下 ：

+ 如果为 `singleton` 则将 `getObject` 得到的对象放入缓存
+ 如果为 `prototype` 则无需将得到的对象放入缓存
+ 在得到对象后都会调用 处理器 `BeanProcessor#postProcessAfterInitialization`

`doGetObjectFromFactoryBean` 这个方法很简单就是调用 `getObject` 方法

```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 直接调用 getObject 方法
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}
```

到此我们就知道了 `FactoryBean` 是个什么东西了，并且还知道其功能是什么，实际上也就和一开始所说的，它是一个工厂对象，用于生产对象的工厂`Bean` ，是个比较特殊的`Bean` 。

实际上在 `spring` 中定义一个属于我们自己的 `工厂bean` 也有其它更灵活的方法，也就是利用标签

```xml
<bean id="Service" class="org.springframework.beans.factory.FactoryBeanTests$Service" factory-bean="ServiceFactoryBean" factory-method="getObject">
	</bean>
	<bean id="ServiceFactoryBean" class="org.springframework.beans.factory.FactoryBeanTests$ServiceFactoryBean" >
	</bean>
```

上面这个配置文件可以看到 `ServiceFactoryBean` 是一个用于生产 `Service ` 的工厂, 则我们只需要在 `Service` 的定义中指定 `Factory-bean` 和 `factory-method `即可

 上述这个过程会在 实例化 `Instance ` 的过程中调用，而 `FactoryBean`则是在 拿到`BeanFactory 的 Instance `后才进行处理. 

关于上面这个例子，留作一个单独的小节进行分析.