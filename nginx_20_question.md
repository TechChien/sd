1. nginx 與 Apache 相比,架構上有什麼根本差異？為什麼 nginx 在高並發場景下效能較好？

2. 請說明 nginx 的 master process 與 worker process 分工方式,以及為什麼要這樣設計。

3. nginx 的 event-driven 模型（epoll）是如何運作的？跟傳統多執行緒/多程序模型有何不同？

4. worker_processes 該如何設定？設成 auto 跟手動指定數字有什麼差別？

5. worker_connections 這個參數代表什麼？它跟 worker_processes 如何一起決定 nginx 能承載的最大並發連線數？

6. 請說明 nginx 的 location 匹配規則與優先順序（= , ^~ , ~ , ~\* , 一般前綴匹配）。

7. proxy_pass 後面加不加斜線（trailing slash）差別在哪？請舉例說明對 URI 拼接的影響。

8. nginx 反向代理時,如何正確傳遞用戶端真實 IP（X-Real-IP、X-Forwarded-For）？為什麼直接用 $remote_addr 可能不準？

9. 請說明 nginx 常見的負載平衡演算法（round robin、least_conn、ip_hash、weight）分別適用什麼情境。

10. upstream 模組中,如何設定健康檢查（health check）與 failover（例如 max_fails、fail_timeout）？

11. 什麼是 nginx 的 blue-green deployment 或灰度發布,實務上怎麼用 nginx 做流量切分？

12. proxy_buffering 開啟與關閉的差異是什麼？什麼情況下需要關閉它（例如 SSE/串流場景)？

13. nginx 如何處理 WebSocket 連線？需要額外設定哪些 header（Upgrade、Connection)？

14. 請說明 nginx 的 gzip 壓縮設定,以及哪些情境不適合開啟 gzip。

15. nginx 的快取機制（proxy_cache）如何運作？cache key、cache 有效期、cache 清除的策略是什麼？

16. 什麼是 nginx 的 rate limiting（limit_req、limit_conn）？burst 與 nodelay 參數的作用是什麼？

17. 請說明 nginx 如何設定 SSL/TLS（憑證、SNI、HTTP/2)，以及常見的效能調校參數。

18. OpenResty 與原生 nginx 的關係是什麼？Lua 腳本（例如 access_by_lua_block）通常用在什麼場景？

19. nginx 的 log 格式（access_log、error_log）如何自訂？如何搭配 Vector/Loki 等工具做集中式日誌收集？

20. 如果 nginx 出現 502/504 錯誤,你的排查思路是什麼？常見原因有哪些（upstream timeout、connection refused、buffer 不足等）？

# nginx 面試題解答

**1. nginx vs Apache 架構差異**
Apache（prefork/worker MPM）為每個連線分配一個 process 或 thread，記憶體消耗隨並發數線性增長；nginx 採用事件驅動（event-driven）+ 非阻塞 I/O，單一 worker process 可用少量記憶體處理數千連線。
**使用場景**：高並發靜態資源伺服、API Gateway、反向代理層——連線數遠大於 CPU 核心數時，nginx 優勢明顯；若需要 `.htaccess` 動態目錄層級設定，Apache 較方便。

**2. master process 與 worker process**
Master 負責讀取設定檔、綁定 port、管理 worker 的建立/回收（reload 時不中斷服務），不處理實際請求；worker 才是真正處理連線與請求的 process。
**使用場景**：執行 `nginx -s reload` 時能理解為何服務不會中斷——master 平滑地啟動新 worker、讓舊 worker 處理完現有連線後退出（graceful shutdown）。

**3. epoll 事件驅動模型**
worker 用單一執行緒搭配 epoll（Linux）監聽大量 socket 事件，事件觸發才處理，不會為每個連線阻塞等待，適合 I/O bound 場景。
**使用場景**：長連線（keep-alive）、WebSocket、大量閒置連線（如 SSE 推播）時，傳統 thread-per-connection 模型記憶體/context switch 成本太高，這時 epoll 模型效益最大。

**4. worker_processes**
設為 `auto` 會偵測 CPU 核心數並自動配置一個 worker 對應一個核心，通常搭配 `worker_cpu_affinity` 綁核心避免 context switch。
**使用場景**：正式環境幾乎都用 `auto`；若同機器還跑其他吃 CPU 的服務（如你的 LiteLLM gateway 或 GPT-OSS backend），可手動限制數量避免搶佔資源。

