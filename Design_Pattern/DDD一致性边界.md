# 规则

- 将真实不变式建模为一致性边界
- 需要数据两个表以上使用外键等约束规则保持书籍一次性一致更改的，都可能需要放入一个聚合
- 区分业务规则与业务流程，涉及到业务流程的长时间事务不能放入聚合
- 聚合跨时间边界太大，但是业务规则与业务流程有时很难区分，业务规则通过业务流程来实现的，也不能放入聚合