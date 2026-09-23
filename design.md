# 設計文件（Design Doc）

> 本文件說明 `index.html` 目前實作的功能現況：哪些功能「已啟用、使用者實際會用到」，哪些功能「程式碼已存在但目前被隱藏/關閉/未串接」。專案是單一 HTML 檔（HTML + CSS + JS 全部寫在同一份 `index.html`），搭配 8th Wall AR 引擎、Three.js 3D 引擎與手機感測器 API，目的是用手機的陀螺儀/加速度計 + 虛擬搖桿操控一台 3D 直升機模型，並疊加在手機鏡頭畫面或 AR 場景中。

## 專案結構

| 檔案 | 用途 |
|---|---|
| `index.html` | 唯一的程式進入點，包含畫面、樣式、控制邏輯、Three.js 3D 渲染、8th Wall AR 串接 |
| `low_poly_helicopter.glb` | 直升機 3D 模型（含旋翼自轉動畫） |
| `city-street_free_for_game.glb` | 城市街景 3D 模型，作為非 AR 模式下的地面場景 |
| `orientation-guide.png` | 「姿態引導遮罩」畫面中顯示的示意圖（教使用者如何垂直拿手機） |
| `引導.png` | **未被程式碼引用的圖片**，目前沒有任何 `<img>` 或 CSS 指向此檔案，屬於未使用的素材 |

---

## 一、目前已啟用的功能

### 1. 啟動流程
- 進入頁面即自動請求 `DeviceOrientationEvent` / `DeviceMotionEvent` 權限（iOS 13+ 需要使用者手勢才能請求，因此若首次自動請求失敗，會監聽第一次點擊/觸碰後重試）。
- 權限畫面（`#screen-permission`）目前在 CSS 中被強制隱藏（`class="hidden"`），實際上不會顯示給使用者看，整個啟動是無感自動完成的。

### 2. 感測器讀值
- 監聽 `deviceorientation`（alpha 朝向、beta 俯仰、gamma 橫滾）與 `devicemotion`（含重力加速度）。
- 啟動當下的 alpha/beta/gamma 會被記錄為 offset 基準值。
- 讀值頻率（Hz）、姿態、加速度、搖桿座標、抓取狀態會即時顯示在偵錯面板的狀態列（`#status-bar`）。

### 3. 虛擬搖桿（水平移動）
- 支援觸控（含多點觸控下的 identifier 追蹤）、滑鼠拖曳（桌機測試用）、鍵盤 WASD / 方向鍵。
- 搖桿輸出 `joyH`（左右）、`joyV`（前後），驅動直升機水平移動速度。
- 放開後有回彈動畫（`snap` transition）。

### 4. 上升按鈕（垂直移動）
- 按住「上升」按鈕（`#alt-btn`）時直升機以固定加速度上升；放開後緩慢下沉（模擬重力，類似 Flappy Bird 但反應更慢、更沉重）。
- 這是取代「原本用陀螺儀傾斜控制上下」的新方案（見 commit fb5d87c）。

### 5. 3D 場景渲染（Three.js）
- 載入直升機與城市街景 GLB 模型，直升機播放內建旋翼動畫。
- 直升機朝向會依「目前飛行方向」平滑轉向（yaw + pitch 分開計算，避免飛正後方時翻滾跳動），且**只在搖桿有水平輸入時才更新朝向**（上升/下沉不影響機頭朝向）。
- 「箭頭錨點 + 彈簧阻尼」的雙層物理效果：
  - 水平方向：機身像被繩子拴在一個沿飛行方向前移的「箭頭錨點」上，用彈簧-阻尼追隨，產生慣性甩尾感。
  - 垂直方向：獨立的「垂直回彈層」與「俯仰角回彈層」，在上升/下降速度變化的瞬間踢一次彈簧衝量，模擬電梯/公車煞車的慣性傾斜與位移，飛行速度穩定時不會有持續偏移。
- 城市場景／直升機的縮放、位置、角度皆可即時調整（見下方偵錯面板）。

