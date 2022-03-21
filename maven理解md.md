## maven简介

>maven 翻译为“专家”，“内行”，基于java项目平台构建的大型开源项目

- 优秀的构建工具
- maven 是跨平台

## 坐标

提供正确的坐标元素，找到对应的构件

- groupId: 定义当前Maven隶属的实际项目 【必须】
- artifactId: 该元素定义实际项目中的一个Maven项目（模块） 默认最后打包成型的名称会以artifactId开头 【必须】
- version: 定义Maven项目所处的版本 【必须】
- packaging: 定义maven项目的打包方式 【可选】
- classifier: 定义构建输出的附属构件 【不能直接定义】

默认构建最终文件名：artifactId - version [-classifier].packaging

## 依赖配置 dependency 标签

- groupId  artifactIdversion 基本坐标
- type: 默认的类型，对应于项目定义的packaging ，大部分情况下，不比声明，默认值为jar
- scope: 依赖范围
- optional: 标记依赖是否可选
- exclusions: 排除传递性依赖 只需要 groupId  和  artifactId

### scope 依赖范围

依赖范围： 控制依赖与三种路径的关系（编译classpath, 测试classpath, 运行的classpath)

可选值

- compile: 编译依赖范围，**默认**，依赖对于上面三种路径均有效
- test: 只对于测试classpath 有效
- provided: 已提供依赖范围，编译classpath, 测试classpath, 都有效，运行时无效，容器已经提供
- runtime: 运行时依赖范围，测试和运行有效，编译主代码无效
- system: 系统依赖范围，该依赖与provided依赖范围一致，但是使用system必须通过systemPath显示指定依赖文件的路径
- import: 导入依赖范围，该依赖范围对于三种classpath不会产生实际的影响

### 传递性依赖

必要的间接依赖以传递依赖引入到当前项目

当产生依赖问题，依赖调节第一原则，路径近者优先

## 生命周期

对所有项目的构建过程进行抽象和统一，项目的构建过程映射到一个个生命周期上，每个生命周期的任务实际上是由**插件**来完成。

Maven有三套相互独立的生命周期

- clean: 清理项目
- default: 构建项目
- site: 建立项目站点

每一个生命阶段包含一些 phase(**阶段**)，阶段有序，后面的阶段依赖前面的阶段

### clean 生命周期：

- preClean 执行清理完成之前的动作
- clean 清理上一次构建完成的文件
- post-clean 执行一些清理后的动作

### default生命周期：

- validate
- initialize
- generate-sources
- process-sources
- generate-resources
- process-resources
- compile 
- process-classes
- generate-test-sources
- process-test-sources
- generaget-test-resources
- process-test-resources
- test-compile
- process-test-classes
- test
- prepare-package
- package
- pre-intergation-test
- intergation-test
- post-intergation-test
- verify
- install
- deploy

### site 生命周期

- pre-site
- site
- post-site
- site-deploy

### 插件

maven命令背后的执行是交给插件执行的，有多插件的目标在编写时候已经默认了绑定阶段

插件网址：https://maven.apache.org/plugins/index.html

## 依赖管理

Maven项目父子之间是可以继承的，但是不确保将来所有的子项目都会用到父项目依赖，这个问题可以使用`<dependencyManagement>`元素标签，只是依赖生命但不会实际引入，子项目需要引入只需要声明**groupId** 和**artifactId**

## 插件管理

前者有`dependencyManagement`，后者有`pluginManagement`可以声明父项目当中，子项目引入声明**groupId** 和**artifactId**

## Maven版本号的约定

主版本-次版本-增量版本-里程碑版本

## Maven属性

### 内置属性

- ${baseDir}：项目根目录
- ${version}：项目版本

### pom属性

- ${project.build.sourceDirctory} : 项目的主源码目录，默认src/main/java
- ${project.build.testSourceDirctory} : 项目的测试源码目录，默认src/test//java
- ${project.build.directory} : 项目的构建输出目录，默认target/
- ${project.build.outputDirctory} : 项目的主代码编译输出目录，默认target/classes/
- ${project.testOutputDirectory}:项目测试代码编译输出，默认target/test-classes/
- ${project.groupId} ：项目的groupId
- ${project.artifactId} ：项目的artifactId
- ${project.version} ：项目的version
- ${project.build.finalName}:项目打包输出的文件名称，默认为${project.artifactId} - ${project.version}

### 自定义属性

`<properties>`标签下面的元素定义

### Settings属性

### Java系统属性

### 环境变量属性

## profile文件激活

### 命令行激活

`mvn clean install -P dev`

### setting文件激活

```xml
<settings>
....
    <activeProfiles>
    	<activeProfile>dev</activeProfile>
    </activeProfiles>
....
</settings>
```

### 系统属性激活

```xml
<profiles>
	<profile>
    	<activation>
        	<name>dev</name>
        </activation>
    </profile>
</profiles>
```

### 操作系统环境激活

```xml
<profiles>
	<profile>
        <activation>
            <name>Window xp</name>
            <family>Windows</family>
            <arch>x86</arch>
        </activation>
        <version>6.1.1220</version>
    </profile>
</profiles>
```

### 文件存在是否激活

```xml
<profiles>
	<profile>
    	<activation>
        	<file>
            	<missing>x.properties</missing>
       			<exists>y.properties</exists>
            </file>
        </activation>
    </profile>
</profiles>
```

