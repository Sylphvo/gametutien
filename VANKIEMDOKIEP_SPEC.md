# VẠN KIẾM ĐỘ KIẾP – ĐẶC TẢ DỰ ÁN (GDD + TDD)

> Bản gộp toàn bộ tài liệu trong `docs/*.html`. Đây là **nguồn chính** cho AI coding agent.
> File `tailieukientrucgame.md` cũ đã bị thay thế, không dùng nữa.
> Khi file này và HTML khác nhau, sửa cả hai cho khớp. Không tự chọn một bên.

---

## 0. QUY TẮC CHO AI CODING AGENT

1. Chỉ làm phần việc của giai đoạn đang được yêu cầu (xem `ANTIGRAVITY_PROMPTS.md`). Không tự làm trước giai đoạn sau.
2. **Không tự bịa số liệu cân bằng.** Mọi con số lấy từ mục 9 (Cân bằng). Nếu thiếu, đặt giá trị tạm, ghi `// TODO(balance): ...` và liệt kê trong báo cáo cuối.
3. Mọi con số nằm trong ScriptableObject hoặc `[SerializeField]`, không hard-code trong logic.
4. Tên cảnh giới thứ 3 là **"Kết Đan"** (không phải "Kim Đan").
5. Không thêm thư viện bên ngoài (DOTween, Newtonsoft, Zenject…). Tween dùng coroutine + `Time.unscaledDeltaTime`. Lưu game dùng `JsonUtility`.
6. Code C# chuẩn, có XML summary comment cho class và method public. Mỗi class một file, tên file trùng tên class.
7. Hiệu năng (mục tiêu 60 FPS Android tầm trung):
   - Không `Instantiate()`/`Destroy()` trong vòng lặp chiến đấu. Dùng `ObjectPooler`.
   - Không `Camera.main`, `GameObject.Find()`, `GetComponent()` trong `Update()`. Cache ở `Awake()`.
   - GC Alloc = 0 trong combat: dùng `Physics2D.OverlapCircleNonAlloc`, cache mảng/list, không LINQ, không nối chuỗi trong `Update()`.
8. Pattern: `Singleton` cho Manager cốt lõi; `Object Pooling` cho đạn, phi kiếm, quái, VFX, số sát thương, Linh Châu; C# events để tách gameplay khỏi UI.
9. Thứ Agent **không làm được** và phải báo cho người dùng làm tay trong Unity Editor: kéo thả tham chiếu, tạo scene, gán sprite. Agent nên viết Editor script tự động hóa những việc này khi có thể (ví dụ `VkdkDataSeeder` tạo ScriptableObject từ bảng số liệu).
10. Sau mỗi giai đoạn, báo cáo: file đã tạo, việc người dùng cần làm tay trong Editor, các `TODO`.
11. **Luôn chơi được:** mọi tính năng phải chạy được khi chưa có hình, âm thanh, lời thoại thật. Dùng sprite tạm (`VkdkPlaceholderArt`), âm thanh tạm (`VkdkPlaceholderAudio`), hội thoại mẫu. Scene và tham chiếu dựng bằng Editor script, người dùng chỉ bấm menu rồi Play.
12. Mọi chữ hiện trên UI lấy từ `StringTable` qua `Loc.Get(key)` (mục 12.5).

---

## 1. TỔNG QUAN

| Mục | Giá trị |
|---|---|
| Tên | Vạn Kiếm Độ Kiếp (*Sword Realm: Tribulation*) |
| Thể loại | 2D Action Roguelike / Arena Survival, tu tiên |
| Engine | Unity 6.3 LTS (6000.3.x), template **Universal 2D** (URP + 2D Renderer) |
| Nền tảng | Android, màn hình ngang 16:9 / 20:9 |
| Kiếm tiền | Free-to-Play + Google AdMob (Rewarded, Interstitial, Banner) |
| VFX | Dựng trong Unreal Niagara → bake flipbook PNG → Particle System trong Unity |

Người chơi nhập vai **Hàn Lập** qua 19 chương ở ba giới diện Nhân Giới → Linh Giới → Tiên Giới. Mỗi chương là một **Bí Cảnh** chơi 10–15 phút: phi kiếm tự động chiến đấu, người chơi di chuyển né đòn, chọn thẻ Kỳ Ngộ, vượt Lôi Kiếp để đột phá và Biến Thân.

**Bản quyền:** cốt truyện, nhân vật, địa danh lấy từ tiểu thuyết *Phàm Nhân Tu Tiên* (Vong Ngữ). Phát hành thương mại cần giấy phép, nếu không phải đổi tên và viết lại nội dung.

### 1.1. Phiên bản & package

| Package | Cách cài | Dùng cho |
|---|---|---|
| Universal RP | Có sẵn trong template | Render, 2D Light, Bloom |
| 2D Sprite, 2D Animation, 2D PSD Importer | Package Manager | Sprite Editor, nhân vật `.psb` có xương |
| Input System | Package Manager; Active Input Handling = Input System Package (New) | Đa chạm, joystick |
| uGUI (có TextMeshPro) | Có sẵn (từ Unity 6, TMP nằm trong uGUI). Import TMP Essential Resources | Chữ |
| Shader Graph | Package Manager | `SG_Sprite_Additive` |
| Google Mobile Ads Unity Plugin 11.x | `.unitypackage` từ github.com/googleads/googleads-mobile-unity/releases (kèm External Dependency Manager). Nhập Android App ID trong *Assets → Google Mobile Ads → Settings*, chạy *Android Resolver → Force Resolve* | Quảng cáo, UMP |

Player Settings: IL2CPP, ARM64, API Compatibility .NET Standard 2.1.

### 1.2. Quyết định đã chốt

| Chủ đề | Quyết định |
|---|---|
| Đột phá | 3 lần Độ Kiếp mỗi trận = 4 bậc cảnh giới. Tên bậc lấy theo giới diện của chương |
| Biến Thân | Lần Độ Kiếp 1/2/3 mở Thanh Lôi / Kim Lôi / Hồng Lôi |
| `OnTransformationChanged` | `public static event Action<Color>` trên `TransformationHandler`. Gửi màu mặc định khi hết Biến Thân |
| `TimeService` | Không có server riêng. Lấy header `Date` từ `UnityWebRequest.Head("https://www.google.com")` |
| Boss | Một class `BossController` chung, mỗi boss là một `BossData` |
| Tên enum giới diện | `WorldPlane` (không dùng `Plane` vì trùng `UnityEngine.Plane`) |

---

## 2. CỐT TRUYỆN & THẾ GIỚI

Hàn Lập: thiếu niên nghèo, tư chất **Tứ Linh Căn** (ngụy linh căn). Nhờ **Chưởng Thiên Bình** và tính cẩn trọng ("thấy lợi không ham, gặp nguy bỏ chạy") đi từ phàm nhân đến Chí Cao Đạo Tổ.

Kể chuyện: hội thoại 2D (chân dung + khung chữ) trước và sau trận, có nút bỏ qua. Boss phút 10 của mỗi chương là nhân vật/kẻ địch trong truyện.

### 2.1. Chương

| Ch. | Bí Cảnh | Nội dung | Cảnh giới trong trận | Boss phút 10 | Mở khóa |
|---|---|---|---|---|---|
| 1 | Thất Huyền Môn | Gia nhập Thất Huyền Môn, kết bạn Lệ Phi Vũ, theo Mặc Đại Phu, nhặt Chưởng Thiên Bình | Luyện Khí → Trúc Cơ → Kết Đan → Nguyên Anh | Yêu Lang Huyết Nhãn | Chưởng Thiên Bình, Động Phủ |
| 2 | Mật thất Mặc phủ | Âm mưu đoạt xá của Mặc Đại Phu | như trên | Mặc Đại Phu | Luyện đan |
| 3 | Huyết Sắc Cấm Địa | Hoàng Phong Cốc, thử luyện lấy dược liệu Trúc Cơ Đan, gặp Nam Cung Uyển | như trên | Huyết Văn Mãng Vương | Huyết Ngọc Tri Nhện |
| 4 | Việt Quốc thất thủ | Ma Đạo Lục Tông tấn công, rời Thiên Nam | như trên | Trưởng lão Hợp Hoan Tông | Thanh Trúc Phong Vân Kiếm |
| 5 | Loạn Tinh Hải | Tinh Cung – Nghịch Tinh Minh, gặp Tử Linh Tiên Tử | Trúc Cơ → Kết Đan → Nguyên Anh → Hóa Thần | Yêu thú biển Ngoại Hải | Phệ Kim Trùng |
| 6 | Hư Thiên Điện | Đoạt Hư Thiên Đỉnh, Càn Lam Băng Diễm; gặp Đại Diễn Thần Quân | như trên | Lão ma Cực Âm Đảo | Thần Luân Hư Thiên Đỉnh |
| 7 | Mộ Lan Thảo Nguyên | Kết Nguyên Anh, bảo vệ Thiên Nam | như trên | Thánh nữ Mộ Lan | Lục Dực Sương Ngô |
| 8 | Âm Minh Chi Địa | Quỷ vật, luyện Tịch Tà Thần Lôi | như trên | Quỷ vương Âm Minh | Đề Hồn Thú |
| 9 | Đại Tấn – Tiết điểm phi thăng | Hóa Thần, phi thăng Linh Giới | như trên | Lão tổ Âm La Tông | Mở Linh Giới |
| 10 | Loạn lưu không gian | Tu vi bị áp chế, rơi gần Thiên Uyên Thành | Hóa Thần → Luyện Hư → Hợp Thể → Đại Thừa | Không gian thú | — |
| 11 | Thiên Uyên Thành | Phòng thủ pháo đài, gặp lại Ngân Nguyệt | như trên | Thống lĩnh Giác Xi | Đồng đội Ngân Nguyệt |
| 12 | Huyết Thiên Đại Lục | Dạ Xoa, Mộc Tộc; Phạm Thánh Chân Ma Công | như trên | Vua Dạ Xoa | Cánh Phong Lôi Sí |
| 13 | Ma Giới thông đạo | Ma Giới xâm lược, đoàn tụ Tử Linh | Luyện Hư → Hợp Thể → Đại Thừa → Độ Kiếp | Thủy Tổ Nguyên Yểm | — |
| 14 | Thiên Kiếp phi thăng | Đánh bại Thủy Tổ Ma Tộc, phi thăng Tiên Giới | như trên | Thủy Tổ Bảo Hoa & Lục Cực | Mở Tiên Giới |
| 15 | Bắc Hàn Tiên Vực | Bị ám toán, mất trí nhớ (khóa một số nâng cấp Động Phủ trong trận) | Chân Tiên → Kim Tiên → Thái Ất → Đại La | Tiên binh Thiên Đình | — |
| 16 | Hắc Thổ Tiên Vực | Khôi phục ký ức, Thời Gian Pháp Tắc | như trên | Tiên tướng Thiên Đình | Thẻ Thời Gian Pháp Tắc |
| 17 | Luân Hồi Điện | Điện Chủ là phân thân/kiếp trước; Man Hoang, Tích Lân Không Cảnh | như trên | Thủ lĩnh Man Hoang | Đồng đội Nam Cung Uyển |
| 18 | Bí mật Chưởng Thiên Bình | Âm mưu luân hồi của Thiên Đình | Kim Tiên → Thái Ất → Đại La → Đạo Tổ | Chấp pháp Thiên Đình | Chưởng Thiên Bình cấp 3 |
| 19 | Trung Thổ – Thiên Đình | Chém Cổ Hoặc Kim, thành Chí Cao Đạo Tổ | như trên | Cổ Hoặc Kim | Kết thúc, chế độ Vô Tận |

Tên boss phụ là đề xuất cho game, không phải nhân vật cụ thể của nguyên tác.

### 2.2. Bản đồ (tileset mỗi Bí Cảnh)

- **Nhân Giới:** Thiên Nam (núi rừng, tông môn), Loạn Tinh Hải (đảo, biển), Mộ Lan Thảo Nguyên (thảo nguyên), Âm Minh Chi Địa (sương tím, mộ địa), Đại Tấn (chùa cổ, ma cung).
- **Linh Giới:** Thiên Uyên Thành (tường thành, sấm sét), Huyết Thiên/Huyền Vũ Đại Lục (đất đỏ, rừng cổ thụ), Ma Giới (dung nham).
- **Tiên Giới:** Bắc Hàn/Hắc Thổ Tiên Vực (băng tuyết/đất đen), Man Hoang Giới Vực, Tích Lân Không Cảnh (không gian vỡ), Trung Thổ & Luân Hồi Điện (cung điện vàng trắng).

### 2.3. Nhân vật

| Nhân vật | Vai trò trong game |
|---|---|
| Hàn Lập | Nhân vật điều khiển |
| Nam Cung Uyển | Hội thoại; đồng đội hỗ trợ từ ch.17 (hồi sinh 1 lần/trận) |
| Tử Linh Tiên Tử (Uông Ngưng) | Hội thoại ch.5, 13 |
| Nguyên Dao | Hội thoại ch.8–11 |
| Ngân Nguyệt | Đồng đội hỗ trợ từ ch.11 (tăng tầm dò phi kiếm) |
| Đại Diễn Thần Quân | NPC mở nhánh nâng cấp Đại Diễn Quyết |
| Mặc Đại Phu | Boss ch.2 |
| Lệ Phi Vũ | NPC ch.1 |
| Cổ Hoặc Kim | Boss cuối |

Thế lực: Thất Huyền Môn, Thất Đại Phái Việt Quốc (Hoàng Phong Cốc…), Ma Đạo Lục Tông, Tinh Cung / Nghịch Tinh Minh / Cực Âm Đảo, tông môn Đại Tấn, Thiên Uyên Thành, Thiên Đình, Luân Hồi Điện, Man Hoang Bát Đại Tộc.

---
## 3. GAMEPLAY

### 3.1. Điều khiển (cảm ứng, đa chạm)

| Thao tác | Hành động |
|---|---|
| Chạm + kéo nửa trái màn hình | Floating Joystick: di chuyển |
| `Btn_Dash` | Lướt né đòn (có Phong Lôi Sí: để lại vệt sét gây sát thương) |
| `Btn_Transform` | Biến Thân, sáng khi đầy Nộ |
| `Btn_Tribulation` | Độ Kiếp, chỉ hiện khi đủ Tu Vi |
| Tự động | Phi kiếm tự tìm và đánh quái |

### 3.2. Vòng lặp trong trận

1. **Khởi đầu:** giữa Bí Cảnh, 3 phi kiếm bay quỹ đạo, cảnh giới = `runRealms[0]` của chương.
2. **Chiến đấu:** người chơi né; phi kiếm dò quái trong `DetectionRadius`, lao tới, quay về.
3. **Linh Châu & Kỳ Ngộ:** quái rơi Linh Châu (EXP) và Linh Thạch. Đầy EXP → `Time.timeScale = 0`, hiện 3 thẻ Kỳ Ngộ.
4. **Độ Kiếp:** đủ Tu Vi → nút hiện. Sống sót 10 s Lôi Kiếp → lên bậc kế, hồi đầy HP, mở Biến Thân mới. Tối đa 3 lần/trận.
5. **Boss:** phút 5 boss tinh anh, phút 10 boss chương. Diệt boss chương = qua Bí Cảnh.
6. **Kết thúc:** `ResultPopup`.

**Rút lui (Độn Thổ Phù):** trong menu tạm dừng. Rút lui giữ 100% Linh Thạch đã nhặt nhưng không qua chương. Chết không hồi sinh giữ 50%.

### 3.3. Meta (ngoài trận)

**Động Phủ – nâng cấp vĩnh viễn bằng Linh Thạch:**

