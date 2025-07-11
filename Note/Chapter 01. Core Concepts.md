# 1. Architecture
- **Storage (HDFS, S3, ...)**
	- Kiến trúc HDFS
		- ***Name Node*** là nơi quản lý metadata (tên file, vị trí block, ...)
		- ***Data Nodes*** là nơi lưu trữ dữ liệu vật lý
		- Dữ liệu file được chia thành các ***Blocks*** (mặc định 128 MB)
		- Mỗi Block được lưu trên nhiều Data Node (thường là 3 bản sao)
- **Data Processing (MapReduce, Spark)**
	- Kiến trúc Spark
		- Là 1 Cluster gồm nhiều Nodes
		- Theo Client Mode, Spark Driver chạy ngoài Cluster, tất cả các Nodes đều là Worker Nodes
		- Theo Cluster Mode, 1 Node được gọi là ***Driver Node*** nếu có Spark Driver chạy trên đó
			- Tạo Spark Session / Spark Context để khởi tạo ứng dụng
			- Tạo DAG, phân chia stage/task, và gửi tới Executor để thực thi
			- Theo dõi tiến trình, nhận kết quả, xử lý lỗi, retry task của Executor
		- Các node còn lại là ***Worker Nodes***, chịu trách nhiệm chạy các tiến trình Executors để xử lý dữ liệu
			- Nếu dùng Standalone, Worker Node chủ yếu để chạy Executor(s)
			- Nếu dùng Yarn, Worker Node có thể chạy nhiều thứ: Node Manager, Executors, Application Master, ...
		- Mỗi ***Executor*** thực thi các tasks song song, với số lượng task đồng thời phụ thuộc vào số core được cấp phát cho Executor đó
- **Resource Management (Yarn, Standalone - riêng Spark, ...)**
	- Kiến trúc Yarn
		- ***Resource Manager***
			- Quản lý tài nguyên toàn bộ Cluster, nằm trên Master Node
			- Chấp nhận hoặc từ chối yêu cầu từ Application Master cho việc cấp phát tài nguyên (CPU, RAM, ...)
		- ***Node Managers***
			- 1 Node Manager chạy trên 1 Worker Node
			- Chọn 1 Node bất kỳ, Node Manager khởi chạy Container cho Application Master
				- Nếu Cluster Mode, AM nằm ở Driver Node
				- Nếu Client Mode, AM nằm ở Worker Node
			- Đối với các Worker Node còn lại, Node Manager sẽ nhận lệnh từ Resource Manager do Application Master yêu cầu và thực hiện phân bổ Container cho Executor `-- thông thường, 1 Executor ứng với 1 Container để tránh cạnh tranh tài nguyên --`
			- Giám sát, báo cáo trạng thái tài nguyên của Container cục bộ về cho Resource Manager `-- vd: node 2 có 3 containers, mỗi container dùng 2 Cores và 1GB RAM --`
		- ***Application Master***
			- AM là lớp trung gian giữa Yarn và ứng dụng
			- Mỗi ứng dụng sẽ có 1 AM riêng
			- Resource/Node Manager chỉ là người quản lý và cấp phát tài nguyên, không có nhu cầu phải hiểu logic của từng loại app  (cách sử dụng tài nguyên)  → xuất hiện AM
			- Analogy: "tao biết app cần gì, tao xin container cho phù hợp!"
			- Theo dõi trạng thái của toàn bộ Executors thuộc cùng ứng dụng ở cấp độ Container. Coi xem Executor nào chết để xin thêm tài nguyên `-- vd: executor 1 ở node 2 bị kill vì thiếu RAM --`
			- Trong Cluster Mode, AM và Spark Driver cùng chia sẻ 1 Container
		- ***Container***
			- Là 1 đơn vị logic đóng gói một phần tài nguyên gồm (CPU, RAM, ...) 
			- Giúp cô lập tài nguyên cho việc cấp phát, quản lý
---
 # 2. API
- RDD
- DataFrame
- Spark SQL
---
# 3. Entry Point
- **Spark Session** 
	- Là Entry Point chính để làm việc với các API của Spark trong Spark 2.0 trở lên
- **Entry Points in version 1.x**
	- ***Spark Context***
		- Là 1 Entry Point làm việc với RDD
		- Chứa cấu hình của ứng dụng Spark `class SparkConf`
		- Kích hoạt quá trình khi gặp action
			- Kết nối với Cluster Manager (Standalone, YARN, Mesos, Kubernetes). Yêu cầu tạo executor(s) với số cores, ram được config 
			- Gửi Job tới DAG Scheduler để xây DAG, chia stages
			- Task Scheduler tiếp tục chia stages thành tasks, phân phối tasks tới executor
			- Executor thực thi và trả kết quả về Spark Driver
		- Trong Spark 2.0+, SparkContext vẫn tồn tại nhưng được bao bọc bởi SparkSession `sc = spark.sparkContext`
	- ***SQL Context***
		- Là 1 Entry Point để làm việc với Structured Data trong Spark 1.x, cho phép thực thi các truy vấn SQL và làm việc với DataFrame
		- Trong Spark 2.0+, SQLContext đã được thay thế bởi SparkSession
	- ***Hive Context***
		- Là một phiên bản mở rộng của SQL Context trong Spark 1.x, giúp
			- Hỗ trợ kết nối với Hive Metastore
			- Thực thi SQL truy vấn trên Hive Table
		- Trong Spark 2.0+, chức năng của HiveContext được tích hợp vào SparkSession khi config `.enableHiveSupport()`
