#  调试 Quarkus 生成的增强类字节码

1. 输出增强类字节码

   可以在插件 goals 中添加 generate-code generate-code-tests ，也可以手动执行，会生成 generated-bytecode.jar，里面包含所有增强类的字节码文件。

   ```
   <plugin> 
       <groupId>${quarkus.platform.group-id}</groupId>
       <artifactId>quarkus-maven-plugin</artifactId>
       <version>${quarkus.platform.version}</version>
       <extensions>true</extensions> 
       <executions>
           <execution>
               <goals>
                   <goal>build</goal>
                   <goal>generate-code</goal>
                   <goal>generate-code-tests</goal>
               </goals>
           </execution>
       </executions>
   </plugin>
   ```

2. Idea 中关联这个 jar

   File -> Project Structure -> Modules -> Dependencies 中添加这个 jar 包。

   之后调试可以跳转到生成的增强类里面。