**5. worker_connections**
單一 worker 能同時處理的最大連線數（含對 client 與對 upstream 的連線），最大並發數約為 `worker_processes × worker_connections`（實際上要除以每個請求佔用的連線數，如反代通常算 2）。
**使用場景**：評估容量規劃時用來計算理論上限，例如你的 runner orchestrator 若同時有大量 session container 連進來，需要調高此值並搭配 `ulimit -n` 提高檔案描述符上限。

**6. location 匹配優先順序**
順序為：`=`（完全匹配）→ `^~`（前綴匹配並停止 regex 比對）→ `~` / `~*`（區分/不區分大小寫正則）→ 一般前綴匹配（取最長者）。
**使用場景**：靜態資源用 `^~ /static/` 避免被後面的正則規則攔截；健康檢查路徑用 `= /health` 確保最快命中、不被其他 regex 誤判。

**7. proxy_pass 尾端斜線**
`proxy_pass http://backend/;` 帶斜線會把 location 匹配到的路徑「替換」掉；不帶斜線則是把原始 URI 整段「附加」到 backend 位址後面。
**使用場景**：反代到子路徑服務時（例如 `/api/` 對應到 backend 根目錄）務必用帶斜線寫法，否則 URL 會多出重複的 `/api/api/...` 而 404。

**8. 傳遞真實 IP**
需設定 `proxy_set_header X-Real-IP $remote_addr;` 與 `X-Forwarded-For $proxy_add_x_forwarded_for;`；因為經過反代/LB後，`$remote_addr` 在 upstream 端看到的是 nginx 自己的 IP，不是原始 client IP。
**使用場景**：日誌審計、rate limiting 依真實用戶 IP 判斷、WAF 規則、地理位置判斷時都要正確設定，尤其是多層反代（CDN→LB→nginx→app）要注意 X-Forwarded-For 是逗號串接的 IP 清單，只信任最後一個可信節點加上去的值。

**9. 負載平衡演算法**
`round robin`（預設，均分）、`least_conn`（導向連線數最少的節點，適合長連線）、`ip_hash`（同 client IP 固定打到同一節點，做 session 黏著）、`weight`（依權重分配流量，適合異質硬體或灰度發布）。
**使用場景**：無狀態 API 用 round robin；WebSocket/長連線服務用 least_conn；沒有做 session 外部化的傳統 web app 用 ip_hash 做 sticky session；金絲雀部署用 weight 逐步調整新舊版本流量比例。

**10. 健康檢查與 failover**
社群版 nginx 用被動式健康檢查：`max_fails=3 fail_timeout=30s` 表示 30 秒內失敗 3 次就標記該節點暫時不可用；主動式健康檢查需 nginx Plus 或搭配 OpenResty 的第三方模組（如 `nginx_upstream_check_module`）。
**使用場景**：你的 Claude runner fleet 若某個 backend container 掛掉，靠 `max_fails`/`fail_timeout` 讓流量自動繞開故障節點，等它恢復後自動納回輪詢。

**11. Blue-green / 灰度發布**
透過兩組 upstream（blue/green）搭配 `map` 或 Lua 判斷（如依 header、cookie、隨機權重）決定流量走向，或直接改 `weight` 逐步切換流量。
**使用場景**：新版本 Claude runner image 上線前，先讓 5% 流量導向新版驗證穩定性，觀察 Prometheus 指標無異常後再逐步拉高比例，最後完全切換。

**12. proxy_buffering**
開啟時 nginx 會先把 upstream 回應完整緩衝再發給 client（節省 upstream 連線時間，但增加延遲）；關閉時則是即時透傳（streaming），response 一有資料就立刻轉發。
**使用場景**：SSE（Server-Sent Events）、Claude API 的 streaming 回應、WebSocket 這類需要低延遲即時推送的場景必須關閉 buffering，否則 client 會感覺卡頓、收不到即時 chunk。這點跟你的 proxy service 攔截 SSE 串流轉發給 client 直接相關。

**13. WebSocket 支援**
需設定 `proxy_set_header Upgrade $http_upgrade;` 與 `proxy_set_header Connection "upgrade";`，並確保 `proxy_read_timeout` 拉長避免長連線被提前斷開。
**使用場景**：任何即時通訊、協作編輯、terminal-over-web（如 Claude Code 的互動式 session）都需要這組設定，否則 handshake 會失敗、被 nginx 當一般 HTTP 請求處理。

