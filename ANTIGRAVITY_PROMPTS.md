# PROMPT TRIỂN KHAI VẠN KIẾM ĐỘ KIẾP VỚI ANTIGRAVITY

Dán lần lượt từng prompt. **Chỉ chuyển sang giai đoạn sau khi giai đoạn trước đạt hết mục "Kiểm tra".**
Mỗi giai đoạn nên mở một cuộc hội thoại mới với agent để ngữ cảnh gọn; luôn đính kèm `VANKIEMDOKIEP_SPEC.md`.

Không dùng file `tailieukientrucgame.md` cũ.

---

## Giai đoạn 0 – Bạn tự làm (không cần agent)

1. Unity Hub → cài **Unity 6.3 LTS** (6000.3.x) + module **Android Build Support** (kèm OpenJDK, SDK & NDK).
2. Tạo project mới, template **Universal 2D**.
3. Package Manager → cài: **2D Animation**, **2D PSD Importer**, **Input System**, **Shader Graph** (2D Sprite và URP đã có sẵn). Khi Unity hỏi bật Input System backend, chọn **Yes** (Editor khởi động lại).
4. *Window → TextMeshPro → Import TMP Essential Resources*.
5. File → Build Profiles → chuyển sang **Android**.
6. Chép `VANKIEMDOKIEP_SPEC.md`, `ANTIGRAVITY_PROMPTS.md` và thư mục `docs/` vào thư mục gốc project (ngang hàng với `Assets/`).
7. Khởi tạo Git với `.gitignore` chuẩn cho Unity, commit lần đầu. Commit sau mỗi giai đoạn để có thể quay lại khi agent làm hỏng.
8. Mở thư mục project bằng Antigravity.

Google Mobile Ads plugin **chưa cài** ở bước này, để tới giai đoạn 5.

---

## Giai đoạn 1 – Nền tảng, dữ liệu, công cụ Editor

```
Đọc toàn bộ VANKIEMDOKIEP_SPEC.md, đặc biệt mục 0 (quy tắc), 8.6, 9, 10.

Nhiệm vụ giai đoạn 1 (chỉ làm phần này):
1. Tạo cấu trúc thư mục theo mục 10.1 (thêm file .gitkeep vào thư mục rỗng).
2. Tạo assembly definition: VKDK.Runtime cho Assets/_Project/Scripts, VKDK.Editor (chỉ Editor) cho Assets/_Project/Editor, tham chiếu VKDK.Runtime, Unity.InputSystem, Unity.TextMeshPro.
3. Viết toàn bộ class ScriptableObject trong mục 10.2 và các SO liệt kê ngay sau đó (PlayerBaseStats, EnemyData, WaveTable/WaveSegment, BossData/BossPhase/BossSkill, SwordData, UpgradeCardData, EvolutionData, TribulationConfig, EconomyConfig, HerbData, PillData, BeastData, DialogueData, AdConfig, UITheme). Trường phải đủ để chứa toàn bộ số liệu mục 9.
4. Viết Core: ObjectPooler, AutoReturnToPool, SaveSystem + PlayerSaveData + SettingsData (mục 10.4), GameManager (singleton, DontDestroyOnLoad), Loc + StringTable (mục 12.5; tạo StringTable "vi" và thêm key dần trong các giai đoạn sau).
   Thêm vào RealmData trường playerStatMultiplier (mục 11.1); EnemyData có behavior, tags, deathPool (mục 11.2–11.3).
5. Viết Editor: VkdkTexturePostprocessor và VkdkFxTool đúng như code mục 8.6.
6. Viết Editor VkdkDataSeeder (menu Tools/VKDK/Seed Data) tạo hoặc cập nhật asset SO cho: 14 RealmData, 3 TransformationData, PlayerBaseStats, TribulationConfig, EconomyConfig, 5 SwordData, 9 UpgradeCardData, 3 EvolutionData, 7 EnemyData, 8 BossData, ChapterData + WaveTable chương 1–4, HerbData, PillData, UITheme, AdConfig (Test ID của Google). Số liệu lấy chính xác từ mục 9. Chạy lại nhiều lần không tạo trùng (tìm theo id).

Không viết gameplay, UI, quảng cáo ở giai đoạn này.

Cuối cùng báo cáo: danh sách file, các TODO, và việc tôi cần làm tay trong Editor.
```