| Nâng cấp | Tên công pháp | Mỗi cấp | Cấp tối đa |
|---|---|---|---|
| HP gốc | Phạm Thánh Chân Ma Công | +5% HP | 20 |
| Công gốc | Thanh Nguyên Kiếm Quyết | +4% sát thương phi kiếm | 20 |
| Tốc độ phi kiếm | Đại Diễn Quyết | +3% tốc độ bay, +2% tầm dò | 20 |
| Tỷ lệ rơi Linh Thạch | Chưởng Thiên Bình | +3% | 10 |

**Linh Căn:** Tứ (×1.00 EXP, khởi đầu) → Tam (×1.10, qua ch.3) → Song (×1.25, qua ch.9) → Thiên (×1.50, qua ch.14).

**Kiếm Trủng – trang bị trước trận:** 1 phi kiếm, 1 Thần Luân, 1 Cánh, 1 Linh Thú.
- Thần Luân Hư Thiên Đỉnh: mỗi 20 s hút quái gần và thiêu bằng Càn Lam Băng Diễm (sát thương + chậm).
- Cánh Phong Lôi Sí: Dash xa hơn, để lại vệt sét.

**Chưởng Thiên Bình & Luyện Đan:**
- Tích 1 giọt Lục Dịch mỗi 8 giờ thực (cấp 2: 6 h, mở ch.10; cấp 3: 4 h, mở ch.18), tối đa 3 giọt. Thời gian lấy từ `TimeService`.
- Linh điền 3 ô. Linh thảo chín sau 24 h thực, hoặc tưới 1 giọt Lục Dịch để chín ngay.
- Mỗi trận dùng tối đa 1 viên đan:

| Đan | Hiệu quả trong trận |
|---|---|
| Trúc Cơ Đan | Lần Độ Kiếp đầu cần ít hơn 20% Tu Vi |
| Giáng Trần Đan | Lôi Kiếp còn 8 s |
| Tuyết Phách Hoàn | Bắt đầu trận với 1 thẻ Kỳ Ngộ ngẫu nhiên |
| Huyền Thiên Đan | Hồi sinh 1 lần không cần quảng cáo |
| Định Nhan Đan | Chỉ đổi ngoại hình |

**Linh Thú:**

| Linh thú | Mở | Hành vi | Tiến hóa |
|---|---|---|---|
| Huyết Ngọc Tri Nhện | ch.3 | Nhả tơ làm chậm, +10% Linh Thạch | 1 giai đoạn |
| Phệ Kim Trùng | ch.5 | Bầy trùng bám quái, giảm giáp, ăn khiên boss | 3 giai đoạn (Lục Dịch + Linh Thạch) |
| Lục Dực Sương Ngô | ch.7 | Định kỳ đóng băng vùng | 2 giai đoạn |
| Đề Hồn Thú | ch.8 | ×2 sát thương lên quỷ vật/ma | nhiều giai đoạn |

**Chế độ Vô Tận:** mở sau ch.19, không giới hạn thời gian, xếp hạng theo thời gian sống.

---

## 4. TU LUYỆN

Hai lớp tiến trình:
- **Cảnh giới trong trận:** 4 bậc, reset mỗi trận.
- **Cảnh giới cốt truyện:** lưu trong tài khoản = bậc cao nhất của chương đã qua. Hiện ở Lobby, mở chương. Không cộng chỉ số.

14 cảnh giới: Nhân Giới (1 Luyện Khí, 2 Trúc Cơ, 3 Kết Đan, 4 Nguyên Anh, 5 Hóa Thần), Linh Giới (6 Luyện Hư, 7 Hợp Thể, 8 Đại Thừa, 9 Độ Kiếp), Tiên Giới (10 Chân Tiên, 11 Kim Tiên, 12 Thái Ất, 13 Đại La, 14 Đạo Tổ).

### 4.1. Lôi Kiếp

1. Tu Vi đủ ngưỡng → `Btn_Tribulation` hiện và rung.
2. Bấm: màn hình tối, quái vẫn đánh.
3. 10 s sét đánh quanh người chơi, mỗi tia có vòng cảnh báo đỏ 0.8 s.
4. Sống sót: lên `runRealms[n+1]`, hồi đầy HP, mở Biến Thân mới.
5. Chết trong Lôi Kiếp: hiện `RevivePopup`.

Số tia, sát thương xem mục 9.8. Nhân theo giới diện `tribulationMultiplier`: Nhân 1, Linh 1.5, Tiên 2.

### 4.2. Biến Thân

Bấm `Btn_Transform` khi đầy Nộ (100) → vào trạng thái cao nhất đã mở trong 12 s, tiêu hết Nộ. HUD đổi màu.

| Trạng thái | Ý nghĩa | Tốc độ kiếm | Sát thương | Màu HUD |
|---|---|---|---|---|
| Thanh Lôi | Lôi lực Thanh Trúc Phong Vân Kiếm | ×1.5 | ×1.2 | `#3FD18F` |
| Kim Lôi | Tịch Tà Thần Lôi (×2 lên ma, quỷ) | ×2.0 | ×1.4 | `#F5C542` |
| Hồng Lôi | Thiên Kiếp chi lôi | ×2.5 | ×1.7 | `#E5484D` |

---

## 5. CHIẾN ĐẤU

### 5.1. Phi kiếm

State machine mỗi kiếm: `Orbiting` → (có quái trong `DetectionRadius`) → `Attacking` → (trúng / hết tầm) → `Returning` → `Orbiting`. Giới hạn 36 kiếm trên màn hình.

### 5.2. Thuộc tính

| Thuộc tính | ×1.5 với | ×0.7 với |
|---|---|---|
| Lôi | Ma tu, quỷ vật, Ma Tộc | Yêu thú hệ Thổ |
| Hỏa | Mộc Tộc, quái Băng | Yêu thú biển |
| Băng | Yêu thú biển, quái Hỏa | Ma Tộc |

Kim Lôi (Tịch Tà Thần Lôi): ×2.0 lên ma tu, quỷ vật.

### 5.3. Tiến hóa Kỳ Ngộ

| Tiến hóa | Điều kiện | Hiệu quả |
|---|---|---|
| Đại Canh Kiếm Trận | Trang bị Thanh Trúc Phong Vân Kiếm + Thanh Nguyên Kiếm Quyết cấp 5 + ≥ 12 kiếm | Mỗi 25 s, mọi kiếm bày trận quanh người, chém liên tục quái trong vùng 4 s. Hiển thị 72 kiếm bằng 1 Particle System |
| Tịch Tà Thần Lôi | Lôi Văn cấp 5 + đã mở Kim Lôi | Sét vàng nhảy 5 mục tiêu, xóa buff quái hệ Ma |
| Càn Lam Băng Diễm | Thần Luân Hư Thiên Đỉnh + Hỏa Văn cấp 5 | Lửa lam vừa thiêu vừa đóng băng, lan khi quái chết |

Thẻ Thời Gian Pháp Tắc (chỉ Tiên Giới): mỗi 15 s ngưng đọng quái trong vùng 2 s.

### 5.4. Boss các chương sau (chưa có số liệu)

| Boss | Ch. | Cơ chế |
|---|---|---|
| Thủy Tổ Nguyên Yểm | 13 | Triệu ma binh, phủ ma khí giảm tầm nhìn |
| Bảo Hoa & Lục Cực | 14 | Hai boss cùng lúc |
| Cổ Hoặc Kim | 19 | Ngưng đọng người chơi 1.5 s (màn hình mất màu dần 1 s trước), tua ngược máu 1 lần ở 30% |

Mọi đòn diện rộng của boss có telegraph ≥ 0.6 s.

---
## 6. VFX

Unity không đọc file Niagara `.uasset`. Hiệu ứng render ra flipbook trong Unreal rồi phát bằng Particle System. Asset Unreal Marketplace/Fab thường chỉ được dùng trong Unreal: kiểm tra license.

**Unreal:** dựng Niagara trên nền đen tuyệt đối, xuất bằng Niagara Baker:
- Ngắn (vệt chém, tia lửa): lưới 4×4, 1024×1024.
- Lớn (cột sét Lôi Kiếp, hào quang Hồng Lôi): lưới 6×6, 2048×2048.

**Unity:**
- Texture: Android override, ASTC 6x6 (tự áp bởi `VkdkTexturePostprocessor`).
- Material: `Universal Render Pipeline/Particles/Unlit`, Transparent, Blending Mode **Additive**, HDR Color Intensity 1.8–2.5. **Không** dùng `Mobile/Particles/Additive` (shader Built-in, hiện hồng trên URP).
- Prefab Particle System + Texture Sheet Animation (Grid) + `AutoReturnToPool`. `VkdkFxTool` tạo tự động.
- Bloom: URP Asset bật HDR, Camera bật Post Processing, Global Volume có Bloom (Threshold ~1, tắt High Quality Filtering).

### 6.1. Danh sách hiệu ứng

| Mã | Hiệu ứng | Cách làm | Lưới |
|---|---|---|---|
| VFX_Slash | Vệt chém phi kiếm | Flipbook, màu theo thuộc tính | 4×4 |
| VFX_Hit | Tia lửa trúng đòn | Flipbook + hạt | 4×4 |
| VFX_SwordTrail | Vệt sau kiếm | Trail Renderer | — |
| VFX_ChainLightning | Sét lan / Tịch Tà Thần Lôi | Line Renderer + flipbook | 4×4 |
| VFX_Burn / VFX_Freeze | Thiêu / đóng băng | Flipbook lặp trên quái | 4×4 |
| VFX_DaiCanhKiemTran | 72 kiếm bày trận | 1 Particle System 72 sprite kiếm + kiếm võng vàng | 6×6 |
| VFX_HuThienDinh | Hư Thiên Đỉnh hút quái | Sprite + xoáy + lửa lam | 6×6 |
| VFX_PhongLoiSi | Lướt sét | Flipbook kéo dài | 4×4 |
| VFX_TimeStop | Thời Gian Pháp Tắc | Vòng đồng hồ + giảm bão hòa | 4×4 |
| VFX_Tribulation_Bolt | Cột sét Lôi Kiếp | Flipbook + flash + rung camera | 6×6 |
| VFX_Telegraph | Vòng cảnh báo đỏ | Sprite vòng fill theo thời gian | — |
| VFX_Breakthrough | Đột phá | Cột sáng + sóng xung kích | 6×6 |
| VFX_Aura_ThanhLoi/KimLoi/HongLoi | Hào quang Biến Thân | Flipbook lặp | 6×6 |
| VFX_PheKimTrung | Bầy trùng | Particle System + Noise | — |
| VFX_DeHonThu | Hút hồn | Flipbook | 4×4 |
| VFX_SuongNgo_Frost | Băng lan | Flipbook | 4×4 |
| VFX_LinhChau_Pickup | Nhặt Linh Châu | Hạt bay về người chơi | — |
| VFX_ChuongThienBinh | Bình tích Lục Dịch (UI) | UI Particle | 4×4 |
| VFX_Revive | Hồi sinh | Sóng đẩy + vòng bất tử | 6×6 |

Màu: Lôi xanh lục ngọc; Kim Lôi vàng kim; Hồng Lôi đỏ cam; Hỏa cam đỏ; Băng lam nhạt; Ma khí đỏ máu/tím đen (blend **Alpha**, không Additive); Thời Gian vàng nhạt – bạc.

Ngân sách: ≤ 200 hạt/Particle System (bầy trùng ≤ 300), ~25 hiệu ứng đồng thời, flipbook tối đa 2048², hào quang ≤ 1/3 chiều cao màn hình.

---

## 7. UI & QUẢNG CÁO

Canvas: Scale With Screen Size, 1920×1080, Match 0.5. `SafeAreaHandler.cs` trên Panel gốc.

### 7.1. Màn hình

- **`LobbyUI`:** Top Bar trái avatar + cảnh giới cốt truyện; phải Linh Thạch + nút `+` (QC) + Cài đặt. Giữa: Hàn Lập thiền, kiếm xoay, linh thú. Bottom: `Kiếm Trủng`, `Động Phủ`, nút lớn `XUẤT CHINH`. Banner dưới cùng, ẩn khi vào trận.
- **`ChapterSelectUI`:** bản đồ cuộn ngang 3 giới diện; chương đã qua chơi lại được; ô chọn 1 viên đan.
- **`DialogueUI`:** chân dung trái/phải, khung chữ dưới, chạm để tiếp, nút `Bỏ qua`.
- **`CombatHUD`:** Top-Left `CharacterStatusHUD` (avatar, HP, EXP, tên cảnh giới; viền đổi màu theo Biến Thân). Top-Center đồng hồ + thanh máu boss. Top-Right nút tạm dừng (có Độn Thổ Phù). Bottom-Left `FloatingJoystick`. Bottom-Right `Btn_Dash`, `Btn_Transform`, `Btn_Tribulation`.
- **`UpgradeCardPopup`:** 3 thẻ; thẻ tiến hóa viền vàng.
- **`RevivePopup`:** đếm 5 s; nếu có Huyền Thiên Đan thì thêm nút hồi sinh miễn phí.
- **`ResultPopup`:** thời gian, cảnh giới, tổng sát thương, Linh Thạch, linh thảo; nút `Nhận Thường`.

### 7.2. Điểm đặt quảng cáo

| # | Loại | Vị trí | Điều kiện | Thưởng |
|---|---|---|---|---|
| — | Banner | Lobby | Ẩn khi vào trận | — |
| — | Rewarded | Lobby nút `+` | Tối đa 5 lần/ngày | +50 Linh Thạch |
| 1 | Rewarded | `UpgradeCardPopup` | `Làm Mới (Xem QC)` | Random lại 3 thẻ |
| 2 | Rewarded | `UpgradeCardPopup` | `Nhận Cả 3 (Xem QC)`, 1 lần/trận | Nhận cả 3 |
| 3 | Rewarded | `RevivePopup` | `Dùng Đan Hồi Sinh (Xem QC)`, 1 lần/trận | 100% HP, đẩy lùi, bất tử 3 s |
| 4 | Rewarded | `ResultPopup` | `Nhận x2 Linh Thạch (Xem QC)` | x2 Linh Thạch |
| 5 | Interstitial | Result → Lobby | Cooldown ≥ 180 s; bỏ ván đầu; không ngay sau Rewarded | — |

Không hiện Interstitial giữa hội thoại hoặc trước trận boss.

---

## 8. BỘ UI & QUY TRÌNH ASSET

### 8.1. Màu (ScriptableObject `UITheme`)

| Token | Mã | Dùng |
|---|---|---|
| Jade.Panel | `#0D2622E6` | Nền khung |
| Gold.Border | `#D4AF37` | Viền, tiêu đề |
| Text.Main | `#F2E8D0` | Chữ chính |
| Text.Muted | `#9FB5AE` | Chữ phụ |
| Bar.HP | `#C0392B` | Máu |
| Bar.EXP | `#3FC1A5` | Linh Lực |
| Bar.Rage | `#E8A33D` | Nộ |
| Theme.ThanhLoi / KimLoi / HongLoi | `#3FD18F` / `#F5C542` / `#E5484D` | Viền HUD khi Biến Thân |

Viền khung vẽ trắng/xám để code tô màu bằng `Image.color`.

### 8.2. Font

TextMeshPro, font hỗ trợ tiếng Việt (gợi ý Be Vietnam Pro / Noto Serif, giấy phép SIL OFL). Font Asset Dynamic. Viền/bóng chữ bằng TMP Material Preset, **không** dùng component Outline/Shadow. Cỡ: tiêu đề 56, nút 44, nội dung 32, chú thích 26, số sát thương 40 (chí mạng 56).

