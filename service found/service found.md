# ServiceLoader服务发现机制[Dubbo服务发现机制基础]

内部定义服务文件路径前缀private static final String PREFIX = "META-INF/services/";

使用步骤

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、编写服务接口

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、编写服务实现类

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、在META-INF/services文件夹下创建服务文件,名称为服务接口全限定名,内容为服务实现类的全限定名[多个直接隔行]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4、通过ServiceLoader.load()并且遍历ServiceLoader[实现了Iterator接口]就可以获取到所有的服务具体实现

原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过扫描META-INF/services下以服务接口全限定名为文件名的文件获取当前服务接口的服务实现全限定名,根据全限定名通过反射的机制根据全限定名创建对象实例并进行强制转换返回[主要的流程在内部LazyIterator中]

hasNextService():判断是否还存在服务实现
```
private boolean hasNextService() {

    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            // 获取文件的全限定名
            String fullName = PREFIX + service.getName();
            if (loader == null){
                // 使用应用程序类加载器进行加载,加载指定的文件读取其中内容
                configs = ClassLoader.getSystemResources(fullName);
            } else {
                configs = loader.getResources(fullName);
            }
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
                return false;
        }
        // 解析资源[就是将文件里的内容全部解析出来],读出来的是服务实现的全限定名
        pending = parse(service, configs.nextElement());
    }
    // 将服务实现名记录在内部
    nextName = pending.next();
    return true;
}
```
nextService():获取服务实例
```
private S nextService() {

    if (!hasNextService()){
        throw new NoSuchElementException();
    }
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        // 通过反射创建出类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,"Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,"Provider " + cn  + " not a subtype");
    }
    try {
        // 强制类型转换[此时已经生成了实例]
        S p = service.cast(c.newInstance());
        // 存放入内部属性
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,"Provider " + cn + " could not be instantiated",x);
    }
    throw new Error();
}
```
优点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;解耦、解依赖

缺点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、如果新增或者删除服务则需要更改文件,此缺点有解决办法[google的auto开源]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、无法根据参数动态的获取实现,而是只能获取到所有的实现并且生成对象实例,如果某个实现不需要则会浪费资源

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、不是单例,相同的类型就会创建两个ServiceLoader对象浪费资源