**Kiểm tra:**
- Console không có lỗi biên dịch.
- Có menu *Tools → VKDK → FX Tool* và *Tools → VKDK → Seed Data*.
- Chạy Seed Data → thư mục `ScriptableObjects/` có asset; mở `ChapterData` chương 2, thấy boss Mặc Đại Phu 4.000 HP.
- Kéo một PNG thử vào `Art/Sprites/` → Inspector đã là Sprite, ASTC 4x4 ở tab Android.

---

## Giai đoạn 2 – Lõi chiến đấu (chơi được bằng hình tạm)

```
Đọc VANKIEMDOKIEP_SPEC.md mục 0, 3.1, 3.2, 5.1, 5.2, 9.2–9.7, 10.3.

Nhiệm vụ giai đoạn 2:
1. PlayerController + PlayerStats (đọc PlayerBaseStats), Dash có bất tử, bất tử 0.5 s sau khi bị đánh.
2. FloatingJoystick (nửa trái màn hình, đa chạm), SafeAreaHandler. Trong Editor cho phép WASD + Space (Dash) để thử nhanh.
3. FlyingSwordController + FlyingSword: quỹ đạo, state machine Orbiting/Attacking/Returning, OverlapCircleNonAlloc, sát thương theo công thức 9.4, hiệu ứng thuộc tính Lôi/Hỏa/Băng.
4. EnemyAI theo 7 loại quái mục 9.6, WaveSpawner đọc WaveTable, trần 150 quái, scale theo phút.
5. LinhChauPickup (hút, gộp khi > 150), EXP/level, Tu Vi, Nộ theo 9.3. DamagePopup bằng TMP từ pool.
6. Editor script VkdkPlaceholderArt (menu Tools/VKDK/Generate Placeholders): sinh sprite tạm (hình tròn/tam giác màu) cho người chơi, kiếm, 7 loại quái, Linh Châu, để chơi được khi chưa có hình.
7. Editor script VkdkSceneBuilder (menu Tools/VKDK/Build Combat Scene): tạo scene Combat với Camera (Post Processing bật), Global Volume có Bloom, Player, ObjectPooler đã cấu hình pool, WaveSpawner gán ChapterData chương 1, Canvas có joystick. Mọi tham chiếu gán bằng code, tôi không phải kéo thả.
8. AudioManager, SoundData, SoundLibrary, AudioMixer (mục 13.1) và Editor VkdkPlaceholderAudio (mục 13.5). Gọi các SFX chiến đấu ở đúng chỗ.
9. DebugMenu (mục 12.6) với các lệnh dùng được ở giai đoạn này: bất tử, tốc độ game, hiện FPS/số quái/đạn/pool, nhảy phút.

Chưa làm: thẻ Kỳ Ngộ, Lôi Kiếp, Biến Thân, boss, HUD đầy đủ.
Tuân thủ hiệu năng mục 0.7. Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- Chạy Generate Placeholders → Generate Placeholder Audio → Build Combat Scene → Play.
- Nghe được tiếng kiếm trúng, không bị chồng tiếng khi nhiều kiếm. F1 mở Debug menu.
- Di chuyển được, 3 kiếm tự đánh Yêu Lang, quái chết rơi Linh Châu, nhặt được, lên cấp (xem log hoặc thanh tạm).
- Mở Profiler khi có ~100 quái: GC Alloc trong `Update` bằng 0 hoặc gần 0.

---

## Giai đoạn 3 – Tiến trình trong trận & boss

```
Đọc VANKIEMDOKIEP_SPEC.md mục 3.2, 4, 5.3, 7.1 (CombatHUD, UpgradeCardPopup, RevivePopup, ResultPopup), 9.5, 9.8, 9.9, 9.12.

