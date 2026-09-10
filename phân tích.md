1. Phân tích tác động khi đổi tên trường (name -> productName)Jackson (thư viện JSON parser mặc định của Spring Boot) ánh xạ key trong JSON response vào thuộc tính của DTO theo tên khớp chính xác. Khi product-service trả về key "productName" trong khi DTO client chỉ khai báo trường name:Mặc định Jackson không tìm thấy key "name" tương ứng $\rightarrow$ giá trị của trường name trong DTO ProductInfo bị gán bằng null.Triệu chứng không phải là lỗi HTTP hay parse exception, mà là lỗi logic âm thầm (Silent Data Corruption) ở cả 3 service:order-service: Khi tạo đơn hàng, trường tên sản phẩm bị lưu thành null (hoặc rỗng) vào bảng orders / order_items. Nếu DB có ràng buộc NOT NULL ở cột product_name, giao dịch tạo đơn lập tức ném DataIntegrityViolationException (gãy flow đặt hàng). Khi gửi email/SMS xác nhận, khách hàng nhận thông báo: "Bạn đã đặt thành công [null] x 1".inventory-service: Khi xuất/nhập kho hoặc in biên bản kiểm kê, nhãn hàng hóa bị mất tên, hệ thống ghi log audit chứa trường null, gây sai lệch thông tin hiển thị trên portal quản trị kho.report-service: Các tác vụ tổng hợp dữ liệu, báo cáo doanh thu theo mặt hàng gom nhóm (GROUP BY) phải nhóm theo null, dẫn đến báo cáo kinh doanh tổng hợp sai lệch toàn bộ hoặc văng NullPointerException nếu có xử lý logic chuỗi (productInfo.name().toUpperCase()).2. Phân tích tác động khi đổi endpoint path (/api/products/{id} -> /api/v2/products/{id})Khi path bị đổi mà không duy trì phiên bản cũ:Ba service gọi endpoint cũ (/api/products/{id}) sẽ nhận về mã phản hồi 404 Not Found từ product-service.FeignClient nhận status 404 và mặc định ném ra FeignException.NotFound (nằm trong cây kế thừa FeignException).Nếu 3 service không có fallback / ErrorDecoder xử lý riêng cho 404, exception này lan truyền ra toàn bộ luồng nghiệp vụ:Toàn bộ API của order-service, inventory-service và report-service có dính dáng tới ProductClient.getById(...) lập tức ném mã lỗi 500 Internal Server Error về phía người dùng/caller.Toàn bộ các flow liên quan tới kiểm tra sản phẩm, tạo đơn, báo cáo bị tê liệt ngay lập tức (Hard Breaking Failure).3. Đề xuất chiến lược API Versioning & Đánh giá Trade-offĐể product-service thay đổi response và endpoint mà không làm gãy các service phụ thuộc, ta triển khai một trong hai chiến lược sau:Chiến lược 1: URI Versioning (Đường dẫn riêng biệt)Duy trì song song hai controller endpoint trên product-service: v1 phục vụ client cũ, v2 phục vụ client mới.Cách triển khai:Giữ nguyên @GetMapping("/api/products/{id}") trả về DTO cũ (name).Tạo mới @GetMapping("/api/v2/products/{id}") trả về DTO mới (productName).Ưu điểm:Rõ ràng, dễ theo dõi, trực quan qua log truy cập (Access Log / APM Gateway).Dễ dàng test trên Postman/cURL hoặc cấu hình routing tại API Gateway (Spring Cloud Gateway, Nginx).Nhược điểm (Trade-off):Nhân bản code (code duplication) ở Controller/Service nếu không refactor khéo.Client phải sửa đường dẫn trong FeignClient (@GetMapping("/api/v2/products/{id}")) khi nâng cấp.Chiến lược 2: Header / Content Negotiation VersioningGiữ nguyên một URI duy nhất (/api/products/{id}), phân biệt phiên bản thông qua Request Header (ví dụ: X-API-VERSION: 2 hoặc Accept: application/vnd.vietmart.v2+json).Cách triển khai:Java@GetMapping(value = "/api/products/{id}", headers = "X-API-VERSION=1")
public ProductInfoV1 getByIdV1(...) { ... }

