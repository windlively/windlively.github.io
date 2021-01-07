---
title: jar包执行时访问资源文件造成的FileSystemNotFoundException问题
date: 2021-01-07 22:36:34
tags:
  - 后端开发
  - Java
  - 踩坑记录
---
## 问题描述  
在Java项目中，若需要访问资源文件，通常使用`getClass().getResource("/xxx")`和`getClass().getResourceAsStream("/")`来获取资源文件，后一种方式获取到的是InputStream，一般不会出现什么问题，但是通过第一种获取资源URL的方式，在IDEA中执行时没有问题，但是在服务器环境执行时，会出现`java.nio.file.FileSystemNotFoundException`，之前遇到这个问题我没有在意，只是换成`getResourceAsStream`后就解决了，但是后来遇到一个需要递归访问资源文件目录，我使用的`Files.walkFileTree`方法，直接获取输入流的方案不适合我，随后发现其实只要是打成jar包执行的时候，通过`Files`和`Path`去访问资源文件，`getResource`就会抛出`FileSystemNotFoundException`，并非服务器环境的问题，因为jar包执行时，文件系统与IDE中直接执行时不一样，所以会抛出这个异常。  
原始的代码如下：
```java
Path configDir = Paths.get(
                    LocalConfigDataFlowManager.class.getResource(
                            "/"
                    ).toURI().getPath(), "flow-config"
            );
List<Map<String, Object>> config = new ArrayList<>();
Files.walkFileTree(
        configDir,
        new HashSet<>(),
        1,
        new FileVisitor<Path>() {
        ......
```
报错信息如下：
```log
java.nio.file.FileSystemNotFoundException
        at jdk.zipfs/jdk.nio.zipfs.ZipFileSystemProvider.getFileSystem(ZipFileSystemProvider.java:169)
        at jdk.zipfs/jdk.nio.zipfs.ZipFileSystemProvider.getPath(ZipFileSystemProvider.java:155)
        at java.base/java.nio.file.Path.of(Path.java:208)
        at java.base/java.nio.file.Paths.get(Paths.java:97)
        at ink.andromeda.dataflow.core.flow.ConfigurableDataFlowManager.<init>(ConfigurableDataFlowManager.java:305)
        at ink.andromeda.dataflow.demo.LocalConfigDataFlowManager.<init>(LocalConfigDataFlowManager.java:25)
        at ink.andromeda.dataflow.demo.DataFlowDemoApplication.dataFlowManager(DataFlowDemoApplication.java:51)
        at ink.andromeda.dataflow.demo.DataFlowDemoApplication$$EnhancerBySpringCGLIB$$73b01d37.CGLIB$dataFlowManager$0(<generated>)
        ......
......
2021-01-07.22:34:39.649 ERROR TraceId[] [main] i.a.d.d.LocalConfigDataFlowManager.visitFileFailed:60 null/flow-config
java.nio.file.NoSuchFileException: null/flow-config
        at java.base/sun.nio.fs.UnixException.translateToIOException(UnixException.java:92)
        at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:111)
        at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:116)
        at java.base/sun.nio.fs.UnixFileAttributeViews$Basic.readAttributes(UnixFileAttributeViews.java:55)
        at java.base/sun.nio.fs.UnixFileSystemProvider.readAttributes(UnixFileSystemProvider.java:149)
        at java.base/java.nio.file.Files.readAttributes(Files.java:1763)
        at java.base/java.nio.file.FileTreeWalker.getAttributes(FileTreeWalker.java:219)
        at java.base/java.nio.file.FileTreeWalker.visit(FileTreeWalker.java:276)
        at java.base/java.nio.file.FileTreeWalker.walk(FileTreeWalker.java:322)
        at java.base/java.nio.file.Files.walkFileTree(Files.java:2716)
        at ink.andromeda.dataflow.demo.LocalConfigDataFlowManager.getFlowConfig(LocalConfigDataFlowManager.java:39)
        at ink.andromeda.dataflow.core.flow.ConfigurableDataFlowManager.reload(ConfigurableDataFlowManager.java:92)
        at ink.andromeda.dataflow.core.flow.ConfigurableDataFlowManager.init(ConfigurableDataFlowManager.java:56)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        ......
```
## 解决方案
刚开始试了如下方案：
```java
Path configDir = FileSystems.getDefault().getPath(new UrlResource(this.getClass().getResource("/flow-config").toURI()).toString());
```
但是仍然有问题：
```log
2021-01-07.23:06:42.037 ERROR TraceId[] [main] i.a.d.d.LocalConfigDataFlowManager.visitFileFailed:63 URL [jar:file:/Users/windlively/IdeaProjects/data-flow/data-flow-demo/target/data-flow-demo-1.0-SNAPSHOT.jar!/BOOT-INF/classes!/flow-config]
java.nio.file.NoSuchFileException: URL [jar:file:/Users/windlively/IdeaProjects/data-flow/data-flow-demo/target/data-flow-demo-1.0-SNAPSHOT.jar!/BOOT-INF/classes!/flow-config]
        at java.base/sun.nio.fs.UnixException.translateToIOException(UnixException.java:92)
        at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:111)
        at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:116)
        at java.base/sun.nio.fs.UnixFileAttributeViews$Basic.readAttributes(UnixFileAttributeViews.java:55)
        at java.base/sun.nio.fs.UnixFileSystemProvider.readAttributes(UnixFileSystemProvider.java:149)
        at java.base/java.nio.file.Files.readAttributes(Files.java:1763)
        at java.base/java.nio.file.FileTreeWalker.getAttributes(FileTreeWalker.java:219)
        at java.base/java.nio.file.FileTreeWalker.visit(FileTreeWalker.java:276)
        at java.base/java.nio.file.FileTreeWalker.walk(FileTreeWalker.java:322)
        at java.base/java.nio.file.Files.walkFileTree(Files.java:2716)
        at ink.andromeda.dataflow.demo.LocalConfigDataFlowManager.getFlowConfig(LocalConfigDataFlowManager.java:42)
        at ink.andromeda.dataflow.core.flow.ConfigurableDataFlowManager.reload(ConfigurableDataFlowManager.java:92)
        at ink.andromeda.dataflow.core.flow.ConfigurableDataFlowManager.init(ConfigurableDataFlowManager.java:56)
```
于是继续查询资料，在stackoverflow上看到了一个利用Spring的`@Value()`注解和Spring Resource来访问资源文件的替代方案，而且提供了遍历资源文件夹的方法。  
首先在Bean中加入一个字段并用Value注解注入资源文件的位置：
```java
@Value("classpath:/flow-config/**")
private Resource[] flowConfigResources;
```
Spring会自动将classes根目录`flow-config`文件夹下的资源文件全部注入到`flowConfigResources`字段中去。
然后修改原先的代码如下：
```java
protected List<Map<String, Object>> getFlowConfig() {
    return Stream.of(flowConfigResources)
            .filter(s -> Objects.nonNull(s.getFilename()))
            .filter(s -> s.getFilename().matches("^sync-config-[\\w-]+?.json$"))
            .map(flowConfigResource -> {
                try (InputStream is = flowConfigResource.getInputStream()) {
                    ByteArrayOutputStream os = new ByteArrayOutputStream();
                    int data;
                    while ((data = is.read()) != -1) os.write(data);
                    //noinspection unchecked
                    return (Map<String, Object>) GSON().fromJson(os.toString(StandardCharsets.UTF_8.name()),
                            new TypeToken<Map<String, Object>>() {
                            }.getType());

                } catch (IOException ex) {
                    throw new IllegalStateException(ex);
                }
            })
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
}
```
过滤和遍历Spring注入的Resource列表，即可读取到此文件夹下所有需要的资源文件，打成jar包之后也可正常运行。