Nhiệm vụ giai đoạn 3:
1. UpgradeSystem + UpgradeCardPopup: dừng game khi lên cấp, 3 thẻ theo trọng số, quy tắc 9.5, tiến hóa (chỉ cần Tịch Tà Thần Lôi và Đại Canh Kiếm Trận ở mức logic; hình tạm).
2. TribulationSystem theo 4.1 và 9.9, có telegraph 0.8 s. Đột phá đổi tên cảnh giới theo runRealms.
3. TransformationHandler: Nộ, 3 trạng thái, static event OnTransformationChanged (mục 1.2).
4. BossController chung + BossData cho 4 boss chương 1–2 (Lang Vương, Yêu Lang Huyết Nhãn, Khôi Lỗi Đồng, Mặc Đại Phu) đủ phase và chiêu theo 9.8. Mỗi chiêu có telegraph.
5. CombatHUD: HP, EXP, tên cảnh giới, đồng hồ, thanh máu boss, Btn_Dash (cooldown radial), Btn_Transform, Btn_Tribulation (chỉ hiện khi đủ Tu Vi, rung nhẹ). ThemeTint đổi màu viền.
6. RevivePopup (tạm thời chưa có quảng cáo: nút "Hồi sinh (test)"), ResultPopup, menu tạm dừng có Độn Thổ Phù (mục 3.2).
7. Cập nhật VkdkSceneBuilder để dựng thêm các UI trên.

Chưa làm: Lobby, meta, quảng cáo. Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- Chơi trọn chương 1 trong Editor: chọn thẻ, Độ Kiếp 3 lần, đổi Thanh/Kim/Hồng Lôi, boss phút 5 và 10 xuất hiện, thắng hiện ResultPopup.
- Đổi ChapterData sang chương 2: Mặc Đại Phu dùng Đoạt Xá khi dưới 50% HP và Dash thoát được.
- Commit Git.

---

## Giai đoạn 4 – Bộ UI, Lobby, cốt truyện, meta cơ bản

```
Đọc VANKIEMDOKIEP_SPEC.md mục 2, 3.3 (Động Phủ, Linh Căn, Kiếm Trủng), 7, 8.1–8.4, 9.10, 10.1 (Scenes), 10.3.

Nhiệm vụ giai đoạn 4:
1. UI Kit runtime: UIPanel, UIButton, SkillButton, UIBar, CurrencyCounter, Toast, ThemeTint, UITheme. Tween bằng coroutine unscaled.
2. Editor VkdkUIKitBuilder (menu Tools/VKDK/Build UI Kit): tạo các prefab UI_* trong Prefabs/UI/Kit theo bảng 8.4, dùng sprite tạm nếu chưa có ảnh trong Art/UI (tên file theo bảng 8.3, khi có ảnh thật thì chạy lại để gán).
3. Scene Boot → Lobby → Combat. Boot đọc save, khởi tạo Manager.
4. LobbyUI, ChapterSelectUI (chương 1–4, khóa/mở theo save), CaveDwelling (4 nâng cấp + Linh Căn, giá theo 9.10), SwordTomb (mở/nâng/trang bị kiếm), ChapterManager, thưởng qua chương/chơi lại.
5. DialogueRunner + DialogueUI; tạo DialogueData mẫu 2–3 dòng cho chương 1–4, đánh dấu TODO(dialogue).
6. Lưu/đọc toàn bộ tiến trình qua SaveSystem.
7. Cập nhật VkdkDataSeeder nếu thiếu dữ liệu. Tạo scene UIKit_Preview.

Chưa làm: quảng cáo, Chưởng Thiên Bình, luyện đan, linh thú. Các nút quảng cáo đặt sẵn nhưng gọi hàm giả trả thưởng ngay (có log "AD STUB").
Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- Mở từ scene Boot: Lobby → chọn chương 1 → chơi → về Lobby, Linh Thạch tăng, chương 2 mở.
- Nâng HP gốc trong Động Phủ, vào trận thấy HP tăng. Tắt game mở lại vẫn còn.
- Device Simulator: thử máy có tai thỏ, UI không bị che.

---

## Giai đoạn 5 – Quảng cáo AdMob

**Trước khi dán prompt, bạn làm tay:**
1. Tải Google Mobile Ads Unity Plugin bản 11.x mới nhất từ https://github.com/googleads/googleads-mobile-unity/releases, import `.unitypackage`.
2. *Assets → Google Mobile Ads → Settings*: nhập Android App ID (khi chưa có app thật, dùng App ID test của Google trong tài liệu AdMob).
3. *Assets → External Dependency Manager → Android Resolver → Force Resolve*. Đợi xong, không lỗi.

```
Đọc VANKIEMDOKIEP_SPEC.md mục 7.2, 9.10, 9.12, 10.3 (AdMobManager), 10.5.