### 8.3. Ảnh UI cần vẽ (tại 1920×1080)

| Thành phần | File | Kích thước | Kiểu Image |
|---|---|---|---|
| Khung pop-up | `ui_panel_jade_9s64.png` | 256×256 | Sliced |
| Khung nhỏ | `ui_frame_small_9s24.png` | 96×96 | Sliced |
| Nút thường (3 trạng thái) | `ui_btn_normal_9s40.png`, `_pressed`, `_disabled` | 320×112 | Sliced |
| Nút XUẤT CHINH | `ui_btn_battle.png` + `ui_btn_battle_glow_fx.png` | 480×160 | Simple |
| Nút kỹ năng | `ui_skill_ring.png` | 200×200 | Simple |
| Lớp cooldown | `ui_skill_cooldown.png` | 200×200 | Filled Radial 360 |
| Khung thanh | `ui_bar_frame_9s20.png` | 600×48 | Sliced |
| Ruột thanh (trắng) | `ui_bar_fill.png` | 560×32 | Filled Horizontal |
| Thẻ Kỳ Ngộ | `ui_card_common/_rare/_evolve.png` | 420×600 | Simple |
| Avatar | `ui_avatar_frame.png` + `ui_avatar_mask.png` | 160×160 | Simple + Mask |
| Icon | `icon_*.png` | 128×128 | Simple |
| Icon tiền | `icon_linhthach.png`, `icon_linhchau.png` | 64×64 | Simple |
| Icon QC | `icon_ad_video.png` | 48×48 | Simple |
| Joystick | `ui_joy_base.png`, `ui_joy_handle.png` | 300 / 120 | Simple |
| Chân dung | `portrait_*.png` | cao 1024 | Simple |
| Khung hội thoại | `ui_dialogue_9s48.png` | 256×160 | Sliced |
| Điểm chương | `ui_chapter_node_locked/open/cleared.png` | 128×128 | Simple |

### 8.4. Prefab dùng chung (`Prefabs/UI/Kit/`, dùng Prefab Variant)

| Prefab | Script |
|---|---|
| `UI_Panel` | `UIPanel` (tween scale 0.9→1 trong 0.15 s, unscaled time) |
| `UI_Button` → `UI_Button_Primary`, `UI_Button_Ad` | `UIButton` (âm thanh, chống bấm liên tục 0.3 s) |
| `UI_SkillButton` | `SkillButton` (`fillAmount`, sáng/rung) |
| `UI_Bar` | `UIBar` (`SetValue(float normalized)`, ruột trễ) |
| `UI_Card` | `UpgradeCardUI` |
| `UI_CurrencyCounter` | `CurrencyCounter` |
| `UI_Avatar` | — |
| `UI_DamagePopup` | `DamagePopup` (pool, tối đa 30) |
| `UI_Joystick` | `FloatingJoystick` |
| `UI_Toast` | `Toast` |

`ThemeTint` (gắn trên Image cần đổi màu):

```csharp
using UnityEngine;
using UnityEngine.UI;

/// <summary>Đổi màu Image theo trạng thái Biến Thân.</summary>
public class ThemeTint : MonoBehaviour
{
    [SerializeField] private Image target;

    private void OnEnable()  => TransformationHandler.OnTransformationChanged += Apply;
    private void OnDisable() => TransformationHandler.OnTransformationChanged -= Apply;

    private void Apply(Color hudColor) => target.color = hudColor;
}
```

Sprite Atlas: `Atlas_UI_Common` (luôn nạp), `Atlas_UI_Lobby`, `Atlas_UI_Combat`, `Atlas_Icons`. Chân dung không vào atlas. Atlas: ASTC 4x4, không mip, Padding 4.

Hiệu năng UI: tách `Canvas_Static` / `Canvas_Dynamic` / `Canvas_Popup`; tắt Raycast Target ở phần trang trí; không Layout Group trên phần tử cập nhật mỗi frame; pop-up ẩn bằng `CanvasGroup`, không Instantiate. Scene kiểm tra: `Scenes/UIKit_Preview`.

### 8.5. Quy trình PNG

| Loại PNG | Tiền xử lý | Thư mục | Hiển thị |
|---|---|---|---|
| Phi kiếm, pháp bảo, vật phẩm | Tách nền; kiếm vẽ mũi hướng **lên** | `Art/Sprites/Weapons/`, `Items/` | SpriteRenderer + `Sprite-Lit-Default` |
| Quái, boss | Tách nền | `Art/Sprites/Enemies/` | `Sprite-Lit-Default` |
| Nhân vật có chuyển động | Tách layer, lưu `.psb` | `Art/Characters/` | 2D PSD Importer → Skinning Editor |
| Hiệu ứng nền đen | **Không** tách nền | `Art/VFX_Unreal/` | Particle System + `M_FX_Additive_Template` |
| Hiệu ứng blend Alpha (ma khí) | Unmultiply → `_alpha.png` | `Art/VFX_Unreal/` | Particle System, Alpha |
| Tấm nhiều icon | — | `Art/UI/Icons/` | Sprite Editor → Slice |
| Khung UI | — | `Art/UI/` | UI Image Sliced |

Pivot phi kiếm: **giữa** (code dùng `transform.up = direction`). Hậu tố `_hilt` = pivot ở chuôi (kiếm vung cận chiến).

**Quy ước tên file** (chữ thường, không dấu, `_`):

| Hậu tố | Ý nghĩa |
|---|---|
| `_CxR` (vd `_4x4`) | Flipbook C cột × R hàng |
| `_fx` | Sprite hiệu ứng tĩnh dùng SpriteRenderer + `SG_Sprite_Additive` |
| `_9s` + số | 9-slice, viền bằng số px |
| `_hilt` | Pivot giữa mép dưới |
| `_px` | Pixel art: Point, không nén |

**Material dùng chung (tạo tay 1 lần):** `M_FX_Additive_Template` (Particles/Unlit, Transparent, Additive, HDR Intensity 2), `M_FX_Alpha_Template` (như trên, Alpha), `SG_Sprite_Additive` (Shader Graph Sprite Unlit, Blend Additive, `_Tint` HDR; nếu bản URP không có Blend Additive cho Sprite Unlit thì dùng Particle System 1 hạt).

**Bảng import:**

| Thư mục / hậu tố | Type | Filter | Mip | Alpha Is Transparency | Android |
|---|---|---|---|---|---|
| `Art/UI/**`, `Art/Sprites/**` | Sprite, PPU 100 | Bilinear | Tắt | Bật | ASTC 4x4, Max 2048 |
| `Art/VFX_Unreal/**` | Default | Bilinear | Tắt | Tắt | ASTC 6x6, Max 2048 |
| `_fx` | Sprite | Bilinear | Tắt | Tắt | ASTC 6x6 |
| `_px` | Sprite | Point | Tắt | Bật | RGBA32 |

Mọi texture: Wrap Mode Clamp.

### 8.6. Công cụ Editor (`Assets/_Project/Editor/AssetTools/`)

**`VkdkTexturePostprocessor.cs`**

```csharp
using System.IO;
using System.Text.RegularExpressions;
using UnityEditor;
using UnityEngine;

/// <summary>Tự áp thiết lập import cho PNG trong Assets/_Project/Art theo thư mục và hậu tố.</summary>
public class VkdkTexturePostprocessor : AssetPostprocessor
{
    private const string Root = "Assets/_Project/Art/";

    private void OnPreprocessTexture()
    {
        if (!assetPath.StartsWith(Root)) return;
        var importer = (TextureImporter)assetImporter;
        if (!importer.importSettingsMissing) return; // chỉ lần import đầu

        string file = Path.GetFileNameWithoutExtension(assetPath).ToLowerInvariant();
        bool isPixel = file.Contains("_px");
        bool isFxSprite = file.Contains("_fx");

        importer.mipmapEnabled = false;
        importer.wrapMode = TextureWrapMode.Clamp;
        importer.filterMode = isPixel ? FilterMode.Point : FilterMode.Bilinear;

        if (assetPath.StartsWith(Root + "VFX_Unreal/"))
        {
            importer.textureType = TextureImporterType.Default;
            importer.alphaIsTransparency = false;
            SetAndroid(importer, TextureImporterFormat.ASTC_6x6, 2048);
            return;
        }

        importer.textureType = TextureImporterType.Sprite;
        importer.spriteImportMode = SpriteImportMode.Single;
        importer.spritePixelsPerUnit = 100;
        importer.alphaIsTransparency = !isFxSprite;

        var settings = new TextureImporterSettings();
        importer.ReadTextureSettings(settings);
        settings.spriteAlignment = file.Contains("_hilt")
            ? (int)SpriteAlignment.BottomCenter
            : (int)SpriteAlignment.Center;

        Match border = Regex.Match(file, @"_9s(\d+)");
        if (border.Success)
        {
            float b = int.Parse(border.Groups[1].Value);
            settings.spriteBorder = new Vector4(b, b, b, b);
            settings.spriteMeshType = SpriteMeshType.FullRect;
        }
        importer.SetTextureSettings(settings);

        if (isPixel)
            SetAndroid(importer, TextureImporterFormat.RGBA32, 2048);
        else
            SetAndroid(importer, isFxSprite ? TextureImporterFormat.ASTC_6x6 : TextureImporterFormat.ASTC_4x4, 2048);
    }

    private static void SetAndroid(TextureImporter importer, TextureImporterFormat format, int maxSize)
    {
        TextureImporterPlatformSettings android = importer.GetPlatformTextureSettings("Android");
        android.overridden = true;
        android.format = format;
        android.maxTextureSize = maxSize;
        importer.SetPlatformTextureSettings(android);
    }
}
```

**`VkdkFxTool.cs`** (menu *Tools → VKDK → FX Tool*): ba nút.
1. *Tạo Material + Particle Prefab*: với mỗi texture đang chọn, đọc lưới từ tên (`_(\d+)x(\d+)`), nhân bản material template, gán `_BaseMap`, lưu `Materials/FX/M_<tên>.mat`; tạo GameObject có `ParticleSystem` (duration = lifetime, không loop, startSpeed 0, maxParticles 1, burst 1 hạt lúc 0, tắt Shape, `stopAction = Callback`, Texture Sheet Animation Grid đúng lưới), gán material, thêm `AutoReturnToPool`, lưu `Prefabs/VFX/VFX_<tên>.prefab`.
2. *Unmultiply*: đọc PNG bằng `File.ReadAllBytes` + `Texture2D.LoadImage`; mỗi pixel `a = max(r,g,b)`, nếu `a = 0` thì trong suốt, ngược lại `rgb = rgb × 255 / a`; ghi `<tên>_alpha.png`.
3. *Kiểm tra ngân sách*: quét `Assets/_Project/Art`, cảnh báo tên không khớp `^[a-z0-9_]+\.(png|psb)$`, Max Size > 2048, chưa override Android, Sprite bật mip map.

```csharp
using System.IO;
using System.Text.RegularExpressions;
using UnityEditor;
using UnityEngine;

/// <summary>Tạo Material + Prefab VFX từ flipbook, Unmultiply nền đen, kiểm tra ngân sách texture.</summary>
public class VkdkFxTool : EditorWindow
{
    private const string MaterialDir = "Assets/_Project/Materials/FX";
    private const string PrefabDir = "Assets/_Project/Prefabs/VFX";
    private const string ArtDir = "Assets/_Project/Art";

    private Material template;
    private int defaultCols = 4, defaultRows = 4;
    private float lifetime = 0.5f;

    [MenuItem("Tools/VKDK/FX Tool")]
    private static void Open() => GetWindow<VkdkFxTool>("VKDK FX Tool");

    private void OnGUI()
    {
        template = (Material)EditorGUILayout.ObjectField("Material mẫu", template, typeof(Material), false);
        defaultCols = EditorGUILayout.IntField("Số cột (nếu tên không có _CxR)", defaultCols);
        defaultRows = EditorGUILayout.IntField("Số hàng (nếu tên không có _CxR)", defaultRows);
        lifetime = EditorGUILayout.FloatField("Thời gian chạy (giây)", lifetime);
        EditorGUILayout.HelpBox("Chọn một hoặc nhiều texture trong cửa sổ Project rồi bấm nút.", MessageType.Info);

        using (new EditorGUI.DisabledScope(template == null))
        {
            if (GUILayout.Button("Tạo Material + Particle Prefab")) CreateVfxForSelection();
        }
        if (GUILayout.Button("Unmultiply nền đen → alpha (tạo file _alpha.png)")) UnmultiplySelection();
        if (GUILayout.Button("Kiểm tra ngân sách texture")) ValidateTextures();
    }

    private void CreateVfxForSelection()
    {
        Directory.CreateDirectory(MaterialDir);
        Directory.CreateDirectory(PrefabDir);

        foreach (Texture2D tex in Selection.GetFiltered<Texture2D>(SelectionMode.Assets))
        {
            string name = Path.GetFileNameWithoutExtension(AssetDatabase.GetAssetPath(tex));
            int cols = defaultCols, rows = defaultRows;
            Match grid = Regex.Match(name, @"_(\d+)x(\d+)");
            if (grid.Success) { cols = int.Parse(grid.Groups[1].Value); rows = int.Parse(grid.Groups[2].Value); }

            var mat = new Material(template);
            mat.SetTexture("_BaseMap", tex);
            AssetDatabase.CreateAsset(mat, $"{MaterialDir}/M_{name}.mat");

            var go = new GameObject("VFX_" + name);
            var ps = go.AddComponent<ParticleSystem>();
            ps.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);

            ParticleSystem.MainModule main = ps.main;
            main.duration = lifetime;
            main.loop = false;
            main.startLifetime = lifetime;
            main.startSpeed = 0f;
            main.maxParticles = 1;
            main.stopAction = ParticleSystemStopAction.Callback;

            ParticleSystem.EmissionModule emission = ps.emission;
            emission.rateOverTime = 0f;
            emission.SetBursts(new[] { new ParticleSystem.Burst(0f, 1) });

            ParticleSystem.ShapeModule shape = ps.shape;
            shape.enabled = false;

            ParticleSystem.TextureSheetAnimationModule sheet = ps.textureSheetAnimation;
            sheet.enabled = true;
            sheet.mode = ParticleSystemAnimationMode.Grid;
            sheet.numTilesX = cols;
            sheet.numTilesY = rows;

            go.GetComponent<ParticleSystemRenderer>().sharedMaterial = mat;
            go.AddComponent<AutoReturnToPool>();

            PrefabUtility.SaveAsPrefabAsset(go, $"{PrefabDir}/VFX_{name}.prefab");
            DestroyImmediate(go);
        }
        AssetDatabase.SaveAssets();
    }

    private static void UnmultiplySelection()
    {
        foreach (Texture2D tex in Selection.GetFiltered<Texture2D>(SelectionMode.Assets))
        {
            string path = AssetDatabase.GetAssetPath(tex);
            var img = new Texture2D(2, 2, TextureFormat.RGBA32, false);
            img.LoadImage(File.ReadAllBytes(path));

            Color32[] px = img.GetPixels32();
            for (int i = 0; i < px.Length; i++)
            {
                Color32 c = px[i];
                int a = Mathf.Max(c.r, Mathf.Max(c.g, c.b));
                px[i] = a == 0
                    ? new Color32(0, 0, 0, 0)
                    : new Color32((byte)(c.r * 255 / a), (byte)(c.g * 255 / a), (byte)(c.b * 255 / a), (byte)a);
            }
            img.SetPixels32(px);
            img.Apply();

            string outPath = Path.Combine(Path.GetDirectoryName(path), Path.GetFileNameWithoutExtension(path) + "_alpha.png");
            File.WriteAllBytes(outPath, img.EncodeToPNG());
            DestroyImmediate(img);
            AssetDatabase.ImportAsset(outPath.Replace('\\', '/'));
        }
    }

    private static void ValidateTextures()
    {
        int warnings = 0;
        foreach (string guid in AssetDatabase.FindAssets("t:Texture2D", new[] { ArtDir }))
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            if (AssetImporter.GetAtPath(path) is not TextureImporter importer) continue;
            TextureImporterPlatformSettings android = importer.GetPlatformTextureSettings("Android");
            string file = Path.GetFileName(path);

            if (!Regex.IsMatch(file, @"^[a-z0-9_]+\.(png|psb)$"))
                { Debug.LogWarning($"Tên file nên viết thường, không dấu, không khoảng trắng: {path}"); warnings++; }
            if (android.overridden && android.maxTextureSize > 2048)
                { Debug.LogWarning($"Max Size > 2048: {path}"); warnings++; }
            if (!android.overridden)
                { Debug.LogWarning($"Chưa override Android (chưa nén ASTC): {path}"); warnings++; }
            if (importer.mipmapEnabled && importer.textureType == TextureImporterType.Sprite)
                { Debug.LogWarning($"Sprite đang bật mip map: {path}"); warnings++; }
        }
        Debug.Log($"VKDK: kiểm tra xong, {warnings} cảnh báo.");
    }
}
```

