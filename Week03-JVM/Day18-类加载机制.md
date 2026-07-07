## Day 18（周四）— 类加载机制

### 📖 理论学习（3h）
- [ ] 类的生命周期（加载→验证→准备→解析→初始化→使用→卸载）
- [ ] 加载阶段：通过全限定名获取二进制字节流
- [ ] 验证阶段：文件格式、元数据、字节码、符号引用验证
- [ ] 准备阶段：静态变量分配内存并设置零值
- [ ] 解析阶段：符号引用→直接引用
- [ ] 初始化阶段：执行\<clinit\>类构造器
- [ ] 双亲委派模型（Bootstrap→Extension→Application ClassLoader）
- [ ] 打破双亲委派的方式（SPI、Tomcat、OSGi）

### 🎯 验收标准
- [ ] 能画出类加载器的层次结构图
- [ ] 能解释双亲委派的工作流程和好处
- [ ] 能说明SPI如何打破双亲委派（线程上下文类加载器）
- [ ] 能解释Tomcat为什么要打破双亲委派（web应用隔离）

### 💻 LeetCode
- [ ] [200. 岛屿数量](https://leetcode.cn/problems/number-of-islands/)（Medium）— DFS/BFS
- [ ] [207. 课程表](https://leetcode.cn/problems/course-schedule/)（Medium）— 拓扑排序

### 📝 学习笔记
```
在此记录学习笔记...
```