**14. gzip 壓縮**
`gzip on;` 搭配 `gzip_types` 指定壓縮的 MIME type，可大幅減少文字類資源（HTML/CSS/JS/JSON）傳輸量；已壓縮格式（圖片、影片、已 gzip 過的檔案）開啟反而浪費 CPU 且效果差。
**使用場景**：API 回傳大量 JSON 資料、前端靜態資源伺服適合開；串流媒體、已壓縮的二進位檔案不需要開，甚至該用 `gzip_types` 排除避免做無用功。

**15. proxy_cache 快取機制**
需先用 `proxy_cache_path` 定義快取儲存路徑與 zone，location 內用 `proxy_cache` 啟用；cache key 預設為請求 URI + 部分 header 組合（可自訂），有效期用 `proxy_cache_valid` 依 status code 分別設定，清除可用 `proxy_cache_purge`（需模組支援）或直接砍檔案。
**使用場景**：API Gateway 前面快取不常變動的查詢結果（如靜態設定、字典資料），減少對 backend/DB 的壓力；動態且個人化的內容（如登入後頁面）不適合快取，需搭配 `proxy_cache_bypass` 排除。

**16. Rate limiting**
`limit_req_zone` 定義以何種 key（通常是 IP）限速多少（如 `rate=10r/s`）；`burst` 允許短時間內超出速率的請求數暫存排隊，`nodelay` 則讓 burst 內的請求立即處理而非排隊延遲。
**使用場景**：保護後端 API 或 LLM inference endpoint 免受突發流量洪水攻擊；`burst` 搭配 `nodelay` 適合允許用戶短暫爆發性請求（如前端一次性載入多個資源）又不想犧牲太多延遲。

**17. SSL/TLS 設定**
需設定 `ssl_certificate`/`ssl_certificate_key`、啟用 `http2`、多網域用 SNI（`server_name` 對應不同 cert）；效能調校包含 `ssl_session_cache`（減少重複 handshake）、`ssl_protocols` 限制版本、`ssl_prefer_server_ciphers`。
**使用場景**：內部服務對外暴露 HTTPS endpoint（如你的內部 MCP server）時，用 `ssl_session_cache shared:SSL:10m` 減少高並發下 TLS handshake 的 CPU 開銷。

**18. OpenResty 與 Lua**
OpenResty 是以 nginx 為核心、整合 LuaJIT 的擴充發行版，讓你能在 nginx 各處理階段（如 `access_by_lua_block`、`content_by_lua_block`）嵌入自訂邏輯，突破原生 nginx 設定檔的靜態限制。
**使用場景**：你做過的 OAuth token 驗證、動態路由判斷（依 header/JWT 決定轉發哪個 upstream）、A/B 測試邏輯、動態黑白名單，這些需要「程式邏輯」而非單純設定檔規則的場景就要用 Lua。

**19. Log 格式與集中收集**
`log_format` 自訂欄位（如加入 `$upstream_response_time`、`$request_id`）；搭配 Vector 這類 log agent tail nginx 的 access_log/error_log，轉發到 Loki 做集中查詢與告警。
**使用場景**：排查跨多台 nginx 節點的請求延遲問題時，自訂 log 加入 upstream 回應時間、trace id，讓 Vector→Loki 可以用 LogQL 快速篩選特定 request 全鏈路。

**20. 502/504 排查思路**
502（Bad Gateway）通常是 upstream 回傳格式錯誤或連線被拒（connection refused，backend 沒起來/port 錯）；504（Gateway Timeout）是 upstream 處理超時，常見於 `proxy_read_timeout` 太短或 backend 本身卡住；也要檢查 `proxy_buffer_size` 是否不足導致 header 過大被拒。排查順序：先看 nginx error_log 的具體錯誤訊息 → 確認 upstream 服務是否存活（`curl` 直連測試）→ 檢查逾時與 buffer 相關參數 → 看 upstream 自己的 log 是否有慢查詢或例外。
**使用場景**：這是維運最常實戰的情境——例如你的 LiteLLM gateway 轉發到 GPT-OSS 120B 時若模型推論時間過長導致 504，就需要拉長 `proxy_read_timeout`；若 runner container 還沒完全啟動就被路由進來則會出現 502 connection refused。