Plugin Google Mobile Ads đã được cài. Nhiệm vụ giai đoạn 5:
1. AdMobManager: xin đồng ý UMP rồi mới Initialize; RaiseAdEventsOnUnityMainThread = true; Banner/Interstitial/Rewarded với Ad Unit ID lấy từ AdConfig (mặc định Test ID chính thức của Google); xử lý mất mạng, chưa load, tự load lại.
2. AdPlacementController: thay toàn bộ "AD STUB" ở giai đoạn 4 bằng quảng cáo thật theo bảng 7.2, gồm giới hạn: Nhận Cả 3 và Hồi sinh 1 lần/trận, nút + ở Lobby 5 lần/ngày, Interstitial cooldown 180 s, bỏ ván đầu, không ngay sau Rewarded, không trong hội thoại hay trước boss.
3. Banner chỉ hiện ở Lobby.
4. Khi quảng cáo không có sẵn: nút xám + Toast "Quảng cáo chưa sẵn sàng".
5. Đảm bảo assembly VKDK.Runtime tham chiếu được thư viện của plugin (GoogleMobileAds, GoogleMobileAds.Ump). Nếu không biên dịch được, giải thích nguyên nhân và sửa asmdef.

Không đổi logic gameplay. Báo cáo như giai đoạn 1.
```

**Kiểm tra (phải chạy trên điện thoại Android thật):**
- Build Development → cài máy → hộp thoại đồng ý UMP hiện (nếu ở khu vực yêu cầu).
- Quảng cáo test hiện ở cả 5 điểm + nút `+`, nhận đúng thưởng.
- Tắt mạng: nút quảng cáo xám, game không treo.

---

## Giai đoạn 6 – Chưởng Thiên Bình, luyện đan, linh thú, tiến hóa

```
Đọc VANKIEMDOKIEP_SPEC.md mục 3.3 (Chưởng Thiên Bình, Luyện Đan, Linh Thú), 5.3, 9.11, 10.3 (TimeService, HeavenlyVial, AlchemySystem, BeastCompanion).

Nhiệm vụ giai đoạn 6:
1. TimeService theo mục 1.2 và 10.3.
2. HeavenlyVial: giọt Lục Dịch theo giờ thực, tối đa 3, tính cả thời gian tắt game.
3. AlchemySystem: 3 ô linh điền, gieo, chín sau 24 h hoặc tưới Lục Dịch, luyện đan theo 9.11. UI trong Động Phủ.
4. Ô chọn đan trước trận ở ChapterSelectUI; áp hiệu ứng đan trong trận (Trúc Cơ Đan, Tuyết Phách Hoàn, Giáng Trần Đan, Huyền Thiên Đan hồi sinh miễn phí).
5. BeastCompanion + Huyết Ngọc Tri Nhện (thưởng chương 3), ô Linh Thú trong Kiếm Trủng.
6. Hoàn thiện 3 tiến hóa mục 5.3 (Càn Lam Băng Diễm cần Thần Luân Hư Thiên Đỉnh: tạo dữ liệu và logic, khóa tới chương 6).
7. Boss chương 3–4 (Huyết Văn Mãng, Huyết Văn Mãng Vương, Hộ pháp Ma Diễm Tông, Trưởng lão Hợp Hoan Tông) theo 9.8.

Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- Chỉnh giờ máy lùi lại 1 ngày: Lục Dịch không tăng bất thường.
- Luyện được Trúc Cơ Đan, dùng trước trận, lần Độ Kiếp đầu cần 240 Tu Vi.
- Chơi được chương 3–4 với đủ boss.

---

## Giai đoạn 7 – Nội dung chương 5–19

