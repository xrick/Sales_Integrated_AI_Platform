● 我將詳細分析這個新出現的問題，並制定完整的修復計畫。

  問題分析

  🔴 問題 #012: NumPy Float32 序列化錯誤

  問題表現

  - 時間: 2025-08-14 15:05:45
  - 錯誤訊息: Object of type float32 is not JSON serializable
  - 前端表現: "發送消息失敗，請重試"
  - 後端狀態: 500 Internal Server Error
  - 影響範圍: 特殊案例知識庫系統運行時的JSON序列化失敗

  詳細資料流分析

  完整調用鏈追蹤:
  1. 用戶輸入: "請幫我介紹便於攜帶，開關機迅速的筆電"
     ↓
  2. MGFD系統處理 ✅
     - 會話ID: 8fd3bd10-e4b5-4152-8ab1-69b12edc777c
     - 增強型槽位提取器啟動
     ↓
  3. 特殊案例知識庫匹配 ✅
     - 找到匹配: DSL001 (相似度: 0.76494026) ← np.float32類型！
     - 匹配類別: difficult_slot_detection
     ↓
  4. 第一次序列化錯誤 ❌
     - Redis狀態管理: 保存會話狀態失敗
     - 錯誤: Object of type float32 is not JSON serializable
     ↓
  5. LLM處理繼續 ✅
     - Act階段完成: 模擬LLM回應
     - ResponseGenerator生成: elicitation類型回應
     ↓
  6. 第二次序列化錯誤 ❌
     - 再次嘗試保存Redis狀態
     - 相同的float32序列化錯誤
     ↓
  7. FastAPI回應失敗 ❌
     - PydanticSerializationError: Unable to serialize unknown type: <class 'numpy.float32'>
     - 返回500狀態碼給前端

  根本原因分析

  A. NumPy數據類型不兼容 (CRITICAL):
  # 問題位置: SpecialCasesKnowledgeBase._calculate_case_similarity()
  similarity = cosine_similarity([query_embedding], [case_embedding])[0][0]
  # cosine_similarity 返回 np.float32 類型
  # 但 JSON.dumps() 無法序列化 np.float32

  B. 雙重序列化失敗點:
  1. Redis狀態管理: RedisStateManager.save_session_state()
  2. FastAPI回應: Pydantic序列化器處理API回應

  C. 數據傳播鏈:
  SpecialCasesKnowledgeBase (np.float32)
  → EnhancedSlotExtractor
  → UserInputHandler
  → RedisStateManager (第1次失敗)
  → MGFDSystem
  → FastAPI Response (第2次失敗)

  技術深度分析

  sentence-transformers返回類型:
  - cosine_similarity() 返回numpy.ndarray
  - 數組元素為numpy.float32類型
  - 需要轉換為Python原生float類型

  影響範圍:
  - 所有使用特殊案例知識庫的查詢
  - 語義相似度計算功能
  - Redis會話狀態持久化
  - API回應序列化

  解決方案設計

  修復策略：NumPy類型轉換標準化

  階段1: 核心序列化修復

  目標: 在SpecialCasesKnowledgeBase中進行類型轉換
  位置: libs/mgfd_cursor/special_cases_knowledge.py

  階段2: 通用序列化處理

  目標: 在Redis和API層級添加numpy類型處理
  位置:
  - libs/mgfd_cursor/redis_state_manager.py
  - FastAPI自定義JSON編碼器

  階段3: 測試驗證

  目標: 確保所有numpy類型都能正確序列化

  完整修復計畫

  修復1: SpecialCasesKnowledgeBase類型轉換

  # 在 _calculate_case_similarity 方法中
  similarity = cosine_similarity([query_embedding], [case_embedding])[0][0]
  return float(similarity)  # 轉換為Python原生float

  # 在所有返回相似度分數的地方添加轉換
  matched_case = {
      **case,
      "matched_category": category_name,
      "similarity_score": float(similarity)  # 確保是float類型
  }

  修復2: Redis狀態管理器增強

  # 在 save_session_state 方法中添加numpy類型處理
  import numpy as np

  def _serialize_numpy_types(self, obj):
      """遞歸處理numpy類型轉換"""
      if isinstance(obj, np.floating):
          return float(obj)
      elif isinstance(obj, np.integer):
          return int(obj)
      elif isinstance(obj, np.ndarray):
          return obj.tolist()
      elif isinstance(obj, dict):
          return {k: self._serialize_numpy_types(v) for k, v in obj.items()}
      elif isinstance(obj, list):
          return [self._serialize_numpy_types(item) for item in obj]
      return obj

  修復3: FastAPI自定義JSON編碼器

  # 在 main.py 中添加
  import numpy as np
  from fastapi.encoders import jsonable_encoder

  class NumpyEncoder(json.JSONEncoder):
      def default(self, obj):
          if isinstance(obj, np.floating):
              return float(obj)
          elif isinstance(obj, np.integer):
              return int(obj)
          elif isinstance(obj, np.ndarray):
              return obj.tolist()
          return super().default(obj)

  # 配置FastAPI使用自定義編碼器

  修復4: 增強型槽位提取器數據清理

  # 在 extract_slots_with_classification 返回前清理數據
  def _clean_numpy_types(self, data):
      """清理返回數據中的numpy類型"""
      # 實現遞歸清理邏輯
      return cleaned_data

  修復5: 知識庫統計更新處理

  # 在 _record_case_usage 中處理統計數據
  stored_case["usage_count"] = int(stored_case.get("usage_count", 0)) + 1
  # 確保所有數值都是Python原生類型

  測試計劃

  測試案例設計

  1. 原始問題重現: "請幫我介紹便於攜帶，開關機迅速的筆電"
  2. 相似度邊界測試: 測試高、中、低相似度查詢
  3. Redis序列化測試: 驗證會話狀態保存
  4. API回應測試: 確認FastAPI正常返回
  5. 知識庫更新測試: 驗證統計數據更新

  驗證標準

  - ✅ 無500錯誤
  - ✅ Redis狀態正常保存
  - ✅ 前端收到正確JSON回應
  - ✅ 特殊案例匹配功能正常
  - ✅ 知識庫統計正確更新

  風險評估

  低風險修復

  - ✅ 類型轉換不會影響業務邏輯
  - ✅ 向後兼容現有功能
  - ✅ 修復範圍明確且集中

  預期效果

  - 完全解決NumPy序列化問題
  - 提升系統穩定性
  - 保持特殊案例知識庫功能完整

  實施優先級

  🔥 Critical (立即修復)

  1. SpecialCasesKnowledgeBase類型轉換
  2. Redis狀態管理器numpy處理

  🔧 High (後續增強)

  3. FastAPI自定義編碼器
  4. 全面numpy類型檢查

  📋 Medium (優化改進)

  5. 測試用例補充
  6. 錯誤日誌改善