---

要不要我針對第 7、12、16、20 題這幾個你在實務上最可能碰到的（proxy_pass 拼接、SSE buffering、rate limit、502/504 排查）出幾個追問題型，模擬面試官會怎麼往下追問？

# 逐題回饋

## 組 1:proxy_pass

**Q1-1** ✅ 正確。`http://backend:8080/v2/` 帶尾斜線 → location 匹配到的 `/api/` 會被「替換」成 `/v2/`,所以 `/api/users` → `/v2/users`,完整結果 `http://backend:8080/v2/users?id=1`。

**Q1-2** ❌ 這裡有誤解,答案應該是 `http://backend:8080/api/users?id=1`——沒錯你寫對了！但邏輯要修正一下:不是「原封不動」這麼簡單,而是因為 `proxy_pass` 後面**完全沒有 URI 路徑**（連根路徑 `/` 都沒有),nginx 就不會做任何路徑替換,直接把 client 請求的**完整原始 URI**（含 location 匹配到的部分）整段接在 backend 位址後面。所以你的答案是對的,只是理解要更精確：**判斷關鍵不是「有沒有尾斜線」,而是「proxy_pass 後面到底有沒有寫路徑」**。

- `proxy_pass http://backend:8080;`（無路徑）→ 原始 URI 整段附加
- `proxy_pass http://backend:8080/;`（只有斜線,也算有路徑)→ 會做替換,等同 `proxy_pass http://backend:8080;` 但語意上是把 location 前綴替換成 `/`

**Q1-3**（不確定,補充答案）
使用場景：**當 backend 本身就是用完整原始路徑在路由**,你不想讓 nginx 做任何路徑轉換,只是單純當一層透明代理。例如：

- backend 是另一套也有自己 routing 邏輯的服務（像你的 LiteLLM gateway 本身可能認得 `/v1/chat/completions` 這種完整路徑),nginx 只是做 TLS termination 或加 header,不改變路徑結構。
- 你想讓 nginx location 只做「攔截判斷」（比如依 header 分流到不同 upstream),但不想讓 URI 結構跟著 location 寫法而改變,減少維護時「這條路徑到底有沒有被 rewrite」的心智負擔。

---

## 組 2:SSE / Buffering

**Q2-1**（只提到「檢查 upstream 是否運作正常」,方向不夠完整,補充）
除了 nginx 端的 `proxy_buffering off`,還要檢查:

1. **`proxy_cache` 有沒有關**——即使 buffering 關了,若同一個 location 有 `proxy_cache` 在跑,還是會等內容完整才回應。SSE/streaming 路徑要 `proxy_cache off;`。
2. **`gzip` 是否開啟**（這正是 Q2-2 要問的,兩題互相關聯）。
3. **應用層本身是否真的邊產生邊 flush**——很多框架（如 Python Flask/FastAPI）預設會把 response 在應用層自己先 buffer 起來,要確認 backend 程式碼有沒有正確用 streaming response（例如 FastAPI 的 `StreamingResponse`）並且每個 chunk 後有 flush。
4. **中間是否還有其他反代層**（CDN、LB）也在做 buffering,這是 Q2-3 要延伸的重點。
5. **`proxy_http_version`**——SSE/長連線建議設 `1.1`,搭配 `proxy_set_header Connection "";` 避免不必要的連線重建。

**Q2-2** ❌ 理解方向不對,修正：
問題不是「封包切 chunk」導致「壓縮後檔案失效」,而是：**gzip 壓縮演算法本身需要一定量的資料塞進壓縮 buffer 才會輸出（尤其 gzip 的壓縮字典/視窗機制需要累積內容才能達到好的壓縮率)**。如果 SSE 一次只送一小段資料（例如一個 token）,gzip 可能會把這些小 chunk 留在自己的內部 buffer 裡等湊到一定大小才吐出,造成**即使 `proxy_buffering off`,資料還是被 gzip 這層卡住,不會即時送到 client**。

實務做法：SSE / streaming 路徑要用 `gzip off;`（或至少排除該 location 不做 gzip),寧可犧牲一點傳輸量也要保證即時性。

**Q2-3**（不確定,補充完整答案）
不只 nginx,**整條鏈路每一層都要確認沒有 buffering 或 caching**：