`AutoReturnToPool` (runtime) phải có trước khi biên dịch file này.

---
## 9. CÂN BẰNG (CHƯƠNG 1–4)

Số khởi điểm, chưa playtest. Đơn vị khoảng cách: unit (PPU 100). Thời gian: giây.

### 9.1. Mục tiêu

| Chương | Người chơi mới | Nâng cấp Động Phủ cần để qua dễ |
|---|---|---|
| 1 | Qua trong 1–2 lần | 0 |
| 2 | 2–3 lần | ~4 cấp tổng |
| 3 | Thường thua phút 8–10 | ~10 cấp tổng |
| 4 | Rất khó | ~18 cấp + 1 phi kiếm nâng cấp |

### 9.2. Chỉ số gốc (`PlayerBaseStats`)

| Chỉ số | Giá trị |
|---|---|
| HP | 100 |
| Công gốc | 10 |
| Tốc độ chạy | 3.5 |
| Bán kính hút Linh Châu | 1.5 |
| Bất tử sau khi bị đánh | 0.5 s |
| Dash: quãng / cooldown / bất tử | 3 / 3 s / 0.25 s |
| Phi kiếm khởi đầu | 3 (tối đa 36) |
| Bán kính quỹ đạo | 1.2 |
| Tốc độ xoay quỹ đạo | 180°/s |
| Tốc độ bay khi tấn công | 12 |
| `DetectionRadius` | 5 |
| Thời gian nghỉ mỗi kiếm | 0.8 s |
| Xuyên mục tiêu | 1 |

### 9.3. EXP, Tu Vi, Nộ

```
EXP lên cấp n → n+1 = 5 + 4 × n
Tu Vi = tổng EXP đã nhặt trong trận
Ngưỡng Độ Kiếp: 300 / 900 / 2000 Tu Vi (≈ cấp 11 / 21 / 30)
```

Linh Châu: nhỏ 1 EXP (quái thường), vừa 5 (tinh anh nhỏ; gộp 5 viên nhỏ khi > 150 viên), lớn 50 (boss tinh anh).

Nộ (tối đa 100): +0.5 mỗi đòn kiếm trúng, +4 mỗi lần bị đánh, +50 khi hạ boss tinh anh.

### 9.4. Phi kiếm

```
SatThuong = CongGoc × (1 + 0.04 × capCongDongPhu) × HeSoKiem × (1 + TongBonusKyNgo) × HeSoThuocTinh × HeSoBienThan
GiaNangKiem(cap) = 100 × 1.35^(cap − 1)        // tối đa cấp 10
```

| Phi kiếm | HeSoKiem | Hiệu ứng | Giá mở | Mỗi cấp |
|---|---|---|---|---|
| Phàm phẩm | 1.0 | — | Mặc định | +5% |
| Thanh Trúc Phong Vân Kiếm | 1.2 | 15% tia điện 0.5× sang 1 quái | Thưởng ch.4 | +6% |
| Lôi Kiếm | 1.1 | Sét lan 2 mục tiêu, 0.4× mỗi lần | 800 | +5% |
| Hỏa Kiếm | 1.0 | Thiêu 3 s, 0.2×/s | 800 | +5% |
| Băng Kiếm | 0.9 | Chậm 30% 2 s; 3 lần trúng đóng băng 1 s (boss chỉ chậm) | 1.000 | +5% |

### 9.5. Thẻ Kỳ Ngộ (tối đa cấp 5)

| Thẻ | Trọng số | Mỗi cấp |
|---|---|---|
| Phân Kiếm Thuật | 10 | +1 kiếm |
| Ngự Phong Quyết | 10 | +10% tốc độ bay, −5% thời gian nghỉ |
| Thanh Nguyên Kiếm Quyết | 10 | +12% sát thương |
| Đại Diễn Quyết | 8 | +10% `DetectionRadius` |
| Lôi Văn | 6 | Cấp 1: sét lan 1 mục tiêu 0.3×; +1 mục tiêu mỗi cấp |
| Hỏa Văn | 6 | Cấp 1: thiêu 2 s 0.15×/s; +0.05× mỗi cấp |
| Luyện Thể | 8 | +10% HP tối đa, hồi 10% HP |
| Thân Pháp | 8 | +6% tốc độ chạy, −8% cooldown Dash |
| Tẩy Luyện | 3 | Đổi 1 thẻ đang có lấy 1 thẻ khác cùng cấp |

3 thẻ không trùng; bỏ thẻ cấp 5; thẻ tiến hóa (đủ điều kiện) chắc chắn chiếm 1 ô; hết thẻ thì thay bằng "Linh Thạch +20" hoặc "Hồi 30% HP".

### 9.6. Quái

| Id | Tên | HP | Chạm | Tốc độ | Hành vi | EXP | Linh Thạch |
|---|---|---|---|---|---|---|---|
| `en_soi` | Yêu Lang | 12 | 6 | 2.2 | Lao thẳng | 1 | 8% rơi 1 |
| `en_soi_dau` | Lang Đầu Đàn | 60 | 10 | 2.6 | Cứ 4 s lao nhanh (cảnh báo 0.6 s) | 5 | 100% rơi 3 |
| `en_doc_trung` | Độc Trùng | 8 | 4 | 3.0 | Zigzag; chết để vũng độc 2 s (2 dmg/0.5 s) | 1 | 6% rơi 1 |
| `en_khoi_loi` | Khôi Lỗi Gỗ | 40 | 8 | 1.4 | Chậm, trâu | 2 | 10% rơi 1 |
| `en_huyet_doi` | Huyết Dơi | 10 | 5 | 3.4 | Bầy 6–8, lượn rồi lao | 1 | 6% rơi 1 |
| `en_ma_tu` | Đệ tử Ma Đạo | 30 | 6 (đạn) | 1.8 | Giữ cách 5, mỗi 2.5 s bắn đạn tốc độ 5 | 3 | 15% rơi 2 |
| `en_ma_tu_no` | Huyết Nô | 20 | 18 (nổ r=1.2) | 2.8 | Cách 1 thì đứng 0.6 s rồi nổ | 2 | 10% rơi 1 |

```
HP thực tế      = HP gốc × enemyStatMultiplier × (1 + 0.12 × phút)
Sát thương thực = Chạm gốc × enemyStatMultiplier × (1 + 0.05 × phút)
```

| Chương | `enemyStatMultiplier` | Thưởng qua lần đầu | Chơi lại |
|---|---|---|---|
| 1 | 1.0 | 150 | 30 |
| 2 | 1.3 | 250 | 50 |
| 3 | 1.7 | 400 | 80 |
| 4 | 2.2 | 600 | 120 |

### 9.7. Wave

Sinh quái trên vòng bán kính 11 quanh người chơi. Trần toàn game: 150 quái, 300 đạn, 150 Linh Châu.

**Ch.1:** 0–2' 3 con/2.0 s, tối đa 40, 100% Yêu Lang · 2–4' 4/1.8 s, 60, 80% Lang 20% Độc Trùng · 4–5' 5/1.6 s, 70, 70/30, phút 4:30 thêm 2 Lang Đầu Đàn · **5': Lang Vương** (ngừng sinh tới khi chết hoặc 60 s) · 5–8' 6/1.5 s, 90, 60% Lang 30% Độc Trùng 10% Khôi Lỗi · 8–10' 8/1.3 s, 110, 50/30/20 + mỗi phút 3 Lang Đầu Đàn · **10': Yêu Lang Huyết Nhãn** (quái thường sinh 50% mật độ).

**Ch.2:** 0–3' 4/1.8 s, 60, 60% Độc Trùng 40% Khôi Lỗi · 3–5' 5/1.6 s, 80, 50/30 + 20% Huyết Dơi · **5': Khôi Lỗi Đồng** · 5–10' 7/1.4 s, 110, 40/30/30 · **10': Mặc Đại Phu**.

**Ch.3:** 0–3' 5/1.6 s, 80, 50% Lang 30% Dơi 20% Độc Trùng · 3–5' 6/1.5 s, 100, 40% Lang 30% Dơi 20% Khôi Lỗi 10% Ma Tu · **5': Huyết Văn Mãng** · 5–10' 8/1.3 s, 130, 30/30/20/20 · **10': Huyết Văn Mãng Vương**.

**Ch.4:** 0–3' 6/1.5 s, 100, 60% Ma Tu 40% Huyết Nô · 3–5' 7/1.4 s, 120, 50/30 + 20% Dơi · **5': Hộ pháp Ma Diễm Tông** · 5–10' 9/1.2 s, 150, 40/35/25 · **10': Trưởng lão Hợp Hoan Tông**.

### 9.8. Boss (HP là giá trị cuối, không nhân hệ số phút)

| Boss | Ch./phút | HP | Chạm | Tốc độ | Chiêu (cooldown) | Thưởng |
|---|---|---|---|---|---|---|
| Lang Vương | 1/5 | 600 | 12 | 2.4 | Lao 8 unit (5 s, cảnh báo đường 0.8 s); hú gọi 6 Yêu Lang (12 s) | Linh Châu lớn, 20 LT |
| Yêu Lang Huyết Nhãn | 1/10 | 2.500 | 15 | 2.6 | Như trên; < 50%: lao 2 lần liên tiếp, cào quạt 90° tầm 2.5 (4 s, cảnh báo 0.7 s) | Qua chương |
| Khôi Lỗi Đồng | 2/5 | 1.000 | 14 | 1.5 | Đập đất r=2.5, 20 dmg (4 s, cảnh báo 0.8 s) | Linh Châu lớn, 30 LT |
| Mặc Đại Phu | 2/10 | 4.000 | 10 | 2.0, giữ cách 4 | P1: phun độc quạt 5 viên (3 s), gọi 4 Khôi Lỗi Gỗ (15 s). P2 < 50%: **Đoạt Xá** – vòng hút r=6 kéo về tâm 2.5 unit/s trong 3 s, tâm 30 dmg/0.5 s, phải Dash ra (12 s, cảnh báo 1.0 s, viền màn hình tím) | Qua chương, mở Luyện đan |
| Huyết Văn Mãng | 3/5 | 1.500 | 16 | 2.2 | Trườn đường cong để vệt độc 3 s (6 s) | Linh Châu lớn, `herb_huyet_sam` |
| Huyết Văn Mãng Vương | 3/10 | 6.000 | 18 | 2.4 | Như trên; phun 3 luồng độc (5 s, cảnh báo 0.8 s); < 40%: chui đất trồi dưới chân (8 s, cảnh báo 1.0 s) | Qua chương, Huyết Ngọc Tri Nhện |
| Hộ pháp Ma Diễm Tông | 4/5 | 2.200 | 16 | 2.0 | Ném 3 cầu lửa vào vị trí người chơi (4 s, cảnh báo 0.8 s) | Linh Châu lớn, 40 LT |
| Trưởng lão Hợp Hoan Tông | 4/10 | 9.000 | 14 | 2.6 | P1: vòng 12 đạn (3 s), dịch chuyển ra sau lưng (7 s, dấu 0.6 s). P2 < 50%: **Mê Hồn** – đảo joystick 3 s (14 s, cảnh báo 1.0 s), 2 ảo ảnh 1 HP | Qua chương, Thanh Trúc Phong Vân Kiếm |

### 9.9. Lôi Kiếp

| Lần | Số tia / 10 s | Sát thương mỗi tia | Bán kính | Ghi chú |
|---|---|---|---|---|
| 1 | 8 | 15% HP tối đa | 1.0 | Rơi đều |
| 2 | 14 | 18% | 1.1 | 30% tia nhắm vị trí người chơi lúc cảnh báo |
| 3 | 22 | 20% | 1.2 | 50% nhắm người chơi; 2 tia cuối cùng lúc |

Sét cũng đánh quái trong vùng (500% Công gốc). Sát thương × `tribulationMultiplier` của chương.

### 9.10. Kinh tế

```
GiaNangDongPhu(chiSo, cap) = Goc[chiSo] × 1.25^(cap − 1)
Goc: HP 50, Công 60, Tốc độ kiếm 50, Rơi Linh Thạch 80
```

Linh Căn: Tam 3.000 (qua ch.3), Song 15.000 (qua ch.9), Thiên 60.000 (qua ch.14). Rewarded ở Lobby: +50 Linh Thạch, 5 lần/ngày. Một trận ch.1: ~230 Linh Thạch lần đầu, ~100 khi chơi lại.

### 9.11. Linh thảo & đan

Linh thảo: `herb_thanh_linh` (boss ch.1, 2), `herb_huyet_sam` (boss ch.3), `herb_tu_diep` (boss ch.4). Boss chương rơi 100% lần đầu, 30% khi chơi lại. Chín sau 24 h hoặc 1 giọt Lục Dịch.

| Đan | Công thức | Mở |
|---|---|---|
| Trúc Cơ Đan | 2 Thanh Linh + 1 Huyết Sâm | Qua ch.2 |
| Tuyết Phách Hoàn | 2 Thanh Linh | Qua ch.2 |
| Giáng Trần Đan | 1 Huyết Sâm + 2 Tử Diệp | Qua ch.4 |

### 9.12. Hồi sinh

Mỗi trận tối đa 1 lần bằng quảng cáo + 1 lần bằng Huyền Thiên Đan. Hồi sinh: 100% HP, đẩy lùi quái r=6, bất tử 3 s, xóa đạn.

### 9.13. Hội thoại

Lời thoại cần người viết. Agent chỉ tạo cấu trúc và dữ liệu mẫu 2–3 dòng có đánh dấu `TODO(dialogue)`.

```csharp
[CreateAssetMenu(menuName = "VKDK/Dialogue")]
public class DialogueData : ScriptableObject
{
    public DialogueLine[] lines;
}

[System.Serializable]
public class DialogueLine
{
    public string speakerId;      // "han_lap", "mac_dai_phu"
    public bool portraitOnLeft;
    [TextArea] public string text; // ≤ 120 ký tự
}
```

---
## 10. KỸ THUẬT

### 10.1. Thư mục

