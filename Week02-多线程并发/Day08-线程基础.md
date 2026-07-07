# Week 2：多线程与并发

## 目标
- 掌握线程创建方式和线程状态
- 理解synchronized底层原理和锁升级
- 掌握volatile、CAS、ReentrantLock
- 理解线程池原理和JUC并发包

---

## Day 8（周一）— 线程基础

### 📖 理论学习（3h）
- [ ] 线程创建方式（Thread、Runnable、Callable、线程池）
- [ ] 线程状态（NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED）
- [ ] start() vs run()区别
- [ ] sleep() vs wait()区别

### 🎯 验收标准
- [ ] 能画出线程状态转换图
- [ ] 能说明sleep()和wait()的区别（持有锁？唤醒方式？）
- [ ] 能解释为什么wait()必须在synchronized块中调用

### 💻 LeetCode
- [ ] [1114. 按序打印](https://leetcode.cn/problems/print-in-order/)（Easy）— 多线程题

### 📝 学习笔记
```
在此记录学习笔记...
```