- **CDN**：多數 CDN 預設會對 response 做 buffer/cache,必須針對該路徑設定 "no buffering" 或直接繞過 CDN（bypass),很多 CDN 甚至完全不支援真正的 streaming passthrough,這時只能對 SSE 路徑做 CDN bypass。
- **nginx**：`proxy_buffering off;` + `gzip off;` + `proxy_cache off;`。
- **LiteLLM gateway**：如果它本身是用某個 Python web framework 包起來（如 FastAPI/uvicorn),要確認它是不是也有自己的 response buffering 設定,以及它轉發給 GPT-OSS backend 時是不是有正確處理 streaming（沒有把整個回應等完再轉發）。
- **GPT-OSS backend（OpenAI-compatible endpoint)**：本身要支援 `stream=true` 且真的逐 token 吐出,而不是內部生成完整才一次回傳。

所以**只改 nginx 這一層絕對不夠**,這是很典型的「單點修正、整條鏈路仍卡頓」的坑,你之前做過的「攔截 SSE 串流轉發」proxy service 這一層也要特別確認有沒有正確 flush,不然 nginx 這邊設定再對也沒用。

---

## 組 3:Rate limiting

**Q3-1** ✅ 大致正確，但用詞需要修正一個關鍵細節：不是「7 個會被 queue,`nodelay` 會立即處理 queue」，正確理解是——**`burst=10` 表示最多允許 10 個請求進入等待佇列（超出 rate 的部分),`nodelay` 則是讓這些排隊中的請求「不用等待延遲時間,立即被放行處理」**,而不是「先 queue 再立即處理」這種兩階段動作,而是同一時間點就直接判斷：只要在 burst 容量內就馬上過,不會有實際延遲感。

實際結果：12 個請求 → 前 5 個以正常速率內直接過 → 剩下 7 個因為 `burst=10` > 7,全部落在 burst 容量內,加上 `nodelay`,所以**全部 12 個都會被處理,沒有人被拒絕**,只是嚴格來說系統內部仍會依 leaky bucket 演算法平滑速率（但因為有 nodelay,client 端感受不到延遲)。如果是 13 個以上超過 `rate + burst` 容量,超出的才會直接回 503。

**Q3-2** ✅ 方向正確。拿掉 `nodelay` 後,7 個排隊請求依然會被接受（因為在 burst 範圍內),但**會被強制依照 `rate=5r/s` 的速率延遲處理**（也就是每 200ms 才放一個出去),而不是立即處理。對使用者體感來說,請求不會被拒絕,但**回應時間會明顯變長**（要等待排隊延遲）,適合「寧可慢一點也不要爆衝打垮 backend」的場景；`nodelay` 則適合「允許短暫爆發,但爆發之後速率仍要收斂」的場景。

**Q3-3** ✅ 正確方向,補充精確一點：`$remote_addr` 是字串型態的 IP（如 `"192.168.1.1"`,約 7-15 bytes 依長度不同),`$binary_remote_addr` 是把 IP 轉成固定長度的二進位表示（IPv4 固定 4 bytes,IPv6 固定 16 bytes)。用 binary 版本除了省記憶體,更重要的是**共享記憶體（`zone=api:10m`）儲存效率更高、雜湊查找更快**,在 rate limit 這種高頻率查表的場景下效能差異會被放大。

**Q3-4** ✅ 觀念正確,但寫法要修正：不能直接「使用 x-forward-for ip addr」當 key,因為 `X-Forwarded-For` 本身可能是逗號分隔的多個 IP（client → 每經過一層反代就會被加一個 IP 進去),如果直接整串當 key 會導致同一個真實用戶因為 proxy chain 長度不同而被視為不同 key,失去限流意義。正確做法：

- 若架構單純（只有一層 CDN/LB),可以用 `$http_x_forwarded_for` 但要小心可能被偽造,建議**信任由你自己 LB 設定的 header**（例如改用你自己 LB 加上的可信 header,像 `X-Real-IP`,由最靠近 client 的可信節點寫入,後面每一層都不再覆蓋)。
- 更嚴謹的做法是搭配 `set_real_ip_from` + `real_ip_header X-Forwarded-For;`（ngx_http_realip_module),讓 nginx 從受信任的 CDN/LB IP 範圍解析出真實 client IP,寫回 `$remote_addr`,這樣後續 `limit_req_zone` 才能安全地繼續用 `$binary_remote_addr`。