```text
Assets/
├── _Project/
│   ├── Art/
│   │   ├── Characters/       # .psb có xương
│   │   ├── Sprites/          # Enemies/, Weapons/, Items/, Beasts/
│   │   ├── Portraits/
│   │   ├── Tilesets/
│   │   ├── UI/               # khung, nút, Icons/
│   │   ├── Atlases/          # Atlas_UI_Common, Atlas_UI_Lobby, Atlas_UI_Combat, Atlas_Icons
│   │   └── VFX_Unreal/       # flipbook nền đen
│   ├── Editor/
│   │   └── AssetTools/       # VkdkTexturePostprocessor, VkdkFxTool, VkdkDataSeeder
│   ├── Audio/                # BGM/, SFX/
│   ├── Materials/            # M_FX_Additive_Template, M_FX_Alpha_Template, SG_Sprite_Additive
│   │   └── FX/
│   ├── Prefabs/              # Player/, Weapons/, Beasts/, Enemies/, VFX/, UI/Kit/
│   ├── Scenes/               # Boot, Lobby, Combat, UIKit_Preview
│   ├── ScriptableObjects/    # Realms/, Chapters/, Transformations/, Swords/, Artifacts/, Beasts/,
│   │                         # Upgrades/, Alchemy/, Bosses/, Enemies/, Waves/, Dialogue/, Config/
│   └── Scripts/
│       ├── Core/             # GameManager, ObjectPooler, AutoReturnToPool, SaveSystem, AudioManager, TimeService
│       ├── Data/             # các class ScriptableObject
│       ├── Story/            # ChapterManager, DialogueRunner
│       ├── Player/           # PlayerController, PlayerStats, TransformationHandler
│       ├── Combat/           # FlyingSwordController, FlyingSword, TribulationSystem, UpgradeSystem, DamagePopup, LinhChauPickup
│       ├── Companion/        # BeastCompanion
│       ├── Enemy/            # EnemyAI, WaveSpawner, BossController
│       ├── Meta/             # CaveDwelling, HeavenlyVial, AlchemySystem, SwordTomb
│       ├── UI/               # FloatingJoystick, SafeAreaHandler, CombatHUD, LobbyUI, ChapterSelectUI,
│       │                     # DialogueUI, UpgradeCardUI, Kit/ (UIPanel, UIButton, SkillButton, UIBar, ThemeTint...)
│       └── Monetization/     # AdMobManager, AdPlacementController
```

Scene: `Boot` (khởi tạo Manager, UMP, AdMob, load save) → `Lobby` → `Combat`. Manager cốt lõi `DontDestroyOnLoad`.

### 10.2. ScriptableObject chính

```csharp
using UnityEngine;

public enum WorldPlane { NhanGioi, LinhGioi, TienGioi }

[CreateAssetMenu(menuName = "VKDK/Realm")]
public class RealmData : ScriptableObject
{
    public string id;            // "realm_ket_dan"
    public string realmName;     // "Kết Đan"
    public int realmIndex;       // 1..14
    public WorldPlane plane;
    public float playerStatMultiplier = 1f; // nhân HP và sát thương người chơi trong trận (mục 11.1)
}

[CreateAssetMenu(menuName = "VKDK/Chapter")]
public class ChapterData : ScriptableObject
{
    public string id;                              // "chapter_01"
    public int chapterIndex;                       // 1..19
    public string biCanhName;
    public WorldPlane plane;
    public RealmData[] runRealms = new RealmData[4];
    public float tribulationMultiplier = 1f;
    public float enemyStatMultiplier = 1f;
    public int firstClearReward;
    public int replayReward;
    public WaveTable waves;
    public BossData eliteBoss;                     // phút 5
    public BossData chapterBoss;                   // phút 10
    public DialogueData introDialogue;
    public DialogueData outroDialogue;
    public ScriptableObject[] unlockRewards;
}

[CreateAssetMenu(menuName = "VKDK/Transformation")]
public class TransformationData : ScriptableObject
{
    public string id;
    public string displayName;
    public float swordSpeedMultiplier = 2f;
    public float damageMultiplier = 1.4f;
    public float duration = 12f;
    public Color hudColor;
    public string auraPoolTag;
}
```

Các SO khác cần tạo theo bảng mục 9: `PlayerBaseStats`, `EnemyData`, `WaveTable` (danh sách `WaveSegment`: thời gian bắt đầu/kết thúc, số con mỗi lần, chu kỳ, tối đa cùng lúc, thành phần theo %, sự kiện đặc biệt), `BossData` (HP, chạm, tốc độ, danh sách phase theo % HP, mỗi phase có danh sách `BossSkill` với cooldown, thời gian cảnh báo, tham số), `SwordData`, `UpgradeCardData`, `EvolutionData`, `TribulationConfig`, `EconomyConfig`, `HerbData`, `PillData`, `BeastData`, `UITheme`. Mọi SO có trường `id` dạng chuỗi để lưu game.

**`VkdkDataSeeder` (Editor):** menu *Tools → VKDK → Seed Data* tạo/cập nhật toàn bộ asset SO của chương 1–4 với số liệu mục 9, để người dùng không phải nhập tay.

### 10.3. Đặc tả script

| Script | Yêu cầu |
|---|---|
| `ObjectPooler` | Singleton. Dictionary `Queue<GameObject>` theo tag (string). `SpawnFromPool(string tag, Vector3 pos, Quaternion rot)`, `ReturnToPool(string tag, GameObject obj)`. Cấu hình pool (tag, prefab, số lượng khởi tạo) trong Inspector. Tự mở rộng khi hết, ghi cảnh báo |
| `AutoReturnToPool` | Trên prefab VFX. `OnParticleSystemStopped()` gọi `ReturnToPool`. Có trường `poolTag` |
| `SaveSystem` | JSON `PlayerSaveData` tại `Application.persistentDataPath`. Ghi file tạm rồi đổi tên. Tự lưu khi `OnApplicationPause(true)`, `OnApplicationQuit`, hết trận. Kiểm tra `saveVersion` để migrate |
| `FlyingSwordController` | Quản lý danh sách `FlyingSword`. Quỹ đạo tròn (`Mathf.Sin`/`Cos`). State `Orbiting` → `Attacking` (mục tiêu bằng `Physics2D.OverlapCircleNonAlloc`) → `Returning`. Khi trúng: `DamagePopup` và VFX chém từ pool. Tối đa 36 |
| `TransformationHandler` | Đọc `TransformationData`. Đổi Animator Override Controller/màu sprite, bật hào quang, nhân chỉ số. `public static event Action<Color> OnTransformationChanged` bắn khi vào và khi hết (màu mặc định) |
| `TribulationSystem` | Tối màn hình; coroutine gọi sét từ pool theo mục 9.9, vòng cảnh báo 0.8 s. Hết 10 s gọi đột phá lên `runRealms[n+1]` |
| `FloatingJoystick` | `IPointerDownHandler`, `IDragHandler`, `IPointerUpHandler`. Hiện tại điểm chạm nửa trái, `HandleRange`, trả `Vector2 Direction` chuẩn hóa |
| `SafeAreaHandler` | Co `RectTransform` theo `Screen.safeArea`, cập nhật khi đổi hướng/kích thước |
| `AdMobManager` | `GoogleMobileAds.Api`. Xin đồng ý qua UMP (`GoogleMobileAds.Ump.Api`) trước khi `MobileAds.Initialize`. Bật `MobileAds.RaiseAdEventsOnUnityMainThread = true`. Load trước Banner/Interstitial/Rewarded. `ShowBanner()`/`HideBanner()`, `ShowInterstitial(Action onAdClosed)` (cooldown 180 s, bỏ ván đầu), `ShowRewardedAd(Action onUserEarnedReward, Action onAdFailed = null)` (xử lý mất mạng/chưa load, tự load lại sau khi đóng). Ad Unit ID trong SO `AdConfig`, mặc định là Test ID của Google |
| `ChapterManager` | Giữ chương đang chơi, mở chương tiếp khi diệt boss chương, cập nhật cảnh giới cốt truyện |
| `DialogueRunner` | Phát `DialogueData` lên `DialogueUI`, hỗ trợ bỏ qua |
| `UpgradeSystem` | Bốc 3 thẻ theo trọng số (mục 9.5), kiểm tra tiến hóa, áp chỉ số |
| `TimeService` | Khi mở game và khi quay lại từ nền: `UnityWebRequest.Head("https://www.google.com")`, đọc header `Date`. Giữa các lần: giờ tin cậy + `Time.realtimeSinceStartupAsDouble`. Mất mạng: dùng giờ máy, nếu lùi so với `lastTrustedUnix` thì không cộng thời gian |
| `HeavenlyVial` | Tính giọt Lục Dịch từ `TimeService`, tối đa 3 |
| `AlchemySystem` | 3 ô linh điền, tưới, luyện đan theo công thức |
| `BeastCompanion` | Đi theo người chơi, hành vi theo `BeastData`, đòn đánh dùng pool |
| `WaveSpawner` | Đọc `WaveTable`, sinh quái từ pool trên vòng r=11, tôn trọng trần 150 |
| `EnemyAI` | Hành vi theo `EnemyData` (lao thẳng, zigzag, bầy, giữ khoảng cách bắn, tự nổ) |
| `BossController` | Phase theo % HP từ `BossData`, chiêu lấy từ thư viện `BossSkill` (mục 11.6). Hỗ trợ boss đôi. Mọi đòn diện rộng có telegraph ≥ 0.6 s |
| `ArtifactController` | Thần Luân, Cánh (mục 11.9) |
| `CompanionPassives` | Ngân Nguyệt, Nam Cung Uyển (mục 11.9) |
| `TutorialManager`, `TutorialStep`, `TutorialOverlay`, `TutorialAnchor` | Mục 12.1 |
| `EndlessMode` | Mục 12.2 |
| `SettingsPopup`, `PausePopup` | Mục 12.3–12.4 |
| `Loc`, `StringTable` | Mục 12.5 |
| `DebugMenu` | Mục 12.6, chỉ `DEVELOPMENT_BUILD \|\| UNITY_EDITOR` |
| `AudioManager`, `SoundData`, `SoundLibrary` | Mục 13 |
| `AnalyticsService` | Mục 14.1 |
| Editor `VkdkPlaceholderArt`, `VkdkPlaceholderAudio`, `VkdkSceneBuilder`, `VkdkUIKitBuilder`, `VkdkBuildSettings` | Xem `ANTIGRAVITY_PROMPTS.md` |

### 10.4. Dữ liệu lưu

```csharp
[System.Serializable]
public class PlayerSaveData
{
    public int saveVersion = 1;
    public int linhThach;
    public int highestChapterCleared;          // 0..19
    public int storyRealmIndex;                // 1..14
    public int linhCanTier;                    // 0 Tứ, 1 Tam, 2 Song, 3 Thiên
    public int[] cavernUpgradeLevels;          // HP, Công, Tốc độ kiếm, Rơi Linh Thạch
    public string[] unlockedSwordIds;
    public int[] swordLevels;                  // cùng thứ tự unlockedSwordIds
    public string[] unlockedArtifactIds;
    public string[] unlockedBeastIds;
    public int[] beastEvolutionStages;
    public string equippedSwordId, equippedRingId, equippedWingId, equippedBeastId;
    public int vialDrops;
    public long vialLastTickUnix;
    public long lastTrustedUnix;
    public HerbPlot[] herbPlots;
    public string[] pillInventory;
    public int lobbyAdCountToday;
    public long lobbyAdDayUnix;
    public bool firstRunCompleted;
    public int artifactLevels_HuThienDinh, artifactLevels_PhongLoiSi;
    public long tutorialFlags;                 // bit theo thứ tự bước hướng dẫn (mục 12.1)
    public float endlessBestSeconds;
    public int endlessBestRealm;
    public SettingsData settings = new SettingsData();
}

[System.Serializable]
public class SettingsData
{
    public float bgmVolume = 0.8f, sfxVolume = 1f;
    public bool highQuality = true;
    public int targetFps = 60;
    public bool vibration = true, screenShake = true, damageNumbers = true;
    public bool fixedJoystick = false;
    public string language = "vi";
}

[System.Serializable]
public class HerbPlot { public string herbId; public long plantedUnix; public bool ready; }
```

Chỉ lưu id chuỗi của SO, không lưu tham chiếu.

### 10.5. Build Android

| Mục | Giá trị |
|---|---|
| Package Name | `com.yourstudio.vankiemdokiep` |
| Minimum API | 24 (Android 7.0) |
| Target API | **36 (Android 16)**: bắt buộc cho app mới và bản cập nhật từ 31/8/2026. Kiểm tra lại Play Console trước khi phát hành |
| Backend / Arch | IL2CPP / ARM64 |
| Định dạng | `.aab` |
| Quảng cáo | Test Ad Unit ID của Google trong suốt quá trình phát triển và Closed Testing |

### 10.6. Lộ trình

| Giai đoạn | Nội dung |
|---|---|
| Prototype | Di chuyển, phi kiếm, 1 Bí Cảnh, Kỳ Ngộ, Lôi Kiếp + Biến Thân, pooling |
| Vertical slice | Chương 1–4, hội thoại, Động Phủ, Kiếm Trủng, lưu game, AdMob test |
| Soft launch | Hết Nhân Giới (ch.1–9), Chưởng Thiên Bình, luyện đan, linh thú, Closed Testing |
| Cập nhật 1 | Linh Giới (ch.10–14) |
| Cập nhật 2 | Tiên Giới (ch.15–19), Vô Tận |

---

## 11. NỘI DUNG CHƯƠNG 5–19

Số khởi điểm, chưa playtest. Tên quái và boss phụ là đề xuất cho game.

### 11.1. Hệ số cảnh giới (người chơi, trong trận)

`RealmData.playerStatMultiplier` nhân **HP tối đa và sát thương** người chơi khi đang ở cảnh giới đó (áp cả chương 1–4). Cảnh giới cốt truyện ở Lobby không cộng chỉ số.

| # | Cảnh giới | Hệ số | # | Cảnh giới | Hệ số |
|---|---|---|---|---|---|
| 1 | Luyện Khí | 1.00 | 8 | Đại Thừa | 2.65 |
| 2 | Trúc Cơ | 1.15 | 9 | Độ Kiếp | 3.00 |
| 3 | Kết Đan | 1.30 | 10 | Chân Tiên | 3.50 |
| 4 | Nguyên Anh | 1.50 | 11 | Kim Tiên | 4.00 |
| 5 | Hóa Thần | 1.75 | 12 | Thái Ất | 4.60 |
| 6 | Luyện Hư | 2.00 | 13 | Đại La | 5.30 |
| 7 | Hợp Thể | 2.30 | 14 | Đạo Tổ | 6.00 |

### 11.2. Hành vi quái (`EnemyBehavior`)

| Hành vi | Mô tả |
|---|---|
| `Charger` | Đi thẳng tới người chơi |
| `Zigzag` | Đi tới, lắc ngang biên độ 1, chu kỳ 0.8 s |
| `Swarm` | Bầy 6–8, lượn vòng r=4 trong 2 s rồi lao vào |
| `Ranged` | Giữ cách 5, mỗi 2.5 s bắn 1 đạn tốc độ 5 |
| `Exploder` | Cách 1 thì đứng 0.6 s (cảnh báo) rồi nổ r=1.2 |
| `Tank` | Như Charger, chậm, HP cao |
| `DashElite` | Mỗi 4 s lao ×2.5 tốc độ trong 0.6 s, cảnh báo đường 0.6 s |
| `Shielder` | Mỗi 6 s cho đồng loại trong r=3 khiên 30% HP gốc (5 s, không cộng dồn) |
| `Teleporter` | Mỗi 4 s dịch 3 unit về phía người chơi, dấu đích 0.6 s |
| `PatternShooter` | Giữ cách 6, mỗi 3 s bắn 8 đạn tỏa tròn tốc độ 4 |
| `Splitter` | Chết tách 2 con 35% HP, tốc độ ×1.3, không tách tiếp |

