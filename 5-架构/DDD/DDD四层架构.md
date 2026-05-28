>DDD(Domain-Driven Design，领域驱动设计)，不是“分层模板”，而是”先把业务模型建对，再让代码围绕业务模型组织“的一套方法论

## 一，分层关系
  >adapter -> application -> domain <- infrastructure。
  
  1. adapter：控制器、DTO、参数校验、协议转换。
  2. application：流程编排、事务边界（比如创建/草稿处理器）。
  3. domain：规则收口，比如草稿和正式提交采用不同强度校验（如 saveDraft() 与 create() 的规则差异）。
  4. infrastructure：实现 domain 定义的 gateway，比如 MyBatis SQL、PostGIS 查询、第三方 API 调用。
可维护性：规则集中在 domain，不会散在 controller/sql 里。
可测试性：application 和 domain 可以用 mock gateway 做单测。
  5 可演进性：换数据库或外部服务时，主要动 infrastructure，不会大面积改业务流程。
     所以 DDD 在这个项目里的价值，不是“看起来分了四层”，而是确实让复杂业务改得动、测得住、讲得清。