---

## 組 4:502/504

**Q4-1** ✅ 正確,補充一點：這個錯誤明確指出卡在「等待 upstream 回應 header」的階段（還沒開始收到任何 response),對應要調的是 `proxy_read_timeout`（也可以連帶檢查 `proxy_connect_timeout` 是否是另一種卡法,但這則訊息明確是 read 階段）。

**Q4-2** ⚠️ 小修正：`Connection refused` 對應的是 **502 Bad Gateway**,你寫的「502 (Bad request)」名稱錯誤——502 全名是 **Bad Gateway**,不是 Bad Request（400 才是 Bad Request）。跟 Q4-1 相比,層面差異是：Q4-1 是「連線建立成功,但等 backend 回應等到逾時」（backend 活著但太慢或卡住);Q4-2 是「連線根本建立不起來」（backend 那個 port 沒有任何程序在監聽）。可能原因：backend container 還沒啟動完成、port 設定錯誤、backend crash 掉了、防火牆/network policy 擋住。

**Q4-3**（方向對但太籠統,補充具體排查點）
「throughput 不夠」是結果不是原因,面試官會想聽你講出**具體的瓶頸點**,常見的有：

1. **upstream keepalive connection pool 沒設定或設太小**——沒有 `keepalive` 指令時,nginx 對每個 upstream 請求預設是「用完就關」,尖峰時期大量 TCP 建立/關閉導致 backend 來不及處理新連線,甚至觸發 backend 端的 connection queue 滿載。解法：在 `upstream {}` block 內加 `keepalive 32;` 並搭配 `proxy_http_version 1.1;` + `proxy_set_header Connection "";`。
2. **upstream 節點數量不足**——你的 runner fleet 若固定 capacity（如你先前提到的 fixed fleet capacity 4),尖峰時所有 worker 都在忙,新請求進來但沒有可用節點,即使個別測試都正常,整體吞吐量還是不夠。
3. **OS 層 file descriptor / ephemeral port 用盡**——高並發短連線下,`ulimit -n` 或 `net.ipv4.ip_local_port_range` 設太小會導致間歇性連不上,單獨用 curl 測不會踩到這個問題。
4. **`worker_connections` 或 `worker_rlimit_nofile` 不足**,導致 nginx 自己這端就先卡住,連轉發都做不到。

**Q4-4**（不清楚,完整補充）
`proxy_next_upstream` 定義**在什麼情況下,nginx 認定這次對 upstream 的請求算「失敗」,應該自動重試下一個 upstream 節點**,例如：

```nginx
proxy_next_upstream error timeout http_502 http_504;
```

代表遇到連線錯誤、逾時,或 upstream 回傳 502/504 時,nginx 會自動換一個 upstream 節點重試,而不是直接把錯誤回給 client。

**副作用（雪崩式重試)**：如果你把判斷條件設得太寬鬆（例如把 `http_500` 也加進去),或是沒有搭配 `proxy_next_upstream_tries` 限制重試次數、`proxy_next_upstream_timeout` 限制總重試時間,可能發生：

- 某個請求其實只是「稍微慢」（還沒真的逾時,但接近臨界值),被誤判需要重試,結果**同一個請求被丟去打第二、第三個 upstream 節點**,等於同時間對後端多打了好幾倍的負載。
- 如果全部 upstream 節點都處於同樣「稍微過載、回應變慢」的邊緣狀態,這種重試機制會讓**每個節點都額外承受其他節點轉嫁過來的重試流量**,形成惡性循環,把原本只是「慢」的問題,放大成全面性的 502/504 雪崩。

**正確做法**：明確限制 `proxy_next_upstream_tries 2;`（最多重試 1 次),並且 `proxy_next_upstream_timeout` 設一個合理上限,避免無止盡重試;同時搭配 `error`（純連線層錯誤）而非把應用層邏輯錯誤（如 500）也納入自動重試範圍,因為 500 通常代表 backend 程式邏輯出錯,重試到另一個節點大概率還是同樣結果,只是白白浪費資源。

---

要不要挑一題（我猜 Q2-3 的多層 streaming 或 Q4-3 的 keepalive pool）,我們做更深入的情境模擬,直接針對你目前 runner orchestrator 的架構去套用？
