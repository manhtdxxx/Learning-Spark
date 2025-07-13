- Cách xem chi tiết: `.explain(True)`
---
### Logical Plan
- Cấu trúc
	- **Parsed Logical Plan**
		- Check syntax
	- **Analyzed Logical Plan**
		- Check column, table, ... liệu có tồn tại không
	- **Optimized Logical Plan**
		- Áp dụng Catalyst Optimizer cho Lazy Evaluation
		- Ví dụ có `.filter()` thì nên ưu tiên dùng trước ...
---
### Physical Plan
- Được tạo nhiều để test trên nhiều chiến thuật, sau đó tính cost model để coi cách nào tối ưu nhất, ít tốn chi phí nhất
- Ví dụ như Join có nhiều loại, cần test để coi nên dùng loại Join nào