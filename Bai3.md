## Bài 3: Đọc hiểu & Dò lỗi qua Prompt và Refactor Clean Code

### Phân tích lỗi vi phạm Clean Code của mã nguồn ban đầu
- Tên phương thức và biến quá ngắn, không rõ nghĩa: `gen`, `f`, `out`, `l`, `t`, `p`, `amt`.
- Chuyên biệt trách nhiệm bị vi phạm: một phương thức vừa đọc file, vừa xử lý dữ liệu, vừa ghi file.
- Hardcode đường dẫn không được trích xuất ra tham số hoặc cấu hình.
- Không có logging, chỉ in ra `System.out.println`.
- Thiếu kiểm tra null/rỗng và xử lý ngoại lệ an toàn.
- Dòng lồng nhiều `if` và `for` khiến code khó đọc và khó bảo trì.

### Chuỗi Prompt Cải tiến đầu ra nâng cao (Refinement Chain)

#### Vòng 1: Robustness
```
Bạn là một Java Engineer có kinh nghiệm. Refactor đoạn mã sau để thêm kiểm tra an toàn:
- Kiểm tra null và chuỗi rỗng trước khi xử lý.
- Bắt và xử lý ngoại lệ `IOException` và `NumberFormatException`.
- Nếu một dòng dữ liệu lỗi, bỏ qua dòng đó mà không làm sập luồng.
- Giữ nguyên luồng xử lý chính của báo cáo.

Đoạn mã:
(import java.io.*; import java.util.*; ...) 
```

#### Vòng 2: Clean Code & SRP
```
Bạn là Senior Java Engineer. Refactor thêm để tách `ReportGenerator` thành các phương thức hoặc lớp nhỏ hơn theo nguyên tắc SRP:
- Một thành phần đọc file.
- Một thành phần xử lý tính toán.
- Một thành phần ghi file.
- Đổi tên biến cho rõ nghĩa theo CamelCase.
- Giữ cấu trúc mã dễ đọc, không lồng nhiều if/for.
```

#### Vòng 3: Logging & Context Tuning
```
Bạn là Software Architect. Tiếp tục refactor `ReportGenerator`:
- Thay `System.out.println` bằng Lombok `@Slf4j`.
- Trích xuất hardcode thành tham số đầu vào.
- Ghi log chi tiết khi dòng bị bỏ qua và khi báo cáo hoàn thành.
- Đảm bảo mã nguồn phù hợp với chuẩn doanh nghiệp.
```

### Mã nguồn Java hoàn chỉnh sau tối ưu
```java
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Slf4j
public class ReportGenerator {

    public void generateReport(String inputFilePath, String outputFilePath) throws IOException {
        Path inputPath = Paths.get(inputFilePath);
        Path outputPath = Paths.get(outputFilePath);

        ReportResult result = processOrders(inputPath);
        writeReport(outputPath, result);

        log.info("Đã tạo báo cáo. Input: {}, Output: {}", inputPath, outputPath);
    }

    private ReportResult processOrders(Path inputPath) throws IOException {
        List<String> qualifiedOrderIds = new ArrayList<>();
        double totalAmount = 0.0;

        List<String> lines = Files.readAllLines(inputPath, StandardCharsets.UTF_8);
        for (int index = 0; index < lines.size(); index++) {
            String line = lines.get(index);
            int lineNumber = index + 1;

            if (line == null || line.isBlank()) {
                log.warn("Bỏ qua dòng trống tại dòng {}", lineNumber);
                continue;
            }

            Optional<OrderRecord> optionalOrder = parseOrderLine(line, lineNumber);
            if (optionalOrder.isEmpty()) {
                continue;
            }

            OrderRecord order = optionalOrder.get();
            if (isCompletedHighValue(order)) {
                qualifiedOrderIds.add(order.orderId());
                totalAmount += order.amount();
            }
        }

        return new ReportResult(totalAmount, qualifiedOrderIds);
    }

    private Optional<OrderRecord> parseOrderLine(String line, int lineNumber) {
        String[] parts = line.split(",");
        if (parts.length < 3) {
            log.warn("Dòng {} không đúng định dạng, bỏ qua: {}", lineNumber, line);
            return Optional.empty();
        }

        String orderId = parts[0].trim();
        String amountText = parts[1].trim();
        String status = parts[2].trim();

        if (orderId.isEmpty()) {
            log.warn("Dòng {} chứa orderId rỗng, bỏ qua.", lineNumber);
            return Optional.empty();
        }

        try {
            double amount = Double.parseDouble(amountText);
            return Optional.of(new OrderRecord(orderId, amount, status));
        } catch (NumberFormatException ex) {
            log.warn("Dòng {} có amount không hợp lệ, bỏ qua: {}", lineNumber, amountText);
            return Optional.empty();
        }
    }

    private boolean isCompletedHighValue(OrderRecord order) {
        return "COMPLETED".equalsIgnoreCase(order.status()) && order.amount() > 100;
    }

    private void writeReport(Path outputPath, ReportResult result) throws IOException {
        List<String> outputLines = new ArrayList<>();
        outputLines.add("Total: " + result.totalAmount());
        for (String orderId : result.orderIds()) {
            outputLines.add("Order ID: " + orderId);
        }
        Files.write(outputPath, outputLines, StandardCharsets.UTF_8);
    }

    private static record OrderRecord(String orderId, double amount, String status) {}

    private static record ReportResult(double totalAmount, List<String> orderIds) {}
}
```