@GetMapping(value = "/api/products/{id}", headers = "X-API-VERSION=2")
public ProductInfoV2 getByIdV2(...) { ... }
Endpoint mặc định (không truyền header) trỏ vào V1 để bảo đảm backward compatibility.Ưu điểm:URI sạch, đúng chuẩn RESTful nguyên bản (resource URI phản ánh thực thể, format do header quy định).URL không bị phân mảnh.Nhược điểm (Trade-off):Phức tạp trong cấu hình FeignClient phía client (phải cấu hình thêm header qua @RequestHeader hoặc RequestInterceptor).Caching ở tầng proxy/CDN phức tạp hơn (phải cấu hình cache key dựa trên Header Vary).4. Thiết kế lại DTO Phía Client (Tương thích 2 chiều)Để cả 3 service (order, inventory, report) có thể parse thành công JSON dù product-service trả về format cũ (name) hay format mới (productName), ta sử dụng annotation @JsonAlias của Jackson.Triển khai với Java RecordJavapackage com.vietmart.common.dto;

import com.fasterxml.jackson.annotation.JsonAlias;
import com.fasterxml.jackson.annotation.JsonProperty;

public record ProductInfo(
        Long id,

        // Đọc được cả key "productName" lẫn "name". 
        // Khi serialize thành JSON thì dùng "productName"
        @JsonProperty("productName")
        @JsonAlias({"name", "productName"})
        String name,

        Long price
) {}
Triển khai với Class thông thường (dùng Lombok)Javapackage com.vietmart.common.dto;

import com.fasterxml.jackson.annotation.JsonAlias;
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@JsonIgnoreProperties(ignoreUnknown = true) // Bỏ qua các trường lạ khác nếu product-service thêm vào
public class ProductInfo {

    private Long id;

    @JsonProperty("productName")
    @JsonAlias({"name", "productName"})
    private String name;

    private Long price;
}
Cơ chế hoạt động:Khi product-service trả về JSON dạng cũ: {"id": 1, "name": "Bánh mì", "price": 15000} $\rightarrow$ Jackson thấy alias "name" và gán giá trị vào biến name.Khi product-service đã nâng cấp trả về JSON mới: {"id": 1, "productName": "Bánh mì", "price": 15000} $\rightarrow$ Jackson nhận diện key "productName" và gán vào biến name.Biến name không bao giờ bị null, giúp việc migration giữa các service diễn ra hoàn toàn độc lập mà không cần deploy đồng loạt (zero-downtime deployment).5. Unit Test minh chứng DTO Migration DesignSử dụng ObjectMapper để kiểm chứng DTO nhận diện chính xác cả hai cấu trúc payload:Javapackage com.vietmart.common.dto;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ProductInfoMigrationTest {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Test
    @DisplayName("Deserialize thành công với JSON cũ chứa trường 'name'")
    void shouldDeserializeSuccessfullyWithOldPayload() throws Exception {
        String oldJson = """
                {
                    "id": 101,
                    "name": "Bột giặt OMO 3kg",
                    "price": 150000
                }
                """;

        ProductInfo productInfo = objectMapper.readValue(oldJson, ProductInfo.class);

        assertThat(productInfo).isNotNull();
        assertThat(productInfo.id()).isEqualTo(101L);
        assertThat(productInfo.name()).isEqualTo("Bột giặt OMO 3kg");
        assertThat(productInfo.price()).isEqualTo(150000L);
    }

    @Test
    @DisplayName("Deserialize thành công với JSON mới chứa trường 'productName'")
    void shouldDeserializeSuccessfullyWithNewPayload() throws Exception {
        String newJson = """
                {
                    "id": 102,
                    "productName": "Nước rửa chén Sunlight 1.5L",
                    "price": 45000
                }
                """;

        ProductInfo productInfo = objectMapper.readValue(newJson, ProductInfo.class);

        assertThat(productInfo).isNotNull();
        assertThat(productInfo.id()).isEqualTo(102L);
        assertThat(productInfo.name()).isEqualTo("Nước rửa chén Sunlight 1.5L");
        assertThat(productInfo.price()).isEqualTo(45000L);
    }
}