---
# 4. Lazy Evaluation
- **Transformation**
	- ***Narrow Transformation***
		- Dữ liệu được transform tại chính partition nó ở, không có hiện tượng dữ liệu trao đổi giữa các Executors hay Nodes
		- Không có hiện tượng shuffle, nên narrow này nhanh và tốn ít tài nguyên hơn wide
		- Methods: `map(), flatMap(), filter(), sample(), ...`
	- ***Wide Transformation***
		- Dữ liệu sau khi transform sẽ được phân phối lại thành các partitions mới giữa các Executors hay Nodes
		- Shuffle là bước tốn thời gian và tài nguyên bởi dữ liệu phải được ghi ra disk, truyền qua mạng, đọc lại từ disk
		- Khi shuffle xảy ra, Stage mới được hình thành
		- Methods: `reduceByKey(), sortByKey(), distinct(), join(), ...`
- **Action**
	- Là các methods kích hoạt các transformations đã định nghĩa
	- Lúc này, Spark sẽ áp dụng Lazy Evaluation, xây dựng 1 DAG sắp xếp các transformations theo một thứ tự tối ưu nhất có thể trước khi thực hiện
	- Mỗi action sẽ được coi là 1 job
	- Methods: `take(), collect(), count(), reduce(), ...`
---
# 5.  Job Execution Flow
- **Cách chia Job**
	- Job xuất hiện khi có action `1 job = 1 action`
	- Job được chia thành các stages `1 stage = 1 group of transformations until shuffle`
	- Stages lại được chia tiếp thành các tasks `1 task = 1 partition`
 - **Quy trình xử lý của Spark khi tích hợp với Yarn**
	1. Người dùng viết ứng dụng Spark và dùng `spark-submit` để gửi job lên YARN Resource Manager
	2. RM nhận yêu cầu, kiểm tra khả năng cấp tài nguyên
	3. Nếu OK, chọn 1 Node, Node Manager trên Node đó, khởi tạo 1 container để chạy Application Master
	4. Spark Driver khởi tạo Spark Context, chứa cấu hình tĩnh `(--num-executors, --executor-cores, --executor-memory)`
	5. AM gửi yêu cầu xin tài nguyên (CPU, RAM, ...) đến RM.
	6. RM gửi yêu cầu tới NM trên các Nodes khác để khởi tạo Container cho Executor
	7. Khi kích hoạt action, lập DAG, chia tasks xong thì Driver phân phối các task xuống các Executor
		- Nếu task treo nhiều, AM xin thêm tài nguyên từ RM
		- Nếu task ít, AM sẽ thông báo cho RM giải phóng bớt tài nguyên
	8. Executor thực hiện các task
	9. Executor gửi trạng thái / kết quả của task về Driver 
	10. Driver tổng hợp kết quả cuối cùng và trả về cho người dùng
	11. Khi Job hoàn tất, AM gửi tín hiệu tới RM để giải phóng tài nguyên
---
# 6. Partition
- File nén như gz, ... -> non-splittable
- **Cách chia file thành partitions**
	- ***Khi đọc 1 file***
		- $$\max(\frac{\text{file size}}{\text{128 MB}},\ \text{number of cores})$$
		- 128 MB là kích thước block mặc định trong HDFS
		- Spark sẽ tạo ít nhất số partition = số core để tận dụng song song (nếu có thể)
		---
	- ***Khi đọc nhiều file***
		- $$\max(\frac{\text{number of files} \cdot \text{(file size + 4 MB)}}{\text{128 MB}},\ \text{number of cores})$$
		- 4 MB overhead là chi phí Spark tính cho metadata mỗi file (tên file, đường dẫn file, loại file, ...)
		- Spark gom nhiều file nhỏ vào 1 partition nếu cần, hoặc chia đều nếu file lớn
		- Nếu file nhỏ mà nhiều → có thể tạo quá nhiều partition → gây overhead quản lý task
 ---
- Check số partitions
	- `rdd.getNumPartitions()`
	- `df.rdd.getNumPartitions()`
- Chia lại (Shuffle lại toàn bộ partitions cũ qua mới → tốn kém )
	- `rdd_2 = rdd.repartition(n_partitions)`
	- `df_2 = df.repartition(n_partitions)`
	
