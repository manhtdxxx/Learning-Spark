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
		- ***Theo Client Mode***, Spark Driver chạy ngoài Cluster, nằm ở trên máy thực thi `spark-submit --deploy-mode client`, tất cả các nodes đều là Worker Nodes 
		- ***Theo Cluster Mode***, Spark Driver nằm bên trong Cluster, chạy trên 1 node riêng biệt do Cluster Manager phân bổ, gọi là ***Driver Node***, có vai trò:
			- Tạo Spark Session / Spark Context để khởi tạo ứng dụng
			- Tạo DAG, phân chia stage/task, và gửi tới Executor để thực thi
			- Theo dõi tiến trình, tổng hợp kết quả từ các Executor
		- ***Worker Nodes*** chịu trách nhiệm chạy các tiến trình Executors để thực thi tasks
			- Nếu dùng Standalone, Worker Node chủ yếu để chạy Executor(s)
			- Nếu dùng Yarn, Worker Node có thể chạy nhiều thứ: Node Manager, Executors, Application Master, ...
		- Mỗi ***Executor*** thực thi các tasks song song, với số lượng task đồng thời phụ thuộc vào số core được cấp phát cho Executor đó
- **Resource Management (Yarn, Standalone, Yarn, Mesos, Kubernetes)**
	- Standalone là Resource Management của chính Spark khi không phụ thuộc vào các bên khác, nằm bên trong Spark Cluster, chạy trên 1 Node riêng biệt, gọi là Master Node / Cluster Manager
	- Kiến trúc Yarn
		- ***Resource Manager***
			- Quản lý tài nguyên toàn bộ Cluster, nằm trên Master Node
			- Chấp nhận hoặc từ chối yêu cầu từ Application Master cho việc cấp phát tài nguyên (CPU, RAM, ...)
		- ***Node Managers***
			- 1 Node Manager chạy trên 1 Worker Node
			- Node Manager sẽ cấp phát
				- 1 container cho Application Master
				- Nhiều containers cho Executors `-- thông thường, 1 Executor ứng với 1 Container để tránh cạnh tranh tài nguyên --`
			- Giám sát, báo cáo trạng thái tài nguyên của Container cục bộ về cho Resource Manager `-- vd: node 2 có 3 containers, mỗi container dùng 2 Cores và 1GB RAM --`
		- ***Application Master***
			- AM do Resource Manager khởi tạo riêng cho mỗi ứng dụng, là thứ giúp ứng dụng giao tiếp được với YARN
			- Tại sao cần?
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
		- Là một phiên bản mở rộng của SQLContext trong Spark 1.x, giúp
			- Hỗ trợ kết nối với Hive Metastore
			- Thực thi SQL truy vấn trên Hive Table
		- Trong Spark 2.0+, chức năng của HiveContext được tích hợp vào SparkSession khi config `.enableHiveSupport()`
---
# 4. Lazy Evaluation
- **Transformation**
	- ***Narrow Transformation***
		- Dữ liệu được transform tại chính partition nó ở, không có hiện tượng dữ liệu trao đổi giữa các Executors/Nodes
		- Không có shuffle, nên narrow này nhanh và tốn ít tài nguyên hơn wide
		- Methods: `map(), flatMap(), filter(), join(broadcast()), ...`
	- ***Wide Transformation***
		- Dữ liệu sau khi transform sẽ được phân phối lại thành các partitions mới giữa các Executors/Nodes
		- Shuffle là bước tốn thời gian và tài nguyên bởi dữ liệu phải được ghi ra disk, truyền qua mạng, đọc lại từ disk
		- Khi shuffle xảy ra, Stage mới được hình thành
		- Methods: `reduceByKey(), sortByKey(), distinct(), join(), groupBy(), ...`
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
	1. Người dùng viết ứng dụng Spark và dùng `spark-submit` để gửi job lên Yarn Resource Manager
	2. RM nhận yêu cầu, kiểm tra khả năng cấp tài nguyên
	3. Nếu khả thi, RM chọn 1 node, Node Manager sẽ khởi tạo 1 container để chạy Application Master
	4. Spark Driver khởi tạo Spark Context, chứa cấu hình tĩnh `(--num-executors, --executor-cores, --executor-memory)`
	5. AM gửi yêu cầu xin tài nguyên đến RM
	6. RM gửi yêu cầu tới NM trên các nodes còn lại để khởi tạo container cho executor
	7. Khi kích hoạt action, lập DAG, chia tasks xong thì Driver phân phối các task xuống các executor
		- Nếu task treo nhiều, AM xin thêm tài nguyên từ RM
		- Nếu task ít, AM sẽ thông báo cho RM giải phóng bớt tài nguyên
	8. Executor thực hiện các task
	9. Executor gửi trạng thái / kết quả của task về Driver 
	10. Driver tổng hợp kết quả cuối cùng và trả về cho người dùng
	11. Khi Job hoàn tất, AM gửi tín hiệu tới RM để giải phóng tài nguyên
---
# 6. Partition
- File nén như gz, ... → non-splittable
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
- **Check số partitions**
	- `rdd.getNumPartitions()`
	- `df.rdd.getNumPartitions()`
- **Tăng số partitions**
	- Dùng `repartition()`, sẽ xảy ra shuffle
		- `rdd_2 = rdd.repartition(n_partitions)`
		- `df_2 = df.repartition(n_partitions)`
- **Giảm số partitions**
	- Dùng `coalesce()`, không gây shuffle, cố gắng gộp partitions trong cùng 1 executor lại
	- Có thể dùng `repartition()`, gây shuffle
---
# 7. Executor
- VD 1 worker node có tài nguyên gồm 16 cores, 64GB ram

| Tiêu chí                    | Thin Executor                                                  | Fat Executor                                     |
| --------------------------- | -------------------------------------------------------------- | ------------------------------------------------ |
| **Số executors**            | 16 executors                                                   | 1 executors                                      |
| **Cores/executor**          | 1 core/executor                                                | 16 cores/executor                                |
| **RAM/executor**            | 4 GB/executor                                                  | 64 GB/executor                                   |
| **Số tasks song song**      | 16 tasks                                                       | 16 tasks                                         |
| **Overhead**                | Cao do nhiều executor → lãng phí tài nguyên quản lý            | Thấp hơn ...                                     |
| **Garbage Collection (GC)** | GC nhanh, pause ngắn nhờ ít ram/executor                       | GC chậm, pause dài → làm chậm việc thực thi task |
| **Shuffle**                 | Nhiều shuffle do ít task/executor → tăng disk I/O, tắc network | Ít shuffle, ...                                  |
| **Fault tolerance**         | Tốt hơn, lỗi 1 executor → mất ít task, ít tính toán lại        | Kém hơn, ...                                     |
| **Phù hợp cho**             | Workload có nhiều task nhỏ, ít shuffle, parallelism cao        | Workload nặng (như ML), join phức tạp            |

- Thông thường, ta sẽ mất 1 core, 1GB ram cho OS → còn lại 15 cores, 63GB ram
- Theo khuyến nghị, số cores tối ưu cho 1 executor là 5
	- 3 executors
	- mỗi executor gồm 5 cores, 21GB ram