### 6. AR 模式（8th Wall）
- 若裝置支援且 8th Wall SDK 載入成功，自動切換為 AR 模式（背景鏡頭改由 8th Wall 接管，取代原本的 `getUserMedia` 視訊背景）。
- SLAM 地板偵測 + 十字準星（reticle），採「連續多幀穩定 + 平滑濾波」判斷地板偵測結果是否可信，避免放置時懸空或陷地。
- 點擊畫面且同時滿足「地板已偵測、偵測結果穩定、手機角度符合姿態引導」三個條件時，才會把直升機放置到該位置（並補償模型旋轉中心/箭頭前移造成的視覺偏移）。
- 長按畫面 2 秒可重置放置位置，重新進入「請點按地板放置」的狀態。
- 若 3 秒內未偵測到 8th Wall 或裝置不支援，會 fallback 回原本非 AR 的 `requestAnimationFrame` 渲染迴圈。

### 7. 姿態引導遮罩
- 直升機尚未放置（AR 模式）或一般模式下，會持續顯示一個「請將手機轉成垂直」的引導遮罩，並用一個圓環 UI 即時顯示目前 β/γ 角度誤差，符合容許範圍才會淡出。
- 橫式持握時會自動把 β/γ 角色互換再計算誤差。

### 8. 版面與操作輔助
- 橫式/直式切換按鈕：切換版面配置，並嘗試呼叫 `screen.orientation.lock()` 鎖定螢幕方向；若手機實際轉回直式，用 CSS `rotate(90deg)` + dvh/dvw 互換技巧把橫式 UI 轉回正確方向。
- 左右對調按鈕：把搖桿與（上升）按鈕的左右位置互換，方便左右手習慣不同的使用者。
- Wake Lock：進入即時畫面後嘗試保持螢幕常亮。
- 停止按鈕：停用感測器監聽、重置搖桿/按鈕狀態、解除螢幕方向鎖定，並立即重新啟動（不會退回權限畫面）。

### 9. 偵錯面板（`#debug-panel`，點左上角「調整面板」開關）
浮動於畫面左側、預設隱藏，內含大量即時可調參數（每個滑桿都可自訂上下限），用於現場微調手感與視覺效果，不影響核心邏輯：
- 相機偏移／看向點／縮放（offsetX/Y/Z、lookX/Y/Z、zoom）
- 城市場景縮放／位置／旋轉角度
- 直升機可移動範圍（X/Z）
- 直升機模型縮放、初始位置（X/Y/Z）、旋轉偏移（X/Y/Z）、旋轉中心偏移（X/Y/Z）
- 箭頭前移量、水平拉力強度/阻尼
- 垂直回彈強度/阻尼/阻尼加成/衝量增益
- 俯仰角回彈強度/阻尼/衝量增益
- 「陀螺儀控制角度／方向」開關（見下方「未啟用功能」）

---

## 二、程式碼存在但目前未啟用 / 已隱藏的功能

### 1. 抓取按鈕（`#grab-btn`）
- HTML 中被加上 `class="hidden"`（見 commit 81cdda5「隱藏抓取鈕」），畫面上看不到、也點不到。
- 但背後邏輯完整保留：`setGrab()`、觸控/滑鼠事件監聽、狀態列 `sb-grab` 顯示「抓取：按下／放開」、`window.controlState.grabPressed` 狀態。
- 目前沒有任何 3D 行為（例如真的抓取/釋放物件）綁定這個狀態，是為未來功能預留的半成品。

### 2. 陀螺儀控制機身角度／飛行方向（實驗性開關）
- 偵錯面板中的核取方塊「陀螺儀控制角度／方向（實驗性，預設關閉）」，對應 `window.gyroControlEnabled`，預設 `false`。
- 開啟後，搖桿的「前後左右」座標系會隨手機傾斜角度（陀螺儀）一起旋轉（`deltaQuat`），也就是搖桿方向會跟著機身姿態變化；關閉時搖桿方向固定不受傾斜影響。
- 目前預設關閉，一般使用者不會用到，僅供內部測試/微調時開啟。

### 3. 朝向指示棒（`headingArrow`，黃色箭頭）
- Three.js `ArrowHelper`，原本用來視覺化「目前搖桿輸入會飛去的方向」，方便偵錯。
- 建立後立刻設定 `headingArrow.visible = false`，畫面上永久隱藏，但其 `position`/`direction` 每幀仍持續更新（作為箭頭錨點物理計算的依據），只是不渲染出來。