╭─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Ready to code?                                                                                                                                                              │
│                                                                                                                                                                             │
│ Here is Claude's plan:                                                                                                                                                      │
│ ╭─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮ │
│ │ NumPy Float32 序列化錯誤修復計畫                                                                                                                                        │ │
│ │                                                                                                                                                                         │ │
│ │ 問題分析                                                                                                                                                                │ │
│ │                                                                                                                                                                         │ │
│ │ 根本原因: SpecialCasesKnowledgeBase使用sentence-transformers計算相似度時返回np.float32類型，但JSON序列化器無法處理numpy類型，導致Redis狀態保存和FastAPI回應均失敗。     │ │
│ │                                                                                                                                                                         │ │
│ │ 修復階段                                                                                                                                                                │ │
│ │                                                                                                                                                                         │ │
│ │ 階段1: 核心類型轉換 (Critical)                                                                                                                                          │ │
│ │                                                                                                                                                                         │ │
│ │ - 檔案: libs/mgfd_cursor/special_cases_knowledge.py                                                                                                                     │ │
│ │ - 修復: 在_calculate_case_similarity()方法中將np.float32轉換為Python原生float                                                                                           │ │
│ │ - 預期: 解決序列化錯誤的根本原因                                                                                                                                        │ │
│ │                                                                                                                                                                         │ │
│ │ 階段2: Redis增強處理 (High)                                                                                                                                             │ │
│ │                                                                                                                                                                         │ │
│ │ - 檔案: libs/mgfd_cursor/redis_state_manager.py                                                                                                                         │ │
│ │ - 修復: 添加numpy類型檢測和轉換邏輯                                                                                                                                     │ │
│ │ - 預期: 防止Redis狀態保存失敗                                                                                                                                           │ │
│ │                                                                                                                                                                         │ │
│ │ 階段3: FastAPI編碼器 (Medium)                                                                                                                                           │ │
│ │                                                                                                                                                                         │ │
│ │ - 檔案: main.py                                                                                                                                                         │ │
│ │ - 修復: 配置自定義JSON編碼器處理numpy類型                                                                                                                               │ │
│ │ - 預期: API層級的最後防護                                                                                                                                               │ │
│ │                                                                                                                                                                         │ │
│ │ 階段4: 測試驗證 (High)                                                                                                                                                  │ │
│ │                                                                                                                                                                         │ │
│ │ - 測試: 原始問題重現、相似度邊界、序列化全鏈路                                                                                                                          │ │
│ │ - 預期: 確保修復完整性和穩定性                                                                                                                                          │ │
│ │                                                                                                                                                                         │ │
│ │ 技術要點                                                                                                                                                                │ │
│ │                                                                                                                                                                         │ │
│ │ - 重點處理cosine_similarity()返回的np.float32                                                                                                                           │ │
│ │ - 保持特殊案例知識庫功能完整                                                                                                                                            │ │
│ │ - 確保向後兼容性                                                                                                                                                        │ │
│ │ - 最小化性能影響                                                                                                                                                        │ │
│ │                                                                                                                                                                         │ │
│ │ 預期結果                                                                                                                                                                │ │
│ │                                                                                                                                                                         │ │
│ │ ✅ 完全解決"發送消息失敗"問題                                                                                                                                            │ │
│ │ ✅ 特殊案例知識庫正常運行✅ Redis狀態管理穩定                                                                                                                             │ │
│ │ ✅ 前端用戶體驗恢復正常                                                                                                                                                  │ │
│ ╰─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯ │
│                                                                                                                                                                             │
│ Would you like to proceed?                                                                                                                                                  │
│                                                                                                                                                                             │
│ ❯ 1. Yes, and auto-accept edits                                                                                                                                             │
│   2. Yes, and manually approve edits                                                                                                                                        │
│   3. No, keep planning                                                                                                                                                      │
│                                            