### 默认激活

```xml
<profiles>
	<profile>
        <id>dev</id>
    	<activation>
        	<activeByDefault>true</activeByDefault>
        </activation>
    </profile>
</profiles>
```

## 编写Maven插件

### 一般步骤

1. 创建一个maven-plugin项目，插件本身就是一个maven项目，规定packaging 必须是maven-plugin
2. 为插件编写目标
3. 为目标提供配置点
4. 编写代码实现目标行为
5. 错误处理及日志
6. 测试插件

### 代码行统计Maven插件案例

1. 创建项目使用maven-archetype-plugin骨架

   `mvn archetype:generate`

   ![image-20220321074733392](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220321074733392.png)

   **选择3**

   **pom.xml**

   ```xml
    <groupId>com.test</groupId>
     <artifactId>hs-plugin</artifactId>
     <version>v1.0</version>
     <packaging>maven-plugin</packaging>
   
     <name>hs-plugin Maven Plugin</name>
   
     <!-- FIXME change it to the project's website -->
     <url>http://maven.apache.org</url>
   
     <properties>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
     </properties>
   
     <dependencies>
       <dependency>
         <groupId>org.apache.maven</groupId>
         <artifactId>maven-plugin-api</artifactId>
         <version>3.5.4</version>
       </dependency>
       <!--注解方式-->
       <dependency>
         <groupId>org.apache.maven.plugin-tools</groupId>
         <artifactId>maven-plugin-annotations</artifactId>
         <version>3.5</version>
         <scope>provided</scope>
       </dependency>
     </dependencies>
   
   
   ```

   **code**

   ```java
   import org.apache.maven.model.Resource;
   import org.apache.maven.plugin.AbstractMojo;
   import org.apache.maven.plugins.annotations.LifecyclePhase;
   import org.apache.maven.plugins.annotations.Mojo;
   import org.apache.maven.plugins.annotations.Parameter;
   
   import java.io.BufferedReader;
   import java.io.File;
   import java.io.FileReader;
   import java.io.IOException;
   import java.util.ArrayList;
   import java.util.List;
   
   @Mojo( name = "count", defaultPhase = LifecyclePhase.INSTALL )
   public class CountMojo extends AbstractMojo {
   
       private static final String[] INCLUDES_DEFAULT = {"java", "xml", "properties"};
   
       @Parameter(defaultValue = "${project.basedir}",required = true, readonly = true)
       private File baseDir;
       @Parameter(defaultValue = "${project.build.sourceDirectory}", required = true, readonly = true)
       private File sourceDirectory;
       @Parameter(defaultValue = "${project.build.testSourceDirectory}", required = true, readonly = true)
       private File testSourceDirectory;
       @Parameter(defaultValue = "${project.build.resources}", required = true, readonly = true)
       private List<Resource> resources;
       @Parameter(defaultValue = "${project.build.testResources}", required = true, readonly = true)
       private List<Resource> testResources;
       @Parameter
       private String[] includes;
   
   
       public void execute()  {
           if (includes == null || includes.length == 0){
               includes = INCLUDES_DEFAULT;
           }
           try {
               countDir(sourceDirectory);
               countDir(testSourceDirectory);
               for (Resource resource : resources) {
                   countDir(new File(resource.getDirectory()));
               }
               for (Resource resource : testResources) {
                   countDir(new File(resource.getDirectory()));
               }
           } catch (IOException e) {
               e.printStackTrace();
           }
       }
   
       private void countDir(File dir) throws IOException {
           if(!dir.exists()){
               return;
           }
           ArrayList<File> collected = new ArrayList<File>();
           collectFiles(collected, dir);
           int lines = 0;
           for (File file : collected) {
               lines += countLine(file);
           }
           String path = dir.getAbsolutePath().substring(baseDir.getAbsolutePath().length());
           getLog().info(path+":"+lines+"lines of code in" + collected.size() +" files");
       }
       private void collectFiles(List<File> collected,File file){
           if(file.isFile()){
               for (String include : includes) {
                   if (file.getName().endsWith("." + include)) {
                       collected.add(file);
                       break;
                   }
               }
           } else {
               for (File sub : file.listFiles()) {
                   collectFiles(collected,sub);
               }
           }
       }
       private int countLine(File file) throws IOException {
           BufferedReader br = new BufferedReader(new FileReader(file));
           int line = 0;
           try {
               while (br.ready()){
                       br.readLine();
                       line++;
               }
           } finally {
               br.close();
           }
           return line;
       }
   }
   ```

   `mvn clean install`

   其他项目引入

   ```xml
   <plugin>
       <groupId>com.test</groupId>
       <artifactId>hs-plugin</artifactId>
       <version>v1.0</version>
   </plugin>
   ```

   `mvn com.test:hs-plugin:v1.0:count`

## 命令

查看当前项目已经解析的依赖： `mvn dependency:list`

查看当前项目的依赖树：`mvn dependency:tree`

分析当前项目的依赖： `mvn dependency:analyze`

`mvn clean package -P dev`

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
            <resource>
                <directory>src/main/sql</directory>
                <filtering>false</filtering>
            </resource>
        </resources>
    </configuration>
</plugin>
```

