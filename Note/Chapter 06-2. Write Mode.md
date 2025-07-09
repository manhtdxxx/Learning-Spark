> - Với Read, Spark có thể đọc 1 file hoặc 1 folder gồm nhiều files
> - Với Write, Spark chỉ có thể ghi xuống thành 1 folder tập hợp các files = số partitions

- **Overwrite**
	- `df.write.csv("folder_path").mode("overwrite").option(..., ...) ...`
	- `df.write.format("csv").mode("overwrite").save("folder_path")`
	- `df.write.format("csv").mode("overwrite").option("path", "folder_path").save()`
	- Ghi đè (xóa dữ liệu cũ, ghi mới hoàn toàn)
	- Nếu folder chưa tồn tại, tự tạo
---
- **Append**
	- `mode("append")`
	- Ghi thêm file mới vào folder hiện có
	- Nếu folder chưa tồn tại, tự tạo
---
- **Ignore**
	- `mode("ignore")`
	- Bỏ qua nếu folder path đã tồn tại
---
- **Error If Exist** 
	- `mode("errorIfExists)`
	- Ném lỗi nếu folder path đã tồn tại
---
### Tối ưu hóa ghi File bằng Partition By cho 1 cột
- `df.write.parquet("base_folder").mode(...).partitionBy("col_name") ..."`
- Đường dẫn file(s) chứa dữ liệu: `base_folder/col_name=value/.parquet`
- Đặc điểm
	- Nếu lưu file dạng không nén như CSV, kích thước file = kích thước partition
	- Dữ liệu trong file sẽ mất cột được `partitionBy` vì Spark mặc định trong file đó cột đó chỉ có 1 giá trị duy nhất
- Tác dụng
	- Khi đọc file với `.where("col_name='value'")` thì sẽ nhanh hơn bởi Spark sẽ biết đọc chính xác file nào, nằm trong folder nào, không phải scan toàn bộ files
	- Chỉ nhanh hơn khi điều kiện bao gồm cột được `partitionBy`
---
### Tối ưu hóa ghi File bằng Partition By cho nhiều cột
- `df.write.parquet("base_folder").mode(...).partitionBy("col_1", col_2") ...`
- Đường dẫn file(s) chứa dữ liệu: `base_folder/col_1=value/col_2=value/.parquet`