Cờ phụ `deathPool`: chết để vũng 2 s, 2 dmg/0.5 s.

### 11.3. Nhãn quái (`EnemyTag`) và khắc chế

| Nhãn | Lôi | Hỏa | Băng | Khác |
|---|---|---|---|---|
| Yêu | ×1 | ×1 | ×1 | |
| Ma | ×1.5 (Kim Lôi ×2) | ×1 | ×0.7 | Đề Hồn Thú ×2 |
| Quỷ | ×1.5 (Kim Lôi ×2) | ×1 | ×1 | Đề Hồn Thú ×2 |
| Thổ | ×0.7 | ×1 | ×1 | |
| Mộc | ×1 | ×1.5 | ×1 | |
| Biển | ×1 | ×0.7 | ×1.5 | |
| Tiên | ×1 | ×1 | ×1 | |

Chương 1–4: Yêu Lang `Charger`/Yêu; Lang Đầu Đàn `DashElite`/Yêu; Độc Trùng `Zigzag`+deathPool/Yêu; Khôi Lỗi Gỗ `Tank`/Mộc; Huyết Dơi `Swarm`/Ma; Đệ tử Ma Đạo `Ranged`/Ma; Huyết Nô `Exploder`/Ma.

### 11.4. Quái mới (giá trị gốc, nhân `enemyStatMultiplier` và hệ số phút)

| Id | Tên | Nhãn | Hành vi | HP | Chạm | Tốc độ | EXP | Linh Thạch |
|---|---|---|---|---|---|---|---|---|
| `en_hai_xa` | Hải Xà | Biển | Zigzag | 14 | 6 | 2.6 | 1 | 8% / 1 |
| `en_ngu_yeu` | Ngư Yêu | Biển | Swarm | 10 | 5 | 3.4 | 1 | 6% / 1 |
| `en_giap_giai` | Giáp Giải | Thổ | Tank | 50 | 9 | 1.3 | 2 | 10% / 1 |
| `en_hai_yeu_phun` | Hải Yêu Phun Nước | Biển | Ranged | 28 | 6 | 1.8 | 3 | 15% / 2 |
| `en_am_hon` | Âm Hồn | Quỷ | Teleporter | 24 | 8 | 2.0 | 2 | 10% / 1 |
| `en_cuc_am_tu` | Đệ tử Cực Âm | Ma | PatternShooter | 32 | 6 | 1.6 | 3 | 15% / 2 |
| `en_mo_lan_ky` | Kỵ Sĩ Mộ Lan | Yêu | Charger | 20 | 8 | 3.0 | 1 | 8% / 1 |
| `en_phap_su_mo_lan` | Pháp Sư Mộ Lan | Yêu | Shielder | 30 | 5 | 1.6 | 3 | 15% / 2 |
| `en_quy_vat` | Quỷ Vật | Quỷ | Swarm | 12 | 6 | 3.2 | 1 | 6% / 1 |
| `en_cuong_thi` | Cương Thi | Quỷ | Tank | 70 | 12 | 1.2 | 3 | 12% / 1 |
| `en_quy_hoa` | Quỷ Hỏa | Quỷ | Exploder + deathPool | 18 | 20 (nổ) | 2.6 | 2 | 10% / 1 |
| `en_am_la_tu` | Đệ tử Âm La Tông | Ma | PatternShooter | 34 | 7 | 1.6 | 3 | 15% / 2 |
| `en_huyet_anh` | Huyết Ảnh | Ma | Teleporter | 26 | 9 | 2.2 | 2 | 10% / 1 |
| `en_khong_gian_trung` | Không Gian Trùng | Yêu | Teleporter | 22 | 8 | 2.4 | 2 | 10% / 1 |
| `en_giac_xi` | Chiến Binh Giác Xi | Yêu | Charger | 26 | 9 | 2.6 | 2 | 10% / 1 |
| `en_giac_xi_khien` | Khiên Binh Giác Xi | Thổ | Shielder | 45 | 8 | 1.6 | 3 | 15% / 2 |
| `en_da_xoa` | Dạ Xoa | Ma | Zigzag | 18 | 8 | 3.4 | 1 | 8% / 1 |
| `en_moc_toc` | Mộc Tộc | Mộc | Splitter | 40 | 9 | 1.6 | 3 | 12% / 1 |
| `en_ma_binh` | Ma Binh | Ma | Tank | 70 | 12 | 1.4 | 3 | 12% / 1 |
| `en_ma_phap` | Ma Pháp Sư | Ma | PatternShooter | 34 | 8 | 1.6 | 3 | 15% / 2 |
| `en_ma_no` | Ma Nô | Ma | Exploder | 24 | 22 (nổ) | 2.8 | 2 | 10% / 1 |
| `en_tien_binh` | Tiên Binh | Tiên | Charger | 28 | 9 | 2.6 | 2 | 10% / 1 |
| `en_tien_cung` | Tiên Cung Thủ | Tiên | Ranged | 30 | 8 | 1.8 | 3 | 15% / 2 |
| `en_tien_ve` | Tiên Vệ Khiên | Tiên | Shielder | 50 | 9 | 1.6 | 3 | 15% / 2 |
| `en_man_hoang` | Chiến Binh Man Hoang | Thổ | Tank | 80 | 13 | 1.5 | 3 | 12% / 1 |
| `en_man_hoang_thu` | Man Hoang Thú | Yêu | Swarm | 14 | 7 | 3.4 | 1 | 6% / 1 |
| `en_hu_khong_anh` | Hư Không Ảnh | Quỷ | Teleporter | 26 | 10 | 2.4 | 2 | 10% / 1 |
| `en_thien_dinh_phap` | Pháp Sư Thiên Đình | Tiên | PatternShooter | 36 | 8 | 1.6 | 3 | 15% / 2 |
| `en_luan_hoi_hon` | Luân Hồi Hồn | Quỷ | Splitter | 36 | 9 | 2.0 | 3 | 12% / 1 |

### 11.5. Thông số chương & wave

| Ch. | `enemyStatMultiplier` | `tribulationMultiplier` | Cảnh giới đầu | Qua lần đầu | Chơi lại | Linh thảo boss |
|---|---|---|---|---|---|---|
| 5 | 2.8 | 1.0 | Trúc Cơ | 800 | 160 | `herb_hai_tao` |
| 6 | 2.9 | 1.0 | Trúc Cơ | 900 | 180 | `herb_hai_tao` |
| 7 | 3.1 | 1.0 | Trúc Cơ | 1.000 | 200 | `herb_tu_diep` |
| 8 | 3.3 | 1.0 | Trúc Cơ | 1.150 | 230 | `herb_am_linh_chi` |
| 9 | 3.5 | 1.0 | Trúc Cơ | 1.300 | 260 | `herb_am_linh_chi` |
| 10 | 5.5 | 1.25 | Hóa Thần | 1.600 | 320 | `herb_loi_minh_truc` |
| 11 | 5.8 | 1.25 | Hóa Thần | 1.800 | 360 | `herb_loi_minh_truc` |
| 12 | 6.0 | 1.25 | Hóa Thần | 2.000 | 400 | `herb_huyet_lien` |
| 13 | 7.2 | 1.25 | Luyện Hư | 2.300 | 460 | `herb_huyet_lien` |
| 14 | 7.5 | 1.25 | Luyện Hư | 2.600 | 520 | `herb_huyet_lien` |
| 15 | 10.5 | 1.5 | Chân Tiên | 3.000 | 600 | `herb_tien_lo` |
| 16 | 14.2 | 1.5 | Chân Tiên | 3.400 | 680 | `herb_tien_lo` |
| 17 | 14.7 | 1.5 | Chân Tiên | 3.800 | 760 | `herb_tien_lo` |
| 18 | 17.4 | 1.5 | Kim Tiên | 4.200 | 840 | `herb_tien_lo` |
| 19 | 18.0 | 1.5 | Kim Tiên | 5.000 | 1.000 | — |

- Linh Thạch rơi từ quái × `floor(1 + 0.1 × (chương − 1))`, tối thiểu 1.
- Chương 15: bonus Động Phủ của Công gốc và Tốc độ phi kiếm chỉ tính 50% trong trận (hiện thông báo).
- `tribulationMultiplier` chương 1–4 = 1.0.

**Khung wave chung ch.5–19:** A 0–3' 6 con/1.5 s, tối đa 100 · B 3–5' 7/1.4 s, 120 · phút 5 boss tinh anh (ngừng sinh tới khi chết hoặc 60 s) · C 5–10' 9/1.2 s, 150 · phút 10 boss chương (50% mật độ C).

| Ch. | A | B | C |
|---|---|---|---|
| 5 | 50 hai_xa, 50 ngu_yeu | 40 hai_xa, 30 ngu_yeu, 30 giap_giai | 30 hai_xa, 25 ngu_yeu, 25 giap_giai, 20 hai_yeu_phun |
| 6 | 40 hai_xa, 30 am_hon, 30 giap_giai | 40 hai_xa, 30 am_hon, 30 hai_yeu_phun | 25 hai_xa, 25 am_hon, 25 cuc_am_tu, 25 giap_giai |
| 7 | 60 mo_lan_ky, 40 soi | 50 mo_lan_ky, 30 soi, 20 phap_su_mo_lan | 45 mo_lan_ky, 25 phap_su_mo_lan, 25 soi, 5 soi_dau |
| 8 | 60 quy_vat, 40 quy_hoa | 40 quy_vat, 30 quy_hoa, 30 cuong_thi | 30 quy_vat, 25 quy_hoa, 25 cuong_thi, 20 huyet_anh |
| 9 | 50 am_la_tu, 50 huyet_anh | 40 am_la_tu, 30 huyet_anh, 30 quy_hoa | 30 am_la_tu, 25 huyet_anh, 25 cuong_thi, 20 quy_hoa |
| 10 | 70 khong_gian_trung, 30 da_xoa | 50 khong_gian_trung, 50 da_xoa | 40 khong_gian_trung, 30 da_xoa, 30 giac_xi |
| 11 | 60 giac_xi, 40 da_xoa | 50 giac_xi, 25 giac_xi_khien, 25 da_xoa | 40 giac_xi, 25 giac_xi_khien, 20 da_xoa, 15 khong_gian_trung |
| 12 | 50 da_xoa, 50 moc_toc | 40 da_xoa, 40 moc_toc, 20 giac_xi_khien | 35 da_xoa, 35 moc_toc, 15 giac_xi, 15 giac_xi_khien |
| 13 | 60 ma_no, 40 ma_binh | 40 ma_no, 30 ma_binh, 30 ma_phap | 35 ma_no, 25 ma_binh, 25 ma_phap, 15 da_xoa |
| 14 | 40 ma_binh, 30 ma_phap, 30 ma_no | 35 ma_binh, 30 ma_phap, 35 ma_no | 30 ma_binh, 30 ma_phap, 25 ma_no, 15 khong_gian_trung |
| 15 | 60 tien_binh, 40 tien_cung | 50 tien_binh, 30 tien_cung, 20 tien_ve | 40 tien_binh, 30 tien_cung, 30 tien_ve |
| 16 | 50 tien_binh, 50 man_hoang_thu | 40 tien_binh, 30 man_hoang_thu, 30 tien_cung | 30 tien_binh, 25 man_hoang_thu, 25 tien_cung, 20 thien_dinh_phap |
| 17 | 50 man_hoang_thu, 50 man_hoang | 40 man_hoang_thu, 30 man_hoang, 30 hu_khong_anh | 30 man_hoang_thu, 25 man_hoang, 25 hu_khong_anh, 20 luan_hoi_hon |
| 18 | 50 luan_hoi_hon, 50 hu_khong_anh | 40 luan_hoi_hon, 30 hu_khong_anh, 30 thien_dinh_phap | 30 luan_hoi_hon, 25 hu_khong_anh, 25 thien_dinh_phap, 20 tien_ve |
| 19 | 40 tien_binh, 30 tien_cung, 30 tien_ve | 35 tien_binh, 25 tien_cung, 20 tien_ve, 20 thien_dinh_phap | 25 tien_binh, 20 tien_cung, 20 tien_ve, 20 thien_dinh_phap, 15 hu_khong_anh |

(Số là %, id bỏ tiền tố `en_`.)

### 11.6. Thư viện chiêu boss (`BossSkill`)

Sát thương ghi theo bội số **Chạm** của boss. Cảnh báo ≥ 0.6 s.

| Chiêu | Tham số | Mô tả |
|---|---|---|
| `Charge` | distance, warn, repeat | Lao thẳng, cảnh báo đường |
| `Summon` | enemyId, count | Gọi quái |
| `Fan` | bullets, angle, speed | Quạt về phía người chơi, 1× mỗi viên |
| `Ring` | bullets, speed | Vòng tròn, 1× mỗi viên |
| `Slam` | radius, damageMul, warn | Đập quanh boss |
| `Rain` | count, radius, damageMul, warn | Rơi vào vị trí người chơi lúc cảnh báo |
| `Trail` | duration, tickDamageMul | Vệt độc/lửa |
| `Teleport` | mode (behind/random), mark | Dịch chuyển, dấu trước |
| `Pull` | radius, speed, duration, centerDamageMul | Hút về tâm |
| `InvertControls` | duration, warn | Đảo joystick |
| `Clone` | count | Ảo ảnh 1 HP dùng cùng chiêu, mất khi đổi phase |
| `Burrow` | warn, damageMul | Chui đất, trồi dưới chân |
| `Fog` | duration, visionRadius | Thu hẹp tầm nhìn |
| `Laser` | width, sweepAngle, duration, damageMul, warn | Tia quét cung |
| `Shield` | percentHp, duration | Khiên (Phệ Kim Trùng phá ×3) |
| `FreezeZone` | radius, duration, warn | Vùng chậm 60% |
| `TimeStop` | duration, warn | Người chơi đứng yên, cảnh báo bằng mất màu dần |
| `Rewind` | seconds, maxHealPercent | Một lần: hồi HP về mức `seconds` giây trước, tối đa `maxHealPercent` |
| `Enrage` | speedMul, cooldownMul | Kích hoạt theo điều kiện |

Boss chương 1–4 cũng khai báo bằng thư viện này.

### 11.7. Boss chương 5–19

HP là giá trị cuối. Quy tắc: tinh anh ≈ 1.000 × hệ số chương, boss chương ≈ 4.000 ×, Chạm ≈ 7 × (tinh anh 80%). P2 khi HP < 50% nếu không ghi khác. Cooldown trong ngoặc.

