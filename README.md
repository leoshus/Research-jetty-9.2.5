###Research-jetty-9.2.5
====================

Research the source code about jetty 9.2.5
---------------
2014/12/05
---------------
* jetty-start->org.eclipse.jetty.start.Main 启动入口
Main的main方法首先会实例化一个自身的对象,然后开始对命令行参数开始解析 进而启动jetty
```
StartArgs startArgs = main.processCommandLine(args);//解释命令行参数 homeDir baseDir
main.start(startArgs);//启动jetty
```
启动jetty之前在Main.start()方法中，首先会设置java.class.path，接着展示jetty usage information、版本信息、jetty配置信息、列表modules等等
最终都会转换成Main.invikeMain()方法中的反射一个Main方法来启动内嵌的jetty Server。
```
public void invokeMain(ClassLoader classloader, StartArgs args) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException, ClassNotFoundException, IOException
    {
        Class<?> invoked_class = null;
        String mainclass = args.getMainClassname();

        try
        {
            invoked_class = classloader.loadClass(mainclass);
        }
        catch (ClassNotFoundException e)
        {
            System.out.println("WARNING: Nothing to start, exiting ...");
            StartLog.debug(e);
            usageExit(ERR_INVOKE_MAIN);
            return;
        }

        StartLog.debug("%s - %s",invoked_class,invoked_class.getPackage().getImplementationVersion());

        CommandLineBuilder cmd = args.getMainArgs(baseHome,false);
        String argArray[] = cmd.getArgs().toArray(new String[0]);
        StartLog.debug("Command Line Args: %s",cmd.toString());

        Class<?>[] method_param_types = new Class[]
        { argArray.getClass() };

        Method main = invoked_class.getDeclaredMethod("main",method_param_types);
        Object[] method_params = new Object[] { argArray };
        StartLog.endStartLog();
        main.invoke(null,method_params);
    }
```

类似于以下的内嵌类。
```
public class Main {
	public static void main(String[] args) throws Exception {

		String contextPath = "/terrace-prototype";
		int port = Integer.getInteger("port", 8080);

		Server server = createServer(contextPath, port);

		try {
			server.start();
			server.join();
		} catch (Exception e) {
			e.printStackTrace();
			System.exit(100);
		}
	}

	private static Server createServer(String contextPath, int port) {
		// use Eclipse JDT compiler
		System.setProperty("org.apache.jasper.compiler.disablejsr199", "true");

		Server server = new Server(port);
		server.setStopAtShutdown(true);

		ProtectionDomain protectionDomain = Main.class.getProtectionDomain();
		URL location = protectionDomain.getCodeSource().getLocation();
		String warFile = location.toExternalForm();

		WebAppContext context = new WebAppContext(warFile, contextPath);
		context.setServer(server);

		// 设置work dir,war包将解压到该目录，jsp编译后的文件也将放入其中。
		String currentDir = new File(location.getPath()).getParent();
		File workDir = new File(currentDir, "work");
		context.setTempDirectory(workDir);

		server.setHandler(context);
		return server;
	}
}
```

首先我们来看看Server(int)这个构造器：
```
public Server(@Name("port")int port)
    {
        this((ThreadPool)null);
        ServerConnector connector=new ServerConnector(this);
        connector.setPort(port);
        setConnectors(new Connector[]{connector});
    }
```
在这个构造器里，将Server本身传给了ServerConnector来构造对象，并设置了ServerConnector的port属性，并且将ServerConnector对象引用放在了Server自身的一个_connectors集合中。
```
private final List<Connector> _connectors = new CopyOnWriteArrayList<>();
```
这里的这个集合使用了CopyOnWriteArrayList集合，顾名思义，读操作时不做同步操作，但是当写操作的时候会把将要修改的内容备份一份再进行修改,这样提高了并发量的同时也降低了写的性能。