### 4. 舊版「陀螺儀傾斜控制上下」邏輯
- 目前上下移動已改為「上升按鈕」（見上方功能 4），commit fb5d87c 提到「上下改用按鈕控制」，代表先前是用手機傾斜角度直接控制上升/下降。
- 目前程式碼中已看不到舊的傾斜控制上下邏輯（已被取代/移除），但 `VERT_FALL_SPEED`／重力式下沉的設計仍保留了「類 Flappy Bird」的操作精神，供未來若想恢復類似手感時參考。

### 5. 未使用的素材檔
- `引導.png`：專案內有這張圖片檔，但 `index.html` 中沒有任何地方引用它，目前是一個未使用的靜態資源。

---

## 三、可能的後續延伸方向（非現有程式碼，僅為觀察到的擴充點）

- 抓取按鈕已有完整前端狀態（`grabPressed`），可考慮串接「抓取城市場景中的物件」等玩法。
- 「陀螺儀控制角度／方向」開關若要正式開放給一般使用者，需要重新設計如何與姿態引導遮罩、AR 模式共存（目前 AR 模式下機頭朝向已改為「只跟移動方向轉」，兩者語意需要對齊）。
- 偵錯面板目前參數眾多且皆為浮點手動調整，未來若手感穩定，可以把目前的預設值直接寫死、移除偵錯面板以精簡上線版本的程式碼與 UI。

---

## 四、AR 追蹤技術現況與已知限制（記錄於 2026-09-23，基準 commit 3329e0d；WebXR 備援路徑於同日加入）

> 本節記錄 8th Wall 架構的技術現況、已知限制、已嘗試過的替代方案，
> 供之後要調整/回滾追蹤架構時，能明確比較差異。

### 1. 目前架構：8th Wall SLAM（單眼視覺追蹤）