| Ch. | Loại | Boss | Nhãn | HP | Chạm | Tốc | Chiêu |
|---|---|---|---|---|---|---|---|
| 5 | Tinh anh | Hải Xà Vương | Biển | 2.800 | 16 | 2.4 | Charge 7, warn 0.8 (5 s); Trail 3 s, 0.3× (8 s) |
| 5 | Chương | Bát Cước Hải Yêu | Biển | 11.000 | 20 | 1.8 | P1: Ring 12, tốc 4 (3 s); Slam r3, 1.5×, warn 0.8 (6 s); Summon hai_xa×6 (14 s). P2: +Fog 5 s, tầm 4 (16 s) |
| 6 | Tinh anh | Âm Hồn Thống Lĩnh | Quỷ | 2.900 | 16 | 2.2 | Teleport random, mark 0.6 (4 s); Fan 5, 60°, tốc 5 (3 s) |
| 6 | Chương | Lão Ma Cực Âm Đảo | Ma | 11.600 | 20 | 2.0 | P1: Ring 16, tốc 4 (3 s); Teleport behind (7 s); Summon am_hon×4 (15 s). P2: +FreezeZone r4, 3 s, warn 0.8 (10 s) |
| 7 | Tinh anh | Mộ Lan Kỵ Tướng | Yêu | 3.100 | 18 | 2.8 | Charge 9, warn 0.7, repeat 2 (6 s) |
| 7 | Chương | Thánh Nữ Mộ Lan | Yêu | 12.400 | 22 | 1.8 | P1: Rain 4, r1.5, 1.5×, warn 0.8 (5 s); Shield 20%, 6 s (18 s); Summon phap_su_mo_lan×3 (16 s). P2: +Laser rộng 0.8, 120°, 2 s, 1×/0.2 s, warn 1.0 (10 s) |
| 8 | Tinh anh | Cương Thi Vương | Quỷ | 3.300 | 18 | 1.6 | Slam r2.5, 1.5×, warn 0.8 (4 s); Summon cuong_thi×2 (12 s) |
| 8 | Chương | Quỷ Vương Âm Minh | Quỷ | 13.200 | 23 | 2.0 | P1: Fog 6 s, tầm 4.5 (18 s); Summon quy_vat×8 (12 s); Fan 7, 90°, tốc 5 (3 s). P2: +Pull r6, 2.5 u/s, 3 s, tâm 1.5×/0.5 s, warn 1.0 (13 s); Clone×2 (khi vào P2) |
| 9 | Tinh anh | Hộ Pháp Âm La Tông | Ma | 3.500 | 20 | 2.0 | Ring 12, tốc 4 (3 s); Teleport random (6 s) |
| 9 | Chương | Lão Tổ Âm La Tông | Ma | 14.000 | 25 | 1.8 | P1: Fan 9, 100°, tốc 5 (3 s); Laser rộng 0.8, 90°, 2 s, warn 1.0 (9 s); Summon am_la_tu×3 (15 s). P2: +Rain 6, r1.4, 1.5×, warn 0.8 (6 s); Shield 25%, 6 s (20 s) |
| 10 | Tinh anh | Không Gian Trùng Mẫu | Yêu | 5.500 | 31 | 2.2 | Teleport random (4 s); Summon khong_gian_trung×4 (10 s) |
| 10 | Chương | Không Gian Thú | Yêu | 22.000 | 39 | 2.2 | P1: Ring 16, tốc 4.5 (3 s); Teleport behind (6 s). P2: +Pull r7, 2.5 u/s, 3 s, tâm 1.5×/0.5 s, warn 1.0 (12 s); Rain 5, r1.3, 1.5×, warn 0.8 (6 s) |
| 11 | Tinh anh | Giác Xi Bách Phu Trưởng | Yêu | 5.800 | 33 | 2.6 | Charge 8, warn 0.7 (5 s); Slam r2.5, 1.5×, warn 0.8 (6 s) |
| 11 | Chương | Thống Lĩnh Giác Xi | Yêu | 23.200 | 41 | 2.4 | P1: Charge 10, warn 0.7, repeat 2 (6 s); Slam r3, 1.8×, warn 0.8 (7 s); Summon giac_xi_khien×3 (15 s). P2: +Shield 25%, 8 s (18 s); Enrage khi khiên vỡ: tốc ×1.3 trong 5 s |
| 12 | Tinh anh | Trưởng Lão Mộc Tộc | Mộc | 6.000 | 34 | 1.6 | Summon moc_toc×3 (10 s); Rain 3, r1.5, 1.5×, warn 0.8 (5 s) |
| 12 | Chương | Vua Dạ Xoa | Ma | 24.000 | 42 | 3.0 | P1: Charge 8, warn 0.6, repeat 3 (7 s); Clone×2 (20 s). P2: +Fan 7, 90°, tốc 5.5 (3 s); Trail 3 s, 0.3× (luôn bật) |
| 13 | Tinh anh | Ma Tướng | Ma | 7.200 | 40 | 2.2 | Slam r2.8, 1.5×, warn 0.8 (5 s); Summon ma_no×4 (10 s) |
| 13 | Chương | Thủy Tổ Nguyên Yểm | Ma | 28.800 | 50 | 2.0 | P1: Summon ma_binh×4 (14 s); Fog 6 s, tầm 4 (18 s); Laser rộng 1.0, 120°, 2.5 s, warn 1.0 (9 s). P2: +Pull r7, 3 u/s, 3 s, tâm 2×/0.5 s, warn 1.0 (12 s) |
| 14 | Tinh anh | Ma Soái | Ma | 7.500 | 42 | 2.4 | Charge 9, warn 0.7 (5 s); Ring 12, tốc 4.5 (4 s) |
| 14 | Chương (đôi) | Thủy Tổ Bảo Hoa | Ma | 17.000 | 53 | 1.8, giữ cách 6 | Ring 18, tốc 4 (3 s); Rain 5, r1.5, 1.5×, warn 0.8 (6 s). Enrage khi Lục Cực chết: tốc ×1.3, cooldown ×0.7 |
| 14 | Chương (đôi) | Thủy Tổ Lục Cực | Ma | 17.000 | 53 | 2.6 | Charge 10, warn 0.7, repeat 2 (6 s); Slam r3, 2×, warn 0.8 (7 s). Enrage khi Bảo Hoa chết |
| 15 | Tinh anh | Đội Trưởng Tiên Binh | Tiên | 10.500 | 59 | 2.6 | Charge 8, warn 0.7 (5 s); Shield 20%, 5 s (15 s) |
| 15 | Chương | Tuần Sát Sứ Thiên Đình | Tiên | 42.000 | 74 | 2.2 | P1: Fan 7, 80°, tốc 5.5 (3 s); Summon tien_ve×2 (15 s); Shield 20%, 6 s (18 s). P2: +Rain 6, r1.4, 1.5×, warn 0.8 (6 s) |
| 16 | Tinh anh | Thạch Tướng Hắc Thổ | Thổ | 14.200 | 79 | 1.6 | Slam r3, 1.8×, warn 0.9 (5 s); Burrow warn 1.0, 2× (9 s) |
| 16 | Chương | Tiên Tướng Thiên Đình | Tiên | 56.800 | 99 | 2.4 | P1: Charge 10, warn 0.7, repeat 2 (6 s); Laser rộng 1.0, 120°, 2 s, warn 1.0 (9 s). P2: +Rain 7, r1.4, 1.5×, warn 0.8 (6 s) |
| 17 | Tinh anh | Man Hoang Thú Vương | Yêu | 14.700 | 82 | 2.8 | Charge 9, warn 0.6, repeat 2 (5 s); Summon man_hoang_thu×8 (12 s) |
| 17 | Chương | Thủ Lĩnh Man Hoang | Thổ | 58.800 | 103 | 2.0 | P1: Slam r4, 2×, warn 1.0 (6 s); Summon man_hoang×3 (15 s). P2: +Charge 10, warn 0.7, repeat 3 (7 s); Shield 25%, 8 s (20 s) |
| 18 | Tinh anh | Chấp Sự Thiên Đình | Tiên | 17.400 | 97 | 2.2 | Ring 16, tốc 4.5 (3 s); Teleport behind (6 s) |
| 18 | Chương | Chấp Pháp Thiên Đình | Tiên | 69.600 | 122 | 2.2 | P1: Laser rộng 1.0, 150°, 2.5 s, warn 1.0 (8 s); Ring 18, tốc 4.5 (3 s). P2: +TimeStop 1.0 s, warn 1.0 (15 s); Clone×2 (khi vào P2) |
| 19 | Tinh anh | Thần Tướng Thiên Đình | Tiên | 18.000 | 101 | 2.4 | Charge 10, warn 0.7, repeat 2 (5 s); Fan 9, 100°, tốc 5.5 (3 s) |
| 19 | Chương | **Cổ Hoặc Kim** | Tiên | 90.000 | 126 | 2.2 | P1 (100–60%): Ring 20, tốc 4.5 (3 s); Fan 9, 100°, tốc 6 (4 s); Teleport behind (7 s). P2 (60–30%): +TimeStop 1.5 s, warn 1.0 (14 s); Laser rộng 1.2, 180°, 2.5 s, warn 1.0 (10 s). P3 (<30%): Rewind 5 s, tối đa +15% HP (khi vào P3); Rain 8, r1.4, 1.5×, warn 0.8 (6 s); Summon tien_ve×3 (18 s) |

Teleport không ghi mark thì mark = 0.6 s. Boss đôi: 2 thanh máu chồng nhau, qua chương khi cả hai chết.

### 11.8. Linh thú

Sát thương tính theo Công gốc người chơi (đã nhân hệ số cảnh giới). Tiến hóa trong Động Phủ.

| Linh thú | GĐ | Hiệu quả | Chi phí | Điều kiện |
|---|---|---|---|---|
| Huyết Ngọc Tri Nhện | 1 | Mỗi 4 s bắn tơ vào quái gần nhất trong 6, nổ r1.5: chậm 40% 2 s. +10% Linh Thạch | — | Qua ch.3 |
| | 2 | Tơ r2.0, chậm 50%, +15% Linh Thạch | 2 Lục Dịch + 1.500 | Qua ch.6 |
| Phệ Kim Trùng | 1 | 6 con, mỗi con cắn 0.3× Công/0.5 s; quái bị cắn nhận +15% sát thương 3 s; sát thương lên khiên ×3 | — | Qua ch.5 |
| | 2 | 10 con, +20% | 3 + 2.000 | Qua ch.9 |
| | 3 (Vương) | 16 con, cắn 0.5×, +30% | 6 + 8.000 | Qua ch.14 |
| Lục Dực Sương Ngô | 1 | Mỗi 8 s vòng băng r3: 1× Công, đóng băng 1.5 s (boss: chậm 50%) | — | Qua ch.7 |
| | 2 | Mỗi 6 s, r3.5, 2× | 3 + 3.000 | Qua ch.12 |
| Đề Hồn Thú | 1 (Khỉ nhỏ) | Hào quang r3: quái Quỷ/Ma nhận 0.5× Công/s (Quỷ ×2); mỗi quái chết trong hào quang hồi 1 HP | — | Qua ch.8 |
| | 2 (Linh Hầu) | r3.5, 0.8×/s | 3 + 4.000 | Qua ch.10 |
| | 3 (Hình Thú) | r4.5, 1.2×/s; mỗi 10 s gầm choáng 1 s quái thường | 6 + 10.000 | Qua ch.15 |

### 11.9. Thần Luân, Cánh, đồng đội

| Trang bị | Ô | Cấp 1 | Mỗi cấp (tối đa 5) | Giá nâng | Mở |
|---|---|---|---|---|---|
| Hư Thiên Đỉnh | Thần Luân | Mỗi 20 s hút quái r4 về giữa trong 1 s, rồi thiêu: 3× Công, chậm 30% 3 s | −1 s cooldown, +0.5× | `500 × 1.5^(cấp−1)` | Thưởng ch.6 |
| Phong Lôi Sí | Cánh | Dash xa +50%, vệt sét 1.5 s, 1× Công/0.3 s | −0.2 s cooldown Dash, +0.2× | `600 × 1.5^(cấp−1)` | Thưởng ch.12 |

| Đồng đội | Mở | Hiệu quả (luôn bật) |
|---|---|---|
| Ngân Nguyệt | Qua ch.11 | +15% `DetectionRadius`, +10% tốc độ bay kiếm; hồ ly bạc bay cạnh |
| Nam Cung Uyển | Qua ch.17 | 1 lần/trận: khi HP về 0 tự hồi sinh 50% HP, bất tử 2 s; không tính vào giới hạn hồi sinh |

**Thẻ Thời Gian Pháp Tắc:** trọng số 5, chỉ ở chương Tiên Giới và khi đã qua ch.16. Cấp 1: mỗi 15 s ngưng đọng quái thường trong r3 trong 2 s, boss chậm 60%. Mỗi cấp (tối đa 5): +0.4 s, +0.3 bán kính.

### 11.10. Linh thảo & đan bổ sung

Linh thảo: `herb_hai_tao` (boss ch.5–6), `herb_am_linh_chi` (ch.8–9), `herb_loi_minh_truc` (ch.10–11), `herb_huyet_lien` (ch.12–14), `herb_tien_lo` (ch.15–18).

| Đan | Công thức | Mở |
|---|---|---|
| Trúc Cơ Đan (thay thế) | 2 Hải Linh Tảo + 1 Thanh Linh Thảo | Qua ch.5 |
| Giáng Trần Đan (thay thế) | 1 Âm Linh Chi + 1 Tử Diệp Hoa | Qua ch.8 |
| Huyền Thiên Đan | 2 Lôi Minh Trúc + 1 Huyết Liên | Qua ch.12 |
| Định Nhan Đan | 1 Tiên Lộ Thảo | Qua ch.15 (trang phục "Thanh Sam") |

Đan có nhiều công thức thì hiện tất cả.

---
## 12. HƯỚNG DẪN, VÔ TẬN, CÀI ĐẶT, DEBUG

### 12.1. Hướng dẫn người chơi mới (`TutorialManager`)

Mỗi bước là `TutorialStep` (SO): điều kiện bắt đầu, nội dung (key chuỗi), `anchorId` phần tử nổi bật, điều kiện kết thúc, có dừng game không. Tiến độ lưu bằng bit trong `tutorialFlags` theo thứ tự bảng. Menu tạm dừng có "Bỏ qua hướng dẫn".

| # | Id | Bắt đầu | Hiển thị | Kết thúc | Dừng game |
|---|---|---|---|---|---|
| 1 | `tut_move` | Vào trận ch.1 lần đầu | Bàn tay kéo ở nửa trái, "Kéo để di chuyển" | Đi tổng 3 unit | Không (chưa sinh quái) |
| 2 | `tut_sword` | Xong 1 | "Phi kiếm tự tấn công yêu thú" 3 s, sinh 5 Yêu Lang phía trước | Hạ 5 con | Không |
| 3 | `tut_pickup` | Linh Châu đầu rơi | Mũi tên chỉ Linh Châu | Nhặt 1 viên | Không |
| 4 | `tut_card` | Lên cấp lần đầu | Nổi bật pop-up, ẩn 2 nút quảng cáo | Chọn thẻ | Có |
| 5 | `tut_dash` | Phút 1:00, sinh 1 Lang Đầu Đàn | Nổi bật `Btn_Dash`, `timeScale` 0.3 khi cảnh báo lao xuất hiện | Bấm Dash | Chậm 0.3 |
| 6 | `tut_tribulation` | Đủ Tu Vi lần đầu | Nổi bật `Btn_Tribulation`, "Vượt Lôi Kiếp để đột phá. Né vòng đỏ!" | Bấm | Có |
| 7 | `tut_tribulation_easy` | Lôi Kiếp đầu tiên của tài khoản | Chỉ 5 tia, sát thương 50% | Hết Lôi Kiếp | Không |
| 8 | `tut_transform` | Lần đầu đầy Nộ khi đã có Thanh Lôi | Nổi bật `Btn_Transform` | Bấm | Có |
| 9 | `tut_boss` | Boss tinh anh đầu tiên | Chữ lớn tên boss + mũi tên chỉ hướng | 2.5 s | Không |
| 10 | `tut_cave` | Về Lobby sau trận đầu | Động Phủ → nâng HP gốc (tặng 100 Linh Thạch nếu thiếu) | Nâng xong | — |
| 11 | `tut_vial` | Qua ch.1 | Mở Chưởng Thiên Bình, tặng 1 giọt | Đóng | — |
| 12 | `tut_alchemy` | Qua ch.2 | Gieo → tưới → luyện Tuyết Phách Hoàn (tặng nguyên liệu thiếu) | Luyện xong | — |
| 13 | `tut_sword_tomb` | Lần đầu đủ 800 Linh Thạch | Kiếm Trủng, mở Lôi Kiếm | Mở xong | — |
| 14 | `tut_pill_select` | Lần đầu có đan | Nổi bật ô chọn đan ở `ChapterSelectUI` | Chạm | — |

