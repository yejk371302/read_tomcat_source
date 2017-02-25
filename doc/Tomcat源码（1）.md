脚本启动过程不再赘述，直接进入启动类。

tomcat的启动类是`org.apache.catalina.startup.Bootstrap`。类中先是执行一个静态代码块，用于设定当前环境的home目录，设定环境变量`catalina.home`和`catalina.base`。

`org.apache.catalina.startup.Bootstrap`类有程序入口方法`main`，静态块执行完成后，便启动入口程序。入口程序中，首先创建了一个`org.apache.catalina.startup.Bootstrap`对象本身，然后调用`org.apache.catalina.startup.Bootstrap`对象的`init`方法进行初始化。

`init`方法如下：
```java
public void init() throws Exception {

        initClassLoaders();

        Thread.currentThread().setContextClassLoader(catalinaLoader);

        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        if (log.isDebugEnabled())
            log.debug("Loading startup class");
        Class<?> startupClass =
            catalinaLoader.loadClass
            ("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.newInstance();

        // Set the shared extensions class loader
        if (log.isDebugEnabled())
            log.debug("Setting startup class properties");
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;

    }
```

`initClassLoaders()`方法是初始化类加载器，加载tomcat所需要的类文件。方法会先创建一个`common.loader`类加载器，加载的路径配置在`${HOME}/conf/catalina.properties`中的`common.loader`中配置。`initClassLoaders()`使用`URLClassLoader`加载配置的jar文件。接着会将`common.loader`作为父加载器，创建`server`和`shared`两个加载器，加载`${HOME}/conf/catalina.properties`中`server.loader`和`shared.loader`中配置的类，并将`server.loader`对应的类加载器作为当前主线程上下文的类加载器和安全类加载器。`SecurityClassLoad.securityClassLoad(catalinaLoader)`方法中在设置安全类加载器时，会加载tomcat核心的类，实现如下：
```java
static void securityClassLoad(ClassLoader loader, boolean requireSecurityManager)
            throws Exception {

        if (requireSecurityManager && System.getSecurityManager() == null) {
            return;
        }

        loadCorePackage(loader);
        loadCoyotePackage(loader);
        loadLoaderPackage(loader);
        loadRealmPackage(loader);
        loadServletsPackage(loader);
        loadSessionPackage(loader);
        loadUtilPackage(loader);
        loadValvesPackage(loader);
        loadJavaxPackage(loader);
        loadConnectorPackage(loader);
        loadTomcatPackage(loader);
    }
```
每个独立的调用方法都是加载一个package下的类，具体可以参考代码实现，不再展开。

剩下的代码逻辑则是使用类加载器加载`org.apache.catalina.startup.Catalina`类，并且创建对象，调用`org.apache.catalina.startup.Catalina`对象的`setParentClassLoader`方法将`shared.loader`赋值给`Catalina`对象。

`init`方法执行完成后，入口程序会解析启动命令传入的参数，执行对应的方法。
```java
try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            } else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null==daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }
```
正常命令启动时，我们一般传入start，比如启动脚本中默认的执行命令如下：
```bash
exec "$PRGDIR"/"$EXECUTABLE" start "$@"
```
传入的参数为`start`，此时会执行
```java
 } else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stop")) {
```
代码逻辑。`daemon`即是当前的`BootStrap`对象，分别执行`BootStrap`对象的`setAwait`方法、`load`方法和`start`方法，对应的实现则都是通过反射调用了`Catalina`对象的相同方法。也就是说`BootStrap`只是一个守护类，实际上的执行类都是`Catalina`，执行完`Catalina`对应的方法，整个Tomcat就启动完毕了。