```
Đọc VANKIEMDOKIEP_SPEC.md mục 11 (toàn bộ) và mục 2.1.

Nhiệm vụ giai đoạn 7:
1. EnemyAI: đủ 11 hành vi mục 11.2 và deathPool; khắc chế theo nhãn mục 11.3 (cập nhật cả quái chương 1–4).
2. Áp RealmData.playerStatMultiplier cho HP và sát thương người chơi trong trận (mục 11.1).
3. Thư viện BossSkill đủ 19 chiêu mục 11.6, mỗi chiêu một class, tham số đọc từ BossData. Chuyển boss chương 1–4 sang dùng thư viện. Hỗ trợ boss đôi chương 14 (2 thanh máu, Enrage).
4. Cập nhật VkdkDataSeeder: 29 EnemyData mục 11.4, ChapterData + WaveTable chương 5–19 mục 11.5, 30 BossData mục 11.7, linh thảo và đan mục 11.10.
5. Cập nhật VkdkPlaceholderArt: sprite tạm cho quái và boss mới (màu theo nhãn, hình theo hành vi), tileset tạm cho mỗi khu vực.
6. Linh thú đủ 4 loại và giai đoạn tiến hóa (mục 11.8); ArtifactController cho Hư Thiên Đỉnh và Phong Lôi Sí; CompanionPassives cho Ngân Nguyệt và Nam Cung Uyển (mục 11.9); thẻ Thời Gian Pháp Tắc.
7. ChapterSelectUI hiện đủ 19 chương theo 3 giới diện; chương 15 áp giảm bonus Động Phủ.
8. DebugMenu: thêm "Mở mọi chương", "+10.000 Linh Thạch", "+Lục Dịch/linh thảo", "Chọn thẻ bất kỳ".

Báo cáo như giai đoạn 1, kèm danh sách chiêu/hành vi còn chưa đúng đặc tả.
```

**Kiểm tra:**
- Debug "Mở mọi chương", chơi thử chương 5, 10, 14, 19 với Debug bất tử: đủ quái, boss đúng chiêu, boss đôi chương 14 qua được khi cả hai chết.
- Cổ Hoặc Kim: có TimeStop với màn hình mất màu dần, Rewind đúng một lần khi xuống dưới 30%.
- Profiler ở phút 10 chương 19: GC Alloc trong Update gần 0.

---

## Giai đoạn 8 – Hướng dẫn, Cài đặt, Tạm dừng, Vô Tận

```
Đọc VANKIEMDOKIEP_SPEC.md mục 12 (toàn bộ).

Nhiệm vụ giai đoạn 8:
1. TutorialManager, TutorialStep (SO), TutorialOverlay, TutorialAnchor; tạo 14 bước mục 12.1 qua VkdkDataSeeder; gắn TutorialAnchor vào các nút liên quan trong VkdkSceneBuilder/VkdkUIKitBuilder.
2. SettingsPopup đủ mục 12.3 (nút Tùy chọn quyền riêng tư chỉ hoạt động khi có plugin quảng cáo; trước đó ẩn).
3. PausePopup mục 12.4, tự mở khi app vào nền giữa trận.
4. EndlessMode mục 12.2, nút vào Vô Tận ở ChapterSelectUI.
5. Chuyển toàn bộ chữ UI còn viết cứng sang Loc.Get.
6. DebugMenu: thêm "Reset hướng dẫn", "Giả lập hết quảng cáo / mất mạng".

Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- Debug "Xóa dữ liệu" (trong Cài đặt) → chơi lại từ đầu: đủ 9 bước hướng dẫn trong trận, Lôi Kiếp đầu chỉ 5 tia; về Lobby có bước Động Phủ.
- Chỉnh âm lượng, chất lượng Thấp: Bloom tắt, FPS 30. Tắt mở game vẫn giữ cài đặt.
- Vô Tận: đổi bản đồ ở phút 5, boss xuất hiện, kỷ lục được lưu.

---

## Giai đoạn 9 – Analytics & báo lỗi

**Bạn làm tay trước:**
1. Tạo project trên console.firebase.google.com, thêm app Android với đúng package name, tải `google-services.json` vào `Assets/`.
2. Tải Firebase Unity SDK bản mới nhất, import `FirebaseAnalytics.unitypackage` và `FirebaseCrashlytics.unitypackage`.
3. *Android Resolver → Force Resolve*.

```
Đọc VANKIEMDOKIEP_SPEC.md mục 14.1.