Trận hướng dẫn: không Interstitial, không nút quảng cáo ở Kỳ Ngộ, `RevivePopup` có 1 lần hồi sinh miễn phí.

UI: `TutorialOverlay` (lớp tối 70% có lỗ quanh phần tử, chặn chạm ngoài lỗ), khung chữ `UI_Panel` nhỏ, mũi tên nhấp nháy. Phần tử nổi bật gắn `TutorialAnchor` với `anchorId`, không tìm theo tên GameObject.

### 12.2. Chế độ Vô Tận (`EndlessMode`)

| Mục | Luật |
|---|---|
| Mở | Qua chương `endlessUnlockChapter` (mặc định 19); Debug menu mở được |
| Cảnh giới đầu | Cảnh giới cốt truyện cao nhất − 3 (tối thiểu 1), đột phá tới Đạo Tổ |
| Bản đồ | Đổi mỗi 5': Thiên Nam → Loạn Tinh Hải → Âm Minh → Thiên Uyên → Ma Giới → Bắc Hàn → Trung Thổ, lặp |
| Quái | Mỗi 5' đổi bộ quái theo bản đồ (thành phần đoạn C của chương tương ứng), trần 150 |
| Hệ số quái | HP × `2 × 1.18^phút`, sát thương × `2 × 1.08^phút` (không nhân `enemyStatMultiplier`) |
| Boss | Mỗi 5' một boss chương ngẫu nhiên đã qua. HP = HP gốc ÷ `enemyStatMultiplier` chương đó × `2 × 1.18^phút`; Chạm tương tự với `1.08^phút` |
| Lôi Kiếp | Sau 3 lần, mỗi +1.500 Tu Vi được Độ Kiếp tiếp; mỗi lần thành công +10% sát thương (cộng dồn) |
| Thưởng | 20 Linh Thạch/phút sống + nhặt được |
| Quảng cáo | Như trận thường |
| Kỷ lục | `endlessBestSeconds`, `endlessBestRealm` trong save. Bảng xếp hạng online để bản sau |

### 12.3. Cài đặt (`SettingsPopup`)

| Mục | Kiểu | Mặc định | Tác dụng |
|---|---|---|---|
| Nhạc nền | 0–100 | 80 | Mixer `BGMVolume` = `20 × log10(v)` dB, v = 0 → −80 dB |
| Hiệu ứng âm thanh | 0–100 | 100 | Mixer `SFXVolume` |
| Chất lượng | Thấp / Cao | Cao (Thấp nếu `SystemInfo.systemMemorySize < 3000`) | Thấp: tắt Bloom, −50% hạt VFX, tắt hào quang lặp, 30 FPS |
| Giới hạn FPS | 30 / 60 | 60 | `Application.targetFrameRate`, `vSyncCount = 0` |
| Rung | Bật/Tắt | Bật | `Handheld.Vibrate()` khi bị đánh mạnh, sét trúng, đột phá |
| Rung màn hình | Bật/Tắt | Bật | Camera shake |
| Số sát thương | Bật/Tắt | Bật | Ẩn `DamagePopup` (vẫn hiện chí mạng) |
| Kiểu joystick | Nổi / Cố định | Nổi | Cố định ở góc trái dưới |
| Ngôn ngữ | Danh sách | Tiếng Việt | Bản đầu chỉ có vi |
| Tùy chọn quyền riêng tư | Nút | — | Chỉ hiện khi `PrivacyOptionsRequirementStatus == Required`; gọi `ConsentForm.ShowPrivacyOptionsForm` |
| Chính sách quyền riêng tư | Nút | — | `Application.OpenURL(privacyPolicyUrl)` (URL trong `AdConfig`) |
| Xóa dữ liệu | Nút | — | Xác nhận 2 lần (lần 2 gõ "XOA"), xóa save, về Boot |
| Phiên bản | Chữ | — | `Application.version` |

Lưu trong `PlayerSaveData.settings`, không dùng `PlayerPrefs`. Mở từ Lobby và menu tạm dừng.

### 12.4. Menu tạm dừng (`PausePopup`)

Mở bằng nút góc phải trên hoặc tự mở khi `OnApplicationPause(true)` giữa trận. Nút: Tiếp tục, Cài đặt, Độn Thổ Phù, Bỏ qua hướng dẫn (khi đang hướng dẫn). Hiện danh sách thẻ Kỳ Ngộ đang có kèm cấp.

### 12.5. Chuỗi chữ (`Loc`, `StringTable`)

Mọi chữ UI lấy từ `StringTable` (SO: danh sách key → text) qua `Loc.Get("key")`. Ví dụ key: `btn.battle`, `popup.revive.ad`, `realm.ket_dan`, `tut.move`. Thêm ngôn ngữ = thêm `StringTable`.

### 12.6. Debug menu (`DebugMenu`)

Chỉ trong `DEVELOPMENT_BUILD || UNITY_EDITOR`. Mở: giữ 3 ngón tay 2 s, hoặc F1 trong Editor.

Lệnh: mở mọi chương; +10.000 Linh Thạch; +3 Lục Dịch và +5 mỗi linh thảo; bất tử; nhảy tới 4:55 / 9:55; +Tu Vi tới ngưỡng Độ Kiếp kế; đầy Nộ; chọn thẻ Kỳ Ngộ bất kỳ; tốc độ ×1/×2/×4; hiện FPS, số quái, đạn, object trong pool; reset hướng dẫn; giả lập hết quảng cáo / mất mạng.

---

## 13. ÂM THANH

### 13.1. Kiến trúc

- AudioMixer `MainMixer`: Master → BGM, SFX, UI. Expose `MasterVolume`, `BGMVolume`, `SFXVolume`. Snapshot `Paused` (BGM −6 dB).
- `AudioManager` (singleton): 2 AudioSource nhạc nền (crossfade 1.5 s, unscaled), pool 16 SFX, 2 UI.
- Gọi theo id: `AudioManager.PlaySfx("sfx_sword_hit")`, `AudioManager.PlayBgm("bgm_nhan_gioi")`. Id → `SoundData` trong `SoundLibrary`.
- `SoundData`: clips (chọn ngẫu nhiên), volume, pitchVariance, mixerGroup, `maxInstances`, `minInterval`.
- Khi `timeScale = 0`: chuyển snapshot `Paused`, tạm dừng SFX gameplay. Khi quảng cáo toàn màn hình mở: `AudioListener.pause = true`, đóng thì bật lại.

### 13.2. Import

| Loại | Nguồn | Load Type | Compression |
|---|---|---|---|
| Nhạc nền | `.ogg`/`.wav` 44.1 kHz stereo, loop liền | Streaming | Vorbis 60–70 |
| SFX < 1 s | `.wav` mono | Decompress On Load | ADPCM, Force To Mono |
| SFX 1–5 s | `.wav` | Compressed In Memory | Vorbis 70 |

Nhạc nền ~−16 LUFS, SFX đỉnh ≤ −1 dBFS. Tên file = id.

### 13.3. Nhạc nền

`bgm_lobby` (Lobby, Động Phủ, Kiếm Trủng) · `bgm_nhan_gioi` (ch.1–4, 7, 9) · `bgm_loan_tinh_hai` (ch.5–6) · `bgm_am_minh` (ch.8) · `bgm_linh_gioi` (ch.10–12) · `bgm_ma_gioi` (ch.13–14) · `bgm_tien_gioi` (ch.15–19) · `bgm_boss` (boss ch.1–12) · `bgm_boss_final` (boss ch.13–19) · `bgm_tribulation` (lớp chồng 10 s khi Lôi Kiếp) · `bgm_endless` · `sting_victory`, `sting_defeat` (3–5 s, không lặp).

### 13.4. Hiệu ứng âm thanh

| Id | Khi nào | maxInstances | minInterval | Pitch ± |
|---|---|---|---|---|
| `sfx_sword_launch` | Kiếm lao đi | 4 | 0.05 | 8% |
| `sfx_sword_hit` | Kiếm trúng | 4 | 0.04 | 10% |
| `sfx_sword_crit` | Chí mạng | 2 | 0.1 | 5% |
| `sfx_lightning_chain` | Sét lan | 2 | 0.1 | 8% |
| `sfx_fire_burn` | Bắt đầu thiêu | 2 | 0.15 | 8% |
| `sfx_freeze` | Đóng băng | 2 | 0.15 | 8% |
| `sfx_enemy_die` | Quái chết | 3 | 0.05 | 12% |
| `sfx_enemy_explode` | Quái nổ | 2 | 0.1 | 6% |
| `sfx_enemy_shoot` | Quái bắn | 3 | 0.08 | 10% |
| `sfx_player_hurt` | Bị đánh | 1 | 0.2 | 5% |
| `sfx_player_dash` | Dash | 1 | 0.1 | 5% |
| `sfx_pickup_exp` | Nhặt Linh Châu | 3 | 0.03 | +2% mỗi viên liên tiếp, tối đa +30% |
| `sfx_pickup_stone` | Nhặt Linh Thạch | 2 | 0.05 | 5% |
| `sfx_level_up` | Lên cấp | 1 | — | 0 |
| `sfx_tribulation_warn` | Vòng cảnh báo sét | 3 | 0.1 | 5% |
| `sfx_tribulation_strike` | Sét đánh | 2 | 0.1 | 8% |
| `sfx_breakthrough` | Đột phá | 1 | — | 0 |
| `sfx_transform` | Biến Thân | 1 | — | 0 |
| `sfx_boss_appear` | Boss xuất hiện | 1 | — | 0 |
| `sfx_boss_warn` | Boss bắt đầu chiêu | 2 | 0.2 | 5% |
| `sfx_time_stop` | Ngưng đọng | 1 | — | 0 |
| `sfx_beast_skill` | Linh thú dùng chiêu | 2 | 0.2 | 8% |
| `sfx_revive` | Hồi sinh | 1 | — | 0 |
| `ui_click` | Bấm nút | 2 | 0.05 | 3% |
| `ui_popup_open`, `ui_popup_close` | Pop-up | 1 | 0.1 | 0 |
| `ui_card_pick` | Chọn thẻ | 1 | — | 0 |
| `ui_upgrade` | Nâng cấp | 1 | — | 0 |
| `ui_coin` | Số Linh Thạch tăng | 1 | 0.05 | tăng dần |
| `ui_error` | Không đủ tiền, QC chưa sẵn | 1 | 0.2 | 0 |
| `ui_dialogue_next` | Sang dòng thoại | 1 | 0.05 | 0 |

### 13.5. Âm thanh tạm

Editor `VkdkPlaceholderAudio` (menu *Tools → VKDK → Generate Placeholder Audio*): tạo `.wav` cho mọi id ở 13.3–13.4 bằng sóng sin/vuông/nhiễu ngắn (ghi bằng PCM 16-bit, 44.1 kHz), tạo `SoundData` + `SoundLibrary`. File thật cùng tên sẽ ghi đè, không sửa code. Âm thanh thật phải có giấy phép thương mại; ghi nguồn trong `Assets/_Project/Audio/LICENSES.md`.

---

## 14. THEO DÕI, QUYỀN RIÊNG TƯ, PHÁT HÀNH

### 14.1. Analytics & báo lỗi

- Firebase Analytics + Crashlytics (Firebase Unity SDK bản mới nhất, `google-services.json` trong `Assets/`). Chạy lại Android Resolver sau khi import.
- `AnalyticsService` là lớp duy nhất gọi Firebase. Khởi tạo ở `Boot` sau UMP: `FirebaseApp.CheckAndFixDependenciesAsync()`, rồi `FirebaseAnalytics.SetConsent` theo lựa chọn người dùng. Editor/Development: chỉ log, không gửi.
- Lỗi không gây crash: `Crashlytics.LogException`.
- Nếu chưa cài Firebase: `AnalyticsService` dùng bản giả (chỉ `Debug.Log`) bọc trong `#if VKDK_FIREBASE` để project vẫn biên dịch.

| Sự kiện | Tham số |
|---|---|
| `tutorial_step` | step_id |
| `chapter_start` | chapter, attempt, pill_id |
| `chapter_end` | chapter, result (win/lose/retreat), duration_s, realm_index, level, kills, revives |
| `tribulation` | chapter, index, result |
| `boss_result` | boss_id, result, time_to_kill_s |
| `card_pick` | card_id, level, offered_ids |
| `upgrade_buy` | type, id, new_level, cost |
| `pill_craft` | pill_id |
| `ad_reward` | placement, result (shown/failed/not_ready) |
| `endless_end` | duration_s, realm_index |

Không gửi thông tin cá nhân. Mục tiêu: crash < 0.5%, ANR < 0.2% (ngưỡng xấu của Google Play khoảng 1.09% và 0.47%). Không khởi tạo SDK hay đọc file lớn đồng bộ trên main thread lúc mở app.

### 14.2. Quyền riêng tư

- UMP: mỗi lần mở app `ConsentInformation.Update()` → `ConsentForm.LoadAndShowConsentFormIfRequired()` → chỉ khi `CanRequestAds()` mới khởi tạo quảng cáo. Nút "Tùy chọn quyền riêng tư" khi `PrivacyOptionsRequirementStatus == Required`. Thử bằng `ConsentDebugSettings` (vùng EEA giả lập).
- Đối tượng người chơi 13+. `RequestConfiguration.MaxAdContentRating = T`.
- Chính sách quyền riêng tư: URL công khai, nêu dữ liệu lưu trên máy, dữ liệu AdMob/Firebase thu thập, mục đích, cách từ chối, không dành cho dưới 13 tuổi, cách xóa dữ liệu, email liên hệ, ngày cập nhật.
- Data Safety: vị trí gần đúng, tương tác trong app, nhật ký sự cố/chẩn đoán, ID thiết bị/ID quảng cáo; mã hóa khi truyền; yêu cầu xóa qua email. Đối chiếu trang hướng dẫn Data Safety của AdMob và Firebase.

### 14.3. Play Console & cửa hàng

- Tài khoản cá nhân tạo sau 13/11/2023: Closed testing ≥ 12 người thử liên tục 14 ngày trước khi xin Production.
- Play App Signing; upload keystore tạo trong Unity, sao lưu, không commit.
- `Bundle Version Code` tăng mỗi lần tải lên.
- Khai có quảng cáo, khai dùng Advertising ID, làm bảng hỏi IARC, đặt `app-ads.txt` trên website.
- Trang cửa hàng: tên ≤ 30 ký tự, mô tả ngắn ≤ 80, mô tả đầy đủ ≤ 4.000, icon 512×512 PNG 32-bit, ảnh nổi bật 1024×500, ≥ 2 ảnh chụp (nên 4–8 ảnh ngang 1920×1080). Icon trong game dùng Adaptive Icon.

### 14.4. Kiểm tra trước phát hành

Thay Test Ad ID bằng ID thật; tắt Debug menu; texture 0 cảnh báo; thử 3 máy (RAM 3 GB, tai thỏ, Android 16); thử mất mạng, vào nền giữa trận/quảng cáo, tắt app khi đang lưu; save cũ đọc được ở bản mới; Crashlytics nhận crash thử; đủ giấy phép (truyện, font, âm thanh, hình, asset Unreal); Closed testing đủ điều kiện.
