# 1. ARCHITECTURE
- CLUSTER
	- NODE(S)
		- DRIVER NODE
		- WORKER NODE(S)
			- EXECUTOR(S)
---
 # 2. API
- RDD (RESILIENT DISTRIBUTED DATASET)
- DATAFRAME
- SPARK SQL
---
# 3. SPARK SESSION
- SPARK CONTEXT
- SQL CONTEXT
---
# 4. LAZY EVALUATION
- TRANSFORMATION
	- NARROW (NO SHUFFLE)
	- WIDE (SHUFFLE)
- ACTION
---
# 5.  EXECUTION MODEL
 - JOB = 1 ACTION
	 - STAGE = 1 GROUP OF TRANSFORMATIONS UNTIL SHUFFLE
		 - TASK = 1 PARTITION
---
# 6. PARTITION
- File nén như gz, ... -> không thể split
- Cách chia
	- Khi đọc 1 file
		- $$\max(\frac{\text{file size}}{\text{128 MB}},\ \text{number of cores})$$
		- **128MB** là kích thước block mặc định
		- Spark sẽ tạo **ít nhất số partition = số core** để tận dụng song song (nếu có thể)
		---
	- Khi đọc nhiều file
		- $$\max(\frac{\text{number of files} \cdot \text{(file size + 4 MB)}}{\text{128 MB}},\ \text{number of cores})$$
		- **4MB overhead** là chi phí Spark tính cho metadata mỗi file (tên file, đường dẫn file, loại file, ...)
		- Spark gom nhiều file nhỏ vào 1 partition nếu cần, hoặc chia đều nếu file lớn
		- Nếu file nhỏ mà nhiều → có thể tạo quá nhiều partition → **gây overhead quản lý task**
 ---
- Check
	- `rdd.getNumPartitions()`
	- `df.rdd.getNumPartitions()`
- Chia lại (Shuffle lại toàn bộ partitions cũ qua mới -> tốn kém )
	- `rdd_2 = rdd.repartition(n_partitions)`
	- `df_2 = df.repartition(n_partitions)`
	
