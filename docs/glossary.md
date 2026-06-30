# Glossary

> Thuật ngữ dùng trong hub này. Domain nghiệp vụ thật xem thêm skill phù hợp với domain của bạn.

## Hub / AI tooling

| Thuật ngữ | Định nghĩa |
|-----------|-----------|
| Knowledge hub | `{{workspace-root}}` — workspace tổng hợp điều phối AI, không phải runtime app |
| Skill | Folder + `SKILL.md`, năng lực AI nạp động theo `description` |
| Plugin | Bó skill + commands + connectors cho 1 vai trò |
| MCP | Model Context Protocol — chuẩn kết nối AI tới tool ngoài qua `.mcp.json` |
| Catalog | Danh mục skill sinh bởi `generate_catalogs.py`, agent đọc để kích hoạt |
| Hook | Script tự động chạy before/after event, cấu hình trong `settings.json` |
| Agent / subagent | Tác nhân chuyên trách (planner, tester, code-reviewer...) trong workflow |
| Memory | Bài học file-based ở `~/.claude/projects/<workspace>/memory/`, index `MEMORY.md` |
| Downstream target | Project thật hub điều khiển AI làm việc lên (các hệ thống của bạn) |
| Work-context path | Git root của project đang sửa; truyền cho subagent thay vì CWD |

## Domain nghiệp vụ (tham chiếu nhanh — ví dụ, tuỳ domain của bạn)

> Các bảng dưới là **ví dụ** thuật ngữ cho một số domain phổ biến. Xoá/thay/bổ sung theo domain thật của bạn.

### Ví dụ: Thương mại điện tử (E-commerce)

| Thuật ngữ | Nghĩa |
| --------- | ----- |
| SKU | Stock Keeping Unit — mã định danh từng biến thể sản phẩm |
| Cart / Checkout | Giỏ hàng / luồng thanh toán |
| Order | Đơn hàng (`{{order-service}}`) |
| Catalog | Danh mục sản phẩm + thuộc tính, giá |
| Inventory | Tồn kho theo SKU/kho |
| Payment gateway | Cổng thanh toán (`{{payment-gateway}}`) |
| Fulfillment | Soạn hàng → đóng gói → bàn giao vận chuyển |

### Ví dụ: Shipping line / Hãng tàu

| Thuật ngữ | Nghĩa |
| --------- | ----- |
| B/L (Bill of Lading) | Vận đơn — chứng từ vận chuyển + sở hữu hàng |
| Container / TEU | Container; TEU = Twenty-foot Equivalent Unit |
| Booking | Đặt chỗ vận chuyển trên tàu |
| Voyage / Vessel | Chuyến tàu / con tàu |
| Port of Loading / Discharge | Cảng xếp hàng / cảng dỡ hàng |
| Demurrage / Detention | Phí lưu container tại cảng / ngoài cảng |

### Ví dụ: Logistics

| Thuật ngữ | Nghĩa |
| --------- | ----- |
| WMS | Warehouse Management System (`{{wms-service}}`) |
| TMS | Transport Management System (`{{tms-service}}`) |
| Last-mile | Chặng giao cuối tới khách hàng |
| Shipment / Waybill | Lô hàng / vận đơn giao nhận |
| 3PL | Third-Party Logistics — bên cung cấp logistics thuê ngoài |
| Cross-docking | Trung chuyển không lưu kho |
| Route optimization | Tối ưu tuyến giao hàng |

### Ví dụ: Tài chính — Ngân hàng — Chứng khoán

| Thuật ngữ | Nghĩa |
| --------- | ----- |
| Ledger / GL | Sổ cái / General Ledger — ghi nhận bút toán kế toán |
| Account / Sub-account | Tài khoản / tiểu khoản giao dịch |
| Debit / Credit | Ghi nợ / ghi có |
| Balance | Số dư (available vs ledger balance) |
| Transaction | Giao dịch tiền/tài sản (`{{transaction-service}}`) |
| Settlement | Quyết toán / thanh toán bù trừ giữa các bên |
| Clearing | Bù trừ — đối chiếu nghĩa vụ trước settlement |
| Reconciliation | Đối soát số liệu giữa hệ thống/đối tác |
| KYC / AML | Định danh khách hàng / chống rửa tiền |
| Interest / APR | Lãi suất / Annual Percentage Rate |
| Collateral | Tài sản đảm bảo cho khoản vay/giao dịch |
| Order | Lệnh mua/bán (`{{order-service}}`) |
| Portfolio | Danh mục tài sản nắm giữ |
| Instrument | Công cụ tài chính (cổ phiếu, trái phiếu, phái sinh...) |
| Equity / Bond | Cổ phiếu / trái phiếu |
| Bid / Ask | Giá đặt mua / giá chào bán |
| Matching | Khớp lệnh mua–bán trên sàn |
| Custody / Depository | Lưu ký / trung tâm ký gửi chứng khoán |
| NAV | Net Asset Value — giá trị tài sản ròng (quỹ) |
| Mark-to-market | Định giá theo giá thị trường hiện tại |