- 追蹤引擎：`@8thwall/engine-binary`（[index.html:7](index.html#L7)），核心 SLAM/world-tracking 是**封閉原始碼二進位檔**，即使 8th Wall 框架其餘部分已在 2026-02-28 開源，這顆追蹤核心仍不在開源範圍內，對外只開放 `XR8.XrController.hitTest()` / `recenter()` / `configure()` 等有限 API，**沒有**任何「餵外部 IMU 數據進去做融合」或「取得平面幾何邊界」的介面。
- 定位方式：純視覺單眼 SLAM，每幀從相機畫面反推 6DoF camera pose（[index.html:2226-2251](index.html#L2226-L2251)），**沒有**跟手機陀螺儀/加速度計做感測器層級融合（VIO）。
- 地板偵測：`hitTest(0.5, 0.5, { DETECTED_SURFACE: true })` 只在「尚未放置」時對螢幕中央做單點命中＋法向量偵測（[index.html:2262](index.html#L2262)），**沒有**回傳平面幾何/邊界，畫面上的偵測平面網格是固定倍率的示意值，不是真實平面範圍（[index.html:1658-1661](index.html#L1658-L1661)）。放置後這個 hit-test 就不再持續執行，物件位置改為完全依賴固定錨點 `arAnchorPos`。

### 2. 已知漂移成因（實機測試結論，見 [[project-ar-slam-drift]]）

- 走 50cm 飄 5cm+、走 2m 飄 20-30cm+（約 10-15%），但**雙腳不動、純轉動手機也會飄**——證實漂移主因是「轉動-位移混淆」：單眼 SLAM 在轉動當下容易把純旋轉誤判成部分平移，把錯誤位移直接寫進 camera position，而非走動距離的累積誤差。

### 3. 目前的因應策略（濾波，非根治）

- `cameraPosFilter`／`cameraQuatFilter`：1€ Filter 濾波器，基礎 beta 用實機調校值（`CAMERA_POS_BASE_BETA = 4.0`，[index.html:1616](index.html#L1616)）。
- 動態壓低 beta：每幀用 SLAM 自己輸出的相機四元數算出前後幀角速度 `rotSpeed`（[index.html:2237-2242](index.html#L2237-L2242)），轉動越快，beta 壓得越低（濾波加重），停止轉動後立刻回到原本高值，不影響直線走動跟手感。強度可用偵錯面板滑桿 `window.camRotDampenDebug` 即時調整。
- **限制**：`rotSpeed` 是從 SLAM 自己「可能已被污染」的輸出反推的次級訊號，並非獨立的原始感測器數據，本質上只是事後補丁，天花板有限。

### 4. 已嘗試並回退的方案

- **背景無感校正**（commit 8ee30cf 加入、b56bc58 回退）：原本嘗試放置後持續對螢幕中央做 hit-test，在偵測穩定時無感校正 `arAnchorPos`。實機測試證實行不通：飛行中準心掃到的地板跟原本放置點無關，拿它校正等於把直升機拖去隨機位置（[index.html:2253-2260](index.html#L2253-L2260)）。結論：沒有 VPS/持久錨點這類獨立參考點，軟體無法自行判斷「現在對準的地板是不是原本那個點」，只有使用者的眼睛判斷得出來，見 [[feedback-ar-autocorrect-needs-user-judgment]]。

### 5. WebXR Device API（已實作為備援路徑，2026-09-23 加入）

- 原理差異：WebXR `immersive-ar` 不是網頁自己做感測器融合，而是把追蹤責任整個交給**作業系統原生的 ARCore**，three.js 的 `renderer.xr` 每幀直接覆寫 `camera` 的 position/quaternion/projectionMatrix，數值來自 ARCore 融合好的 pose。這是跟原生 APP 同一套底層引擎，理論上能解決 8th Wall 純視覺 SLAM 的漂移問題。
- **平台限制（關鍵，決定了必須做成「備援」而非「取代」）**：
  - Android：Chrome 走 ARCore，支援 `immersive-ar`，且僅限 Google [ARCore 支援裝置清單](https://developers.google.com/ar/devices) 內、經過官方校正的機型。
  - iOS：**完全不支援**。原因是 Apple 規定所有 iOS 瀏覽器（含 Chrome/Firefox for iOS）底層都必須用 WebKit 引擎，不能用自己的引擎；WebKit 未實作 `immersive-ar`，所以無論裝哪個瀏覽器結果都一樣。歐盟 DMA 雖然從 iOS 17.4 起理論上允許他牌引擎，但實務上尚未有廠商鋪開且 WebXR AR 支援未經驗證，不能當作現階段依據。
- **評估過但放棄的「自己抓陀螺儀+加速度計融合」方案**：`DeviceMotionEvent` 在網頁上確實抓得到 `rotationRate`/`acceleration`，但：
  1. 時間戳跟相機畫格對不齊（瀏覽器事件迴圈延遲、iOS 降頻），融合時間沒對齊反而可能引入新誤差；
  2. 缺乏相機-IMU 外部校正參數（廠商出廠校正資料，網頁 API 不會給）；
  3. 就算自己算得再準，8th Wall 的 SLAM 是封閉二進位，沒有介面可以把外部 IMU 數據餵回去融合，最多只能做二次濾波補丁（跟目前 `camRotDampenDebug` 同等級，非根治）。
  - 結論：此方案效益有限，**不是**真正的解法，只有 WebXR（交給 ARCore 原生融合）才是根治方向，但受限於上述 iOS 限制。

### 6. WebXR 備援路徑的實作方式

因應上述平台限制（用戶決定：保留 8th Wall 當備援，見決策記錄），採用「WebXR 優先、失敗或不支援時自動退回 8th Wall／桌機」的架構，兩套邏輯共存於同一份 `index.html`：

- **啟動流程**（[index.html:2493-2516](index.html#L2493-L2516)）：頁面載入時先呼叫 `navigator.xr.isSessionSupported('immersive-ar')` 探測支援度。
  - 不支援（iOS、非 ARCore Android、桌機）：完全不受影響，直接照原本流程跑 `_startXR()`（8th Wall）或桌機 fallback，使用者不會看到任何差異。
  - 支援：顯示 `#webxr-enter-overlay`「點一下畫面進入 AR」提示——這是必要的，因為 `requestSession('immersive-ar')` 依規範需要真正的使用者手勢（transient activation）才能呼叫，無法像 8th Wall 一樣無感自動啟動。使用者點擊後才呼叫 `requestSession`，成功則進入 `_startWebXR()`；失敗（例如裝置回報支援但這次沒給到 `hit-test`）則自動退回 8th Wall／桌機模式，並補呼叫 `startCameraBackground()`（因為預期用 WebXR 時會先跳過 `getUserMedia`，避免兩邊搶後鏡頭）。
- **`_startWebXR()`**（[index.html:2419-2457](index.html#L2419-L2457)）：`renderer.xr.setSession(session)` 後，camera pose 完全交給 three.js 自動管理，不需要 1€ Filter／旋轉壓低 beta 這類補丁（這正是 WebXR 相對 8th Wall 的核心優勢）。地板偵測改用 `XRHitTestSource`（`session.requestHitTestSource({ space: viewerSpace })`），每幀在 `renderer.setAnimationLoop` 的 callback 內用 `frame.getHitTestResults()` 取得命中結果。
- **共用邏輯重構**：8th Wall 與 WebXR 的 hit-test 結果格式不同（前者是 `XR8.XrController.hitTest()` 回傳陣列，後者是 `XRHitTestResult.getPose()`），但後續「reticle 平滑＋連續多幀穩定度判斷＋變色」邏輯完全相同，已抽成共用函式 `updateReticleFromHit(hp, hr, t)`（[index.html:1733-1780](index.html#L1733-L1780)）；點按放置／長按重置流程也抽成 `attachPlacementHandlers()`（[index.html:1786-1840](index.html#L1786-L1840)），兩種後端呼叫同一份，不重複維護兩套。
- **`window._arBackend`**：新增旗標區分 `'8thwall'` / `'webxr'` / `null`，只用在渲染迴圈最後「清 buffer」的分支。其餘所有「是不是在 AR 模式」的判斷仍沿用原本的 `window._xrActive`（兩種後端都會設為 `true`），不用逐一改判斷式。
- **必要的 `dom-overlay` 依賴**：`requestSession` 有帶 `optionalFeatures: ['dom-overlay']`，讓搖桿/按鈕/偵錯面板等既有 DOM UI 能疊在 WebXR 沉浸式畫面上（不然畫面會整個被系統接管，UI 全部消失）。Chrome for Android 對 dom-overlay 的支援度算普遍，但**這是實機測試時第一個要確認的點**：如果進入 AR 後操作 UI 整個不見，代表這支裝置的 Chrome 不支援 dom-overlay，這條路要重新設計 UI 呈現方式。
- **已知未處理的邊界情況**：WebXR session 結束時（`session.addEventListener('end', ...)`）目前只是停止渲染迴圈，沒有接回桌機模式或提示重新整理，這是刻意先做最小可行版本，實測沒問題再補。

### 6a. 目前暫時開啟的除錯覆寫（排查走動漂移問題用，記得之後評估要不要恢復）

排查 WebXR 走動漂移問題期間，為了能反覆快速重新放置測試，暫時拿掉了「必須把手機轉成垂直角度才能放置」的限制。這是**兩處**改動，缺一不可（一開始只改了第一處，發現點擊還是沒反應，才發現第二處才是真正擋住點擊的地方）：

1. **放置判斷式本身**（[index.html:1846](index.html#L1846)）：`attachPlacementHandlers()` 的 touchend 判斷式，`&& window.orientGuideOk` 被註解掉（`/* && window.orientGuideOk */`）。
2. **姿態引導遮罩的點擊穿透**（[index.html:1220](index.html#L1220) 附近，`updateOrientGuide()` 內）：`#orient-guide` 這個 `position:fixed;inset:0` 的遮罩，原本只有角度符合（`ok`）時才會加上 `.clickthrough`（CSS 對應 `pointer-events:none`），角度沒對齊時會**真的擋住點擊事件**，點了畫面完全沒反應——這才是「網格圈圈都出現了、點了卻放不下去」的真正原因，跟放置判斷式邏輯本身無關。改成 `orientGuideEl.classList.add('clickthrough');` 讓遮罩永遠穿透點擊。

視覺上藍色框線引導、震動回饋、文字提示都還在（沒有拿掉，只是不再擋放置），純粹是「現在網格一出現、不管手機角度多歪都能立刻點按放置」，方便測試漂移時不用一直把手機轉正。

**要恢復限制**：兩處都要改回去——[index.html:1846](index.html#L1846) 拿掉註解、[index.html:1220](index.html#L1220) 附近改回 `orientGuideEl.classList.toggle('clickthrough', ok);`。
- **⚠️ `renderer.autoClear` 陷阱**：全域的 `renderer.autoClear = false`（[index.html:1513](index.html#L1513)）是為了 8th Wall——它自己把相機畫面畫進 WebGL color buffer，若讓 `render()` 自動清除會蓋掉那張畫面，所以原本的做法是在 `render()` 之前手動呼叫 `clear()`/`clearDepth()`。這個手動 clear 對 WebXR 是錯的：WebXR session 進行中，`renderer.render()` 內部才會把 framebuffer 換綁到「真正會顯示給使用者看」的 XR framebuffer，在 `render()` 之前手動呼叫的 `clear()` 清的是還沒換綁、沒人看得到的預設 framebuffer。修法：`_startWebXR()` 內把 `renderer.autoClear` 暫時開回 `true`，讓 `render()` 自己在正確換綁後的 framebuffer 上清除，session 結束時要記得改回 `false`。**這個修法本身是對的（framebuffer 綁定時機的問題確實存在），但實機測試證實它不是唯一問題**，見下一條。
- **⚠️ dom-overlay 不透明背景陷阱（實機測試踩過兩輪，2026-09-23）**：這才是「相機是黑的」真正的主因。`domOverlay: { root: document.body }` 讓 Chrome 把**整棵 DOM 樹**（不只是 `<canvas>` 本身）當成一張 2D 圖層疊在 AR 相機影像上面——只要任何一層祖先元素有不透明背景色，就會整片蓋住鏡頭透視，跟 canvas 本身透不透明、alpha 清得對不對完全無關。這個專案的 `body { background: #0f0f0f }` 與 `#three-wrap { background: #111 }`（原本是給 8th Wall／桌機模式用的深色底）正好符合這個條件。
  - 第一輪實機測試：當時 `#screen-live`／`#three-wrap` 的版面因為另一個 viewport 高度量測問題（見下一條）只佔了畫面上半部，所以只有「上半部」被這個不透明背景蓋住，下半部因為完全沒有 DOM 內容覆蓋，反而露出了沒被遮住的真實鏡頭畫面——當時誤判成「上半部還沒清乾淨」，其實是兩個問題疊在一起，恰好讓下半部看起來正常。
  - 第二輪：修了 viewport 高度問題、讓版面正確撐滿全螢幕之後，這個不透明背景色也跟著蓋滿了「整個」畫面，症狀從「一半黑」變成「全黑」——才真正暴露出背景色本身才是根因。
  - 修法：新增 CSS class `body.webxr-presenting`（連帶 `#three-wrap`）在 WebXR session 中強制 `background: transparent !important`，`_startWebXR()` 開場加上這個 class、`session.addEventListener('end', ...)` 移除，還原給 8th Wall／桌機模式用的深色底。
  - **教訓**：dom-overlay 的第一個檢查項目應該永遠是「這個 DOM 樹（含所有祖先）背景色透不透明」，而不是 canvas / WebGL 層級的清除邏輯——後者（`autoClear` 陷阱）是真的問題，但視覺上兩者症狀幾乎一樣（「看不到鏡頭」），容易先抓錯層次去修。
- **⚠️ viewport 高度量測陷阱**：Chrome 進入 immersive-ar 時，`window.innerHeight` 有時不會重新量測，導致 `position:fixed;inset:0` 的 `#screen-live` 被鎖在進場前（含網址列）較矮的高度，UI 被擠在畫面上半部。修法：`_startWebXR()` 內用 `window.screen.height`（不受此問題影響）強制校正 `html`/`body` 高度並觸發 `resize`，多個時間點重複執行；session 結束還原。

### 7. 若未來要切換架構的決策點

- 目前狀態即為「Android/Chrome+ARCore 用 WebXR、其餘裝置維持 8th Wall／桌機」的雙軌並行，兩套邏輯都在維護中，不是互斥的單選題。
- 若之後實測穩定、且確定不再需要跨平台（例如產品定位改成只鎖 Android），可以考慮拿掉 8th Wall 整套（`@8thwall/engine-binary`、`_startXR`、`chopperARModule`），只留 WebXR＋桌機模式，簡化程式碼。
- 若 WebXR 實測後發現 dom-overlay 不支援或精度不如預期，回滾方式：拿掉「AR 啟動策略」那段的 `navigator.xr.isSessionSupported` 判斷，直接呼叫 `_startFallback()`，即可完全恢復成本節第 1-4 點描述的純 8th Wall 行為（`_startWebXR`／共用函式留著不影響原有邏輯，也可以之後用 `git diff` 對照本次改動的 commit 整段還原）。
