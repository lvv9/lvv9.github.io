# IntelliJ IDEA
## Plugin
SequenceDiagram 方便看时序图。  

## Project & Moduel
一个项目可以分为多个模块，相关设置存储在.idea目录和.iml文件中。比如用IDEA Rebuild时进行的一些Mark/Unmark操作。  

## Recompile & Build & Rebuild
> You can compile a single file, use the incremental build for a module or a project, and rebuild a project from scratch  

## Maven
将IDEA编译按钮分配给Maven。
> Delegate build and run actions to Maven

## Test
一般运行JUnit单元测试都需要一个Test Runner来运行@test注解的测试方法，不过可以用IDEA自带的Runer进行测试（Maven的叫Surefire，并且即使IDEA勾选了将编译按钮分配给Maven也不会使用）：
> Right-click a test class in the Project tool window or open it in the editor, and right-click the background. From the context menu, select Run <class name> or Debug....
> For a test method, open the class in the editor and right click anywhere in the method. The context menu suggests the command Run / Debug <method name>.
