>DDD(Domain-Driven Design，领域驱动设计)，不是“分层模板”，而是”先把业务模型建对，再让代码围绕业务模型组织“的一套方法论

## 一，分层关系
  >adapter -> application -> domain <- infrastructure。
  
  1. adapter：控制器、DTO、参数校验、协议转换。
  2. application：流程bin、事务边界（比如创建/草稿处理器）。
  3. domain：规则收口（如 saveDraft() 与 create() 的规则差异）。
  4. infrastructure：MyBatis/PostGIS/第三方 API 等实现细节。
  关键点：application 依赖 domain 的接口，不直接依赖 infrastructure 实现，这就是依赖倒置。