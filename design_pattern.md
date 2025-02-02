# 设计原则

- 单一职责原则
  - 类对外只提供一种功能
  - 引起类变化的原因只应有一个
- 开闭原则
  - 对修改关闭
  - 对扩展开放
  - 类的改动是通过增加代码实现的，而不是修改源代码
- 里氏代换原则
  - 任何抽象类出现的地方都可以用它的实现类进行替换
- 依赖倒转原则
  - 依赖于抽象
  - 不要依赖具体的实现
- 接口隔离原则
  - 一个接口只应提供一种对外功能
  - 不要把所有操作都封装到一个接口
- 合成复用原则
  - 如果使用继承，会导致父类的任何变换都可能影响到子类的行为
  - 如果使用对象组合，就降低了这种依赖关系
  - 优先使用组合而不是继承
- 迪米特法则
  - 一个对象对其他对象尽可能少的了解，从而降低对象间耦合，提高系统可维护性

# 工厂模式