Firebase đã được cài. Nhiệm vụ giai đoạn 9:
1. AnalyticsService bọc Firebase, khởi tạo ở Boot sau UMP, đặt consent mode. Thêm scripting define VKDK_FIREBASE; khi không có define thì dùng bản giả chỉ log.
2. Gửi đủ 10 sự kiện mục 14.1 ở đúng chỗ.
3. Crashlytics.LogException cho lỗi đọc save, lỗi tải quảng cáo lặp lại.
4. DebugMenu: nút "Crash thử" (chỉ Development).
Báo cáo như giai đoạn 1.
```

**Kiểm tra:** build Development lên máy thật, bật DebugView của Firebase, thấy sự kiện `chapter_start`/`chapter_end`; bấm "Crash thử", Crashlytics nhận báo cáo sau khi mở lại app.

---

## Giai đoạn 10 – Gắn hình thật, tối ưu, build phát hành

**Bạn làm tay trước:** đưa ảnh thật vào `Art/` theo quy ước tên (mục 8.3, 8.5), flipbook vào `Art/VFX_Unreal/`, tạo `M_FX_Additive_Template`, `M_FX_Alpha_Template`, `SG_Sprite_Additive`, dùng FX Tool tạo prefab VFX.

```
Đọc VANKIEMDOKIEP_SPEC.md mục 6, 8, 10.5.

Nhiệm vụ giai đoạn 10:
1. Thay sprite tạm bằng sprite thật theo tên file (viết Editor script ánh xạ tên → slot trong SO/prefab; báo cáo những slot còn thiếu ảnh).
2. Đăng ký mọi prefab VFX trong Prefabs/VFX vào ObjectPooler, gọi đúng hiệu ứng ở các chỗ tương ứng (bảng 6.1).
3. Tạo Sprite Atlas theo mục 8.4.
4. Editor script VkdkBuildSettings (menu Tools/VKDK/Apply Android Settings): package name, min API 24, target API 36, IL2CPP, ARM64, .aab.
5. Rà hiệu năng: liệt kê chỗ còn cấp phát bộ nhớ trong Update, Canvas bị rebuild mỗi frame, overdraw lớn.

Báo cáo như giai đoạn 1.
```

**Kiểm tra:**
- *Tools → VKDK → FX Tool → Kiểm tra ngân sách texture*: 0 cảnh báo.
- Máy Android tầm trung giữ khoảng 60 FPS ở phút 10 chương 4 (150 quái).
- Build `.aab` thành công, tải lên Closed Testing trên Play Console.
- Trước khi bật quảng cáo thật: thay Test ID trong `AdConfig` bằng ID thật, **không** tự bấm quảng cáo thật.
- Làm hết danh sách mục 14.2–14.4 của spec (chính sách quyền riêng tư, Data Safety, Play Console, trang cửa hàng, Closed testing 12 người × 14 ngày).

---

## Prompt dùng khi gặp lỗi

```
Unity báo lỗi sau (dán nguyên văn từ Console, gồm file và số dòng):
<dán lỗi>

Sửa lỗi này, không đổi thiết kế trong VANKIEMDOKIEP_SPEC.md và không sửa file không liên quan.
Giải thích ngắn nguyên nhân.
```

## Prompt khi agent làm sai đặc tả

```
Phần <tên tính năng> chưa đúng VANKIEMDOKIEP_SPEC.md mục <số mục>:
- Đặc tả: <trích>
- Hiện tại: <mô tả hành vi thấy trong game>
Sửa cho khớp đặc tả. Nếu bạn cho rằng đặc tả có vấn đề, nói rõ lý do và hỏi tôi trước khi đổi.
```

## Prompt khi muốn đổi thiết kế

```
Tôi muốn đổi: <mô tả>.
1. Cập nhật VANKIEMDOKIEP_SPEC.md và file HTML tương ứng trong docs/ cho khớp nhau.
2. Liệt kê script và asset bị ảnh hưởng.
3. Chờ tôi xác nhận rồi mới sửa code.
```
