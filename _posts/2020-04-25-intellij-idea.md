# IntelliJ IDEA

## Plugin
SequenceDiagram 方便看时序图。<br>
Maven Helper 查看Maven工程依赖情况。

## Project & Module
一个项目可以分为多个模块，相关设置存储在.idea目录和.iml文件中。比如用资源的Mark/Unmark操作。

## Recompile & Build & Rebuild
> You can compile a single file, use the incremental build for a module or a project, and rebuild a project from scratch

## Maven
将IDEA编译按钮分配给Maven。
> Delegate build and run actions to Maven

### Module
IDEA对Maven模块有较好的支持，添加模块后会自动对POM进行处理。<br>
关于Maven的“模块”，可以理解为继承和组合，即大工程可以是小工程的父工程，也可以是小工程的组合（聚合），或者都是。<br>
可以参考：https://my.oschina.net/mzdbxqh/blog/846018

### Build
Maven挺复杂的，但IDEA对Maven的构建也有一定程度的支持，如支持资源filter。<br>
关于Maven Resources Filtering，见：https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html

## Test
一般运行JUnit单元测试都需要一个Test Runner来运行@test注解的测试方法，不过可以用IDEA自带的Runner进行测试（Maven的叫Surefire，并且即使IDEA勾选了将编译按钮分配给Maven也不会使用）：
> Right-click a test class in the Project tool window or open it in the editor, and right-click the background. From the context menu, select Run <class name> or Debug....
> For a test method, open the class in the editor and right click anywhere in the method. The context menu suggests the command Run / Debug <method name>.
